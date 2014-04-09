---
layout: post
title: "JNDI环境初始化深入分析"
date: 2013-07-27 05:27
comments: true
categories: 
---

##JNDI是什么
jndi: Java Naming and Directory Interface

jndi是j2ee提供的用于对象查找和获取的接口规范，该规范需要j2ee容器去实现以提供jndi的功能，不同的容器或者说jndi实现者对jndi里的对象的注册方式和内部管理方式都不完全一样，但是获取方式都完全一样（jndi规定了怎么获取）,如：

``` java
InitalContext context = new InitialContext();
Object yourObject = context.lookup(name);
```

当然jndi中还包含了LDAP等更高层次的接口规范，这里不进行描述。

##JNDI中可用的环境变量
JNDI中预定义了一些环境变量，这些环境变量可以控制JNDI环境的行为（包括初始化过程），变量的定义在javax.naming.Context接口中，对应的javadoc有对各变量的说明，但是如果不看源码很难准确理解这些变量的意义。下面的片段截取了一些变量：

``` java
    String INITIAL_CONTEXT_FACTORY = "java.naming.factory.initial";
    String OBJECT_FACTORIES = "java.naming.factory.object";
    String URL_PKG_PREFIXES = "java.naming.factory.url.pkgs";
    String APPLET = "java.naming.applet";
```

##InitalContext
javax.naming.InitalContext是客户端使用jndi的入口，也是jndi容器初始化的入口，因此jndi容器(环境)的初始化是lazy的，只有到需要的时候才会进行初始化。InitalContext的构造函数有两个：

1.不接受任何初始化环境变量
``` java
    public InitialContext() throws NamingException {
        init(null);
    }
```

2.主动设置初始化环境变量(就是在上一节提到的环境变量)
``` java
    public InitialContext(Hashtable<?,?> environment)throws NamingException
    {
        if (environment != null) {
            environment = (Hashtable)environment.clone();
        }
        init(environment);
    }
```

接下来会调用init函数来初始化InitalContext.defaultInitCtx变量，该变量也是Context类型，并且由具体的容器来实现，它才是真正的实现者。待defaultInitCtx创建好之后，InitalContext的所有操作就会被分发给defaultInitCtx执行，InitalContext本质上讲就是defaultInitCtx的工厂和代理。

##初始化过程
``` java
    protected void init(Hashtable<?,?> environment)throws NamingException {
        myProps = ResourceManager.getInitialEnvironment(environment);

        if (myProps.get(Context.INITIAL_CONTEXT_FACTORY) != null) {
            // user has specified initial context factory; try to get it
            getDefaultInitCtx();
        }
    }
```

``` java
    public static Hashtable getInitialEnvironment(Hashtable env)
            throws NamingException
    {
        String[] props = VersionHelper.PROPS;   // system/applet properties
        if (env == null) {
            env = new Hashtable(11);
        }
        Object applet = env.get(Context.APPLET);

        // Merge property values from env param, applet params, and system
        // properties.  The first value wins:  there's no concatenation of
        // colon-separated lists.
        // Read system properties by first trying System.getProperties(),
        // and then trying System.getProperty() if that fails.  The former
        // is more efficient due to fewer permission checks.
        String[] jndiSysProps = helper.getJndiProperties();
        for (int i = 0; i < props.length; i++) {
            Object val = env.get(props[i]);
            if (val == null) {
                if (applet != null) {
                    val = AppletParameter.get(applet, props[i]);
                }
                if (val == null) {
                    // Read system property.
                    val = (jndiSysProps != null)
                        ? jndiSysProps[i]
                        : helper.getJndiProperty(i);
                }
                if (val != null) {
                    env.put(props[i], val);
                }
            }
        }

        // Merge the above with the values read from all application
        // resource files.  Colon-separated lists are concatenated.
        mergeTables(env, getApplicationResources());
        return env;
    }
```


这个函数实际上就是从一些环境变量（参数）源搜集jndi需要的环境变量，然后再以map的形式返回。

1. InitalContext(environment)构造函数传递的环境变量。
2. 如果a中指定了javax.naming.applet对象，则转换为Applet对象，这个Applet对象也成了源(applet)
3. System.getProperties()提供的环境变量(systemProperties)
4. 应用程序提供的环境变量(application)。


这些环境变量的优先级如下：2和3只能够提供以下的环境变量，其他的变量将被忽略：
``` java
    final static String[] PROPS = new String[] {
        javax.naming.Context.INITIAL_CONTEXT_FACTORY,
        javax.naming.Context.OBJECT_FACTORIES,
        javax.naming.Context.URL_PKG_PREFIXES,
        javax.naming.Context.STATE_FACTORIES,
        javax.naming.Context.PROVIDER_URL,
        javax.naming.Context.DNS_URL,
        // The following shouldn't create a runtime dependence on ldap package.
        javax.naming.ldap.LdapContext.CONTROL_FACTORIES
    };
```

优先级是environment > applet > systemProperties

最后的env会和application（应用指定的）环境变量合并（如果env没有提供就采用application的，如果env有了，那么就会和application的合并，用逗号分割），如下：

``` java
    private static void mergeTables(Hashtable props1, Hashtable props2) {
        Enumeration keys = props2.keys();

        while (keys.hasMoreElements()) {
            String prop = (String)keys.nextElement();
            Object val1 = props1.get(prop);
            if (val1 == null) {
                props1.put(prop, props2.get(prop));
            } else if (isListProperty(prop)) {
                String val2 = (String)props2.get(prop);
                props1.put(prop, ((String)val1) + ":" + val2);
            }
        }
    }
```

最后来看下怎么获取application(应用级别)的环境变量:
``` java
    private static Hashtable getApplicationResources() throws NamingException {

        ClassLoader cl = helper.getContextClassLoader();

        synchronized (propertiesCache) {
            Hashtable result = (Hashtable)propertiesCache.get(cl);
            if (result != null) {
                return result;
            }

            try {
                NamingEnumeration resources =
                    helper.getResources(cl, APP_RESOURCE_FILE_NAME);
                while (resources.hasMore()) {
                    Properties props = new Properties();
                    props.load((InputStream)resources.next());

                    if (result == null) {
                        result = props;
                    } else {
                        mergeTables(result, props);
                    }
                }

                // Merge in properties from file in <java.home>/lib.
                InputStream istream =
                    helper.getJavaHomeLibStream(JRELIB_PROPERTY_FILE_NAME);
                if (istream != null) {
                    Properties props = new Properties();
                    props.load(istream);

                    if (result == null) {
                        result = props;
                    } else {
                        mergeTables(result, props);
                    }
                }

            } catch (IOException e) {
                NamingException ne = new ConfigurationException(
                        "Error reading application resource file");
                ne.setRootCause(e);
                throw ne;
            }
            if (result == null) {
                result = new Hashtable(11);
            }
            propertiesCache.put(cl, result);
            return result;
        }
    }
```

实际上就是使用当前线程的ContextClassLoader去尝试加载两个peropery形式的配置文件：

1. APP_RESOURCE_FILE = jndi.properties
2. <java.home>/lib/jndi.properties

这个步骤对于某些jndi容器来说是相当重要的，因为前面提到了环境变量指定方式都需要应用做额外的配置或者写特定的代码，这个步骤就可以让容器的实现者可以将自己的配置放在约定的配置文件里，供jndi使用。比如jetty8的jndi容器实现就是通过`classpath://jndi.properties`来做配置的:

``` java
java.naming.factory.url.pkgs=org.eclipse.jetty.jndi
java.naming.factory.initial=org.eclipse.jetty.jndi.InitialContextFactory
```

之前有个朋友问他的jetty中的应用为什么InitalContext.lookup总是报错，配置不对。实际上就是没有将jetty-jndi-[version].jar依赖进来，这个里面就包含了jndi.properties的配置文件，除非使用jetty-all.jar的集合包，否则要想让jetty提供jndi的功能就一定需要该jar包。

好了通过上面的分析，最后搜集到了所有的环境变量的值，并保存在InitialContext.myProps中，继续初始化defaultInitCtx...

``` java
    protected void init(Hashtable<?,?> environment)throws NamingException {
        myProps = ResourceManager.getInitialEnvironment(environment);

        if (myProps.get(Context.INITIAL_CONTEXT_FACTORY) != null) {
            // user has specified initial context factory; try to get it
            getDefaultInitCtx();
        }
    }
```

如果环境变量里指明了INITAL_CONTEXT_FACTORY(java.naming.factory.inital)就开始初始化默认的Context:

``` java
    protected Context getDefaultInitCtx() throws NamingException{
        if (!gotDefault) {
            defaultInitCtx = NamingManager.getInitialContext(myProps);
            gotDefault = true;
        }
        if (defaultInitCtx == null)
            throw new NoInitialContextException();

        return defaultInitCtx;
    }
```

``` java
    public static Context getInitialContext(Hashtable<?,?> env)
        throws NamingException {
        InitialContextFactory factory;

        InitialContextFactoryBuilder builder = getInitialContextFactoryBuilder();
        if (builder == null) {
            // No factory installed, use property
            // Get initial context factory class name

            String className = env != null ?
                (String)env.get(Context.INITIAL_CONTEXT_FACTORY) : null;
            if (className == null) {
                NoInitialContextException ne = new NoInitialContextException(
                    "Need to specify class name in environment or system " +
                    "property, or as an applet parameter, or in an " +
                    "application resource file:  " +
                    Context.INITIAL_CONTEXT_FACTORY);
                throw ne;
            }

            try {
                factory = (InitialContextFactory)
                    helper.loadClass(className).newInstance();
            } catch(Exception e) {
                NoInitialContextException ne =
                    new NoInitialContextException(
                        "Cannot instantiate class: " + className);
                ne.setRootCause(e);
                throw ne;
            }
        } else {
            factory = builder.createInitialContextFactory(env);
        }

        return factory.getInitialContext(env);
    }
```


可以看出创建Context的过程实际上是一个工厂模式，要先找到这个工厂(InitalContextFactory)

1. 首先检查NamingManager中是否配置的有IntialContextFactoryBuilder（Context工厂的工厂），如果有的话就用指定的factoryBuilder创建一个ContextFactory
2. 否则就用环境变量里java.naming.factory.inital指定的工厂。
3. 得到工厂后，就用这个工厂去创建这个defaultContext(InitalContext就是这个真正的Context的代理）。

至于这个defaultContext是怎么实现的就依赖具体的容器了，只要符合jndi的规范就可以了，但是实际上规范中对这块的要求也是很少的，只是对一些概念和接口做了确定。

回头看来其实jndi的初始化是很简单的，就是简单的指定一些环境变量，特别是`INITAL_CONTEXT_FACTORY(java.naming.factory.inital)`参数，特工一个真正Context的工厂就行了，连创建的过程都交给具体容器去实现了。

