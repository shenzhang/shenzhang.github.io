---
layout: post
title: "maven杂谈(生命周期,插件绑定,effective-xx)"
date: 2013-09-15 01:27
comments: true
categories: 
---

##1.生命周期
###Clean

	pre-clean
	clean
	post-clean

###Default
	validate
	initialize
	generate-sources
	process-sources
	generate-resources
	process-resources
	compile
	process-classes
	generate-test-sources
	process-test-sources
	generate-test-resources
	process-test-resources
	test-compile
	process-test-classes
	test
	prepare-package
	package
	pre-integration-test
	integration-test
	post-integration-test
	verify
	install
	deploy

###Site
	pre-site
	site
	post-site
	site-deploy

##2.插件绑定
maven在不同的生命周期中会按顺序进入不同的阶段，每个阶段又会执行与该阶段绑定的plugins。如何将plugins绑定到生命周期中呢，maven-compile-plugin又是怎么知道在每次的compile阶段执行的呢？

plugin绑定到生命周期中需要在pom.xml中的<build>=><plugins>中进行声明，有些是在pom.xml中，有些可能是在super pom中，有些甚至是maven默认的绑定，但是最终都会出现在effective pom中（mvn help:effective-pom查看)。

那么具体是在哪个阶段执行呢？看effective pom中对maven-compile-plugin的描述：

``` xml
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.3.2</version>
        <executions>
          <execution>
            <id>default-testCompile</id>
            <phase>test-compile</phase>
            <goals>
              <goal>testCompile</goal>
            </goals>
          </execution>
          <execution>
            <id>default-compile</id>
            <phase>compile</phase>
            <goals>
              <goal>compile</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
```

主要是在phase节点和goals节点，上面的配置说明了maven-compile-plugin被分别绑定到了compile阶段和test-compile阶段，并且分别执行该插件的compile goal和testCompile goal。

另外有些plugin可能并没有在pom.xml中说明具体的执行阶段，那么就要看该插件中的/META-INF/maven/plugin.xml插件描述文件了，该描述文件说明了该plugin的前缀(goalPrefix)，所有的mojos或者说是goal，并且这些goal的名字以及默认的执行阶段，比如compile:testCompile节点就说明了&lt;phase&gt;test-compile&lt;/phase&gt;。因此如果没有在pom.xml中说明执行阶段的话，就按照该plugin的goal的自描述中的phase进行绑定。

##3.effective-xx
maven有关于自己运行的配置文件settings.xml，该文件可以是$MAVEN_HOME/conf/settings.xml或者$USER/.m2/setttings.xml，并且后者会覆盖前者，可以通过mvn help:effective-settings查看相对于默认的settings.xml所改变的部分。


同样，前面提到的effective-pom也是由多个部分合并而来，首先是项目下的pom.xml，然后是maven的super pom($MAVEN_HOME/lib/maven-model-builder-x.x.x.jar#org/apache/maven/model/pom-x.x.x.xml)，还有一些是maven针对不同的生命周期默认绑定的插件信息($MAVEN_HOME/lib/maven-core-x.x.x.jar#/META-INF/plexus/components.xml)。最终的pom可以通过help:effective-pom查看)


