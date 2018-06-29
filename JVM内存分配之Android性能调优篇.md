## [JVM内存分配之Android性能调优篇](https://www.cnblogs.com/ldq2016/p/8036464.html)

### 一、JVM内存模型

  ##### 1.1.1 内存模型图

​      平时我们对于Java内存都有一个比较粗略的概念，就是分堆和栈，但实际上还是复杂得多，以下给出完整内存模型： 

![](D:\AndroidFile\Photo\JVM内存分析\jvm_memory_1.png)

##### 1.1.2 内存模型图

​       相对应区域的内容为：

![](D:\AndroidFile\Photo\JVM内存分析\stack_heap_info.png)

#### 1.2 内容模型

##### 1.2.1 程序计数器PC(**Program Counter Register** )

**定义**：

```
    这是一块比较小的内存，不在Ram上，而是直接划分在CPU上的，程序员无法直接操作它，它的作用是：JVM在解释字节码文件（.class）时，存储当前线程所执行的字节码的行号，只是一种概念模型，各种JVM所采用的方式不同，字节码解释器工作时，就是通过改变程序计数器的值来选取下一条要执行的指令，分支、循环、跳转、等基础功能都是依赖此技术区完成的。还有一种情况，就是我们常说的Java多线程方面的，多线程就是通过现程轮流切换而达到的，同一时刻，一个内核只能执行一个指令，所以，对于每一个程序来说，必须有一个计数器来记录程序的执行进度，这样，当现程恢复执行的时候，才能从正确的地方开始，所以，每个线程都必须有一个独立的程序计数器，这类计数器为线程私有的内存。如果一个线程正在执行一个Java方法，则计数器记录的是字节码的指令的地址，如果执行的一个Native方法，则计数器的记录为空，此内存区是唯一一个在Java规范中没有任何OutOfMemoryError情况的区域。
```

​       这一个区域我概括了以下几个要点：

```
1.这一区域不会出现OOM（Out Of Memory）错误的情况

2.属于线程私有，因为每一个线程都有自己的一个程序计数器，来表示当前线程执行的字节码行号

3.标识Java方法的字节码地址，而不是Native方法

4.处于CPU上，我们无法直接操作这块区域
```

#####1.2.2 虚拟机栈**（JVM Stacks）** 

定义：

```
    JVM虚拟机栈就是我们常说的堆栈的栈（我们常常把内存粗略分为堆和栈），和程序计数器一样，也是线程私有的，生命周期和线程一样，每个方法被执行的时候会产生一个栈帧，用于存储局部变量表、动态链接、操作数、方法出口等信息。方法的执行过程就是栈帧在JVM中出栈和入栈的过程。局部变量表中存放的是各种基本数据类型，如boolean、byte、char、等8种，及引用类型（存放的是指向各个对象的内存地址），因此，它有一个特点：内存空间可以在编译期间就确定，运行期不在改变。这个内存区域会有两种可能的Java异常：StackOverFlowError和OutOfMemoryError。
```

​      这个区域也是我们平时口中说的堆栈的栈，关于这个块区域有如下要点：

```
1.属于线程私有，与线程的生命周期相同

2.每一个java方法被执行的时候，这个区域会生成一个栈帧

4.栈帧中存放的局部变量有8种基本数据类型，以及引用类型（对象的内存地址）

5.java方法的运行过程就是栈帧在虚拟机栈中入栈和出栈的过程

6.当线程请求的栈的深度超出了虚拟机栈允许的深度时，会抛出StackOverFlow的错误

7.当Java虚拟机动态扩展到无法申请足够内存时会抛出OutOfMemory的错误
```

##### 1.2.3 本地方法栈**（Native Method Stacks）** 

​        这个区域，属于线程私有，顾名思义，区别于虚拟机栈，这里是用来处理Native方法（Java本地方法）的，而虚拟机栈是处理Java方法的。对于Native方法，Object中就有不少的Native的方法，hashCode,wait等，这些方法的执行很多时候都是借助于操作系统。 但是JVM需要对他们做一些规范，来处理他们的执行过程。此区域，可以有不同的实现方法，向我们常用的Sun的JVM就是本地方法栈和JVM虚拟机栈是同一个。 

**这一区域也有可能抛出StackOverFlowError 和 OutOfMemoryError** 

##### 1.2.4 Java堆**（Heap）** 

​        我们平时说得最多，关注得最多的一个区域，就是他了。我们后期进行的性能优化主要针对这部分内存，GC的主战场，这个地方存放的几乎所有的对象和数组数据,**不存放基本类型和对象引用**。这里我大概进行了如下概括：

```
1.Java堆属于线程共享区域，所有的线程共享这一块内存区域

2.从内存回收角度，Java堆可被分为新生代和老年代，这样分能够更快的回收内存

3.从内存分配角度，Java堆可划分出线程私有的分配缓存区（Thread Local Allocation Buffer,TLAB）,这样能够更快的分配内存

4.当Java虚拟机动态扩展到无法申请足够内存时会抛出OutOfMemory的错误
```

#####1.2.5 方法区**（Method Area）** 

定义：

- 方法区又叫静态区，跟堆一样，被所有的线程共享，方法区包含所有的class和static变量
- 方法区中包含的都是在整个程序中永远唯一的元素，如class，static变量
- 是系统分配的一个内存逻辑区域，是JVM在装载类文件时，用于存储类型信息的(类的描述信息)。 

方法区存放的信息包括： 

**类的基本信息**：

1. 每个类的全限定名
2. 每个类的直接超类的全限定名(可约束类型转换)
3. 该类是类还是接口
4. 该类型的访问修饰符
5. 直接超接口的全限定名的有序列表

**已装载类的详细信息**

1. 运行时常量池：在方法区中，每个类型都对应一个常量池，存放该类型所用到的所有常量，常量池中存储了诸如文字字符串、final变量值、类名和方法名常量。
2. 字段信息：字段信息存放类中声明的每一个字段的信息，包括字段的名、类型、修饰符。
3. 方法信息：类中声明的每一个方法的信息，包括方法名、返回值类型、参数类型、修饰符、异常、方法的字节码。(在编译的时候，就已经将方法的局部变量、操作数栈大小等确定并存放在字节码中，在装载的时候，随着类一起装入方法区。)
4. 静态变量：类变量，类的所有实例都共享，我们只需知道，在方法区有个静态区，静态区专门存放静态变量和静态块。
5. 到类classloader的引用：到该类的类装载器的引用。
6. 到类class 的引用：虚拟机为每一个被装载的类型创建一个class实例，用来代表这个被装载的类。



```
方法区是所有线程共享的内存区域，用于存储已经被JVM加载的类信息、常量、静态变量等数据，一般来说，方法区属于

持久代（关于持久代，会在GC部分详细介绍，除了持久代，还有年轻代和老年代），也难怪Java规范将方法区描述为堆

的一个逻辑部分，但是它不是堆。方法区的垃圾回收比较棘手，就算是Sun的HotSpot VM在这方面也没有做得多么完

美。此处引入方法区中一个重要的概念：运行时常量池。主要用于存放在编译过程中产生的字面量（字面量简单理解就

是常量）和引用。一般情况，常量的内存分配在编译期间就能确定，但不一定全是，有一些可能就是运行时也可将常量

放入常量池中，如String类中有个Native方法intern()<关于intern()的详细说明，请看另一篇文章
```

[Java之美[从菜鸟到高手演变]之字符串](https://blog.csdn.net/zhangerqing/article/details/8093919)

​        方法区主要存放的是已被虚拟机加载的类信息、常量、静态变量、编译器编译后的代码等数据。GC在该区域出现的比较少。概括如下：

```
1.方法区属于线程共享区域

2.习惯性叫他持久代

3.垃圾回收很少光顾这个区域，不过也是需要回收的，主要针对常量池回收，类型卸载

4.常量池用于存放编译期生成的各种字节码和符号引用，常量池具有一定的动态性，
  里面可以存放编译期生成的常量

5.运行期间的常量也可以添加进入常量池中，比如string的intern()方法。
```

##### 1.2.6 运行时常量池

​        运行时常量池也是方法区的一部分，用于存放编译器生成的各种字面量和符号引用。单独拿出来说明一下，是因为我们平时使用String比较多，涉及到这一块的知识，但这一块区域不会抛出OutOfMemoryError ，在方法区中，每个类型都对应一个常量池，存放该类型所用到的所有常量，常量池中存储了诸如文字字符串、final变量值、类名和方法名常量。



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



### 三、常量池分析

#### 3.1 预备知识

​        基本类型和基本类型的包装类。基本类型有：byte、short、char、int、long、boolean。基本类型的包装类分别是：Byte、Short、Character、Integer、Long、Boolean。注意区分大小写。二者的区别是：基本类型体现在程序中是普通变量，基本类型的包装类是类，体现在程序中是引用变量。因此二者在内存中的存储位置不同：基本类型存储在栈中，而基本类型包装类存储在堆中。上边提到的这些包装类都实现了常量池技术，另外两种浮点数类型的包装类则没有实现。另外，String类型也实现了常量池技术。 

#### 3.2 示例

```
public class test {  
    public static void main(String[] args) {      
        objPoolTest();  
    }  
  
    public static void objPoolTest() {  
        int i = 40;  
        int i0 = 40;  
        Integer i1 = 40;  
        Integer i2 = 40;  
        Integer i3 = 0;  
        Integer i4 = new Integer(40);  
        Integer i5 = new Integer(40);  
        Integer i6 = new Integer(0);  
        Double d1=1.0;  
        Double d2=1.0;  
          
        System.out.println("i=i0\t" + (i == i0));  
        System.out.println("i1=i2\t" + (i1 == i2));  
        System.out.println("i1=i2+i3\t" + (i1 == i2 + i3));  
        System.out.println("i4=i5\t" + (i4 == i5));  
        System.out.println("i4=i5+i6\t" + (i4 == i5 + i6));      
        System.out.println("d1=d2\t" + (d1==d2));   
          
        System.out.println();          
    }  
}  

```

#### 3.3 结果

```
i=i0    true  
i1=i2   true  
i1=i2+i3        true  
i4=i5   false  
i4=i5+i6        true  
d1=d2   false  
```

#### 3.4 结果分析

**1.**i和i0均是普通类型(int)的变量，所以数据直接存储在栈中，而栈有一个很重要的特性：**栈中的数据可以共享**。当我们定义了int i = 40;，再定义int i0 = 40;这时候会自动检查栈中是否有40这个数据，如果有，i0会直接指向i的40，不会再添加一个新的40。

**2.**i1和i2均是引用类型，在栈中存储指针，因为Integer是包装类。由于Integer包装类实现了常量池技术，因此i1、i2的40均是从常量池中获取的，均指向同一个地址，因此i1=12。

**3.**很明显这是一个加法运算，**Java的数学运算都是在栈中进行的**，**Java会自动对i1、i2进行拆箱操作转化成整型**，因此i1在数值上等于i2+i3。

**4.i**4和i5均是引用类型，在栈中存储指针，因为Integer是包装类。但是由于他们各自都是new出来的，因此不再从常量池寻找数据，而是从堆中各自new一个对象，然后各自保存指向对象的指针，所以i4和i5不相等，因为他们所存指针不同，所指向对象不同。

**5.**这也是一个加法运算，和3同理。

**6.**d1和d2均是引用类型，在栈中存储指针，因为Double是包装类。但Double包装类没有实现常量池技术，因此Doubled1=1.0;相当于Double d1=new Double(1.0);，是从堆new一个对象，d2同理。因此d1和d2存放的指针不同，指向的对象不同，所以不相等。

**小结：**

**1.**以上提到的几种基本类型包装类均实现了常量池技术，但他们维护的常量仅仅是【-128至127】这个范围内的常量，如果常量值超过这个范围，就会从堆中创建对象，不再从常量池中取。比如，把上边例子改成Integer i1 = 400; Integer i2 = 400;，很明显超过了127，无法从常量池获取常量，就要从堆中new新的Integer对象，这时i1和i2就不相等了。

**2.**String类型也实现了常量池技术，但是稍微有点不同。String型是先检测常量池中有没有对应字符串，如果有，则取出来；如果没有，则把当前的添加进去。

凡是涉及内存原理，一般都是博大精深的领域，切勿听信一家之言，多读些文章。我在这只是浅析，里边还有很多猫腻，就留给读者探索思考了。希望本文能对大家有所帮助！

### 四、案例分析

#### 4.1、分析static的内存分配

```
public class Dome_Static {
 
	public static void main(String[] args) {
		Person p1 = new Person();
		p1.name = "xiaoming";
		p1.country = "chinese";
		Person p2 = new Person();
		p2.name = "xiaohong";
		p1.speak();
		p2.speak();
	}
	
}
class Person {
	String name;
	static String country;
	public void speak() {
		System.out.println("name:"+name+",country:"+country);
	}
}

//输出结果
Output:
name:xiaoming,country:chinese
name:xiaohong,country:chinese
```

1.首先，先加载Dome_Static，然后其main函数入栈，之后Person被加载。static声明的变量会随着类的加载而加载，所以在内存中只会存在一份，实例化多个对象，都共享同一个static变量，会默认初始化 

![](D:\AndroidFile\Photo\JVM内存分析\jvm_memory_4.1.01.png)

2.在栈内存为 p1 变量申请一个空间,在堆内存为Person对象申请空间,初始化完毕后将其地址值返回给p1，通过p1.name和p1.country修改其值 

![](D:\AndroidFile\Photo\JVM内存分析\jvm_memory_4.1.02.png)



3.在栈内存为 p2 变量申请一个空间,在堆内存为Person对象申请空间,初始化完毕后将其地址值返回给p2，仅仅通过p2.name修改其值 

![](D:\AndroidFile\Photo\JVM内存分析\jvm_memory_4.1.03.png)

4.打印show方法，进栈，这里就不画图了，对于栈相关的概念不清楚的可以看看在之前发的博客。简单口述下：p1.show()  show方法入栈，在方法的内部有个指向堆内存的this引用，通过该引用可找到堆内存实体，打印country时，可通过该堆内存对象找到对应的类，读取对应静态区中的字段值

 #### 4.2 最后给大家一道面试题练练手，要求写出其结果(笔试) 

```
public class StaticTest {
    
    public static int k = 0;
    public static StaticTest t1 = new StaticTest("t1");
    public static StaticTest t2 = new StaticTest("t2");
    public static int i = print("i");
    public static int n = 99;
    public int j = print("j");
     
    {
        print("构造块");
    }
     
    static{
        print("静态块");
    }
     
    public StaticTest(String str) {
        System.out.println((++k) + ":" + str + " i=" + i + " n=" + n);
        ++n;
        ++i;
    }
     
    public static int print(String str) {
        System.out.println((++k) + ":" + str + " i=" + i + " n=" + n);
        ++i;
        return ++n;
    }
    public static void main(String[] args) {
        StaticTest t = new StaticTest("init");
    }
 
}

```

结果:

```
1:j i=0 n=0
2:构造块 i=1 n=1
3:t1 i=2 n=2
4:j i=3 n=3
5:构造块 i=4 n=4
6:t2 i=5 n=5
7:i i=6 n=6
8:静态块 i=7 n=99
9:j i=8 n=100
10:构造块 i=9 n=101
11:init i=10 n=102

```

这个留给大家去思考，如果一眼便能便知道为什么是这样的输出结果，那么静态方面知识应该比较扎实了

提示一下 ：

```
1.加载的顺序：先父类的static成员变量 -> 子类的static成员变量 -> 父类的成员变量 -> 父类构造 -> 子类成员变量 -> 子类构造

2.static只会加载一次，所以通俗点讲第一次new的时候，所有的static都先会被全部载入(以后再有new都会忽略)，进行默认初始化。在从上往下进行显示初始化。这里静态代码块和静态成员变量没有先后之分，谁在上，谁就先初始化

3.构造代码块是什么？把所有构造方法中相同的内容抽取出来，定义到构造代码块中，将来在调用构造方法的时候，会去自动调用构造代码块。构造代码快优先于构造方法。


```



### 五、JVM内存分配小结

​      跟着上面四个步骤，走一遍，会发现其实也不会那么复杂，掌握思想就能摸到门路了，我们平时注意区分一下基本数据类型的变量和引用数据类型变量，以下进行了几点概括：

```

1.分清什么是实例什么是对象。Class a= new Class();此时a叫实例，而不能说a是对象。实例在栈中，对象在堆中，操作实例实际上是通过实例的指针间接操作对象。多个实例可以指向同一个对象。

2.栈中的数据和堆中的数据销毁并不是同步的。方法一旦结束，栈中的局部变量立即销毁，但是堆中对象不一定销毁。因为可能有其他变量也指向了这个对象，直到栈中没有变量指向堆中的对象时，它才销毁，而且还不是马上销毁，要等垃圾回收扫描时才可以被销毁。

3.以上的栈、堆、代码段、数据段等等都是相对于应用程序而言的。每一个应用程序都对应唯一的一个JVM实例，每一个JVM实例都有自己的内存区域，互不影响。并且这些内存区域是所有线程共享的。这里提到的栈和堆都是整体上的概念，这些堆栈还可以细分。

4.类的成员变量在不同对象中各不相同，都有自己的存储空间(成员变量在堆中的对象中)。而类的方法却是该类的所有对象共享的，只有一套，对象使用方法的时候方法才被压入栈，方法不使用则不占用内存。

5.局部变量中的引用数据类型在栈中存的是引用类型的指针（地址）

6.局部变量中的基本数据类型的值直接存栈中
```

参考的博客：

[Android性能调优篇之探索JVM内存分配](https://www.cnblogs.com/ldq2016/p/8036464.html)

[Java之美[从菜鸟到高手演变]之JVM内存管理及垃圾回收](https://blog.csdn.net/zhangerqing/article/details/8214365)

[Java 内存分配全面浅析](https://blog.csdn.net/shimiso/article/details/8595564)

[Jvm内存模型](http://gityuan.com/2016/01/09/java-memory/)

[Java基础-方法区以及static的内存分配图](https://blog.csdn.net/wang_1997/article/details/52267688)




