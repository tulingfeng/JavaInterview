## 1.构造器Constructor是否可以被重写？

不能被重写，可以被重载。



## 2.重载和重写的区别

* 重载

发生在同一个类中，方法名相同，参数类型不同、个数不同、顺序不同

```text
StringBuilder message = new StringBuilder();
StringBuilder message = new StringBuilder("hello world");
```

* 重写

子类对父类的允许访问的方法的实现过程进行重新编写，发生在子类中，方法名、参数列表必须相等。



## 3.String StringBuilder StringBuffer区别

* 可变性

`String`由`final`修饰，具有不变性

`StringBuilder、StringBuffer`都继承`AbstractStringBuilder`类，`AbstractStringBuilder `类中使用数组保存字符串`char[]value`但没有被`final`修饰，所以可变

* 线程安全

`String`常量，线程安全

`StringBuilder`线程不安全

`StringBuffer`对方法加了同步锁，线程安全

```java
public class StrigTest {
    public static void main(String[] args) {
        String s1 = "abc";
        String s2 = "abc";
        String s3 = new String("abc")
        System.out.println(s1 == s2); // true 都在字符串常量区(堆中)
        System.out.println(s1 == s3); // false 一个在字符串常量区 一个在堆中
        
        String s4 = s2.intern(); // 手动将一个字符串对象的值转移到字符串常量池中，有助于节省内存空间
        System.out.println(s1 == s3);  // true
    }
}
```





## 4.在一个静态方法内调用一个非静态成员为什么是非法的？

由于静态方法可以不通过对象进行调用



## 5.在Java中定义一个不做事且没有参数的构造方法的作用

Java在执行子类的构造方法之前，如果没有用`super()`来调用父类特定的构造方法，则会调用父类中”没有参数的构造方法”。目的是帮助子类做初始化工作。



## 6.==与equals

== 判断的是对象地址是否相等，基本数据类型比较的是值，引用数据类型比较的是地址

`equals`判断的是两个对象是否相等，两种情况：

```text
1.类没有覆盖equals方法，等价于==
2.类覆盖了equals方法，比较对象内容是否相等
```

```java
public class test1 {
    public static void main(String[] string) {
        String a = new String("ab"); // 堆
        String b = new String("ab");
        String aa = "ab"; // 常量池
        String bb = "ab";
        System.out.println(aa==bb); // true
        System.out.println(a==b); // false
        System.out.println(a.equals(b)); // true
    }
}
// String中的equals方法被重写过
```



## 7.hashcode与equals

`hashcode`是获取哈希码，实际返回的是一个`int`整数，作用是确定该对象在哈希表中索引位置。

`hashCode()`是定义在`JDK`中的`Object.java`中，这意味着`Java`中的任何类都会包括`hashCode()`函数。

**为什么要有hashCode**

判断对象是否存在哈希表的同一位置，如果`hashcode`值不同，说明对象不重复，如果相同，需要进一步比较`equals`方法。

**重写equals为什么必须重写hashcode?**

重写`equals`判断对象是否相等，如果`hashcode`没被重写，相等的对象可能没有相同的`hashcode`值，这是不能接受的。



## 8.Java中只有值传递

**Java 程序设计语言总是采用按值调用。也就是说，方法得到的是所有参数值的一个拷贝，也就是说，方法不能修改传递给它的任何参数变量的内容。**



## 9.final、static、this、super关键字总结

* final

修饰变量，**如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；如果是引用类型的变量，则在对其初始化之后便不能再让其指向另一个对象。**

修饰类，不能被继承。

修饰方法，第一个原因是把方法锁定，以防任何继承类修改它的含义；第二个原因是效率。在早期的Java实现版本中，会将final方法转为内嵌调用。但是如果方法过于庞大，可能看不到内嵌调用带来的任何性能提升（现在的Java版本已经不需要使用final方法进行这些优化了）。类中所有的private方法都隐式地指定为final。



* static

**修饰成员变量和成员方法**，被 static 修饰的成员属于类，不属于单个这个类的某个对象，被类中所有对象共享，可以并且建议通过类名调用。被static 声明的成员变量属于静态成员变量，静态变量 存放在 Java 内存区域的方法区。调用格式：`类名.静态变量名` `类名.静态方法名()`

**静态代码块:** 静态代码块定义在类中方法外, 静态代码块在非静态代码块之前执行(静态代码块—>非静态代码块—>构造方法)。 该类不管创建多少对象，静态代码块只执行一次。

**静态内部类（static修饰类的话只能修饰内部类）：** 静态内部类与非静态内部类之间存在一个最大的区别: 非静态内部类在编译完成之后会隐含地保存着一个引用，该引用是指向创建它的外围类，但是静态内部类却没有。没有这个引用就意味着：1. 它的创建是不需要依赖外围类的创建。2. 它不能使用任何外围类的非static成员变量和方法。

**静态导包(用来导入类中的静态资源，1.5之后的新特性):** 格式为：`import static` 这两个关键字连用可以指定导入某个类中的指定静态资源，并且不需要使用类名调用类中静态成员，可以直接使用类中静态成员变量和成员方法。



* this

this关键字用于引用类的当前实例。 例如：

```java
class Manager {
    Employees[] employees;
     
    void manageEmployees() {
        int totalEmp = this.employees.length;
        System.out.println("Total employees: " + totalEmp);
        this.report();
    }
     
    void report() { }
}

// this.employees.length：访问类Manager的当前实例的变量。
// this.report（）：调用类Manager的当前实例的方法。
```



* super

super关键字用于从子类访问父类的变量和方法。 例如：

```java
public class Super {
    protected int number;
     
    protected showNumber() {
        System.out.println("number = " + number);
    }
}
 
public class Sub extends Super {
    void bar() {
        super.number = 10;
        super.showNumber();
    }
}
```



## 10.深拷贝与浅拷贝

```text
1.浅拷贝
对基本数据类型进行值传递，对引用数据类型进行引用传递
2.深拷贝
对基本数据类型进行值传递，对引用数据类型创建一个新对象，并复制其内容
```



## 11.sleep()和wait()方法的区别

```text
1.sleep没有释放锁，wait释放锁
2.都可以暂停线程执行
3.sleep被用于暂停执行，wait被用于线程交互、通信
4.sleep调用后线程会自动苏醒，wait需要配合notify/notifyAll方法进行唤醒
```



## 12.为什么调用start()方法会执行run()方法，为什么不直接调用run()方法？

调用`start`方法可启动线程让线程进入就绪状态，而`run`方法只是`thread`方法的一个普通方法调用，还是在主线程中调用。



## 13.Java内存模型

线程不是直接在主内存中进行读写，线程可以把变量拷贝到本地内存。

`volatile`关键字

