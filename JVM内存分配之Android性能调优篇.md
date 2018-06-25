## [JVM内存分配之Android性能调优篇](https://www.cnblogs.com/ldq2016/p/8036464.html)

### 一、JVM内存模型

  ##### 1.1.1 内存模型图

​      平时我们对于Java内存都有一个比较粗略的概念，就是分堆和栈，但实际上还是复杂得多，以下给出完整内存模型： 

![](D:\AndroidFile\Photo\JVM内存分析\jvm_memory_1.png)

##### 1.1.2 内存模型图

​       相对应区域的内容为：

![](D:\AndroidFile\Photo\JVM内存分析\stack_heap_info.png)

#### 1.2 内容模型

##### 1.2.1 程序计数器PC

​       这一个区域我概括了以下几个要点：

```
1.这一区域不会出现OOM（Out Of Memory）错误的情况

2.属于线程私有，因为每一个线程都有自己的一个程序计数器，来表示当前线程执行的字节码行号

3.标识Java方法的字节码地址，而不是Native方法

4.处于CPU上，我们无法直接操作这块区域
```

#####1.2.2 虚拟机栈

​      这个区域也是我们平时口中说的堆栈的栈，关于这个块区域有如下要点：

```
1.属于线程私有，与线程的生命周期相同

2.每一个java方法被执行的时候，这个区域会生成一个栈帧

4.栈帧中存放的局部变量有8种基本数据类型，以及引用类型（对象的内存地址）

5.java方法的运行过程就是栈帧在虚拟机栈中入栈和出栈的过程

6.当线程请求的栈的深度超出了虚拟机栈允许的深度时，会抛出StackOverFlow的错误

7.当Java虚拟机动态扩展到无法申请足够内存时会抛出OutOfMemory的错误
```

##### 1.2.3 本地方法栈

​        这个区域，属于线程私有，顾名思义，区别于虚拟机栈，这里是用来处理Native方法（Java本地方法）的，而虚拟机栈是处理Java方法的。对于Native方法，Object中就有不少的Native的方法，hashCode,wait等，这些方法的执行很多时候都是借助于操作系统。 

**这一区域也有可能抛出StackOverFlowError 和 OutOfMemoryError** 

##### 1.2.4 Java堆

​        我们平时说得最多，关注得最多的一个区域，就是他了。我们后期进行的性能优化主要针对这部分内存，GC的主战场，这个地方存放的几乎所有的对象实例和数组数据。这里我大概进行了如下概括：

```
1.Java堆属于线程共享区域，所有的线程共享这一块内存区域

2.从内存回收角度，Java堆可被分为新生代和老年代，这样分能够更快的回收内存

3.从内存分配角度，Java堆可划分出线程私有的分配缓存区（Thread Local Allocation Buffer,TLAB）,这样能够更快的分配内存

4.当Java虚拟机动态扩展到无法申请足够内存时会抛出OutOfMemory的错误
```

#####1.2.5 方法区

​        方法区主要存放的是已被虚拟机加载的类信息、常量、静态变量、编译器编译后的代码等数据。GC在该区域出现的比较少。概括如下：

```
1.方法区属于线程共享区域

2.习惯性加他永久代

3.垃圾回收很少光顾这个区域，不过也是需要回收的，主要针对常量池回收，类型卸载

4.常量池用于存放编译期生成的各种字节码和符号引用，常量池具有一定的动态性，
  里面可以存放编译期生成的常量

5.运行期间的常量也可以添加进入常量池中，比如string的intern()方法。
```

##### 1.2.6 运行时常量池

​        运行时常量池也是方法区的一部分，用于存放编译器生成的各种字面量和符号引用。单独拿出来说明一下，是因为我们平时使用String比价多，涉及到这一块的知识，但这一块区域不会抛出OutOfMemoryError 



### 二、JVM内存源码示例说明

#### 2.1 代码演示

```
public class MemoryAllocateDemo {
    public static void main(String[] args){ //JVM自动寻找main方法
        /**
         * 执行第一句代码，创建一个Test实例test，在栈中分配一块内存，存放一个指向堆区实例对象的指针
         */
        Test test = new Test();
        
        /**
         * 执行第二句代码，声明定义一个int型变量（8种基本数据类型），在栈区直接分配一块内存存储这个变量的值
         */
        int date = 9;
        
        /**
         * 执行第三句代码，创建一个BirthDate实例bd1,在栈中分配一块内存，存放一个指向堆区实例对象的指针
         */
        BirthDate bd1 = new BirthDate(13,6,1991);
        
        /**
         * 执行第四句代码，创建一个BirthDate实例bd2,在栈中分配一块内存，存放一个指向堆区实例对象的指针
         */
        BirthDate bd2 = new BirthDate(30,4,1991);
        
        /**
         * 执行第五句代码，方法test1入栈帧，执行完出栈
         */
        test.test1(date);
        
        /**
         * 执行第六句代码，方法test2入栈帧，执行完出栈
         */
        test.test2(bd1);
        
        /**
         * 执行第七句代码，方法test3入栈帧，执行完出栈
         */
        test.test3(bd2);

    }
}
```

#### 2.2 演示过程一

```
1.JVM自动寻找main方法，执行第一句代码，创建一个Test类的实例test，
  在栈中分配一块内存，存放一个指向堆区对象的指针110925。

2.创建一个int型的变量date，由于是基本类型，直接在栈中存放date对应的值9。

3.创建两个BirthDate类的实例bd1、bd2，在栈中分别存放了对应的指针指向各自的对象
  ,他们在实例化时调用了有参数的构造方法，因此对象中有自定义初始值。
```

图解如下： 

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step1.png)



**内存分配调用演示一 **

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step2.png.png)

#### 2.3 演示过程二

```
1.test1方法入栈帧，以date为参数

2.value为局部变量，把value放在栈中，并且把date的值赋值给value

3.把123456赋值给value局部变量

4.test1方法执行完，value内存被释放，test1方法出栈
```

#####2.3.1 内存分配调用演示二

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step3.png) 

#####2.3.2 内存分配调用演示二

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step4.png)



#### 2.4 演示过程三

```
1.test2方法入栈帧，以实例bd1为参数

2.birthDate为局部变量，把birthDate放在栈中，把bd1的引用的值赋值给birthDate，
  也就是bd1与birthDate的地址都是指向同一个堆区的实例
3.在堆区new了一个对象，并且把这个堆区的指针保存在栈区中birthDate对应的内存空
  间，这个时候，bd1与birthDate指向了不同的堆区，那么birthDate的改变，并不会对
  bd1造成影响

4.test2方法执行完，栈中的birthDate空间被释放，test2方法出栈，但堆区的内存空间
  则要等待系统自动回收
```

##### 2.4.1 内存分配调用演示三

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step5.png)

#####2.4.2  内存分配调用演示三

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step6.png)

#####2.4.3  内存分配调用演示三

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step7.png)



#### 2.5 演示过程四

```
1.test3方法入栈帧，以实例bd2为参数

2.birthDate为局部变量，把birthDate放在栈中，把bd2的引用的值赋值给birthDate，
  也就是bd2与birthDate的地址都是指向同一个堆区的实例
3.调用birthDate的setDay方法，因为birthDate与bd2指向的是同一个对象，也就是bd2调用了setDay方法，所以，也会bd2造成影响

4.test3方法执行完，栈中的birthDate空间被释放，test3方法出栈
```

##### 2.5.1内存分配调用演示四 

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step8.png)

#####2.5.2内存分配调用演示四

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step9.png)

#####2.5.3内存分配调用演示四

![](D:\AndroidFile\Photo\JVM内存分析\memory_allocate_step10.png)



### 三、JVM内存分配小结

​      跟着上面四个步骤，走一遍，会发现其实也不会那么复杂，掌握思想就能摸到门路了，我们平时注意区分一下基本数据类型的变量和引用数据类型变量，以下进行了几点概括：

```
1.局部变量中的基本数据类型的值直接存栈中

2.局部变量中的引用数据类型在栈中存的是引用类型的指针（地址）

3.栈中的数据与堆中的数据内存回收并不是同步的，栈中的只要方法运行完，就会直接
  销毁局部变量，但堆中的对象不一定立即销毁
  
4.类的成员变量在不同对象中各不相同，都有自己的存储空间(成员变量在堆中的对象中
  )。而类的方法却是该类的所有对象共享的，只有一套，对象使用方法的时候方法才被
  压入栈，方法不使用则不占用内存
```

参考的博客：

[Android性能调优篇之探索JVM内存分配](https://www.cnblogs.com/ldq2016/p/8036464.html)

[Java之美[从菜鸟到高手演变]之JVM内存管理及垃圾回收](https://blog.csdn.net/zhangerqing/article/details/8214365)

[Java 内存分配全面浅析](https://blog.csdn.net/shimiso/article/details/8595564)

[Jvm内存模型](http://gityuan.com/2016/01/09/java-memory/)



































































































































