---
title: 如何读懂一个 class 文件
layout: post
date: 2019-02-28
comments: true
categories: Java
tags: [字节码] 
---

**阅读本文你能收获到**

* 了解class字节码文件的结构
* 熟悉 jvm 时如何读取解析字节码

*tip* : 本文请结合 [Java字节码内容的那些表](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/) 和 [Java字节码的那些指令](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E6%8C%87%E4%BB%A4/) 进行学习, 使用到[16进制转UTF-8字符串](https://www.bejson.com/convert/ox2str/)工具。

任何一个Class文件都对应着一个类或者接口的定义信息。但是类和接口也可以通过类加载其直接生成，所以不一定定义在文件里面。 
Class文件是一组以8字节为基础的单位的二进制流，各个数据项目严格按照顺序紧凑地排列在Class文件中，中间不会添加任何分隔符。

下面以简单例子分析

```
//TestClass.java 文件
public class TestClass {
    private int m;
    public int inc() {
        return m + 1;
    }
}
```

使用 `javac` 编译 `TestClass.java` 输出 `TestClass.Class`，得到的二进制流文件可以通过工具查看其内容。笔者在 MAC 平台上使用 `iHex-Hex Editor` 以十六进制格式查看。

```
CAFEBABE 00000034 00130A00 04000F09 00030010 07001107 00120100 016D0100 01490100 063C696E 
69743E01 00032829 56010004 436F6465 01000F4C 696E654E 756D6265 72546162 6C650100 03696E63 
01000328 29490100 0A536F75 72636546 696C6501 000E5465 7374436C 6173732E 6A617661 0C000700 
080C0005 00060100 09546573 74436C61 73730100 106A6176 612F6C61 6E672F4F 626A6563 74002100 
03000400 00000100 02000500 06000000 02000100 07000800 01000900 00001D00 01000100 0000052A 
B70001B1 00000001 000A0000 00060001 00000001 0001000B 000C0001 00090000 001F0002 00010000 
00072AB4 00020460 AC000000 01000A00 00000600 01000000 06000100 0D000000 02000E
```
上述的内容看起来像 "天书" 无从下手， 这时候就要用到 [Java字节码内容的那些表](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/)。JVM为了能统一解析这些 “天书”， 要求其内容必须按照严格的格式来排版。查看 [Class文件结构表](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#1) 可知上述内容有规律可循，为了增强上述内容的可读性，我按照 Class 文件结构重新排序了内容并标注了行数，方便后续分析。

```
line 1  : CAFEBABE   
line 2  : 00000034 
line 3  : 0013
line 4  : 0A 0004 000F
line 5  : 09 0003 0010 
line 6  : 07 0011 
line 7  : 07 0012 
line 8  : 01 0001 6D
line 9  : 01 0001 49 
line 10 : 01 0006 3C 69 6E 69 74 3E 
line 11 : 01 0003 28 29 56 
line 12 : 01 0004 43 6F 64 65 
line 13 : 01 000F 4C 69 6E 65 4E 75 6D 62 65 72 54 61 62 6C 65 
line 14 : 01 0003 69 6E 63 
line 15 : 01 0003 28 29 49 
line 16 : 01 000A 53 6F 75 72 63 65 46 69 6C 65 
line 17 : 01 000E 54 65 73 74 43 6C 61 73 73 2E 6A 61 76 61 
line 18 : 0C 0007 0008
line 19 : 0C 0005 0006 
line 20 : 01 0009 54 65 73 74 43 6C 61 73 73 
line 21 : 01 0010 6A 61 76 61 2F 6C 61 6E 67 2F 4F 62 6A 65 63 74          
line 22 : 0021    
line 23 : 0003   
line 24 : 0004  
line 25 : 0000  
line 26 : 0001 
line 27 : 0002 0005 0006 0000 
line 28 : 0002  
line 29 : 0001  0007 0008 0001  
line 30 : 0009  0000001D  0001 0001 00000005 2A B7 0001 B1 
line 31 : 0000 0001 
line 32 : 000A 00000006 0001 0000 0001 
line 33 : 0001 000B 000C 0001 
line 34 : 0009 0000001F 0002 0001 00000007  2A B4  0002 04 60 AC  
line 35 : 0000 0001 
line 36 : 000A 00000006 0001 0000 0006 
line 37 : 0001 
line 38 : 000D 00000002 000E 
```
> 以下解析流程涉及的外链比较多, 在解析过程中需要跳转查阅对应内容。基本的逻辑都某个索引指向某个数据结构, 该数据结构对应一张表格的内容,于是向下解析该表格内容长度的内容。

**line 1** 为 `魔数`, 主要是用于确认这个文件是否能被虚拟机加载， CAFEBABE 其实就是 `Baristas咖啡`。

**line 2** 为 `主次版本号` , 从 [java-class版本对应](https://en.wikipedia.org/wiki/Java_class_file#General_layout) 可知表示 Java SE 10, 向下兼容到 JDK1.1。

**line 3** 为 `常量池中常量数量` , 由于从1开始计数，第0项预留用于表示“不引用任何一个常量池项目”， 转化 16 进制之后可知常量池有 18 项常量。 每一项常量都对应 [常量表](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#2) 的某一项， 按照表中规定的每一项常量对应各自的结构。

**line 4** 为 `第 1 项常量`，为类中方法的符号引用, 格式为 *{第4项常量}.{第15项常量}* ，为 “java/lang/Object.<init>:()V”

**line 5** 为 `第 2 项常量`，为字段的符号引用, 格式为 *{第3项常量}.{第16项常量}* ， 为 "TestClass.m:I"

**line 6** 为 `第 3 项常量`，为类或接口的符号引用, 指向第 17 项常量，为 “TestClass”

**line 7** 为 `第 4 项常量`，为类或接口的符号引用, 指向第 18 项常量，为 “java/lang/Object”

**line 8** 为 `第 5 项常量`，为 UTF-8 编码的字符串, 长度为1, 转化得到 “m”

**line 9** 为 `第 6 项常量`，为 UTF-8 编码的字符串, 长度为1, 转化得到 “I”

**line 10** 为 `第 7 项常量`，为 UTF-8 编码的字符串, 长度为6, 转化得到 “<init>” 

**line 11** 为 `第 8 项常量`，为 UTF-8 编码的字符串, 长度为3, 转化得到 “()V”

**line 12** 为 `第 9 项常量`，为 UTF-8 编码的字符串, 长度为4, 转化得到 “Code”

**line 13** 为 `第 10 项常量`，为 UTF-8 编码的字符串, 长度为15, 转化得到 “LineNumberTable”

**line 14** 为 `第 11 项常量`，为 UTF-8 编码的字符串, 长度为3, 转化得到 “inc” 

**line 15** 为 `第 12 项常量`，为 UTF-8 编码的字符串, 长度为3, 转化得到 “()I”

**line 16** 为 `第 13 项常量`，为 UTF-8 编码的字符串, 长度为14, 转化得到 “SourceFile”

**line 17** 为 `第 14 项常量`，为 UTF-8 编码的字符串, 长度为10, 转化得到 “TestClass.java”

**line 18** 为 `第 15 项常量`，为字段或方法的部分符号引用, 格式为 *{第7项常量}:{第8项常量}* "<init>:()V"

**line 19** 为 `第 16 项常量`，为字段或方法的部分符号引用，格式为 *{第5项常量}:{第6项常量}*, 为 "m:I"

**line 20** 为 `第 17 项常量`，为 UTF-8 编码的字符串, 长度为9, 转化得到 “TestClass”

**line 21** 为 `第 18 项常量`，为 UTF-8 编码的字符串, 长度为16, 转化得到 “java/lang/Object”

**line 22** 为 `访问标志`， 查看 [类访问标志](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#3_1) 可知为 0x0021（0x0001|0x0020）表明这个是一个普通类，既不是接口，枚举也不是注解，被public关键字修饰但没有被声明为final和abstract

**line 23** 为 `类索引`, 对应 `第 3 项常量`, 为 “TestClass”

**line 24** 为 `父类索引`, 对应 `第 4 项常量`, 为 “java/lang/Object”

**line 25** 为 `实现接口的数目`, 0 表示没有实现任何接口

**line 26** 为 `字段的数目`, 存在一个字段需要解析

**line 27** 为 `第 1 个字段`, 查看 [字段表](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#4) 可知 访问标志（0002）为 *private*, 名字（0005）为 *m*, 描述（0006）为 *I*, 没有属性（0000）

**line 28** 为 `方法的数目`, 存在两个方法需要解析

**line 29-32** 为 `第 1 个方法`, 查看 [方法表](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#5) 可知 访问标志（0001）为 *public*, 名字（0007）为 *<init>*, 描述（0008）为 *()V*, 一个属性（0001）。由于存在一个属性，继续查看 [属性表](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#6)。从 30 行开始解析属性 0009解析为 `第 9 项常量` Code, 查看 [属性表-Code属性结构](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#6_1)及结合[Java字节码的那些指令](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E6%8C%87%E4%BB%A4/) 继续可知。

*以下为Code属性所有内容*

* 0009, 起始标示 “code”
* 0000001D, 该属性的长度为 29（ 2+2+4+5+2+2+2+4+2+2+2）, 也就是该属性后面的所有内容长度之和
* 0001, 操作数栈深度的最大值，也就是说方法执行的任意时刻操作栈不会超过深度 1
* 0001, 局部变量表需要的存储空间，单位 Slot，这个使用 1 个 Slot
* 00000005, 字节码指令长度为 5
* 2A B7 0001 B1, 代表 “ aload_0 invokespecail  java/lang/Object."<init>":()V  return”
* 0000, 需要处理的异常为 0个
* 0001, 有一个属性
	* 000A, 起始标志为 “LineNumberTable” ,用于描述Java源代码行号与字节码行号的对应关系，可查看 [LineNumberTable](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#6_4)
	* 00000006, 该属性的长度为 6
	* 0001 lineNumberInfo 长度为 1 
		* 0000, 字节码行号 0
		* 0001, Java源代码行号 1	 


**line 33-36** 为 `第 2 个方法`, 查看 [方法表]() 可知 访问标志（0001）为 *public*, 名字（000B）为 *inc*, 描述（000C）为 *()I*, 一个属性（0001）。也存在一个属性。从 34 行开始解析属性 0009解析为 `第 9 项常量` Code, 继续看该 Code 的值。

*以下为Code属性所有内容*

* 0009, 起始标示 “code”
* 0000001F, 该属性的长度为 31（2+2+4+7+2+2+2+4+2+2+2）
* 0002, 操作数栈深度的最大值，也就是说方法执行的任意时刻操作栈不会超过深度 2
* 0001, 局部变量表需要的存储空间，单位 Slot，这个使用 1 个 Slot
* 00000007, 字节码指令长度为 7
* 2A B4  0002 04 60 AC, 代表 “aload_0 getfield TestClass.m:I  iconst_1 iadd ireturn”
* 0000, 需要处理的异常为 0个
* 0001, 有一个属性
	* 000A, 起始标志为 “LineNumberTable” ,用于描述Java源代码行号与字节码行号的对应关系，可查看 [LineNumberTable](http://yummylau.com/2019/02/28/java_2019-02-28_%E5%AD%97%E8%8A%82%E7%A0%81%E5%86%85%E5%AE%B9%E8%A1%A8%E6%A0%BC/#6_4)
	* 00000006, 该属性的长度为 6
	* 0001 lineNumberInfo 长度为 1
		* 0000, 字节码行号 0
		* 0001, Java源代码行号 6	 

**line 37** 为 `属性的数目`, 存在一个属性

**line 38** 为 为 `第 1 个属性`, 000D解析为 `第 13 项常量` "sourceFile", 解析 sourceFile属性得到 “ TestClass.java”

为了验证我们的思路是否正确，可以通过 `javap` 查看 `TestClass.class` 的结构来进行对比。

```
gih-d-14238:java yummy$ javap -verbose TestClass
Classfile /Users/yummy/workspace/java/TestClass.class
  Last modified Feb 12, 2019; size 275 bytes
  MD5 checksum c287135973ee287548a00413f187a523
  Compiled from "TestClass.java"
public class TestClass
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#15         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#16         // TestClass.m:I
   #3 = Class              #17            // TestClass
   #4 = Class              #18            // java/lang/Object
   #5 = Utf8               m
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               inc
  #12 = Utf8               ()I
  #13 = Utf8               SourceFile
  #14 = Utf8               TestClass.java
  #15 = NameAndType        #7:#8          // "<init>":()V
  #16 = NameAndType        #5:#6          // m:I
  #17 = Utf8               TestClass
  #18 = Utf8               java/lang/Object
{
  public TestClass();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public int inc();
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field m:I
         4: iconst_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 6: 0
}
SourceFile: "TestClass.java"

```
实际上 `javap` 也是按照我们分析字节码的逻辑来进行内容的输出, 感兴趣的伙伴可以查看 `javap` 内部实现。
