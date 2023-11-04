---
title: 字符串常量池的测试和分析
date: 2023-11-03 15:57:47
tags: 
    - Java
    - jvm
---

如果你准备过 Java 的面试，应该看到过一个问题：“`String s1 = new String("abc");` 这个语句创建了几个字符串对象”。这个问题曾经困扰我，当时的我不能理解这个问题想要考察的是什么？
答案中或许提及了字符串常量池，但是如果细究起来，会发现答案并不完善，有些令人困惑，甚至问题本身就有一定的误导作用。它很容易让初学者以为创建一个字符串对象和创建一个其他类型的对象在过程上是有一些区别的。
其实关键的地方在于 "abc" 而不是 `new String("abc")`。


## 字符串常量池的作用

### 字符串字面量
字面量(literal)是用于表达源代码中的一个固定值的表示法(notion)，比如代码中的整数、浮点数、字符串。简而言之，字符串字面量就是双引号包裹的字符串，例如：
```java
String s1 = "a";
```

在 Java 中，字符串对象就是一个 String 类型的对象，因此在程序运行时，String 类型的变量 s1 指向的一定是一个 String 对象。**字面量 "a" 在某一个时刻，没有经过 new 关键字，变成了一个 String 对象**。

接下来我们来思考一个问题，程序中每一个字符串字面量都要对应着生成一个单独的 String 对象吗？考虑到 Java 中 String 对象是不可变的，显然相同的字符串字面量完全可以共用一个 String 对象从而避免重复创建对象。JVM 也是这样设计的，这些可以共用的 String 对象组成了一个字符串常量池。

1. 第一次遇到某一个字符串字面量时，会在字符串常量池中创建一个 String 对象，以后遇到相同的字符串字面量，就复用该对象，不再重复创建。
2. 每一次 new 都会创建一个新的 String 对象。

ps: 以上的“遇到某一个字符串字面量”就是很纯粹地指代程序的源代码中出现用双引号括起来的字符串字面量。

### 进入字符串常量池的两种情况
因此，**如果字符串常量池中没有值为 "abc" 的 String 对象**，`new String("abc")` 语句将涉及两个 String 对象的创建，第一个是因为括号里的 "abc" 而在字符串常量池中生成的，第二个才是 new 关键字在堆中创建的；否则只会涉及一个 String 对象的创建。
为什么上面改用**如果字符串常量池中没有值为 "abc" 的 String 对象**呢？这是因为，字符串常量池里保留的 String 对象有两种产生来源：
1. 因为第一次遇到字符串字面量而生成的字符串对象。
2. 使用 `java.lang.String#intern` 主动地尝试将字符串对象放入字符串常量池。


## 常量池的分类
1. Class 文件中的常量池(Constant Pool)
2. 运行时常量池(Runtime Constant Pool)
3. 字符串常量池

```java
public class StringTableTest_1 {  
    public static void main(String[] args) {  
        String s1 = "a";  
        String s2 = "b";  
        String s3 = "ab";  
    }
}
```
使用 `javap -v .\StringTableTest_1.class` 进行反编译，摘取重要部分：
```console
Constant pool:
   #1 = Methodref          #6.#24         // java/lang/Object."<init>":()V
   #2 = String             #25            // a
   #3 = String             #26            // b
   #4 = String             #27            // ab

  #25 = Utf8               a
  #26 = Utf8               b
  #27 = Utf8               ab



0: ldc           #2                  // String a
2: astore_1
3: ldc           #3                  // String b
5: astore_2
6: ldc           #4                  // String ab
8: astore_3
9: return
```
- Class 文件中的常量池 Constant pool 会记录代码中出现的字面量（文本文件）。
- 运行时常量池是方法区的一部分，Class 文件中的常量池的内容，在类加载后，就进入了运行时常量池中（内存中的数据）。
- 字符串常量池，记录 interned string 的一个全局表，JDK 6 前在方法区，后移到堆中。

## 字符串常量池的位置和形式
在《深入理解Java虚拟机》提到：字符串常量池的位置从 JDK 7 开始，从永久代中移到了堆中。在这句话中，字符串常量池像是一个特定的内存区域，存储了 interned string 的实例。


{% asset_img "Pasted image 20231103113047.png" 字符串常量池的位置 %}

### 验证字符串常量池的位置
书中使用了以下方式来验证字符串常量池的位置。

```java
public class StringTableTest_8 {  
  
    // JDK 1.8 设置 -Xmx10m -XX:-UseGCOverheadLimit    
    // JDK 1.6 设置 -XX:MaxPerSize=10m
    public static void main(String[] args) {  
        List<String> list = new ArrayList<>();  
        int i = 0;  
        try {  
            for (int j = 0; j < 260000; j++) {  
                list.add(String.valueOf(j).intern());  
                i++;  
            }  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            System.out.println(i);  
        }  
    }  
}
```
在 JDK 8 中异常如下：
```console
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```
在 JDK 6 中异常如下：
```console
java.lang.OutOfMemoryError: PermGen space
```

同时书中也提到了，在字符串常量池的位置改变后，它只用保存第一次出现时字符串对象的引用。JDK 8 中的 intern 方法可以印证该说法，方法注释中提到：如果字符串常量池中已存在相等(equals)的字符串，那就返回已存在的对象（这样原先准备加入的对象就可以释放）；否则，将字符串对象加入字符串常量池中，直接返回对该对象的引用（不用像 JDK 6 时，复制一个对象加入常量池，返回该复制对象的引用）。

### 关于 intern 的实验

```java
public class StringTableTest_5 {  
  
    public static void main(String[] args) {  
        // "a"、"b" 作为字符串字面量，会解析得到字符串对象放入字符串常量池      
        // 但是 new String("a") 创建出来的字符串对象，不会进入字符串常量池        
        String s1 = new String("a") + new String("b");  
        // intern 方法尝试将 s1 放入 StringTable，无则放入，返回该对象引用，有则返回已存在对象的引用  
        String s2 = s1.intern();  
  
        String x = "ab";  
  
		System.out.println(s2 == x);  
		System.out.println(s1 == x); 
    }  
}

public class StringTableTest_6 {  
  
    public static void main(String[] args) {  
        // 将 "ab" 的赋值语句提前到最开始，"ab" 生成的字符串对象进入字符串常量池       
        String x = "ab";  
        String s1 = new String("a") + new String("b");  
        // intern 方法尝试将 s1 放入 StringTable，无则放入，返回该对象引用，有则返回已存在对象的引用  
        String s2 = s1.intern();  
  
        System.out.println(s2 == x);  
        System.out.println(s1 == x);  
    }  
}
```
实验结果证实了上述说法。

### 字符串常量池到底是什么？
但是 [xinxi](https://www.zhihu.com/people/logirl.cc) 提及：字符串常量池，也称为 StringTable，本质上是一个惰性维护的哈希表，是一个纯运行时的结构，只存储对 `java.lang.String` 实例的引用，而不存储 String 对象的内容。当我们提到一个字符串进入字符串常量池其实是说在这个 StringTable 中保存了对它的引用，反之，如果说没有在其中就是说 StringTable 中没有对它的引用。
[zyplanke](https://blog.csdn.net/zyplanke) 分析 StringTable 在内存中的形式时，也表达了类似的观点。
{% asset_img "Pasted image 20231103194544.png" StringTable 的内存形式 %}

尽管这个疑问似乎不妨碍我们理解很多东西，但是深究之后，真的让人困惑，网上也没有搜集到更多的信息。字符串常量池和 StringTable 是否等价？字符串常量池更准确的说法是否是“一个保存引用的 StringTable 加上分布在堆（JDK 6 以前的永久代）中的字符串实例”？
已经好几次打开 jvm 的源码，却看不懂它到底什么意思啊！！！！！难道是时候开始学 C++ 了吗。


## 进入字符串常量池的时机
前面提到了第一次遇到的字符串字面量会在某一个时刻，生成对应的字符串对象进入字符串常量池，同时也提到了，字符串常量池（StringTable）的维护是懒惰的，那么这些究竟是什么时候发生的呢？

```java
public class StringTableTest_12 {

    public static void main(String[] args) throws IOException {
        new String("ab");
    }
}
```

```console
 0: new           #2                  // class java/lang/String
 3: dup
 4: ldc           #3                  // String ab
 6: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
 9: pop
10: return
```

[RednaxelaFX](https://www.zhihu.com/people/rednaxelafx) 的文章提到：
> 在类加载阶段，JVM 会在堆中创建对应这些 class 文件常量池中的字符串对象实例，并在字符串常量池中驻留其引用。具体在 resolve 阶段执行。这些常量全局共享。

[xinxi](https://www.zhihu.com/people/logirl.cc) 的文章中补充到：
>这里说的比较笼统，没错，是 resolve 阶段，但是并不是大家想的那样，立即就创建对象并且在字符串常量池中驻留了引用。 **JVM规范里明确指定resolve阶段可以是lazy的。**
……
就 HotSpot VM 的实现来说，加载类的时候，那些字符串字面量会进入到当前类的运行时常量池，不会进入全局的字符串常量池（即在 StringTable 中并没有相应的引用，在堆中也没有对应的对象产生）。

《深入理解Java虚拟机》中提到：
>《Java虚拟机规范》之中并未规定解析阶段发生的具体时间，只要求了在执行ane-warray、checkcast、getfield、getstatic、instanceof、invokedynamic、invokeinterface、invoke-special、invokestatic、invokevirtual、ldc、ldc_w、ldc2_w、multianewarray、new、putfield和putstatic这17个用于操作符号引用的字节码指令之前，先对它们所使用的符号引用进行解析。所以虚拟机实现可以根据需要来自行判断，到底是在类被加载器加载时就对常量池中的符号引用进行解析，还是等到一个符号引用将要被使用前才去解析它。

综上可知，字符串字面量的解析是属于类加载的解析阶段，但是《Java虚拟机规范》并未规定解析发生的具体时间，只要求在执行一些字节码指令前进行，其中包括了 ldc 指令。虚拟机的具体实现，比如 Hotspot 就在执行 `ldc #indexNumber` 前触发解析，根据字符串常量池中是否已存在字符串对象决定是否创建对象，并将对象推送到栈顶。
这也证实了前文中提到的字符串字面量生成字符串对象和 new 关键字无关。

### 验证延迟实例化

使用 IDEA memory 功能，观察字符串对象的个数逐个变化。
1. 直到第一次运行到字符串字面量时，才会创建对应的字符串对象。
2. 相同的字符串常量，不会重复创建字符串对象。

```java
public class StringTableTest_4 {  
  
    public static void main(String[] args) {  
        System.out.println();  
         
        System.out.println("1");  
        System.out.println("2");  
        System.out.println("3");  
        System.out.println("4");  
        System.out.println("5");  
        System.out.println("6");  
        System.out.println("7");  
        System.out.println("8");  
        System.out.println("9");  
        System.out.println("0");  
        System.out.println("1");  
        System.out.println("2");  
        System.out.println("3");  
        System.out.println("4");  
        System.out.println("5");  
        System.out.println("6");  
        System.out.println("7");  
        System.out.println("8");  
        System.out.println("9");  
        System.out.println("0");  
    }  
}
```

{% asset_img "Pasted image 20231102225445.png" 字符串字面量延迟实例化 %}

## 字符串常量池的垃圾回收和性能优化

### 垃圾回收
前文提到字符串常量池在 JDK 7 开始移到堆中，是因为考虑在方法区中的垃圾回收是比较困难的，同时随着字节码技术的发展，CGLib 等会大量动态生成类的技术的运用使得方法区的内存紧张，将字符串常量池移到堆中，可以有效提高其垃圾回收效率。

```java
public class StringTableTest_9 {  
  
    // -Xmx10m -XX:+PrintStringTableStatistics -XX:+PrintGCDetails -verbose:gc  
    public static void main(String[] args) {  
        int i = 0;  
        try {  
            // 0->100->10000，观察统计信息中数量的变化以及垃圾回收记录
            for (int j = 0; j < 10000; j++) {  
                String.valueOf(j).intern();  
                i++;  
            }  
        } catch (Exception e) {  
            e.printStackTrace();  
        } finally {  
            System.out.println(i);  
        }  
    }  
}
```


```console
[GC (Allocation Failure) [PSYoungGen: 2048K->488K(2560K)] 2048K->856K(9728K), 0.0007745 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 


StringTable statistics:
Number of buckets       :     60013 =    480104 bytes, avg   8.000
Number of entries       :      7277 =    174648 bytes, avg  24.000
Number of literals      :      7277 =    421560 bytes, avg  57.930
Total footprint         :           =   1076312 bytes
Average bucket size     :     0.121
Variance of bucket size :     0.125
Std. dev. of bucket size:     0.354
Maximum bucket size     :         3
```

### 性能优化

#### 调整 buckets size

当 size 过小，哈希碰撞增加，链表变长，效率会变低，需要增大 buckets size。
```java
public class StringTableTest_10 {  
  
    // -XX:StringTableSize=200000 -XX:+PrintStringTableStatistics  
    // 默认->200000->1009(最小值)，观察耗时  
    public static void main(String[] args) {  
        try (BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("src/main/resources/linux.words"), StandardCharsets.UTF_8))) {  
            String line = null;  
            long start = System.nanoTime();  
            while (true) {  
                line = br.readLine();  
                if (line == null) {  
                    break;  
                }  
                line.intern();  
            }  
            System.out.println("cost: " + (System.nanoTime() - start) / 1000000) ;  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```

#### 主动运用 intern 的场景

当你需要大量缓存重复的字符串时，使用 intern 可以大大减少内存占用。
```java
public class StringTableTest_11 {  
  
    // -Xms500m -Xmx500m -XX:StringTableSize=200000 -XX:+PrintStringTableStatistics  
    public static void main(String[] args) throws IOException {  
        List<String> words = new ArrayList<>();  
        System.in.read();  
        for (int i = 0; i < 10; i++) {  
            try (BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("src/main/resources/linux.words"), StandardCharsets.UTF_8))) {  
                String line = null;  
                long start = System.nanoTime();  
                while (true) {  
                    line = br.readLine();  
                    if (line == null) {  
                        break;  
                    }    
                    // words.add(line);  
                    words.add(line.intern());  
                }  
                System.out.println("cost: " + (System.nanoTime() - start) / 1000000) ;  
            }  
        }  
        System.in.read();  
    }  
}
```

使用 VisualVM 观察字符串和 char[] 内存占用情况，可以发现提升显著。

{% asset_img "Pasted image 20231103131707.png" intern 减少内存占用 %}

## 字符串拼接

### 变量的拼接

字符串变量的拼接，底层是使用 StringBuilder 实现：`new StringBuilder().append("a").append("b").toString()`，而 toString 方法使用拼接得到的 char 数组创建一个新的 String 对象，因此 s3 和 s4 是不相同的两个对象。

```java
public class StringTableTest_2 {  
   
    public static void main(String[] args) {  
        String s1 = "a";  
        String s2 = "b";  
        String s3 = "ab";  
        String s4 = s1 + s2;  
    }  
}
```

```console
 0: ldc           #2                  // String a
 2: astore_1
 3: ldc           #3                  // String b
 5: astore_2
 6: ldc           #4                  // String ab
 8: astore_3
 9: new           #5                  // class java/lang/StringBuilder
12: dup
13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
16: aload_1
17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
20: aload_2
21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
27: astore        4
29: return
```

### 常量的拼接

字符串常量的拼接是在编译期间，因为已知结果而被优化为一个字符串常量。又因为 "ab" 字符串在 StringTable 中是已存在的，所以不会重新创建新对象。

```java
public class StringTableTest_3 {

    public static void main(String[] args) {
        String s1 = "a";
        String s2 = "b";
        String s3 = "ab";
        String s4 = s1 + s2;
        String s5 = "a" + "b";
    }
}
```

```console
 0: ldc           #2                  // String a
 2: astore_1
 3: ldc           #3                  // String b
 5: astore_2
 6: ldc           #4                  // String ab
 8: astore_3
 9: new           #5                  // class java/lang/StringBuilder
12: dup
13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
16: aload_1
17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
20: aload_2
21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
27: astore        4
29: ldc           #4                  // String ab
31: astore        5
33: return
```

## 参考文章
1. [Java 中new String("字面量") 中 "字面量" 是何时进入字符串常量池的? - xinxi的回答 - 知乎](https://www.zhihu.com/question/55994121/answer/147296098)
2. [请别再拿“String s = new String("xyz");创建了多少个String实例”来面试了吧](https://www.iteye.com/topic/774673)
3. [JVM中字符串常量池StringTable在内存中形式分析](https://blog.csdn.net/zyplanke/article/details/108699727)