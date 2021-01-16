# 面试官：JVM运行时数据区包含哪几部分？作用是啥？

## JDK，JRE，JVM的联系是啥？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190118193517872.PNG?)
JVM Java Virtual Machine
JDK Java Development Kit
JRE Java Runtime Environment
看上图官方的介绍讲的很清楚
## JVM的作用是啥？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121174837892.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p6dGlfZXJsaWU=,size_16,color_FFFFFF,t_70)
JVM有2个特别有意思的特性，语言无关性和平台无关性。

语言无关性是指实现了Java虚拟机规范的语言对可以在JVM上运行，如Groovy，和在大数据领域比较火的语言Scala，因为JVM最终运行的是class文件，只要最终的class文件复合规范就可以在JVM上运行。

平台无关性是指安装在不同平台的JVM会把class文件解释为本地的机器指令，从而实现Write Once，Run Anywhere
## JVM运行时数据区
Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干个不同的数据区域。这些区域都有各自的用途，以及创建和销毁的时间，有的区域随着虚拟机进程的启动而存在，有些区域则依赖用户线程的启动和结束而建立和销毁。Java虚拟机所管理的内存将会包括以下几个运行时数据区域
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190118194119929.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p6dGlfZXJsaWU=,size_16,color_FFFFFF,t_70)
其中方法区和堆是所有线程共享的数据区
程序计数器，虚拟机栈，本地方法栈是线程隔离的数据区，画一个逻辑图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190120182439808.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p6dGlfZXJsaWU=,size_16,color_FFFFFF,t_70)
### 程序计数器
程序计数器是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器

为什么要记录当前线程所执行的字节码的行号？直接执行完不就可以了吗？ 

因为代码是在线程中运行的，线程有可能被挂起。即CPU一会执行线程A，线程A还没有执行完被挂起了，接着执行线程B，最后又来执行线程A了，CPU得知道执行线程A的哪一部分指令，线程计数器会告诉CPU。

### 虚拟机栈
虚拟机栈存储当前线程运行方法所需要的数据，指令，返回地址。虚拟机栈描述的是Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧用于存储局部变量表，操作数栈，动态链接，方法出口等信息。每个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈道出栈的过程。

**局部变量表**存储存储局部变量，是一个定长为32位的局部变量空间。其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据类型只占用一个。引用类型（new出来的对象）如何存储?看下图

```java
public int methodOne(int a, int b) {
	Object obj = new Object();
	return a + b;
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190120180006852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p6dGlfZXJsaWU=,size_16,color_FFFFFF,t_70)
如果局部变量是Java的8种基本基本数据类型，则存在局部变量表中，如果是引用类型。如String，局部变量表中存的是引用，而实例在堆中。

假如methodOne方法调用methodTwo方法时， 虚拟机栈的情况如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190120182918856.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p6dGlfZXJsaWU=,size_16,color_FFFFFF,t_70)
当虚拟机栈无法再放下栈帧的时候，就会出现StackOverflowError，演示一下

```java
public class JavaVMStackSOF {

    private int stackLength = 1;

    public void stackLeak() {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) throws Throwable {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length: " + oom.stackLength);
            throw e;
        }
    }
}
```
在idea中设置运行时的线程的堆栈大小为如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019012021220829.png?)

-Xss 参数的作用是设置每个线程的堆栈大小
运行输出为
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190120212415141.png)
-Xss参数的值越大，打印输出的深度越大

接着解释一下**操作数栈**，还是比较容易理解的
假如Test.java中有如下方法，
```java
public int getSum(int a, int b) {
	return a + b;
}
```
反编译生成的Test.class文件，并输出到show.txt中

```shell
javap -v Test.class > show.txt
```
show.txt的内容如下
```java
public int getSum(int, int);
  descriptor: (II)I
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=3, args_size=3
       0: iload_1
       1: iload_2
       2: iadd
       3: ireturn
    LineNumberTable:
      line 12: 0
```
解释一下上面的语句

```java
iload_1:局部变量1压栈
iload_2:局部变量2压栈
iadd:栈顶2个元素相加，计算结果压栈
```
简单2个数相加都会用到栈，这个栈就是操作数栈，更不用说复杂的语法了
### 本地方法栈
本地方法栈（Native Method Stack）与虚拟机栈锁发挥的作用是非常相似的，他们之间的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务。
### Java堆
对于大多数应用来说，Java堆（Java Heap）是Java虚拟机锁管理的内存中最大的一块。Java堆是所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存
### 方法区
方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息，常量，静态变量，即时编译器编译后的代码等数据。
## JVM内存模型
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121172050704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p6dGlfZXJsaWU=,size_16,color_FFFFFF,t_70)
由颜色可以看出，jdk1.8之前，堆内存被分为新生代，老年代，永久带，jdk1.8及以后堆内存被分成了新生代和老年代。新生代的区域又分为eden区，s0区，s1区，默认比例是8:1:1，元空间可以理解为直接的物理内存

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190829233955325.png?)