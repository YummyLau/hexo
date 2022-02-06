---
title: 如何读懂晦涩的Class文件｜进阶必备
layout: post
date: 2020-07-27
comments: true
categories: Android
tags: [Java]
---
<!--more-->

Android 开发者日功能常开发几乎都是面向 Java/Kotlin 语法编程，对于.class 文件的关注相对较少。 当你反编译 .class 文件或在 Android 程序编译期间修改字节码做代码注入时，读懂字节码是一道绕不开的槛。

文章主要给出快速读懂一个 class 文件的方式，涉及到的 JVM 指令及字节码结构已做了整理，这部分知识平时用到的时候查一下便可，用多了自然记住了。 即使你是一个新手，按照下面的思路整合，你也可以从 0 上手。
 
读完本篇文章你会收获：

1. class 文件结构长啥样
2. JVM 操作指令有哪些
3. 如何从二进制流中读懂 class 文件


<h3 id="1"> 举个栗子🌰 带你入门</h3>

编写一个简单的 .java 文件

![demo](https://user-gold-cdn.xitu.io/2020/5/26/1724fd0979fa91c9?w=768&h=256&f=png&s=28323)

使用 `javac` 编译 `TestClass.java` 输出 `TestClass.Class`，得到的二进制流文件。可以通过工具查看其内容，在 MAC 平台上推荐使用 `iHex-Hex Editor` 以十六进制格式查看，大概长这个样子。

![demo_bytes](https://user-gold-cdn.xitu.io/2020/5/26/1724fd182c607f5b?w=939&h=275&f=png&s=80261)

这看起来像 “天书”，无从下手。实际上，任何编程产物最终都会演化成二进制，它必定是按照某种规则来申明某种逻辑的。 Java 虚拟机为了能够解析这个文件，要求其内容必须按照严格格式来排版，这种结构格式便是 **Class 文件结构**。但是单单直到里面有哪些内容还不够，虚拟机还需要一套规则来操作这些内容，这些规则便是 **字节码操作指令** 。 

要读懂这些 “天书”，先得了解 “天书” 是怎么写出来的。

<h3 id="2"> class 文件结构长啥样</h3>

`.java 文件`经过 JAVA编译器（javac）编译成中间代码字节码 `.class 文件` 。 `.class 文件` 是一个二进制文件，里面的内容已经是严格按照 Class 文件结构规则排列的。 

下面表示是 Class 的文件结构表，依次按照行从上到下解析，也就是说文件开头优先解析 magic（魔数）。

| 类型|名称|描述|数量|
|:---:|:---:|:---:|:---:|
|u4（4个字节）    | magic        |确定该文件是否为一个能被虚拟机接受的Class文件，类似于ID        |        1|
|u2（2个字节）    | minot_version        |次版本号     |        1|
|u2（2个字节）    | mahor_version        |主版本号        |        1|
|u2（2个字节）    | constant_pool_count        |常量池容量计数值，从1开始计算，0则表示不引用任何一个常量池项目        |        1|
|cp_info                | constant_pool        |常量池       |        constant_pool_count-1|
|u2（2个字节）    | access_flags        | 访问标志       |        1|
|u2（2个字节）    | this_class        |类索引       |        1|
|u2（2个字节）    | super_class        |父类索引       |        1|
|u2（2个字节）    | interfaces_count        |实现接口的数目       |        1|
|u2（4个字节）    | interfaces        |接口索引        |        interfaces_count|
|u2（4个字节）    | fields_count        |字段的数目       |        1|
|field_info    | fields        |字段内容        |        fields_count|
|u2（2个字节）    | methods_count        |方法的数目       |        1|
|method_info    | methods        |方法内容        |        methods_count|
|u2（2个字节）    | attributes_count        |属性的数目       |        1|
|attribute_info    |attributes        |属性内容        |        attributes_count|

单靠上面的表还不够，其中描述列中部分内容在字节码层面的描述，还需要根据特定表格进行查询解析，具体如下：

1. 常量池对应 **[常量表](#2_1)** 约束
2. 访问标志对应 **[访问标志表](#2_2)** 约束
3. 字段对应 **[字段表](#2_3)** 约束
4. 方法对应 **[方法表](#2_4)** 约束
5. 属性对应 **[属性表](#2_5)** 约束，同时属性内可能还需要进一步划分，对应 **Code属性结构**, **异常属性结构** 等表约束
6. 还有一些特殊字符串格式约束，比如 **[特殊字符串表](#2_6)** 等等

<h4 id="2_1">常量表</h4>

常量池主要存放两种类型

* 字面量，包含文本字符串，final的常量值等
* 符号引用，类和接口的全限定名，字段的名称和描述符，方法的名称和描述符

Class文件只保存各个方法，字段信息，不保存内存信息。只有经过运行期转换才能得到真正的内存入口。当虚拟机运行时，需要从常量池中获取到对应的符号引用，再经过类创建者运行时解析，得到具体的内存地址。

| 类型|子结构|标志|描述|
|:---:|:---:|:---:|:---:|
|CONSTANT_Utf8_info         |tag|u1 = 1|UTF-8编码的字符串|      
|-      |lenght|u2|UTF-8编码的字符串占用的字节数|      
|-           |bytes|u1|长度为lenght的UTF-8编码的字符串|      
|CONSTANT_Integer_info         |tag|u1=3|整型字面量|      
|-          |bytes|u4|按照高位在前存储的int值|      
|CONSTANT_Float_info        |tag|u1=4|浮点型字面量|      
|-         |bytes|u4|按照高位在前存储的float值|      
|CONSTANT_Long_info        |tag|u1=5|长整型字面量|      
|-          |bytes|u8|按照高位在前存储的long值|       
|CONSTANT_Double_info       |tag|u1=6|双精度浮点型字面量|      
|-         |bytes|u8|按照高位在前存储的double值|      
|CONSTANT_Class_info         |tag|u1=7|类或接口的符号引用|     
|-        |bytes|u2|指向全限定名常量项的索引|   
|CONSTANT_String_info          |tag|u1=8|字符串类型字面量|     
|-          |bytes|u2|指向字符串字面量的索引| 
|CONSTANT_Fieldref_info         |tag|u1=9|字段的符号引用|     
|-          |index|u2|指向声明字段的类或者接口描述符 CONSTANT_Class_info 的索引项| 
|-          |index|u2|指向声明字段的类或者接口描述符CONSTANT_NameAndType_info 的索引项|    
|CONSTANT_Methodred_info         |tag|u1=10|类中方法的符号引用|
|-           |index|u2|指向声明字段的类或者接口描述符 CONSTANT_Class_info 的索引项| 
|-        |index|u2|指向声明字段的类或者接口描述符CONSTANT_NameAndType_info 的索引项|    
|CONSTANT_InterfaceMethodref_info        |tag|u1=11|接口中方法的符号引用|     
|-          |index|u2|指向声明字段的类或者接口描述符 CONSTANT_Class_info 的索引项| 
|-          |index|u2|指向声明字段的类或者接口描述符CONSTANT_NameAndType_info 的索引项|    
|CONSTANT_NameAndType_info        |tag|u1=12|字段或方法的部分符号引用|     
|-         |index|u2|指向该字段或方法名称常量项的索引| 
|-          |index|u2|指向该字段或方法名称常量项的索引|    
|CONSTANT_MethodHandle_info         |tag|u1=15|表示方法句柄|     
|-            |reference_kind|u1|值必须在[1，9]中，它决定了方法句柄的类型。方法句柄类型的值表示方法句柄的字节码行为| 
|-          |reference_index|u2|值必须是对常量池的有效索引|   
|CONSTANT_MethodType_info        |tag|u1=16|识别方法类型|     
|-           |descriptor_index|u2|值必须是对常量池的有效索引，常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示方法的描述符| 
|CONSTANT_InvokeDynamic_info          |tag|u1=18|表示一个动态方法调用点|     
|-          |bootstrap_method_attar_index|u2|值必须是对当前Class文件中引导方法表的 bootstrap_methods[]数组的有效索引| 
|-          |name_and_type_index|u2|值必须是对当前常量池的有效索引，常量池在该索引处的值必须是CONSTANT_NameAndType_info结构，表示方法名和方法描述符|    


<h4 id="2_2">访问标志表</h4>

针对类，字段表，方法表中的访问标志进行划分。

* 类访问标志，用于识别一些类或者接口层次的访问信息， 包括这个Class是类还是接口，是否被定义成public类型，是否被定义成abstract类类型，如果是类的话，是否被声明为final等等

    | 标志名称|标志值|描述|
    |:---:|:---:|:---:|
    |ACC_PUBLIC| 0x0001| 是否为public类型|
    |ACC_FINAL| 0x0010| 是否被声明为final，只有类可设置|
    |ACC_SUPER| 0x0020| 是否允许使用invokespecial字节码指令的新语意，invokespecial指令的语意在JDK1.0.2发生过变化，为了区别这条指令使用哪种语意，JDK1.0.2之后编译出来的类的这个标识必须都为真|
    |ACC_INTERFACE| 0x0200| 标识这个是一个接口|
    |ACC_ABSTRACT| 0x0400| 是否为abstract类型，对于接口或者抽象类来说，此标志的值都为真，其他类型为假|
    |ACC_SYNTHETIC| 0x1000| 标识这个类并非由用户代码产生的|
    |ACC_ANNOTATION| 0x2000| 标识这是一个注解|
    |ACC_ENUM| 0x4000| 标识这是一个枚举|

* 内部类访问标志

    | 标志名称|标志值|描述|
    |:---:|:---:|:---:|
    |ACC_PUBLIC| 0x0001| 内部类是否为public|
    |ACC_PRIVATE| 0x0002| 内部类是否为private|
    |ACC_PROTECTED| 0x0004| 内部类是否为protected|
    |ACC_STATIC| 0x0008| 内部类是否为protected|
    |ACC_FINAL| 0x0010| 内部类是否为protected|
    |ACC_INTERFACE| 0x0020| 内部类是否为接口|
    |ACC_ABSTRACT| 0x0400| 内部类是否为abstract|
    |ACC_SYNTHETIC| 0x1000| 内部类是否并非由用户代码产生|
    |ACC_ANNOTATION| 0x2000| 内部类是否是一个注解|
    |ACC_ENUM| 0x4000| 内部类是否是一个枚举|

* 字段访问标志

    | 标志名称|标志值|描述|
    |:---:|:---:|:---:|
    |ACC_PUBLIC| 0x0001| 字段是否为public|
    |ACC_PRIVATE| 0x0002| 字段是否为private|
    |ACC_PROTECTED| 0x0004| 字段是否为protected|
    |ACC_STATIC| 0x0008| 字段是否为static
    |ACC_FINAL| 0x0010| 字段是否为final|
    |ACC_VOLATILE| 0x0040| 字段是否为volatile|
    |ACC_TRANSIENT| 0x0080| 字段是否为transient|
    |ACC_SYNTHETIC| 0x1000| 字段是否由编译器自动产生的|
    |ACC_ENUM| 0x4000| 字段是否为enum|

* 方法访问标志

    | 标志名称|标志值|描述|
    |:---:|:---:|:---:|
    |ACC_PUBLIC| 0x0001| 方法是否为public|
    |ACC_PRIVATE| 0x0002| 方法是否为private|
    |ACC_PROTECTED| 0x0004| 方法是否为protected|
    |ACC_STATIC| 0x0008| 方法是否为static
    |ACC_FINAL| 0x0010| 方法是否为final|
    |ACC_SYNCHRONIZED| 0x0020| 方法是否为synchronized|
    |ACC_BRIDGE| 0x0040| 方法是否由编译器产生的桥接方法|
    |ACC_VARARGS| 0x0080| 方法是否接受不定参数|
    |ACC_NATIVE| 0x0100| 方法是否为native|
    |ACC_ABSTRACT| 0x0400| 方法是否为abstract|
    |ACC_STRICTFP| 0x0800| 方法是否为strictfp|
    |ACC_SYNTHETIC| 0x1000| 方法是否由编译器自动产生的|


<h4 id="2_3">字段表</h4>

用于描述接口和类中声明的变量，包括类级别变量以及实例级别变量

| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|access_flags|1|
|u2|name_index|1|
|u2|descriptor_index|1|
|u2|attributes_count|1|
|u2|attributes|attributes_count|
其中 access_flags 见上面访问标志表中的字段访问标志

<h4 id="2_4">方法表</h4>

方法表包含访问标志，名称索引和描述符索引，属性信息等几项

| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|access_flags|1|
|u2|name_index|1|
|u2|descriptor_index|1|
|u2|attributes_count|1|
|attribute_info|attributes|attributes_count|
其中方法的access_flags见上述的方法访问标志

<h4 id="2_5">属性表</h4>

属性表用于解释Class文件中字段表，方法表中携带的属性表集合，用于描述某些场景专有的信息。

| 属性名称|使用位置|含义|
|:---:|:---:|:---:|
|Code| 方法表| Java代码编译成的字节码指令|
|ConstantValue| 字段表| final关键字定义的常量值|
|Deprecated| 类，方法表，字段表| final关键字定义的常量值|
|Exceptions| 方法表| final方法抛出的异常|
|EnclosingMethod| 类文件| 仅当一个类为局部类或者匿名类时才能拥有这个属性，这个属性用于标识这个类所在的外围方法|
|InnerClasses| 类文件| 内部类列表|
| LineNumberTable   |  Code属性 | Java源码的行号与字节码指令的对应关系  |
| LocalVariableTable  | Code属性   | 方法的局部变量描述  |
| StackMapTable   | Code属性  |   JDK1.6中新增的属性，供新的类型检查校验器（Type Checker）检查和处理目标方法的局部变量和操作数栈锁需要的类型是否匹配|
| Signature   | 类，方法表，字段表  | JDK1.5中新增的属性，这个属性用于支持泛型情况下的方法签名，在java语言中，任何类，接口，初始化方法或成员的泛型签名如果包含了类型变量（Type Variables）或者参数化类型（Parameterized Types），则Signature属性会为它记录泛型签名信息。由于java的泛型采用擦除法实现，在为了类型信息被擦除后导致签名混乱，需要这个属性记录泛型中的相关信息   |
| SourceFile   | 类文件  |记录源文件名称   |
| SourceDebugExtension   | 类文件  | JDK1.6中新增的属性，SourceDebugExtension属性用于存储额外的调试信息。譬如在进行JSP文件调试时，无法通过Java堆栈来定位到JSP文件的行号，JSR-45规范为这些非Java语言编写，却需要编译成字节码并运行在Java虚拟机中的程序提供了一个进行调试的标准机制，使用SourceDebugExtension属性就可以用于存储这个标准所新加入的调试信息  |
| Synthetic   | 类，方法表，字段表  | 标识方法或者字段是否为编译器自动生成的   |
| LocalVariableTypeTable   | 类  |JDK1.5中新增的属性，它使用特征签名代替描述符，是为了引入泛型语法之后能描述泛型参数化类型而添加的   |
| RuntimevisibleAnnotations   | 类，方法表，字段表    | JDK1.5中新增的属性，为动态注解提供支持。RuntimevisibleAnnotations 属性用于指明哪些注解是运行时（实际上运行时就是进行反射调用）可见的  |
| RuntimeInvisibleAnnotations   |类，方法表，字段表   | JDK1.5中新增的属性，与 RuntimevisibleAnnotations 属性作用刚好相反， 用于指明哪些注解是运行时不可见的  |
|  RuntimeVisibleParameterAnnotations  | 方法表  |  JDK1.5中新增的属性，作用与 RuntimevisibleAnnotations 属性类似，只不过作用对象为方法参数 |
|  RuntimeInvisibleParameterAnnotations  |  方法表 |  JDK1.5中新增的属性，作用与 RuntimeInvisibleAnnotations 属性类似，只不过作用对象为方法参数 |
|   AnnotationDetault |  方法表 | JDK1.5中新增的属性，用于记录注解类元素的默认值  |
|  BootstrapMethods  |  类文件 | JDK1.5中新增的属性，用于保存 invokedynamic 指令引用的引导方法限定符  |

上述的每一个属性，都需要从常量池中引用一个 CONSTANT_Utf8_info类型常量来标示。还包含attribute_length（u4）用于标示属性值所占用的位数，后面再跟着属性内容。下面为一些常见的属性子表结构。

* Code属性结构表，用于描述代码块

| 类型|名称|数量|
|:---:|:---:|:---:|
|u2| attribute_name_index| 1|
|u4| attribute_length| 1|
|u2|max_stack|1|
|u2|max_locals|1|
|u4|code_length|1|
|u1|code|code_lenght|
|u2|exception_table_lenght|1|
|exception_info|exception_table|exception_table_length|
|u2|attributes_count|1|
|attribute_info|attributes|attributes_count|

* 异常属性结构表，用于描述异常信息

| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|start_pc|1|
|u2|end_pc|1|
|u2|handler_pc|1|
|u2|catch_type|1|

* Exceptions属性结构表

区别与异常表，该表主要是列举中方法中可能抛出的受检查异常，也就是方法描述时throws关键字列举的异常
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|number_of_exceptions|1|
|u2|exception_index_table|number_of_exceptions|

* LineNumberTable属性结构表

用于描述Java源码行号与字节码行号之间的对应关系，默认声称到Class文件中。
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|line_number_table_length|1|
|line_number_info|line_number_table|line_number_table_length|

其中line_number_info包含start_pc和line_number两个u2类型的数据项。


*  LocalVariableTable属性结构表

用于描述栈帧中局部变量表中的变量与Java源码中定义的变量之间的关系，默认生成到Class文件中
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|local_variable_table_lenght|1|
|local_variable_info|local_variable_table|local_variable_table_lenght|

其中 local_variable_info是代表栈帧与源码中局部变量的关联，见下表

| 类型|名称|含义|数量|
|:---:|:---:|:---:|:---:|
|u2|start_pc|局部变量的生命周期开始的字节码偏移量|1|
|u2|length|局部变量的生命周期开始的作用范围覆盖长度|1|
|u2|name_index|指向常量池 CONSTANT_Utf8_info 索引|1|
|u2|descriptor_index|指向常量池 CONSTANT_Utf8_info 索引|1|
|u2|index|局部变量在栈帧局部变量表中Slot的位置|1|


* SourceFile属性结构表

用于记录生成这个Class文件的源码文件名称
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|sourcefile_index|1|
    
其中 sourcefile_index为指向常量池 CONSTANT_Utf8_info 索引

* ConstantValue属性结构表

用于通知虚拟机自动为静态变量赋值。只有被static关键字修饰的变量才可以使用这项属性。
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|constant_index|1|


* InnerClasses属性结构表

用于记录内部类与宿主类之间的关联，如果一个类中定义了内部类，编译器则会为它生成内部类INnerClasses属性
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|number_of_classes|1|
|inner_classes_info|inner_classes|number_of_classes|
    
每一个inner_classes_info代表一个内部类信息，结构如下
    
|类型|名称|含义|数量|
|:---:|:---:|:---:|:---:|
|u2|inner_class_info_index|指向常量池 CONSTANT_Class_info 索引|1|
|u2|outer_class_info_index|指向常量池 CONSTANT_Class_info 索引|1|
|u2|inner_name_index|指向常量池 CONSTANT_Utf8_info 索引，代表这个内部类的名称，如果匿名则为0|1|
|u2|inner_class_access_flags|内部类的访问标志，见上述访问标志篇章|1|

* Deprecated/Synthetic属性结构表

前者是用于标示某个类，字段或者方法是否不再推荐使用
    
后者是用于标示字段或者方法不是由Java源码直接产生，所有由非用户代码生成的方法都需要设置Synthetic属性或者ACC_SYNTHETIC标志，但是<init>和<clinit>除外。他们的结构如下
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|


* StackMapTable属性结构表

于JDK1.6之后添加在Class规范中，位于Code属性表中，该属性会在虚拟机类加载的字节码校验阶段被新类型检查检验器（Type Checker）使用。

| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|number_of_entries|1|
|stack_map_frame|stack_map_frame_entries|number_of_entries|

* Signature属性结构表

于JDK1.5发布之后添加到Class规范中，它是一个可选的定长属性，可以出现在类，属性表，方法表结构的属性表中。该属性会记录泛型签名信息，在Java语言中泛型采用的是擦除法实现的伪泛型，在字节码（Code属性）中，泛型信息编译之后都统统被擦除掉。由于无法像C#等运行时支持获取真泛型类型，添加该属性用于弥补该缺陷，现在Java反射已经能获取到泛型类型。
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|signature_index|1|
    
其中 signature_index 值必须是一个对常量池的有效索引且为 CONSTANT_Utf8_info，表示类签名，方法类型签名或字段类型签名。如果当前Signature属性是类文件的属性，则这个结构表示类签名，如果当前Signature属性是方法表的属性，则表示方法类型签名，如果当前Signature属性是字段表的属性，则表示字段类型签名。


* BootstrapMethods属性结构表

于JDK1.7发布后添加到Class文件规范中，是一个复杂变长的属性，位于类文件的属性表中。
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|attribute_name_index|1|
|u4|attribute_length|1|
|u2|num_bootstrap_methods|1|
|bootstrap_method|bootstrap_methods|num_bootstrap_methods|
    
其中bootstrap_method结构如下
    
| 类型|名称|数量|
|:---:|:---:|:---:|
|u2|bootstrap_method_ref|1|
|u2|num_bootstrap_arguments|1|
|u2|bootstrap_arguments|num_bootstrap_arguments|


<h4 id="2_6">特殊字符串</h4>

* 全限定名，比如 com/yummylau/TestClass 其实就是把类全名的 "." 换成 "/"，可以使用多个 "；"分割多个全限定名
* 简单名称，没有类型和参数修饰的方法或者字段的名字，比如方法 inc() 和字段 m 分别标示为 inc 和 m
* 描述符

| 标识字符|含义|
|:---:|:---:|
|B| 基本类型 byte| 
|C| 基本类型 char| 
|D| 基本类型 double| 
|F| 基本类型 float| 
|I| 基本类型 int| 
|J| 基本类型 long| 
|S| 基本类型 short| 
|Z| 基本类型 boolean| 
|V| 基本类型 void| 
|L| 对象类型，比如 Ljava/lang/Object| 

* 针对数组，每一个维度使用一个前置的"["字符来描述，比如定义一个 “java.lang.String[][]”数组，被记录为“[[java.lang.String;”一个整型数组 “int[]” 被记录为[I
* 针对方法，使用以下描述符
    
| 方法场景|描述符|
|:---:|:---:|
|void inc（）| （）V|
|java.lang.String toString（）| （）Ljava/lang/String;|
|int indexOf（char[]source，int sourceOffest，int sourceCount，char[] target，int targetOffset，int targetCOunt，int formIndex）| （[CII[CIII）I|

表格比较多，但是一般在使用的过程中逐步查找就可以了。

<h3 id="3"> 重新认识字节天书</h3>

![demo_bytes](https://user-gold-cdn.xitu.io/2020/5/26/1724fd182c607f5b?w=939&h=275&f=png&s=80261)

这份字节天书，我重新按照上述的 [class文件结构表](#2) 的字节区间规律，重新对字节进行了排版。

![class_bytes_2](https://user-gold-cdn.xitu.io/2020/5/26/1724fd4ddd13a39f?w=768&h=864&f=png&s=121096)

行从上到下一一对应 [class文件结构表](#2)中的行。下面开始解析每行字节的含义，基本的逻辑都是索引到某个数据结构，该数据结构对应上述的某一张表格。

**第1行** 为 `魔数`, 主要是用于确认这个文件是否能被虚拟机加载， CAFEBABE 其实就是 `Baristas咖啡`。

**第2行** 为 `主次版本号` , 从 [java-class版本对应](https://en.wikipedia.org/wiki/Java_class_file#General_layout) 可知表示 Java SE 10, 向下兼容到 JDK1.1。

**第3行** 为 `常量池中常量数量` , 由于从 1 开始计数，第 0 项预留用于表示“不引用任何一个常量池项目”， 转化 16 进制之后可知常量池有 18 项常量。 每一项常量都对应 [常量表](#2_1) 的某一项， 按照表中规定的每一项常量对应各自的结构。

**第4行** 为 `第 1 项常量`，为类中方法的符号引用, 格式为 *{第4项常量}.{第15项常量}* ，为 “java/lang/Object.< init >:()V”

**第5行** 为 `第 2 项常量`，为字段的符号引用, 格式为 *{第3项常量}.{第16项常量}* ， 为 "TestClass.m:I"

**第6行** 为 `第 3 项常量`，为类或接口的符号引用, 指向第 17 项常量，为 “TestClass”

**第7行** 为 `第 4 项常量`，为类或接口的符号引用, 指向第 18 项常量，为 “java/lang/Object”

**第8行** 为 `第 5 项常量`，为 UTF-8 编码的字符串, 长度为1, 转化得到 “m”

**第9行** 为 `第 6 项常量`，为 UTF-8 编码的字符串, 长度为1, 转化得到 “I”

**第10行** 为 `第 7 项常量`，为 UTF-8 编码的字符串, 长度为6, 转化得到 “< init >” 

**第11行** 为 `第 8 项常量`，为 UTF-8 编码的字符串, 长度为3, 转化得到 “()V”

**第12行** 为 `第 9 项常量`，为 UTF-8 编码的字符串, 长度为4, 转化得到 “Code”

**第13行** 为 `第 10 项常量`，为 UTF-8 编码的字符串, 长度为15, 转化得到 “LineNumberTable”

**第14行** 为 `第 11 项常量`，为 UTF-8 编码的字符串, 长度为3, 转化得到 “inc” 

**第15行** 为 `第 12 项常量`，为 UTF-8 编码的字符串, 长度为3, 转化得到 “()I”

**第16行** 为 `第 13 项常量`，为 UTF-8 编码的字符串, 长度为14, 转化得到 “SourceFile”

**第17行** 为 `第 14 项常量`，为 UTF-8 编码的字符串, 长度为10, 转化得到 “TestClass.java”

**第18行** 为 `第 15 项常量`，为字段或方法的部分符号引用, 格式为 *{第7项常量}:{第8项常量}* "< init >:()V"

**第19行** 为 `第 16 项常量`，为字段或方法的部分符号引用，格式为 *{第5项常量}:{第6项常量}*, 为 "m:I"

**第20行** 为 `第 17 项常量`，为 UTF-8 编码的字符串, 长度为9, 转化得到 “TestClass”

**第21行** 为 `第 18 项常量`，为 UTF-8 编码的字符串, 长度为16, 转化得到 “java/lang/Object”

**第22行** 为 `访问标志`， 查看 [访问标志表](#2_2) 可知为 0x0021（0x0001|0x0020）表明这个是一个普通类，既不是接口，枚举也不是注解，被public关键字修饰但没有被声明为final和abstract

**第23行** 为 `类索引`, 对应 `第 3 项常量`, 为 “TestClass”

**第24行** 为 `父类索引`, 对应 `第 4 项常量`, 为 “java/lang/Object”

**第25行** 为 `实现接口的数目`, 0 表示没有实现任何接口

**第26行** 为 `字段的数目`, 存在一个字段需要解析

**第27行** 为 `第 1 个字段`, 查看 [字段表](#2_3) 可知 访问标志（0002）为 *private*, 名字（0005）为 *m*, 描述（0006）为 *I*, 没有属性（0000）

**第28行** 为 `方法的数目`, 存在两个方法需要解析

**第29-32行** 为 `第 1 个方法`, 查看 [方法表](#2_4) 可知 访问标志（0001）为 *public*, 名字（0007）为 *<init>*, 描述（0008）为 *()V*, 一个属性（0001）。由于存在一个属性，继续查看 [属性表](#2_5)。从 30 行开始解析属性 0009解析为 `第 9 项常量` Code, 查看 [属性表-Code属性结构表](#2_5)。但是发现 code 块内部分字节难以解析。这是因为部分还需要结合 JVM 操作字节码指令才可以。这里先 mark 暂停。

**第33-36行** 为 `第 2 个方法`, 查看 [方法表](#2_4) 可知 访问标志（0001）为 *public*, 名字（000B）为 *inc*, 描述（000C）为 *()I*, 一个属性（0001）。也存在一个属性。从 34 行开始解析属性 0009解析为 `第 9 项常量` Code, 同上，mark 暂停。

**第37行** 为 `属性的数目`, 存在一个属性

**第38行** 为 为 `第 1 个属性`, 000D解析为 `第 13 项常量` "sourceFile", 解析 sourceFile属性得到 “ TestClass.java”

经过上述的解析，我们可得到

![class_before_code](https://user-gold-cdn.xitu.io/2020/5/26/1724fe63c02ad90f?w=768&h=845&f=png&s=134809)

到此，基本读懂了这份字节天书。 但是 Code 属性内容还是缺失。这时候我们需要一份 **字节码指令总表** 来帮助我们进一步解析 Code 里面涉及哪些指令及信息。

<h3 id="4">JVM 操作指令有哪些</h3>

下面是 JVM 操作指令表

| 字节码|助记符|指令含义|
|:---:|:---:|:---:|
|0x00 | nop | 什么都不做|
|0x01 | aconst_null | 将 null 推送至栈顶 |
|0x02 | iconst_m1 | 将 int 型 -1 推送至栈顶 |
|0x03 | iconst_0 | 将 int 型 0 推送至栈顶 |
|0x04 | iconst_1 | 将 int 型 1 推送至栈顶 |
|0x05 | iconst_2 | 将 int 型 2 推送至栈顶 |
|0x06 | iconst_3 | 将 int 型 3 推送至栈顶 |
|0x07 | iconst_4 | 将 int 型 4 推送至栈顶 |
|0x08 | iconst_5 | 将 int 型 5 推送至栈顶 |
|0x09 | lconst_0 | 将 long 型 0 推送至栈顶 |
|0x0a | lconst_1 | 将 long 型 1 推送至栈顶 |
|0x0b | fconst_0 | 将 float 型 0 推送至栈顶 |
|0x0c | fconst_1 | 将 float 型 1 推送至栈顶 |
|0x0d | fconst_2 | 将 float 型 2 推送至栈顶 | 
|0x0e | dconst_0 | 将 double 型 0 推送至栈顶 |
|0x0f | dconst_1 | 将 double 型 1 推送至栈顶 | 
|0x10 | bipush | 将单字节的常量（-128 - 127）推送至栈顶 |
|0x11 | sipush | 将一个短整形常量常量（-32768 - 32767）推送至栈顶 | 
|0x12 | ldc | 将 int, float, String 型常量值从常量池中推送至栈顶 | 
|0x13 | ldc_w | 将 int, float, String 型常量值从常量池中推送至栈顶（宽索引） | 
|0x14 | ldc2_w | 将 long 或 float 型常量值从常量池中推送至栈顶（宽索引） | 
|0x15 | iload | 将指定的 int 型本地变量推送至栈顶 | 
|0x16 | lload | 将指定的 long 型本地变量推送至栈顶 | 
|0x17 | fload | 将指定的 float 型本地变量推送至栈顶 | 
|0x18 | dload | 将指定的 dload 型本地变量推送至栈顶 | 
|0x19 | aload | 将指定的引用类型本地变量推送至栈顶 | 
|0x1a | iload_0 | 将第一个 int 型本地变量推送至栈顶 | 
|0x1b | iload_1 | 将第二个 int 型本地变量推送至栈顶 | 
|0x1c | iload_2 | 将第三个 int 型本地变量推送至栈顶 | 
|0x1d | iload_3 | 将第四个 int 型本地变量推送至栈顶 | 
|0x1e | lload_0 | 将第一个 long 型本地变量推送至栈顶 | 
|0x1f | lload_1 | 将第二个 long 型本地变量推送至栈顶 | 
|0x20 | lload_2 | 将第三个 long 型本地变量推送至栈顶 | 
|0x21 | lload_3 | 将第四个 long 型本地变量推送至栈顶 | 
|0x22 | fload_0 | 将第一个 float 型本地变量推送至栈顶 | 
|0x23 | fload_1 | 将第二个 float 型本地变量推送至栈顶 | 
|0x24 | fload_2 | 将第三个 float 型本地变量推送至栈顶 | 
|0x25 | fload_3 | 将第四个 float 型本地变量推送至栈顶 | 
|0x26 | dload_0 | 将第一个 double 型本地变量推送至栈顶 | 
|0x27 | dload_1 | 将第二个 double 型本地变量推送至栈顶 | 
|0x28 | dload_2 | 将第三个 double 型本地变量推送至栈顶 | 
|0x29 | dload_3 | 将第四个 double 型本地变量推送至栈顶 | 
|0x2a | aload_0 | 将第一个引用类型本地变量推送至栈顶 | 
|0x2b | aload_1 | 将第二个引用类型本地变量推送至栈顶 | 
|0x2c | aload_2 | 将第三个引用类型本地变量推送至栈顶 | 
|0x2d | aload_3 | 将第四个引用类型本地变量推送至栈顶 | 
|0x2e | iaload | 将 int 型数组指定索引的值推送至栈顶 | 
|0x2f | laload | 将 long 型数组指定索引的值推送至栈顶 | 
|0x30 | faload | 将 float 型数组指定索引的值推送至栈顶 | 
|0x31 | daload | 将 double 型数组指定索引的值推送至栈顶 | 
|0x32 | aaload | 将引用型数组指定索引的值推送至栈顶 | 
|0x33 | baload | 将 boolean 或 byte 型数组指定索引的值推送至栈顶 | 
|0x34 | caload | 将 char 型数组指定索引的值推送至栈顶 | 
|0x35 | saload | 将 short 型数组指定索引的值推送至栈顶 | 
|0x36 | istore | 将栈顶 int 型数值存入指定本地变量 | 
|0x37 | lstore | 将栈顶 long 型数值存入指定本地变量 | 
|0x38 | fstore | 将栈顶 float 型数值存入指定本地变量 | 
|0x39 | dstore | 将栈顶 double 型数值存入指定本地变量 | 
|0x3a | astore | 将栈顶引用型数值存入指定本地变量 | 
|0x3b | istore_0 | 将栈顶 int 型数值存入第一个本地变量 | 
|0x3c | istore_1 | 将栈顶 int 型数值存入第二个本地变量 | 
|0x3d | istore_2 | 将栈顶 int 型数值存入第三个本地变量 | 
|0x3e | istore_3 | 将栈顶 int 型数值存入第四个本地变量 | 
|0x3f | lstore_0 | 将栈顶 long 型数值存入第一个本地变量 | 
|0x40 | lstore_1 | 将栈顶 long 型数值存入第二个本地变量 | 
|0x41 | lstore_2 | 将栈顶 long 型数值存入第三个本地变量 | 
|0x42 | lstore_3 | 将栈顶 long 型数值存入第四个本地变量 | 
|0x43 | fstore_0 | 将栈顶 float 型数值存入第一个本地变量 | 
|0x44 | fstore_1 | 将栈顶 float 型数值存入第二个本地变量 | 
|0x45 | fstore_2 | 将栈顶 float 型数值存入第三个本地变量 | 
|0x46 | fstore_3 | 将栈顶 float 型数值存入第四个本地变量 | 
|0x47 | dstore_0 | 将栈顶 double 型数值存入第一个本地变量 | 
|0x48 | dstore_1 | 将栈顶 double 型数值存入第二个本地变量 | 
|0x49 | dstore_2 | 将栈顶 double 型数值存入第三个本地变量 | 
|0x4a | dstore_3 | 将栈顶 double 型数值存入第四个本地变量 | 
|0x4b | astore_0 | 将栈顶引用型数值存入第一个本地变量 | 
|0x4c | astore_1 | 将栈顶引用型数值存入第二个本地变量 | 
|0x4d | astore_2 | 将栈顶引用型数值存入第三个本地变量 | 
|0x4e | astore_3 | 将栈顶引用型数值存入第四个本地变量 | 
|0x4f | iastore | 将栈顶 int 型数值存入指定数组的指定索引位置 | 
|0x50 | lastore | 将栈顶 long 型数值存入指定数组的指定索引位置 | 
|0x51 | fastore | 将栈顶 float 型数值存入指定数组的指定索引位置 | 
|0x52 | dastore | 将栈顶 double 型数值存入指定数组的指定索引位置 | 
|0x53 | aastore | 将栈顶引用型数值存入指定数组的指定索引位置 | 
|0x54 | bastore | 将栈顶 boolean 或 byte 型数值存入指定数组的指定索引位置 | 
|0x55 | castore | 将栈顶 char 型数值存入指定数组的指定索引位置 | 
|0x56 | sastore | 将栈顶 short 型数值存入指定数组的指定索引位置 | 
|0x57 | pop | 将栈顶数值弹出（数值不能是 long 或 double 类型） | 
|0x58 | pop_2 | 将栈顶的一个（对于 long 或 double 类型）或两个数值（对于非 long 或 double 的其他类型）弹出 | 
|0x59 | dup | 复制栈顶数值并将复制值压入栈顶 | 
|0x5a | dup_x1 | 复制栈顶数值并将两个复制值压入栈顶  | 
|0x5b | dup_x2 | 复制栈顶数值并将三个（或两个）复制值压入栈顶  | 
|0x5c | dup_2 | 复制栈顶一个（对于 long 或 double 类型）或两个（非 long 或 double 的其他类型）数值并将复制值压入栈顶 ） | 
|0x5d | dup_2_x1 | dup_x1 指令的双倍版本 | 
|0x5e | dup_2_x2 | dup_x2 指令的双倍版本 | 
|0x5f | swap | 将栈最顶端的两个数值互换（数值不能是 long 或 double 类型）| 
|0x60| iadd | 将栈顶两 int 型数值相加并将结果压入栈顶 |
|0x61| ladd | 将栈顶两 long 型数值相加并将结果压入栈顶 |
|0x62| fadd | 将栈顶两 float 型数值相加并将结果压入栈顶 |
|0x63| dadd | 将栈顶两 double 型数值相加并将结果压入栈顶 |
|0x64| isub | 将栈顶两 int 型数值相减并将结果压入栈顶 |
|0x65| lsub | 将栈顶两 long 型数值相减并将结果压入栈顶 |
|0x66| fsub | 将栈顶两 float 型数值相减并将结果压入栈顶 |
|0x67| dsub | 将栈顶两 double 型数值相减并将结果压入栈顶 |
|0x68| imul | 将栈顶两 int 型数值相乘并将结果压入栈顶 |
|0x69| lmul | 将栈顶两 long 型数值相乘并将结果压入栈顶 |
|0x6a| fmul | 将栈顶两 float 型数值相乘并将结果压入栈顶 |
|0x6b | dmul | 将栈顶两 double 型数值相乘并将结果压入栈顶 |
|0x6c| idiv | 将栈顶两 int 型数值相除并将结果压入栈顶 |
|0x6d| ldiv | 将栈顶两 long 型数值相除并将结果压入栈顶 |
|0x6e| fdiv | 将栈顶两 float 型数值相除并将结果压入栈顶 |
|0x6f | ddiv | 将栈顶两 double 型数值相除并将结果压入栈顶 |
|0x70| irem | 将栈顶两 int 型数值作取模运算并将结果压入栈顶 |
|0x71| lrem | 将栈顶两 long 型数值作取模运算并将结果压入栈顶 |
|0x72| frem | 将栈顶两 float 型数值作取模运算并将结果压入栈顶 |
|0x73 | drem | 将栈顶两 double 型数值作取模运算并将结果压入栈顶 |
|0x74| ineg | 将栈顶两 int 型数值作负并将结果压入栈顶 |
|0x75| lneg | 将栈顶两 long 型数值作负并将结果压入栈顶 |
|0x76| fneg | 将栈顶两 float 型数值作负并将结果压入栈顶 |
|0x77 | dneg | 将栈顶两 double 型数值作负并将结果压入栈顶 |
|0x78| ishl | 将栈顶两 int 型数值左移位指定位数并将结果压入栈顶 |
|0x79| lshl | 将栈顶两 long 型数值左移位指定位数并将结果压入栈顶 |
|0x7a| ishr | 将栈顶两 int 型数值右（带符号）移位指定位数并将结果压入栈顶 |
|0x7b | lshr | 将栈顶两 long 型数值右（带符号）移位指定位数并将结果压入栈顶 |
|0x7c| iushr | 将栈顶两 int 型数值右（无符号）移位指定位数并将结果压入栈顶 |
|0x7d | lushr | 将栈顶两 long 型数值右（无符号）移位指定位数并将结果压入栈顶 |
|0x7e| iand | 将栈顶两 int 型数值作 “按位与” 并将结果压入栈顶 |
|0x7f | land | 将栈顶两 long 型数值作 “按位与” 并将结果压入栈顶 |
|0x80| ior | 将栈顶两 int 型数值作 “按位或” 并将结果压入栈顶 |
|0x81 | lor | 将栈顶两 long 型数值作 “按位或” 并将结果压入栈顶 |
|0x82| ixor | 将栈顶两 int 型数值作 “按位异或” 并将结果压入栈顶 |
|0x83 | lxor | 将栈顶两 long 型数值作 “按位异或” 并将结果压入栈顶 |
|0x84 | iinc | 直接对 int 型变量增加指定值（如i++， i--， i+=2等） |
|0x85 | i2l | 将栈顶 int 型数值强制转成 long 型数值并将结果压入栈顶 |
|0x86 | i2f | 将栈顶 int 型数值强制转成 float 型数值并将结果压入栈顶 |
|0x87 | i2d | 将栈顶 int 型数值强制转成 double 型数值并将结果压入栈顶 |
|0x88 | l2i | 将栈顶 long 型数值强制转成 int 型数值并将结果压入栈顶 |
|0x89 | l2f | 将栈顶 long 型数值强制转成 float 型数值并将结果压入栈顶 |
|0x8a | l2d | 将栈顶 long 型数值强制转成 double 型数值并将结果压入栈顶 |
|0x8b | f2i | 将栈顶 float 型数值强制转成 int 型数值并将结果压入栈顶 |
|0x8c | f2l | 将栈顶 float 型数值强制转成 long 型数值并将结果压入栈顶 |
|0x8d | f2d | 将栈顶 float 型数值强制转成 double 型数值并将结果压入栈顶 |
|0x8e | d2i | 将栈顶 double 型数值强制转成 int 型数值并将结果压入栈顶 |
|0x8f | d2l | 将栈顶 double 型数值强制转成 long 型数值并将结果压入栈顶 |
|0x90 | d2f | 将栈顶 double 型数值强制转成 float 型数值并将结果压入栈顶 |
|0x91 | i2b | 将栈顶 int 型数值强制转成 byte 型数值并将结果压入栈顶 |
|0x92 | i2c | 将栈顶 int 型数值强制转成 char 型数值并将结果压入栈顶 |
|0x93 | i2s | 将栈顶 int 型数值强制转成 short 型数值并将结果压入栈顶 |
|0x94 | lcmp | 比较栈顶两 long 型数值的大小，并将结果（1， 0 或 -1）压入栈顶 |
|0x95 | fcmpl | 比较栈顶两 float 型数值的大小，并将结果（1， 0 或 -1）压入栈顶; 当其中一个数值为 “NaN” 时，将 -1 压入栈顶 |
|0x96 | fcmpg | 比较栈顶两 float 型数值的大小，并将结果（1， 0 或 -1）压入栈顶; 当其中一个数值为 “NaN” 时，将 1 压入栈顶|
|0x97 | dcmpl | 比较栈顶两 double 型数值的大小，并将结果（1， 0 或 -1）压入栈顶; 当其中一个数值为 “NaN” 时，将 -1 压入栈顶 |
|0x98 | dcmpg | 比较栈顶两 double 型数值的大小，并将结果（1， 0 或 -1）压入栈顶; 当其中一个数值为 “NaN” 时，将 1 压入栈顶|
|0x99 | ifeg | 当栈顶 int 型数值等于 0 时跳转|
|0x9a | ifne | 当栈顶 int 型数值不等于 0 时跳转|
|0x9b | iflt | 当栈顶 int 型数值小于 0 时跳转|
|0x9c | ifge | 当栈顶 int 型数值大于或等于 0 时跳转|
|0x9d | ifgt | 当栈顶 int 型数值大于 0 时跳转|
|0x9e | ifle | 当栈顶 int 型数值小于或等于 0 时跳转|
|0x9f | if_icmpeq | 比较栈顶两 int 型数值的大小，当结果等于 0 时跳转|
|0xa0 | if_icmpne | 比较栈顶两 int 型数值的大小，当结果不等于 0 时跳转|
|0xa1 | if_icmplt | 比较栈顶两 int 型数值的大小，当结果小于 0 时跳转|
|0xa2 | if_icmpge | 比较栈顶两 int 型数值的大小，当结果大于或等于 0 时跳转|
|0xa3 | if_icmpgt | 比较栈顶两 int 型数值的大小，当结果大于 0 时跳转|
|0xa4 | if_icmple | 比较栈顶两 int 型数值的大小，当结果小于或等于 0 时跳转|
|0xa5 | if_icmpeq | 比较栈顶两引用型数值，当结果相等时跳转|
|0xa6 | if_icmpnc | 比较栈顶两引用型数值，当结果不相等时跳转|
|0xa7 | goto | 无条件跳转|
|0xa8 | jsr | 跳转至指定的 16 位 offset 位置，并将 jsr 的下一条指令地址压入栈顶|
|0xa9 | ret | 返回至本地变量指定的 index 的指令位置（一般与 jsr 或 jsr_w 联合使用）|
|0xaa | tableswitch | 用于 switch 条件跳转， case 值连续（可变长度指令）|
|0xab | lookupswitch | 用于 switch 条件跳转， case 值连不续（可变长度指令）|
|0xac | ireturn | 从当前方法返回 int |
|0xad | lreturn | 从当前方法返回 long |
|0xae | freturn | 从当前方法返回 float |
|0xaf | dreturn | 从当前方法返回 double |
|0xb0 | areturn | 从当前方法返回对象引用 |
|0xb1 | return | 从当前方法返回 void |
|0xb2 | getstatic | 获取指定类的静态域，并将其值压入栈顶|
|0xb3 | putstatic | 为指定的类的静态域赋值|
|0xb4| getfield| 获取指定类的实例域，并将其值压入栈顶|
|0xb5| putfield | 为指定的类的实例域赋值|
|0xb6| invokevirtual | 调用实例方法 | 
|0xb7| invokespecial | 调用超类构造方法， 实例初始化方法，私有方法|
|0xb8| invokestatic| 调用静态方法 |
|0xb9| invokeinterface| 调用接口方法 |
|0xba| invokedynamic| 调用动态方法 |
|0xbb| new| 创建一个对象，并将其引用值压入栈顶 |
|0xbc| newarray | 创建一个指定的原始类型（如 int， float等）的数组，并将其引用值压入栈顶 |
|0xbd| anewarray | 创建一个引用型（如 类，接口，数组）的数组，并将其引用值压入栈顶 |
|0xbe| arraylength | 获得数组的长度值并压入栈顶 | 
|0xbf| athrow | 将栈顶的异常抛出 |
|0xc0| checkcast | 检验类型转换， 检验未通过将抛出 ClassCastException | 
|0xc1 | instanceof| 检验对象是否时指定类的实例， 如果是， 则将 1 压入栈顶，否则将 0 压入栈顶|
|0xc2 | monitorenter | 获得对象的锁，用于同步方法或同步块|
|0xc3 | monitorexit | 释放对象的锁，用于同步方法或同步块|
|0xc4 | wide | 扩展本地变量的宽度 |
|0xc5| multianewarray| 创建指定类型和指定维度的多维数组（执行该指令时，操作栈中必须包含各维度的长度值），并将其引用值压入栈顶|
|0xc6| ifnull | 为 null 时跳转|
|0xc7| ifnonnull | 不为 null 时跳转|
|0xc8 | goto_w | 无条件跳转（宽索引）|
|0xc9 | jsr_w| 跳转至指定的 32 位 offset 位置，并将 jsr_w 的下一条指令地址压入栈顶 |

上述表格内容更为具体的信息可参考 [官方JVM指令文档](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html)，除上述表外，还需要认识一些数据类型及转化对应规则，同时再对上述指令的使用场景做一些总结划分。

<h4 id="4_1">数据类型在指令中的转化</h4>

| 数据类型|byte|short|int|long|float|double|char|reference|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|简化转化|b|s|i|l|f|d|c|a|


<h4 id="4_2">指令集支持的数据类型</h4>

下面表格中T+指令构成 opcode, T 为上面表格各数据类型的简化转化。

| opcode|byte|short|int|long|float|double|char|reference|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|Tipush| bipush    |    sipush |    |   |    |    |    |     |
|Tconst| |   |  iconst  | lconst  | fconst   | dconst   |    | aconst   |
|Tload| |   |  iload  | lload  | fload   | dload   |    | aload   |
|  Tstore   |       |          | istore      | lstore      |  fstore      | dstore      |      | astore        |
|   Tinc  |       |      |        | iinc      |      |        |       |      |         |
| Taload    |  baload     | saload      | iaload       | laload      |  faload      | daload      | caload     | aaload        |
|  Tastore   |  bastore     |  sastore      | iastore       | lastore      |  fastore      | dastore      | castore     | aastore        |
|  Tadd   |       |       | iadd        | ladd       |  fadd       | dadd       |      |         |
|  Tsub   |       |        | isub       | lsub     |  fsub       | dsub       |      |        |
| Tmul    |       |       | imul       | lmul      |  fmul      | dmul      |      |         |
|  Tdiv   |       |        | idiv       | ldiv      |  fdiv      | ddiv      |      |        |
|  Trem   |       |         | irem       | lrem      |  frem      | drem      |     |        |
|  Tneg   |       |        | ineg       | lneg      |  fneg      | dneg      |      |         |
|  Tshl   |       |        | ishl       | lshl      |        |      |      |         |
| Tshr    |       |         | ishr       | lshr      |        |       |      |        |
|  Tushr   |       |         | iushr       | lushr      |        |       |      |         |
|  Tand   |       |       | iand       | land      |        |       |      |        |
|   Tor  |       |        | ior       | lor      |        |      |      |        |
|   Txor  |       |        | ixor       | lxor      |       |       |     |         |
|  i2T   |  i2b     |  i2s      |        | i2l      |  i2f      | i2d      |      |         |
|  l2T   |       |        | l2i       |       |  l2f      | l2d      |     |         |
|  f2T   |       |         | f2i       | f2l      |        | f2d      |      |         |
|   d2T  |       |       | d2i       | d2l      |  d2f      |       |     |         |
|  Tcmp   |       |         |        |    lcmp   |     |     |      |         |
|  Tcml   |       |         |        |       |  fcml      | dcml      |      |         |
|  Tcmpg   |       |       |       |       |  fcmpg     | dcmpg      |      |         |
|  if_TcmpOP   |       |       | if_icmpOP       |       |        |       |      | if_acopOP        |
| Treturn    |       |       | ireturn       | lreturn      |  freturn      | dreturn      |      | areturn       |

大部分指令没有支持byte，char和short甚至是boolean，编译器会在编译器或者运行期把这类数据扩展为 int类型数据。

<h4 id="4_3">加载/存储指令</h4>

加载/存储指令用于将数据在栈帧中的局部变量表和操作数栈之间来回传输。

* 将一个局部变量加载到操作栈： `Tload` ， `Tload_n` 后者表示是一组指令。
* 将一个数值从操作数栈存储到局部变量表： `Tstore` ， `Tstore_n` 后者表示是一组指令。
* 将一个常量加载到操作数栈： `Tipush` ， `ldc` ，`T_const` 等
* 扩充局部变量表的访问索引指令：`wide`

<h4 id="4_4">运算指令</h4>

对操作数栈的数值进行运算之后把结果重新存入操作栈栈顶。

* 加法指令 `Tadd`
* 减法指令 `Tsub`
* 乘法指令 `Tmul`
* 除法指令 `Tdiv`
* 求余指令 `Trem`
* 取反指令 `Tneg`
* 位移指令 `Tshl`, `Tshr`, `Tushr`
* 按位或指令 `Tor`
* 按位与指令 `Tand`
* 按位异或指令 `Txor`
* 局部变量自增指令 `Tinc`
* 比较指令 `Tcmpg` ,`Tcmpl` 

<h4 id="4_5">类型转化指令</h4>

类型转化指令用于将两种不同的数值类型进行相互转换，这种转换操作一般用于实现用户代码中的显式转换操作，或者用于处理字节码指令集中数据类型相关指令无法与数据类型一一对应的问题。

* int类型转其他 `i2T`
* long类型转其他 `l2T`
* float类型转其他 `f2T`
* double类型转其他 `d2T`

<h4 id="4_6">对象创建与访问指令</h4>

尽管类实例和数组都是对象，但Java虚拟机对类实例和数组的创建与操作使用了不同的字节码指令。

* 创建类实例 `new`
* 创建数组 `newarray`, `anewarray`, `multianewarray`
* 访问类变量和实例变量 `getfield`, `putfield`，`getstatic`，`putstatic`
* 把一个数组元素加载到操作数栈 `Taload`
* 将一个操作数栈的值存储到数组元素中 `Tastore`
* 取数组长度的指令 `arraylength`
* 检查类实例类型 `instanceof`, `checkcast`

<h4 id="3_7">操作数栈管理指令</h4>

    * 将操作数栈栈顶一个或者两个元素出栈 `pop`，`pop2`
    * 复制栈顶一个或两个数值并将复制值重新压入栈顶 `dup`，`dup2`, `dup_x1`，`dup2_x1`,`dup_x2`，`dup2_x2`
    * 将栈最顶端两个数值互换 `swap`

<h4 id="3_8">控制转移指令</h4>

让虚拟机可以有条件或者无条件地从特定位置指令执行程序而不是在控制转移指令的下一条指令执行程序。

* 条件分支 `ifeq`, `ifit`, `ifle`, `ifne`, `ifgt`, `ifge`, `ifull`, `ifnonnull`, `if_icmpeq`, `if_icmpne`, `if_icmplt`, `if_icmpgt`, `if_icmple`, `if_icmpge`, `if_acmpeq`, `if_acmpne`
* 复合条件分支 `tableswitch`, `lookupswitch` 
* 无条件分支 `goto`, `goto_w`, `jsr`, `jsr_w`, `ret`

<h4 id="4_9">方法调用和返回指令</h4>

* 调用对象的实例方法 `invokevirtual`，根据对象的实际类型进行分派
* 调用接口方法 `invokeinterface` , 会在运行时搜索一个实现了这个接口的方法的对象，找到适合的方法进行调用
* 调用一些需要特殊处理的实例方法 `invokespecial`，包括实例初始化方法，私有方法和父类方法
* 调用类方法 `invokestatic` 用于调用static方法
* 运行时动态解析处调用点限定符所引用的方法并执行该方法 `invokedynamic` ，区别于前面4条指令，它们都在固化在jvm内部，而该指令的分派逻辑是由用户所设定的引导方法决定的。

<h4 id="4_10">异常处理指令</h4>

`athrow` 指令用于完成显式抛出异常（throw语句）的操作，除了用throw语句之外，JVM还规定在运行时会在其他 JVM指令检测到异常状况的时候自动抛出。比如当除数为0的时候，JVM会在 `idiv`或 `ldiv` 中抛出 *ArithmeticException* 异常。

<h4 id="4_11">同步指令</h4>

JVM的同步有一下场景，都是使用管程（Monitor）来支持

* 方法级的同步，不需要字节码控制，实现于方法调用和返回操作志宏。从方法表中 ACC_SYNCHRONIZED 得到一个方法是否是同步，如果被设置，则执行线程需要先持有管程才能执行，执行完之后释放管程。
* 方法内部一段指令序列的同步，由`synchronized`和指令`monitorenter` 和 `monitorexit`来支持`synchronized`共同完成

我们的 🌰 非常简单，实际上用不到这么多指令的，其他的可备份用于查询。


<h3 id="5">指令集辅助解析 Code</h3>

有了上述指令集帮助及膝，回到 [重新认识字节天书](#3) 中的第 29-36 行内容重新解析。

**第29-32行** 为 `第 1 个方法`, 查看 [方法表](#2_4) 可知 访问标志（0001）为 *public*, 名字（0007）为 *<init>*, 描述（0008）为 *()V*, 一个属性（0001）。由于存在一个属性，继续查看 [属性表](#2_5)。从 30 行开始解析属性 0009解析为 `第 9 项常量` Code, 查看 [属性表-Code属性结构表](#2_5) 及结合指令集中操作符信息， Code 属性最终的内容如下。（看到这里，你应该尝试过一遍哦 😄）

![code_1](https://user-gold-cdn.xitu.io/2020/5/26/1724ff3dff6cb7dd?w=1024&h=408&f=png&s=110380)

**第33-36行** 为 `第 2 个方法`, 查看 [方法表](#2_4) 可知 访问标志（0001）为 *public*, 名字（000B）为 *inc*, 描述（000C）为 *()I*, 一个属性（0001）。也存在一个属性。从 34 行开始解析属性 0009解析为 `第 9 项常量` Code, 内容如下。


![code_2](https://user-gold-cdn.xitu.io/2020/5/26/1724ff4266252022?w=1024&h=408&f=png&s=103703)
 
至此，.class 文件的内容基本确定可知。

![class_after_code](https://user-gold-cdn.xitu.io/2020/5/26/1724fedad96ff285?w=858&h=864&f=png&s=175363)

为了验证我们的思路是否正确，可以通过 `javap` 查看 `TestClass.class` 的结构来进行对比。


![javap_result](https://user-gold-cdn.xitu.io/2020/5/26/1724ff541eea2613?w=923&h=1168&f=png&s=226253)

除了 `javap` 帮我们做了格式化的工作外，也是按照我们分析字节码的逻辑来进行内容的输出, 感兴趣的伙伴可以查看 `javap` 内部实现。

当你读懂 Class 文件之后，你就可以进一步做很多工作了，比如借助 ASM 框架入侵 gradle 构建流程注入静态代码等，更多场景等你挖掘。

如果文章对您有帮助，欢迎点赞评论支持，若其中有错误，欢迎评论指出哦。

文章首发于 [掘金-如何读懂晦涩的Class文件｜进阶必备](https://juejin.im/post/5ecc89d7e51d4578761ff8cb) 欢迎关注我 👏 
