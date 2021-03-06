# 二、Java内存区域与内存溢出异常

## 2.1 运行时数据区域

### 2.1.1 程序计数器

- 执行的字节码的行号解释器，字节码解释器通过改变计数器的值选取下一条要执行的指令
- ==**线程具有独立的程序计数器**==，保证在切换后恢复到正确位置
- Java方法，记录的是对应字节码的地址；对于Native方法，计数器为空
- 虚拟机规范中没有规定此区域的OOM异常（唯一一个）



### 2.1.2 Java 虚拟机栈

- ==线程私有==，生命周期与线程相同
- 描述Java方法执行的内存模型：每个方法执行会创建栈帧（Stack Frame），用于存储局部变量表，操作数栈，动态链接，方法出口等，每一个方法从调用到执行完成，对应一个栈帧入栈到出栈的过程
- 局部变量表存储**编译期可知**的数据：
  - 基本数据类型：如boolean、byte等
  - 对象引用（reference）：可能指向对象起始地址，可能指向一个对象的句柄
- 两种异常：
  - StackOverflowException：线程请求栈深度超过虚拟机允许的深度
  - OOM：虚拟机栈动态扩展时无法申请到足够的内存

### 2.1.3 本地方法栈

与虚拟机栈发挥作用类似，为Native方法服务



### 2.1.4 Java堆

- 内存管理中最大的一部分，被**所有的线程所共享**，虚拟机启动时创建，用于存放对象实例，几乎所有的对象实例都在这里分配内存
- 垃圾收集器管理的主要区域，GC堆。根据收集器的分代收集算法，堆中也可细分为：新生代和老年代；更加细致一点则可分为：Eden空间，From Survivor空间， To Survivor空间。从内存分配的角度，共享的堆可能分出多个线程私有的TLAB。但是划分是为了更好的回收或者分配内存，本质上==存储的还是对象实例==
- 逻辑上连续，物理内存中可以不连续
- 未完成实例分配并且无法扩展时，抛出OOM异常



### 2.1.5 方法区

- **各线程共享**，存放虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据
- 物理内存上可不连续，可以不实现垃圾收集。内存回收的主要目标是**常量池的回收以及类型的卸载**，但是类型卸载的条件十分严苛
- 无法满足内存需求时抛出OOM异常



### 2.1.6 运行时的常量池

- 方法区的一部分，存储编译期生成的各种字面量和符号引用
- 动态性，运行期间也可能将新的常量放入池中，比如String的intern()
- 无法申请到内存时跑出OOM异常



### 2.1.7 直接内存

- 不是Java运行时数据区的一部分，也不是虚拟机规范中定义的内存区域
- NIO类可以通过通道与缓冲区的I\O方式，使用Native函数库直接分配堆外内存，然后再通过堆内的一个DirectByteBuffer引用这块内存，避免在Java堆与Native堆中来回复制数据
- 出现OOM异常



## 2.2 HotSpot中对象机制

### 2.2.1 对象创建

1. 检查指令参数能否在常量池中定位到一个类的符号引用，并且这个类是否已经被加载，解析和初始化。如果没有会进行相应的加载过程

2. 为新生对象分配内存，对象所需要的内存是完全确定的，两种分配方式：

   - 指针碰撞：对应规整的Java内存
   - 空闲列表：对应不规整的Java内存

   分配内存不是线程安全的，解决方法有：

   - 同步处理，虚拟机采用CAS方式保证操作的原子性
   - 内存分配在特定线程的TLAB中进行

3. 将内存空间都初始化为零值，如果使用TLAB可以提前到分配TLAB时进行

4. 对对象进行必要的设置，例如类的元数据信息，哈希吗，GC分代等等，保存在对象头中

5. 执行init方法

### 2.2.2 对象的内存布局

**1）对象头**，分为两部分

第一部分存储运行时数据，包括哈希吗，GC分代，锁状态，线程持有锁等等，这一部分也叫Mark Word，32位或者是64位，实际存储的数据往往大于64位，这使得这一块通常使用一个可变的数据结构

第二部分存储类型指针，指向类的元数据的指针，通过这个指针表明这个对象属于哪个类，但也不一定保留类型指针，Java数组也会存数组长度信息

**2）实例部分**，存储代码定义中的字段内容，无论从父类继承还是子类定义的，存储的顺序受到虚拟机分配策略参数以及字段在Java源码中的定义顺序影响

**3）对齐填充**，不是必然存在，仅起占位符作用，有的内存管理系统要求对象的其实地址必须是8字节的整数倍



### 2.2.3 对象的访问定位

通过栈上的reference数据来操作堆上的具体对象，主流的访问方式有：

- 句柄访问：reference存储句柄地址，堆中划分出一个句柄池，句柄包含实例数据与类型数据各自的地址信息。**优点**：稳定的句柄地址，对象被移动时，reference也不会改变，只改变句柄内容
- 直接地址访问：reference存储对象地址。**优点**：快速，节省了一次指针定位的开销。**缺点**：堆对象布局中需要考虑如何放置访问类型数据的相关信息











### 类加载过程

七个阶段：

加载-验证-准备-解析-初始化-使用-卸载



**初始化的时机（有且只有“主动引用”时）：**

- new, getstatic, putstatic, invokestatic 字节码，初始化对象，读取或设置静态字段
- java.lang.reflect反射时
- 初始化一个类，其父类还没有初始化
- 虚拟机启动时，包含main方法的主类
- jdk1.7 中...

一些特殊情况

- 静态字段只有直接定义这个字段的类才被初始化
- 对于数组，会初始化一个由虚拟机生成的类，这个类代表了一个元素类型.*class的一维数组，数组的方法和属性都在这个类里，因此java对于数组的访问更安全
- 一个类调用其他类的常量字段时，会在编译过程中通过常量优化将这个字段存至调用的类的常量池中



#### 7.3 类加载过程

##### 7.3.1 加载

**加载完成以下三件事：**

（1）通过类的全限定名获得定义这个类的**二进制字节流**

（2）将字节流所代表的**静态存储结构**转化为方法区的**运行时数据结构**

（3）在内存中生成一个代表这个类的**java.lang.Class对象**，作为方法区数据的访问入口

对于（1），并没有指明来源于方式，因此，有各种方式：

- 从zip包，jar，ear，war
- 网络获取 applet
- 运行时计算生成，动态代理技术，“*$Proxy”
- 其他文件读取，典型是jsp文件

对于非数组类型，程序员可以自行编写类加载器loadclass()，控制字节流的获取方式

对于数组类型，创建过程：

（1）组件类型为引用类型：递归地采用加载过程，数组将会被相应的类加载器类名称空间上被标志

（2）组件类型为基本类型：数组会标记为与引导类加载器关联

##### 7.3.2 验证

目的：确保class文件的字节流中包含的信息符合当前虚拟机要求，并不会危害虚拟机安全

主要完成四个阶段的验证工作：

（1）文件格式规范：

​				魔数、主次版本号、常量中是否有不被支持的常量类型、

（2）元数据验证，字节码描述信息进行语义分析：

​				类是否有父类、是否继承了不允许被继承的类、如果不是抽象类，是否实现了要求实现的方法

（3）字节码验证：验证数据流，字节流，确定程序语义是否合法、符合逻辑

​				操作数栈的数据类型能匹配指令代码序列，跳转不会跳到方法体以外的字节码指令上

（4）符号引用验证：符号引用转化为直接引用

​				通过字符串描述的全限定名能否找到对应的类

​				指定类中是否存在符合方法的字段描述符

​				类，字段、方法的访问性

##### 7.3.3 准备

为类变量分配内存并设定初始值：

**类变量**：被static修饰的变量

初始值：除final以外，其他值有特定的初始值，如int为0，而final修饰的字段则会在类字段中表现为ConstantValue会被初始化为特定的值

##### 7.3.4 解析

**将常量池内的符号引用替换成直接引用的过程**

（1）符号引用：可以是任何形式的字面量，与虚拟机的内存布局无关，符号引用的字面量形式定义在java虚拟机规范的class文件格式中

（2）直接引用：直接指向目标的指针，相对偏移量或是间接定位到目标的句柄。与虚拟机实现的内存布局相关

对同一个符号引用进行多次解析请求很常见，此时虚拟机会缓存解析结果



解析方法主要针对 类或接口、字段、类方法、接口方法、方法类型、方法句柄、调用点限定符进行

**（1）类或接口解析：**

​		将D中的符号引用N解析为类或接口C的直接引用：

​				1、C不是数组类型，将C的全限定名传给D的类加载器加载

​				2、C是数组类型，且元素类型为引用对象，则会加载加载数组的元素类型，并由虚拟机生成一个代表数组维度与元素的数组对象

​				3、确认D是否有对C的访问权限，否则抛出**java.lang.IllegalAccessError**

**(2)   字段解析：**

​		首先会对字段所属类进行解析，解析成功则进行：

​				1、C本身包含简单名称与字段描述符都匹配的字段，则直接返回这个字段的直接引用

​				2、否则，如果C中实现了接口，则会按照继承关系从下往上递归搜索

​				3、否则，如果C不是OBJECT类，则会按照继承关系递归搜索父类

​				4、否则，抛出**java.lang.NoSuchFieldError**异常

​	**成功返回后查看权限**

**（3）类方法解析**：

​		首先会对方法所属类进行解析，解析成功则进行：

​				1、如果发现C是接口，则抛出**java.lang.InCompatibleClassChangeError**异常

​				2、在类C查找是否有匹配

​				3、在类C的父类中递归查找是否有匹配

​				4、否则，在类C实现的接口中递归查找，如果找到则说明C为抽象类，抛出**java.lang.AbstractMethodError**异常

​				5、否则抛出**java.lang.NoSuchMethodError**异常

​		**查找结束后验证权限**

**（4）接口方法解析：**

​		首先会对方法所属类进行解析，解析成功则进行：

​				1、如果发现C是类，则抛出**java.lang.InCompatibleClassChangeError**异常

​				2、在接口C中查找是否有匹配

​				3、在C的父接口中递归查找是否有匹配

​				4、否则查找结束，抛出**java.lang.NoSuchMethodError**异常

​		不存在访问权限的问题，接口中所有方法都是public



##### 7.3.5 初始化

​		初始化时执行类构造器<clinit>()的过程

​		<clinit>()方法是编译器自动收集类中所有类变量的赋值动作和静态语句块顺序所决定的





















