---
title: JAVA类初始化
date: 2022-02-01 15:37:02
category:
- Java
tags:
- Class
---

### JAVA类初始化

1. 静态块只会调用一次
2. 可以触发静态块的有:调用类静态非final属性;Class.forName方法;new关键字
3. 非静态代码块,每次调用new关键字的时候都会调用
4. 调用类字面量时和编译器常量时,不会触发静态代码块和非静态代码块

### 示例代码

```java
[]import java.util.*;

class Initable{
    static final int staticFinal = 47;
    static final int staticFinal2 = ClassInitialization.rand.nextInt(1000);
    static {
        System.out.println("Initializing Initable");
    }

    {
      System.out.println("non static block initializing!!!!");
    }

}

class Initable2{
    static int staticNonFinal = 147;
    static{
        System.out.println("Initializing Initable2");
    }
}

class Initable3{
    static int staticNonFinal = 74;
    static{
        System.out.println("Initializing Initable3");
    }
}

public class ClassInitialization{
    public static Random rand = new Random();
    public static void main(String[]arvg){
      //调用类字面量不会调用static块,也不会调用非static块
      // Class initable = Initable.class;
      // System.out.println("11111111111111111111111111");
      // //调用类的编译期常量也不会调用类的static块,也不会调用非static块
      // System.out.println(Initable.staticFinal);
      // System.out.println("22222222222222222222222222");
      // //编译期的常量并非只要static和final修饰就是,但是这是必要的条件,静态量的调用不会调用非static块
      // System.out.println(Initable.staticFinal2);
      System.out.println("33333333333333333333333333");
      //每次使用new方法会都调用非static块和第一次调用的静态块
      Initable demo = new Initable();
      System.out.println(Initable.staticFinal2);
      System.out.println("44444444444444444444444444");
      Initable demo1 = new Initable();
      System.out.println("555555555555555555555555555555");
      //调用静态非final属性会调用static块
      System.out.println(Initable2.staticNonFinal);
      System.out.println("6666666666666666666666666666666");
      try{
        //使用Class.forName会调用static块
        Class initable3 = Class.forName("Initable3");
      }catch(Exception e){}

        System.out.println("77777777777777777777777777777777");
      try{
        Class initable4 = Class.forName("Initable3");
      }catch(Exception e){}
      System.out.println("88888888888888888888888888888888888");
      System.out.println(Initable3.staticNonFinal);

//       33333333333333333333333333
// Initializing Initable
// non static block initializing!!!!
// 836
// 44444444444444444444444444
// non static block initializing!!!!
// 555555555555555555555555555555
// Initializing Initable2
// 147
// 6666666666666666666666666666666
// Initializing Initable3
// 77777777777777777777777777777777
// 88888888888888888888888888888888888
// 74
 }
}
```

