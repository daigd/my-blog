---
title: "Java常量池总结"
date: 2020-07-15T15:10:12+08:00
tags: ["Java基础", "Java常量池"]
draft: false
---

在Java体系中，一共有三种常量池，分别是Class常量池、字符串常量池和运行时常量池。

## Class常量池

什么是Class常量池？可以理解成Class文件的资源仓库，用于存放编译器生成的各种`字面量（Literal）`和`符号引用（Symbolic References）`。

> 字面量：在计算机科学中，字面量用于表示源代码中一个固定值的表示法，如整数，浮点数，字符串等。
>
> 符号引用：属于编译原理中的概念，相对于直接引用而言。主要包括三种常量：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符。

用一个例子来说明一下，对于以下Java代码：

```java
public class HelloWorld
{
    private static String name = "test";
    
    public static void main(String[] args)
    {
        String msg = "World";
        System.out.println("Hello," + name + "," + msg);
    }
}
```

执行`javac HelloWorld.java`对其进行编译，得到`HelloWorld.class`文件，我们来查看一下这个字节码文件的内容，执行以下命令：

```bash
javap -v HelloWorld.class
```

得到如下内容：

```java
  Last modified 2020-7-15; size 774 bytes
  MD5 checksum bd438b8d35e4266f3713e53bf1037f8c
  Compiled from "HelloWorld.java"
public class com.demo.test.HelloWorld
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #14.#26        // java/lang/Object."<init>":()V
   #2 = String             #27            // World
   #3 = Fieldref           #28.#29        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = Class              #30            // java/lang/StringBuilder
   #5 = Methodref          #4.#26         // java/lang/StringBuilder."<init>":()V
   #6 = String             #31            // Hello,
   #7 = Methodref          #4.#32         // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
   #8 = Fieldref           #13.#33        // com/demo/test/HelloWorld.name:Ljava/lang/String;
   #9 = String             #34            // ,
  #10 = Methodref          #4.#35         // java/lang/StringBuilder.toString:()Ljava/lang/String;
  #11 = Methodref          #36.#37        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #12 = String             #38            // test
  #13 = Class              #39            // com/demo/test/HelloWorld
  #14 = Class              #40            // java/lang/Object
  #15 = Utf8               name
  #16 = Utf8               Ljava/lang/String;
  #17 = Utf8               <init>
  #18 = Utf8               ()V
  #19 = Utf8               Code
  #20 = Utf8               LineNumberTable
  #21 = Utf8               main
  #22 = Utf8               ([Ljava/lang/String;)V
  #23 = Utf8               <clinit>
  #24 = Utf8               SourceFile
  #25 = Utf8               HelloWorld.java
  #26 = NameAndType        #17:#18        // "<init>":()V
  #27 = Utf8               World
  #28 = Class              #41            // java/lang/System
  #29 = NameAndType        #42:#43        // out:Ljava/io/PrintStream;
  #30 = Utf8               java/lang/StringBuilder
  #31 = Utf8               Hello,
  #32 = NameAndType        #44:#45        // append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #33 = NameAndType        #15:#16        // name:Ljava/lang/String;
  #34 = Utf8               ,
  #35 = NameAndType        #46:#47        // toString:()Ljava/lang/String;
  #36 = Class              #48            // java/io/PrintStream
  #37 = NameAndType        #49:#50        // println:(Ljava/lang/String;)V
  #38 = Utf8               test
  #39 = Utf8               com/demo/test/HelloWorld
  #40 = Utf8               java/lang/Object
  #41 = Utf8               java/lang/System
  #42 = Utf8               out
  #43 = Utf8               Ljava/io/PrintStream;
  #44 = Utf8               append
  #45 = Utf8               (Ljava/lang/String;)Ljava/lang/StringBuilder;
  #46 = Utf8               toString
  #47 = Utf8               ()Ljava/lang/String;
  #48 = Utf8               java/io/PrintStream
  #49 = Utf8               println
  #50 = Utf8               (Ljava/lang/String;)V
{
  public com.demo.test.HelloWorld();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: ldc           #2                  // String World
         2: astore_1
         3: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         6: new           #4                  // class java/lang/StringBuilder
         9: dup
        10: invokespecial #5                  // Method java/lang/StringBuilder."<init>":()V
        13: ldc           #6                  // String Hello,
        15: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        18: getstatic     #8                  // Field name:Ljava/lang/String;
        21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        24: ldc           #9                  // String ,
        26: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        29: aload_1
        30: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        33: invokevirtual #10                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        36: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        39: return
      LineNumberTable:
        line 9: 0
        line 10: 3
        line 11: 39

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: ldc           #12                 // String test
         2: putstatic     #8                  // Field name:Ljava/lang/String;
         5: return
      LineNumberTable:
        line 5: 0
}
SourceFile: "HelloWorld.java"

```

可以看到，`Constant pool`包含的内容，一共有50个常量，包含的内容也印证了我们之前提到的，包括字面量和符号引用。

字面量举例：                  

> #2 = String             #27            // World
>
>  #27 = Utf8               World

三种符号引用举例：

- 类和接口的全限定名

> #13 = Class              #39            // com/demo/test/HelloWorld
>
> #39 = Utf8               com/demo/test/HelloWorld

- 字段的名称和描述符

> #15 = Utf8               name
>
> #33 = NameAndType        #15:#16        // name:Ljava/lang/String;
>
> #16 = Utf8               Ljava/lang/String;

- 方法的名称和描述符

> #5 = Methodref          #4.#26         // java/lang/StringBuilder."<init>":()V
>
> #4 = Class              #30            // java/lang/StringBuilder
>
> #26 = NameAndType        #17:#18        // "<init>":()V
>
>  #30 = Utf8               java/lang/StringBuilder
>
> #17 = Utf8               <init>
>
>  #18 = Utf8               ()V

Class常量池的符号引用在JVM加载Class文件时替换为直接引用，因为Class常量池里并没有各个方法、字段的内存地址信息，当虚拟机运行时，创建Class对象实例时或运行时需要将这些符号引用转换为真正的内存入口地址，这样才能访问到相关的数据。

## 字符串常量池

本质上是一个`HashSet<String>`,虚拟机加载`Class`文件时，`Class`常量池里的字符串会"进入"到字符串常量池，注意，这里的进入加双引号意指，字符串常量池存储的是`String`对象的引用，而不是字符串本身的内容，真正的字符串存储在堆中。

## 运行时常量池

运行时常量池是方法区的一部分，虚拟机在每个`Class`文件完成后，除了字符串的其它字面量和符号引用都会进入到运行时常量池，它会将`Class`常量池的符号引用转化为直接引用，比如类的静态方法或私有方法，构造方法，父类方法等，因为这些不能被重写，所以在加载的时候就可以将符号引用转变为直接引用，而其它一些方法则是在被第一次调用的时候才会将符号引用转成直接引用。

注：运行时常量池的内容可以动态添加。

三大常量池示意图如下：

![Java三大常量池示意图](/java/img/Java常量池的区别.png)

参考：

[字符串常量池、class常量池和运行时常量池](https://blog.csdn.net/qq_26222859/article/details/73135660)

[Class常量池](https://hollischuang.github.io/toBeTopJavaer/#/basics/java-basic/class-contant-pool)

[Java中的几种常量池](https://blog.csdn.net/chenkaibsw/article/details/80848069)

[Class文件中的常量池详解](https://blog.csdn.net/luanlouis/article/details/39960815)

[Java关于常量池的问题](https://www.zhihu.com/question/55328596/answer/144027981)

[可能是把Java内存区域讲的最清楚的一篇文章](https://zhuanlan.zhihu.com/p/42717913)

