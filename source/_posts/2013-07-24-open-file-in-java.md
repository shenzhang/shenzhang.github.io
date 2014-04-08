---
layout: post
title: "java中的文件打开操作"
date: 2013-07-24 02:27
comments: true
categories: 
---

java中的文件操作一般都是要通过FileOutputStream/FileInputStream，这两个类可以获得channel通过NIO的手段来操作，或者是通过RandomAccessFile.getChannel()来获得channel并使用NIO。

其实不论FileOutputStream还是RandomAccessFile，都会有一个打开文件的动作，打开文件的结果都会是产生一个FileDescriptor，可以通过`FileOutputStream.getDescrptor()`或`RandomAccessFile.getDescriptor()`来获得。

``` java
    /**
     * The system dependent file descriptor.
     */
    private final FileDescriptor fd;
```

实际上就如同名字所表示的，这个就是一个文件描述符，或者在windows里面讲就是一个文件句柄（HANDLE），所有的系统调用都需要这个文件描述符作为参数。

再来看以下几个构造函数：

*RandomAccessFile(String name, String mode):* 该方法会调用RandomAccessFile(File name, String mode)

*RandomAccessFile(File name, String mode):* 根据File对象打开文件，产生FileDescriptor，但是目标File不会做改动，position的位置在0

*FileOutputStream(File file):* 该方法会调用FileOutputStream(File file, false)

*FileOutputStream(File file, boolean append):* 根据File打开文件,如果append = true，目标file不变，position在文件尾；如果append = false,目标文件会被truncate为0，等同于删除并重新创建了这个文件，也就是清空这个文件。并在最后都生成对应的FileDescriptor

*FileOutputStream(FileDescriptor fd):* 由于参数是FileDescriptor，说明这个文件已经是被打开了的，因此该构造函数实际上不会做任何操作，只是将这个fd保存下来，供以后的操作使用。

因此，我们最经常使用的FileOutputStream(File file)这个构造函数是会自动将文件进行truncate操作的，这个需要注意。

对文件的操作实际上都是使用FileDescriptor，因此普通文件操作和获得channel后的nio操作都共享一个fd，也就是说在普通写操作后，对应channel.getPosition也会同样被移动了。