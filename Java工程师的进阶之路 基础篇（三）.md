> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 基础篇（一）](https://juejin.im/post/6844904152779210760)<br>
> [Java工程师的进阶之路 基础篇（二）](https://juejin.im/post/6844904153550946311)<br>
> [Java工程师的进阶之路 基础篇（三）](https://juejin.im/post/6844904153827770376)<br>

## 1. Java中的构造方法的作用和特性

#### 构造方法作用： 

定义在java类中的一个用来初始化对象的方法，用new+构造方法，创建一个新的对象，并可以给对象中的实例进行赋值。

#### 构造方法语法规则：

1. 方法名必须与类名相同
2. 无返回值类型，也不能用void修饰（有任何返回值类型的方法都不是构造方法）
3. 可以指定参数，也可以不指定参数；分为有参构造方法和无参构造方法

#### 构造方法的特点：

1. 当没有指定构造方法时，系统会自动添加无参的构造方法
2. 构造方法可以重载：方法名相同，但参数不同的多个方法，调用时会自动根据不同的参数选择相应的方法
3. 构造方法是不被继承的
4. 当我们手动的指定了构造方法时，无论是有参的还是无参的，系统都将不会再添加无参的构造方法
5. 构造方法不但可以给对象的属性赋值，还可以保证给对象的属性赋一个合理的值
6. 构造方法不能被static、final、synchronized、abstract和native修饰

## 2. Java中的 final 关键字

在Java中，final关键字可以用来修饰类、方法和变量（包括成员变量和局部变量）。

#### 修饰类

当用final修饰一个类时，表明这个类不能被继承。也就是说，如果一个类你永远不会让他被继承，就可以用final进行修饰。final类中的成员变量可以根据需要设为final，但是要注意final类中的所有成员方法都会被隐式地指定为final方法。

#### 修饰方法

final修饰的方法表示此方法已经是“最后的、最终的”含义，亦即此方法不能被重写（可以重载多个final修饰的方法）。此处需要注意的一点是：因为重写的前提是子类可以从父类中继承此方法，如果父类中final修饰的方法同时访问控制权限为private，将会导致子类中不能直接继承到此方法，因此，此时可以在子类中定义相同的方法名和参数，此时不再产生重写与final的矛盾，而是在子类中重新定义了新的方法。
 
#### 修饰变量

当final修饰一个基本数据类型时，表示该基本数据类型的值一旦在初始化后便不能发生变化；如果final修饰一个引用类型时，则在对其初始化之后便不能再让其指向其他对象了，但该引用所指向的对象的内容是可以发生变化的。本质上是一回事，因为引用的值是一个地址，final要求值，即地址的值不发生变化。

final修饰一个成员变量（属性），必须要显示初始化。这里有两种初始化方式，一种是在变量声明的时候初始化；第二种方法是在声明变量的时候不赋初值，但是要在这个变量所在的类的所有的构造函数中对这个变量赋初值。

当函数的参数类型声明为final时，说明该参数是只读型的。即你可以读取使用该参数，但是无法改变该参数的值。

## 3. Java中的 static 关键字

static可以用来修饰类的成员方法、类的成员变量，另外可以编写static代码块来优化程序性能。

```
public class Person {

    private Date birthDate;
    private static Date startDate, endDate;
    
    static {
        startDate = Date.valueOf("1990");
        endDate = Date.valueOf("2090");
    }
     
    public Person(Date birthDate) {
        this.birthDate = birthDate;
    }
     
    boolean isBornBoomer() {
        return birthDate.compareTo(startDate)>=0 && birthDate.compareTo(endDate) < 0;
    }
}
```

#### static方法

static方法一般称作静态方法，由于静态方法不依赖于任何对象就可以进行访问，因此对于静态方法来说，是没有this的，因为它不依附于任何对象，既然都没有对象，就谈不上this了。并且由于这个特性，在静态方法中不能访问类的非静态成员变量和非静态成员方法，因为非静态成员方法/变量都是必须依赖具体的对象才能够被调用。但是要注意的是，虽然在静态方法中不能访问非静态成员方法和非静态成员变量，但是在非静态成员方法中是可以访问静态成员方法/变量的。

#### static变量

static变量也称作静态变量，静态变量和非静态变量的区别是：静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初次加载时会被初始化。而非静态变量是对象所拥有的，在创建对象的时候被初始化，存在多个副本，各个对象拥有的副本互不影响。static成员变量的初始化顺序按照定义的顺序进行初始化。

#### static代码块

static关键字还有一个比较关键的作用就是用来形成静态代码块以优化程序性能。static块可以置于类中的任何地方，类中可以有多个static块。在类初次被加载的时候，会按照static块的顺序来执行每个static块，并且只会执行一次。为什么说static块可以用来优化程序性能，是因为它的特性:只会在类加载的时候执行一次。

## 4. Java中的 volatile 关键字

#### volatile可见性

线程本身并不直接与主内存进行数据的交互，而是通过线程的工作内存来完成相应的操作。这也是导致线程间数据不可见的本质原因。因此要实现volatile变量的可见性，直接从这方面入手即可。对volatile变量的写操作与普通变量的主要区别有两点：

1. 修改volatile变量时会强制将修改后的值刷新的主内存中。
2. 修改volatile变量后会导致其他线程工作内存中对应的变量值失效。因此，再读取该变量值的时候就需要重新从读取主内存中的值。

#### volatile有序性原理

volatile之所以能够阻止指令重排，是因为底层JVM里面利用了内存屏障来实现的，内存屏障主要有三点功能：

1. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
2. 它会强制将对缓存的修改操作立即写入主存；
3. 如果是写操作，它会导致其他CPU中对应的缓存行无效。

![](https://user-gold-cdn.xitu.io/2020/5/11/172039cbde5f3b43?w=417&h=392&f=png&s=11012)

## 5. Java中的 transient 关键字

1. 一旦变量被transient修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问。
2. transient关键字只能修饰变量，而不能修饰方法和类。注意，本地变量是不能被transient关键字修饰的。变量如果是用户自定义类变量，则该类需要实现Serializable接口。
3. 被transient关键字修饰的变量不再能被序列化，一个静态变量不管是否被transient修饰，均不能被序列化(如果反序列化后类中static型变量还有值，则值为当前JVM中对应static变量的值)

## 6. Java中的 synchronized 关键字

synchronized是Java中的关键字，是一种同步锁。它修饰的对象有以下几种：

1. 修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码，作用的对象是调用这个代码块的对象；
2. 修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法，作用的对象是调用这个方法的对象；
3. 修改一个静态的方法，其作用的范围是整个静态方法，作用的对象是这个类的所有对象；
4. 修改一个类，其作用的范围是synchronized后面括号括起来的部分，作用主的对象是这个类的所有对象。

#### 修饰一个代码块

1. 一个线程访问一个对象中的synchronized(this)同步代码块时，其他试图访问该对象的线程将被阻塞。
```
/**
 * 同步线程
 */
public class SyncThread implements Runnable {
   private static int count;

   public SyncThread() {
      count = 0;
   }

   public  void run() {
      synchronized(this) {
         for (int i = 0; i < 5; i++) {
            try {
               System.out.println(Thread.currentThread().getName() + ":" + (count++));
               Thread.sleep(100);
            } catch (InterruptedException e) {
               e.printStackTrace();
            }
         }
      }
   }

   public int getCount() {
      return count;
   }
}
```
2. 当一个线程访问对象的一个synchronized(this)同步代码块时，另一个线程仍然可以访问该对象中的非synchronized(this)同步代码块。
```
public class Counter implements Runnable{
   private int count;

   public Counter() {
      count = 0;
   }

   public void countAdd() {
      synchronized(this) {
         for (int i = 0; i < 5; i ++) {
            try {
               System.out.println(Thread.currentThread().getName() + ":" + (count++));
               Thread.sleep(100);
            } catch (InterruptedException e) {
               e.printStackTrace();
            }
         }
      }
   }

   // 非synchronized代码块，未对count进行读写操作，所以可以不用synchronized
   public void printCount() {
      for (int i = 0; i < 5; i ++) {
         try {
            System.out.println(Thread.currentThread().getName() + " count:" + count);
            Thread.sleep(100);
         } catch (InterruptedException e) {
            e.printStackTrace();
         }
      }
   }

   public void run() {
      String threadName = Thread.currentThread().getName();
      if (threadName.equals("A")) {
         countAdd();
      } else if (threadName.equals("B")) {
         printCount();
      }
   }
}
```
3. 指定要给某个对象加锁
```
/**
 * 银行账户类
 */
public class Account {
   String name;
   float amount;

   public Account(String name, float amount) {
      this.name = name;
      this.amount = amount;
   }
   // 存钱
   public  void deposit(float amt) {
      amount += amt;
      try {
         Thread.sleep(100);
      } catch (InterruptedException e) {
         e.printStackTrace();
      }
   }
   // 取钱
   public  void withdraw(float amt) {
      amount -= amt;
      try {
         Thread.sleep(100);
      } catch (InterruptedException e) {
         e.printStackTrace();
      }
   }

   public float getBalance() {
      return amount;
   }
}

/**
 * 账户操作类
 */
public class AccountOperator implements Runnable{
   private Account account;
   public AccountOperator(Account account) {
      this.account = account;
   }

   public void run() {
      synchronized (account) {
         account.deposit(500);
         account.withdraw(500);
         System.out.println(Thread.currentThread().getName() + ":" + account.getBalance());
      }
   }
}
```

#### 修饰一个方法

synchronized修饰一个方法很简单，就是在方法的前面加synchronized，synchronized修饰方法和修饰一个代码块类似，只是作用范围不一样，修饰代码块是大括号括起来的范围，而修饰方法范围是整个函数。
```
public synchronized void run() {
   for (int i = 0; i < 5; i ++) {
      try {
         System.out.println(Thread.currentThread().getName() + ":" + (count++));
         Thread.sleep(100);
      } catch (InterruptedException e) {
         e.printStackTrace();
      }
   }
}
```

#### 修饰一个静态的方法

静态方法是属于类的而不属于对象的。同样的，synchronized修饰的静态方法锁定的是这个类的所有对象。
```
/**
 * 同步线程
 */
public class SyncThread implements Runnable {
   private static int count;

   public SyncThread() {
      count = 0;
   }

   public synchronized static void method() {
      for (int i = 0; i < 5; i ++) {
         try {
            System.out.println(Thread.currentThread().getName() + ":" + (count++));
            Thread.sleep(100);
         } catch (InterruptedException e) {
            e.printStackTrace();
         }
      }
   }

   public synchronized void run() {
      method();
   }
}
```

#### 修饰一个类

synchronized作用于一个类T时，是给这个类T加锁，T的所有对象用的是同一把锁。
```
/**
 * 同步线程
 */
public class SyncThread implements Runnable {
   private static int count;

   public SyncThread() {
      count = 0;
   }

   public static void method() {
      synchronized(SyncThread.class) {
         for (int i = 0; i < 5; i ++) {
            try {
               System.out.println(Thread.currentThread().getName() + ":" + (count++));
               Thread.sleep(100);
            } catch (InterruptedException e) {
               e.printStackTrace();
            }
         }
      }
   }

   public synchronized void run() {
      method();
   }
}
```

## 7. Java中的异常处理和类层次结构

#### Java异常简介

程序运行时，发生的不被期望的事件，它阻止了程序按照程序员的预期正常执行，这就是异常。异常发生时，是任程序自生自灭，立刻退出终止。在Java中即，Java在编译或运行或者运行过程中出现的错误。

Java提供了更加优秀的解决办法：异常处理机制。异常处理机制能让程序在异常发生时，按照代码的预先设定的异常处理逻辑，针对性地处理异常，让程序尽最大可能恢复正常并继续执行，且保持代码的清晰。

Java中的异常可以是函数中的语句执行时引发的，也可以是程序员通过throw 语句手动抛出的，只要在Java程序中产生了异常，就会用一个对应类型的异常对象来封装异常，JRE就会试图寻找异常处理程序来处理异常。Java异常机制用到的几个关键字：**try、catch、finally、throw、throws**。

1. **try** -- 用于监听。将要被监听的代码(可能抛出异常的代码)放在try语句块之内，当try语句块内发生异常时，异常就被抛出。
2. **catch** -- 用于捕获异常。catch用来捕获try语句块中发生的异常。
3. **finally** -- finally语句块总是会被执行。它主要用于回收在try块里打开的物力资源(如数据库连接、网络连接和磁盘文件)。只有finally块，执行完成之后，才会回来执行try或者catch块中的return或者throw语句，如果finally中使用了return或者throw等终止方法的语句，则就不会跳回执行，直接停止。
4. **throw** -- 用于抛出异常。
5. **throws** -- 用在方法签名中，用于声明该方法可能抛出的异常。主方法上也可以使用throws抛出。如果在主方法上使用了throws抛出，就表示在主方法里面可以不用强制性进行异常处理，如果出现了异常，就交给JVM进行默认处理，则此时会导致程序中断执行。

三种类型的异常：

1. **检查性异常**：最具代表的检查性异常是用户错误或问题引起的异常，这是程序员无法预见的。例如要打开一个不存在文件时，一个异常就发生了，这些异常在编译时不能被简单地忽略。
2. **运行时异常**： 运行时异常是可能被程序员避免的异常。与检查性异常相反，运行时异常可以在编译时被忽略。
3. **错误**： 错误不是异常，而是脱离程序员控制的问题。错误在代码中通常被忽略。例如，当栈溢出时，一个错误就发生了，它们在编译也检查不到的。

#### Java异常的分类

异常的根接口Throwable，其下有2个子接口，Error和Exception。

1. **Error**：指的是JVM错误，这时的程序并没有执行，无法处理；
2. **Exception**：指的是程序运行中产生的异常，用户可以使用处理格式处理。

![](https://user-gold-cdn.xitu.io/2020/5/11/17203bad59fdb462?w=498&h=1024&f=jpeg&s=66888)

## 8. Java中的hashcode()方法和equals()方法

#### HashCode定义

从Object角度看，JVM每new一个Object，它都会将这个Object丢到一个Hash表中去，这样的话，下次做Object的比较或者取这个对象的时候（读取过程），它会根据对象的HashCode再从Hash表中取这个对象。这样做的目的是提高取对象的效率。若HashCode相同再去调用equal。

1. HashCode的存在主要是用于查找的快捷性，如Hashtable，HashMap等，HashCode是用来在散列存储结构中确定对象的存储地址的；
2. 如果两个对象相同， equals方法一定返回true，并且这两个对象的HashCode一定相同；除非重写了方法；
3. 如果对象的equals方法被重写，那么对象的HashCode也尽量重写，并且产生HashCode使用的对象，一定要和equals方法中使用的一致，否则就会违反上面提到的第2点；
4. 两个对象的HashCode相同，并不一定表示两个对象就相同，也就是equals方法不一定返回true，只能够说明这两个对象在散列存储结构中，如Hashtable，他们存放在同一个篮子里。

#### HashCode作用

Java中的集合（Collection）有两类，一类是List，再有一类是Set。前者集合内的元素是有序的，元素可以重复；后者元素无序，但元素不可重复。 equals方法可用于保证元素不重复，但是，如果每增加一个元素就检查一次，如果集合中现在已经有1000个元素，那么第1001个元素加入集合时，就要调用1000次equals方法。这显然会大大降低效率。

于是，Java采用了哈希表的原理。

哈希算法也称为散列算法，是将数据依特定算法直接指定到一个地址上。这样一来，当集合要添加新的元素时，先调用这个元素的HashCode方法，就一下子能定位到它应该放置的物理位置上。

1. 如果这个位置上没有元素，它就可以直接存储在这个位置上，不用再进行任何比较了；
2. 如果这个位置上已经有元素了，就调用它的equals方法与新元素进行比较，相同的话就不存了；
3. 不相同的话，也就是发生了Hash key相同导致冲突的情况，那么就在这个Hash； key的地方产生一个链表，将所有产生相同HashCode的对象放到这个单链表上去，串在一起（很少出现）。这样一来实际调用equals方法的次数就大大降低了，几乎只需要一两次。

#### hashcode()方法和equals()方法

HashCode是用于查找使用的，而equals是用于比较两个对象的是否相等的。

1. 例如内存中有这样的位置 ：0 1 2 3 4 5 6 7。
而我有个类，这个类有个字段叫ID，我要把这个类存放在以上8个位置之一，如果不用HashCode而任意存放，那么当查找时就需要到这八个位置里挨个去找，或者用二分法一类的算法。但如果用HashCode那就会使效率提高很多。 定义我们的HashCode为ID％8，比如我们的ID为9，9除8的余数为1，那么我们就把该类存在1这个位置，如果ID是13，求得的余数是5，那么我们就把该类放在5这个位置。依此类推。

2. 但是如果两个类有相同的HashCode，例如9除以8和17除以8的余数都是1，也就是说，我们先通过 HashCode来判断两个类是否存放某个桶里，但这个桶里可能有很多类，比如 hashtable，那么我们就需要再通过 equals 在这个桶里找到我们要的类。

## 9. Java中IO流以及BIO,NIO,AIO的区别

#### 以字节为导向的 stream InputStream/OutputStream

InputStream 和 OutputStream 是两个 abstact 类，对于字节为导向的 stream 都扩展这两个基类

![](https://user-gold-cdn.xitu.io/2020/5/11/17203d06c8f44e31?w=462&h=199&f=gif&s=8037)
![](https://user-gold-cdn.xitu.io/2020/5/11/17203d0a3b78c3b0?w=458&h=142&f=gif&s=6193)

#### 以字符为导向的 stream Reader/Writer

以 Unicode 字符为导向的 stream ，表示以 Unicode 字符为单位从 stream 中读取或往 stream 中写入信息。

![](https://user-gold-cdn.xitu.io/2020/5/11/17203d0d16f8dc2d?w=462&h=169&f=gif&s=6500)
![](https://user-gold-cdn.xitu.io/2020/5/11/17203d0ef3f5965f?w=465&h=222&f=gif&s=6529)

#### BIO,NIO,AIO 之间的区别

1. **BIO (Blocking I/O)**: 同步阻塞 I/O 模式，数据的读取写入必须阻塞在一个线程内等待其完成。在活动连接数不是特别高（小于单机 1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也不用过多考虑系统的过载、限流等问题。线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。
2. **NIO (Non-blocking/New I/O)**: NIO 是一种同步非阻塞的 I/O 模型，在 Java 1.4 中引入了 NIO 框架，对应 java.nio 包，提供了 Channel , Selector，Buffer 等抽象。NIO 中的 N 可以理解为 Non-blocking，不单纯是 New。它支持面向缓冲的，基于通道的 I/O 操作方法。 NIO 提供了与传统 BIO 模型中的 Socket 和 ServerSocket 相对应的 SocketChannel 和 ServerSocketChannel 两种不同的套接字通道实现,两种通道都支持阻塞和非阻塞两种模式。阻塞模式使用就像传统中的支持一样，比较简单，但是性能和可靠性都不好；非阻塞模式正好与之相反。对于低负载、低并发的应用程序，可以使用同步阻塞 I/O 来提升开发速率和更好的维护性；对于高负载、高并发的（网络）应用，应使用 NIO 的非阻塞模式来开发。
![](https://user-gold-cdn.xitu.io/2020/5/11/17203d4bb3423a1c?w=396&h=266&f=jpeg&s=15699)
3. **AIO (Asynchronous I/O)**: AIO 也就是 NIO 2。在 Java 7 中引入了 NIO 的改进版 NIO 2,它是异步非阻塞的 IO 模型。异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。AIO 是异步 IO 的缩写，虽然 NIO 在网络操作中，提供了非阻塞的方法，但是 NIO 的 IO 行为还是同步的。对于 NIO 来说，我们的业务线程是在 IO 操作准备好时，得到通知，接着就由这个线程自行进行 IO 操作，IO 操作本身是同步的。查阅网上相关资料，我发现就目前来说 AIO 的应用还不是很广泛，Netty 之前也尝试使用过 AIO，不过又放弃了。
![](https://user-gold-cdn.xitu.io/2020/5/11/17203d6bdb9ce3cd?w=504&h=325&f=png&s=13215)

## 10. 线程和进程的基本概念以及之间的关系

#### 基本概念

**线程**是进程中的一个实体,是被系统独立调度和分派的基本单位,线程自己不拥有操作系统资源,但是该线程可与同属进程的其他线程共享该进程所拥有的全部资源。

**进程**是计算机中的软件程序关于某数据集合上的一次运行活动,是系统进行资源分配和调度的基本单位,是操作系统结构的基础。
在早期面向进程设计的计算机结构中,进程是程序的基本执行实体,在当代面向线程设计的计算机结构中,进程是线程的容器。软件程序是对指令、数据及其组织形式的描述,面进程是程序的实体,通常而言,把运行在系统中的软件程序称之为进程。

#### 线程和进程的关系

线程 是 进程 划分成的更小的运行单位。线程和进程最大的不同在于基本上各进程是独立的，而各线程则不一定，因为同一进程中的线程极有可能会相互影响。从另一角度来说，进程属于操作系统的范畴，主要是同一段时间内，可以同时执行一个以上的程序，而线程则是在同一程序内几乎同时执行一个以上的程序段。

#### 线程的基本状态

1. **新建(new)**：新创建了一个线程对象。
2. **可运行(runnable)**：线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获 取cpu的使用权。
3. **运行(running)**：可运行状态(runnable)的线程获得了cpu时间片（timeslice），执行程序代码。
4. **阻塞(block)**：阻塞状态是指线程因为某种原因放弃了cpu使用权，也即让出了cpu timeslice，暂时停止运行。直到线程进入可运行(runnable)状态，才有 机会再次获得cpu timeslice转到运行(running)状态。阻塞的情况分三种：
(一). 等待阻塞：运行(running)的线程执行o.wait()方法，JVM会把该线程放 入等待队列(waiting queue)中。
(二). 同步阻塞：运行(running)的线程在获取对象的同步锁时，若该同步 锁 被别的线程占用，则JVM会把该线程放入锁池(lock pool)中。
(三). 其他阻塞: 运行(running)的线程执行Thread.sleep(long ms)或t.join()方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入可运行(runnable)状态。
5. **死亡(dead)**：线程run()、main()方法执行结束，或者因异常退出了run()                      方法，则该线程结束生命周期。死亡的线程不可再次复生。
![](https://user-gold-cdn.xitu.io/2020/5/11/17203dddc2edb18c?w=977&h=548&f=png&s=192064)

> [Java工程师的进阶之路 基础篇（一）](https://juejin.im/post/6844904152779210760)<br>
> [Java工程师的进阶之路 基础篇（二）](https://juejin.im/post/6844904153550946311)<br>
> [Java工程师的进阶之路 基础篇（三）](https://juejin.im/post/6844904153827770376)<br>