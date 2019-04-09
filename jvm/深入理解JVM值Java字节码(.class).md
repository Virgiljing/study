[TOC]

# 深入理解JVM值Java字节码(.class)

## 什么事Class文件

Java字节码类文件（.class）是Java编译器编译Java源文件（.java）产生的“目标文件”。它是一种8位字节的二进制流文件， 各个数据项按顺序紧密的从前向后排列， 相邻的项之间没有间隙， 这样可以使得class文件非常紧凑， 体积轻巧， 可以被JVM快速的加载至内存， 并且占据较少的内存空间（方便于网络的传输）。

Java源文件在被Java编译器编译之后， 每个类（或者接口）都单独占据一个class文件， 并且类中的所有信息都会在class文件中有相应的描述， 由于class文件很灵活， 它甚至比Java源文件有着更强的描述能力。

class文件中的信息是一项一项排列的， 每项数据都有它的固定长度， 有的占一个字节， 有的占两个字节， 还有的占四个字节或8个字节， 数据项的不同长度分别用u1, u2, u4, u8表示， 分别表示一种数据项在class文件中占据一个字节， 两个字节， 4个字节和8个字节。 可以把u1, u2, u3, u4看做class文件数据项的“类型” 。

## Class 文件的结构

![](D:\用户目录\我的文档\jvm\jvm\image\20141008125149484.png)

一个典型的class文件分为：MagicNumber，Version，Constant_pool，Access_flag，This_class，Super_class，Interfaces，Fields，Methods 和Attributes这十个部分，用一个数据结构可以表示如下：

| descriptor     | type                                 |
| :------------- | :----------------------------------- |
| u4             | magic                                |
| u2             | minor_version                        |
| u2             | major_version                        |
| u2             | constant_pool_count                  |
| cp_info        | constant_pool[constant_pool_count-1] |
| u2             | access_flags                         |
| u2             | this_class                           |
| u2             | super_class                          |
| u2             | interfaces_count                     |
| u2             | interfaces[interfaces_count]         |
| u2             | fields_count                         |
| field_info     | fields[fields_count]                 |
| u2             | methods_count                        |
| method_info    | methods[methods_count]               |
| u2             | attributes_count                     |
| attribute_info | attributes[attributes_count]         |

下面对class文件中的每一项进行详细的解释：

1、**magic** 
在class文件开头的四个字节， 存放着class文件的魔数， 这个魔数是class文件的标志，他是一个固定的值： 0XCAFEBABE 。 也就是说他是判断一个文件是不是class格式的文件的标准， 如果开头四个字节不是0XCAFEBABE， 那么就说明它不是class文件， 不能被JVM识别。

2、**minor_version 和 major_version** 
紧接着魔数的四个字节是**class文件的此版本号和主版本号**。 
随着Java的发展， class文件的格式也会做相应的变动。 版本号标志着class文件在什么时候， 加入或改变了哪些特性。 举例来说， 不同版本的javac编译器编译的class文件， 版本号可能不同， 而不同版本的JVM能识别的class文件的版本号也可能不同， 一般情况下， 高版本的JVM能识别低版本的javac编译器编译的class文件， 而低版本的JVM不能识别高版本的javac编译器编译的class文件。 如果使用低版本的JVM执行高版本的class文件， JVM会抛出java.lang.UnsupportedClassVersionError 。具体的版本号变迁这里不再讨论， 需要的读者自行查阅资料。

3、**constant_pool** 
在class文件中， 位于版本号后面的就是常量池相关的数据项。 常量池是class文件中的一项非常重要的数据。 常量池中存放了文字字符串， 常量值， 当前类的类名， 字段名， 方法名， 各个字段和方法的描述符， 对当前类的字段和方法的引用信息， 当前类中对其他类的引用信息等等。 常量池中几乎包含类中的所有信息的描述， class文件中的很多其他部分都是对常量池中的数据项的引用，比如后面要讲到的this_class, super_class, field_info, attribute_info等， 另外字节码指令中也存在对常量池的引用， 这个对常量池的引用当做字节码指令的一个操作数。此外，常量池中各个项也会相互引用。

常量池是一个类的结构索引，其它地方对“对象”的引用可以通过索引位置来代替，我们知道在程序中一个变量可以不断地被调用，要快速获取这个变量常用的方法就是通过索引变量。这种索引我们可以直观理解为“内存地址的虚拟”。我们把它叫静态池的意思就是说这里维护着经过编译“梳理”之后的相对固定的数据索引，它是站在整个JVM（进程）层面的共享池。

class文件中的项constant_pool_count的值为1, 说明每个类都只有一个常量池。 常量池中的数据也是一项一项的， 没有间隙的依次排放。常量池中各个数据项通过索引来访问， 有点类似与数组， 只不过常量池中的第一项的索引为1, 而不为0, 如果class文件中的其他地方引用了索引为0的常量池项， 就说明它不引用任何常量池项。class文件中的每一种数据项都有自己的类型， 相同的道理，常量池中的每一种数据项也有自己的类型。 常量池中的数据项的类型如下表：



每个数据项叫做一个XXX_info项，比如，一个常量池中一个CONSTANT_Utf8类型的项，就是一个CONSTANT_Utf8_info 。除此之外， 每个info项中都有一个标志值（tag），这个标志值表明了这个常量池中的info项的类型是什么， 从上面的表格中可以看出，一个CONSTANT_Utf8_info中的tag值为1，而一个CONSTANT_Fieldref_info中的tag值为9 。

Java程序是动态链接的， 在动态链接的实现中， 常量池扮演者举足轻重的角色。 除了存放一些字面量之外， 常量池中还存放着以下几种符号引用： 
（1） 类和接口的全限定名 
（2） 字段的名称和描述符 
（3） 方法的名称和描述符 
我们有必要先了解一下class文件中的特殊字符串， 因为在常量池中， 特殊字符串大量的出现，这些特殊字符串就是上面说的全限定名和描述符。对于常量池中的特殊字符串的了解，可以参考此文档：[Java class文件格式之特殊字符串_动力节点Java学院整理](http://www.jb51.net/article/116313.htm)

4、**access_flag** 保存了当前类的访问权限

5、**this_cass** 保存了当前类的全局限定名在常量池里的索引

6、**super class** 保存了当前类的父类的全局限定名在常量池里的索引

7、**interfaces** 保存了当前类实现的接口列表，包含两部分内容：interfaces_count 和interfaces[interfaces_count] 
interfaces_count 指的是当前类实现的接口数目 
interfaces[] 是包含interfaces_count个接口的全局限定名的索引的数组

8、**fields** 保存了当前类的成员列表，包含两部分的内容：fields_count 和 fields[fields_count] 
fields_count是类变量和实例变量的字段的数量总和。 
fileds[]是包含字段详细信息的列表。

9、**methods** 保存了当前类的方法列表，包含两部分的内容：methods_count和methods[methods_count] 
methods_count是该类或者接口显示定义的方法的数量。 
method[]是包含方法信息的一个详细列表。

10、**attributes** 包含了当前类的attributes列表，包含两部分内容：attributes_count 和 attributes[attributes_count] 
class文件的最后一部分是属性，它描述了该类或者接口所定义的一些属性信息。attributes_count指的是attributes列表中包含的attribute_info的数量。 
属性可以出现在class文件的很多地方，而不只是出现在attributes列表里。如果是attributes表里的属性，那么它就是对整个class文件所对应的类或者接口的描述；如果出现在fileds的某一项里，那么它就是对该字段额外信息的描述；如果出现在methods的某一项里，那么它就是对该方法额外信息的描述。

## **通过示例代码来手动分析class文件**

上面大致讲解了一下class文件的结构，这里，我们拿一个class文件做一个简单的分析，来验证上面的文件结构是否确实是如此。

我们在这里新建一个java文件，TestJvm.java，具体内容如下：

```java
package club.virgilin.jvm;

import java.io.Serializable;

/**
 * TestJvm
 *
 * @author virgilin
 * @date 2019/3/4
 */
public class TestJvm  implements Serializable,Runnable {
    static String name = "virgilin";
    static int a ;
    public static byte b;
    public long c;
    @Override
    public void run() {
        System.out.println("run");
    }

    public static void main(String[] args) {
        TestJvm testJvm = new TestJvm();
        new Thread(testJvm).start();
    }

    public static void setA(int a) {
        TestJvm.a = a;
    }
}
```

**class文件二进制流如下：通过Sublime可以将.class文件直接打开**

```java 
cafe babe 0000 0034 常量开始 0041 0a00 0d00 2a09
002b 002c 0800 1f0a 002d 002e 0700 2f0a
0005 002a 0700 300a 0007 0031 0a00 0700
3209 0005 0033 0800 3409 0005 0035 0700
3607 0037 0700 3801 0004 6e61 6d65 0100
124c 6a61 7661 2f6c 616e 672f 5374 7269
6e67 3b01 0001 6101 0001 4901 0001 6201
0001 4201 0001 6301 0001 4a01 0006 3c69
6e69 743e 0100 0328 2956 0100 0443 6f64
6501 000f 4c69 6e65 4e75 6d62 6572 5461
626c 6501 0012 4c6f 6361 6c56 6172 6961
626c 6554 6162 6c65 0100 0474 6869 7301
001b 4c63 6c75 622f 7669 7267 696c 696e
2f6a 766d 2f54 6573 744a 766d 3b01 0003
7275 6e01 0004 6d61 696e 0100 1628 5b4c
6a61 7661 2f6c 616e 672f 5374 7269 6e67
3b29 5601 0004 6172 6773 0100 135b 4c6a
6176 612f 6c61 6e67 2f53 7472 696e 673b
0100 0774 6573 744a 766d 0100 0473 6574
4101 0004 2849 2956 0100 083c 636c 696e
6974 3e01 000a 536f 7572 6365 4669 6c65
0100 0c54 6573 744a 766d 2e6a 6176 610c
0018 0019 0700 390c 003a 003b 0700 3c0c
003d 003e 0100 1963 6c75 622f 7669 7267
696c 696e 2f6a 766d 2f54 6573 744a 766d
0100 106a 6176 612f 6c61 6e67 2f54 6872
6561 640c 0018 003f 0c00 4000 190c 0012
0013 0100 0876 6972 6769 6c69 6e0c 0010
0011 0100 106a 6176 612f 6c61 6e67 2f4f
626a 6563 7401 0014 6a61 7661 2f69 6f2f
5365 7269 616c 697a 6162 6c65 0100 126a
6176 612f 6c61 6e67 2f52 756e 6e61 626c
6501 0010 6a61 7661 2f6c 616e 672f 5379
7374 656d 0100 036f 7574 0100 154c 6a61
7661 2f69 6f2f 5072 696e 7453 7472 6561
6d3b 0100 136a 6176 612f 696f 2f50 7269
6e74 5374 7265 616d 0100 0770 7269 6e74
6c6e 0100 1528 4c6a 6176 612f 6c61 6e67
2f53 7472 696e 673b 2956 0100 1728 4c6a
6176 612f 6c61 6e67 2f52 756e 6e61 626c
653b 2956 0100 0573 7461 7274 常量结束   0021(access_flag) 0005(this_class)
000d(super_class) 0002(interfaces_count) 000e 000f    0004(fields_count) 0008 0010 0011
0000 0008 0012 0013 0000    0009 0014 0015
0000 0001 0016 0017 0000  0005(methods_count)  (method1)0001 0018
0019 0001 001a 0000 002f 0001 0001 0000
0005 2ab7 0001 b100 0000 0200 1b00 0000
0600 0100 0000 0b00 1c00 0000 0c00 0100
0000 0500 1d00 1e00 0000     (method2)0100 1f00 1900
0100 1a00 0000 3700 0200 0100 0000 09b2
0002 1203 b600 04b1 0000 0002 001b 0000
000a 0002 0000 0012 0008 0013 001c 0000
000c 0001 0000     (method3)0009 001d 001e 0000 0009
0020 0021 0001 001a 0000 0050 0003 0002
0000 0014 bb00 0559 b700 064c bb00 0759
2bb7 0008 b600 09b1 0000 0002 001b 0000
000e 0003 0000 0016 0008 0017 0013 0018
001c 0000 0016 0002 0000 0014 0022 0023
0000 0008 000c 0024 001e 0001     (method4)0009 0025
0026 0001 001a 0000 0033 0001 0001 0000
0005 1ab3 000a b100 0000 0200 1b00 0000
0a00 0200 0000 1b00 0400 1c00 1c00 0000
0c00 0100 0000 0500 1200 1300 00   (method5)00 0800
2700 1900 0100 1a00 0000 1e00 0100 0000
0000 0612 0bb3 000c b100 0000 0100 1b00
0000 0600 0100 0000 0c   (attribute)00 0100 2800 0000
0200 29
```

### magic: 

CA FE BA BE ，代表该文件是一个字节码文件，我们平时区分文件类型都是通过后缀名来区分的，不过后缀名是可以随便修改的，所以仅靠后缀名不能真正区分一个文件的类型。区分文件类型的另个办法就是magic数字，JVM 就是通过 CA FE BA BE 来判断该文件是不是class文件

### version字段： 

00 00 00 34，前两个字节00是minor_version，后两个字节0034是major_version字段，对应的十进制值为52，也就是说当前class文件的主版本号为52，次版本号为0。下表是jdk 1.6 以后对应支持的 Class 文件版本号：

| 编译器版本       | -target参数               | 十六进制版本      | 十进制版本 |
| ----------- | ----------------------- | ----------- | ----- |
| JDK1.6.0_01 | 不带（默认-target1.6）        | 00 00 00 32 | 50.0  |
| JDK1.6.0_01 | -target 1.5             | 00 00 00 31 | 49.0  |
| JDK1.6.0_01 | -target 1.4 -source 1.4 | 00 00 00 30 | 48.0  |
| JDK 1.7.0   | 不带（默认 -target 1.7）      | 00 00 00 33 | 51.0  |
| JDK 1.7.0   | -target 1.6             | 00 00 00 32 | 50    |
| JDK 1.7.0   | -target 1.4 -source 1.4 | 00 00 00 30 | 48.0  |
| JDK 1.8.0   | 不带（默认 -target 1.8）      | 00 00 00 34 | 52.0  |

### 常量池，constant_pool:

3.1.  `constant_pool_count` 

紧接着version字段下来的两个字节是：00 41代表常量池里包含的常量数目，因为字节码的常量池是从1开始计数的，这个常量池包含64个（0x0041-1）常量。

3.2. `constant_pool` 

表类型数据集合，即常量池中每一项常量都是一个表，共有11种结构各不相同的表结构数据。这11种表都有一个共同的特点，即均由一个u1类型的标志位开始，可以通过这个标志位来判断这个常量属于哪种常量类型，常量类型及其数据结构如下表所示：接下来就是分析这64个常量:

| 类型                               | 简介           | 项目     | 类型   | 描述                                      |
| -------------------------------- | ------------ | ------ | ---- | --------------------------------------- |
| CONSTANT_Utf8_info               | utf-8缩略编码字符串 | tag    | u1   | 值为1                                     |
|                                  |              | length | u2   | utf-8缩略编码字符串占用字节数                       |
|                                  |              | bytes  | u1   | 长度为length的utf-8缩略编码字符串                  |
| CONSTANT_Integer_info            | 整形字面量        | tag    | u1   | 值为3                                     |
|                                  |              | bytes  | u4   | 按照高位在前储存的int值                           |
| CONSTANT_Float_info              | 浮点型字面量       | tag    | u1   | 值为4                                     |
|                                  |              | bytes  | u4   | 按照高位在前储存的float值                         |
| CONSTANT_Long_info               | 长整型字面量       | tag    | u1   | 值为5                                     |
|                                  |              | bytes  | u8   | 按照高位在前储存的long值                          |
| CONSTANT_Double_info             | 双精度浮点型字面量    | tag    | u1   | 值为6                                     |
|                                  |              | bytes  | u8   | 按照高位在前储存的double值                        |
| CONSTANT_Class_info              | 类或接口的符号引用    | tag    | u1   | 值为7                                     |
|                                  |              | index  | u2   | 指向全限定名（参见备注一）常量项的索引                     |
| CONSTANT_String_info             | 字符串类型字面量     | tag    | u1   | 值为8                                     |
|                                  |              | index  | u2   | 指向字符串字面量的索引                             |
| CONSTANT_Fieldref_info           | 字段的符号引用      | tag    | u1   | 值为9                                     |
|                                  |              | index  | u2   | 指向声明字段的类或接口描述符CONSTANT_Class_info的索引项   |
|                                  |              | index  | u2   | 指向字段描述符CONSTANT_NameAndType_info的索引项    |
| CONSTANT_Methodref_info          | 类中方法的符号引用    | tag    | u1   | 值为10                                    |
|                                  |              | index  | u2   | 指向声明方法的类描述符CONSTANT_Class_info的索引项      |
|                                  |              | index  | u2   | 指向名称及类型描述符CONSTANT_NameAndType_info的索引项 |
| CONSTANT_InterfaceMethodref_info | 接口中方法的符号引用   | tag    | u1   | 值为11                                    |
|                                  |              | index  | u2   | 指向声明方法的接口描述符CONSTANT_Class_info的索引项     |
|                                  |              | index  | u2   | 指向名称及类型描述符CONSTANT_NameAndType_info的索引项 |
| CONSTANT_NameAndType_info        | 字段或方法的部分符号引用 | tag    | u1   | 值为12                                    |
|                                  |              | index  | u2   | 指向该字段或方法名称常量项的索引                        |
|                                  |              | index  | u2   | 指向该字段或方法描述符常量项的索引                       |



0041 共64个常量

`const #1` 0a 000d 002a                                                    #13.#42

`const #2` 09 002b 002c                                                    #43.#44

`const #3` 08 001f                                                               #31

`const #4` 0a 002d 002e                                                    #45.#46

`const #5` 07 002f                                                               #47

`const #6` 0a 0005 002a                                                     #5.#42

`const #7` 07 0030                                                               #48

`const #8` 0a 0007 0031                                                     #7.#49

`const #9` 0a 0007 0032                                                     #7.#50

`const #10` 09 0005 0033                                                   #7.#51

`const #11` 08 0034                                                             #52

`const #12` 09 0005 0035                                                   #5.#53

`const #13` 07 0036                                                            #54

`const #14` 07 0037                                                            #55

`const #15` 07 0038                                                            #56

`const #16` 01 0004 6e61 6d65                                         name

`const #17` 01 0012 4c 6a61 7661 2f6c 616e 672f 5374 72696e67 3b

`const #18` 01 0001 61

`const 19` 01 0001 49

`const #20` 01 0001 62

`const #21` 01 0001 42

`const #22` 01 0001 63

`const #23` 01 0001 4a

`const #24` 01 0006 3c696e69 743e 

`const #25` 01 0003 28 2956 

`const #26` 01 0004 43 6f6465

`const #27` 01 000f 4c69 6e65 4e75 6d62 6572 5461626c 65

`const #28` 01 0012 4c6f 6361 6c56 6172 6961626c 6554 6162 6c65 

`const #29` 01 0004 74 6869 73

`const #30` 01 001b 4c63 6c75 622f 7669 7267 696c 696e 2f6a 766d 2f54 6573 744a 766d 3b

`const #31` 01 0003 7275 6e

`const #32` 01 0004 6d61 696e 

`const #33` 01 0016 28 5b4c 6a61 7661 2f6c 616e 672f 5374 7269 6e67 3b29 56

`const #34` 01 0004 6172 6773 

`const #35` 01 0013 5b 4c6a 6176 612f 6c61 6e67 2f53 7472 696e 673b

`const #36` 01 0007 74 6573 744a 766d  

`const #37` 01 0004 7365 7441

`const #38` 01 0004 2849 2956 

`const #39` 01 0008 3c 636c 696e6974 3e

`const #40` 01 000a 536f 7572 6365 4669 6c65

`const #41` 01 000c 54 6573 744a 766d 2e6a 6176 61

`const #42` 0c 0018 0019 

`const #43` 07 00 39

`const #44` 0c 003a 003b 

`const #45` 07 00 3c

`const #46` 0c 003d 003e 

`const #47` 01 0019 63 6c75 622f 7669 7267 696c 696e 2f6a 766d 2f54 6573 744a 766d

`const #48` 01 0010 6a 6176 612f 6c61 6e67 2f54 6872 6561 64 

`const #49` 0c 0018 003f 

`const #50` 0c 00 40 00 19

`const #51` 0c 0012 0013 

`const #52` 01 00 08 76 6972 6769 6c69 6e

`const #53` 0c 0010 0011 

`const #54` 01 0010 6a 6176 612f 6c61 6e67 2f4f 626a 6563 74 

`const #55` 01 0014 6a61 7661 2f69 6f2f 5365 7269 616c 697a 6162 6c65 

`const #56` 01 0012 6a 6176 612f 6c61 6e67 2f52 756e 6e61 626c65 

`const #57` 01 0010 6a61 7661 2f6c 616e 672f 5379 7374 656d 

`const #58` 01 0003 6f 7574 

`const #59` 01 0015 4c 6a61 7661 2f69 6f2f 5072 696e 7453 7472 6561 6d3b 

`const #60` 01 0013 6a 6176 612f 696f 2f50 7269 6e74 5374 7265 616d 

`const #61` 01 0007 70 7269 6e74 6c6e 

`const #62` 01 0015 28 4c6a 6176 612f 6c61 6e67 2f53 7472 696e 673b 2956 

`const #63` 01 0017 28 4c6a 6176 612f 6c61 6e67 2f52 756e 6e61 626c 653b 2956 

`const #64` 01 0005 73 7461 7274 

通过javap -v TestJvm.class指令得到

```java
Classfile /C:/Users/Administrator/Desktop/myword/Concurrency-Progamming-Tuitor/target/classes/club/virgilin/jvm/TestJvm.class
  Last modified 2019-3-4; size 1043 bytes
  MD5 checksum 9d55cf39f28737580a11e5d26376b260
  Compiled from "TestJvm.java"
public class club.virgilin.jvm.TestJvm implements java.io.Serializable,java.lang.Runnable
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #13.#42        // java/lang/Object."<init>":()V
   #2 = Fieldref           #43.#44        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #31            // run
   #4 = Methodref          #45.#46        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #47            // club/virgilin/jvm/TestJvm
   #6 = Methodref          #5.#42         // club/virgilin/jvm/TestJvm."<init>":()V
   #7 = Class              #48            // java/lang/Thread
   #8 = Methodref          #7.#49         // java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
   #9 = Methodref          #7.#50         // java/lang/Thread.start:()V
  #10 = Fieldref           #5.#51         // club/virgilin/jvm/TestJvm.a:I
  #11 = String             #52            // virgilin
  #12 = Fieldref           #5.#53         // club/virgilin/jvm/TestJvm.name:Ljava/lang/String;
  #13 = Class              #54            // java/lang/Object
  #14 = Class              #55            // java/io/Serializable
  #15 = Class              #56            // java/lang/Runnable
  #16 = Utf8               name
  #17 = Utf8               Ljava/lang/String;
  #18 = Utf8               a
  #19 = Utf8               I
  #20 = Utf8               b
  #21 = Utf8               B
  #22 = Utf8               c
  #23 = Utf8               J
  #24 = Utf8               <init>
  #25 = Utf8               ()V
  #26 = Utf8               Code
  #27 = Utf8               LineNumberTable
  #28 = Utf8               LocalVariableTable
  #29 = Utf8               this
  #30 = Utf8               Lclub/virgilin/jvm/TestJvm;
  #31 = Utf8               run
  #32 = Utf8               main
  #33 = Utf8               ([Ljava/lang/String;)V
  #34 = Utf8               args
  #35 = Utf8               [Ljava/lang/String;
  #36 = Utf8               testJvm
  #37 = Utf8               setA
  #38 = Utf8               (I)V
  #39 = Utf8               <clinit>
  #40 = Utf8               SourceFile
  #41 = Utf8               TestJvm.java
  #42 = NameAndType        #24:#25        // "<init>":()V
  #43 = Class              #57            // java/lang/System
  #44 = NameAndType        #58:#59        // out:Ljava/io/PrintStream;
  #45 = Class              #60            // java/io/PrintStream
  #46 = NameAndType        #61:#62        // println:(Ljava/lang/String;)V
  #47 = Utf8               club/virgilin/jvm/TestJvm
  #48 = Utf8               java/lang/Thread
  #49 = NameAndType        #24:#63        // "<init>":(Ljava/lang/Runnable;)V
  #50 = NameAndType        #64:#25        // start:()V
  #51 = NameAndType        #18:#19        // a:I
  #52 = Utf8               virgilin
  #53 = NameAndType        #16:#17        // name:Ljava/lang/String;
  #54 = Utf8               java/lang/Object
  #55 = Utf8               java/io/Serializable
  #56 = Utf8               java/lang/Runnable
  #57 = Utf8               java/lang/System
  #58 = Utf8               out
  #59 = Utf8               Ljava/io/PrintStream;
  #60 = Utf8               java/io/PrintStream
  #61 = Utf8               println
  #62 = Utf8               (Ljava/lang/String;)V
  #63 = Utf8               (Ljava/lang/Runnable;)V
  #64 = Utf8               start
{
  static java.lang.String name;
    descriptor: Ljava/lang/String;
    flags: ACC_STATIC

  static int a;
    descriptor: I
    flags: ACC_STATIC

  public static byte b;
    descriptor: B
    flags: ACC_PUBLIC, ACC_STATIC

  public long c;
    descriptor: J
    flags: ACC_PUBLIC

  public club.virgilin.jvm.TestJvm();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 11: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lclub/virgilin/jvm/TestJvm;

  public void run();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String run
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 18: 0
        line 19: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lclub/virgilin/jvm/TestJvm;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #5                  // class club/virgilin/jvm/TestJvm
         3: dup
         4: invokespecial #6                  // Method "<init>":()V
         7: astore_1
         8: new           #7                  // class java/lang/Thread
        11: dup
        12: aload_1
        13: invokespecial #8                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
        16: invokevirtual #9                  // Method java/lang/Thread.start:()V
        19: return
      LineNumberTable:
        line 22: 0
        line 23: 8
        line 24: 19
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      20     0  args   [Ljava/lang/String;
            8      12     1 testJvm   Lclub/virgilin/jvm/TestJvm;

  public static void setA(int);
    descriptor: (I)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=1, args_size=1
         0: iload_0
         1: putstatic     #10                 // Field a:I
         4: return
      LineNumberTable:
        line 27: 0
        line 28: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0     a   I

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: ldc           #11                 // String virgilin
         2: putstatic     #12                 // Field name:Ljava/lang/String;
         5: return
      LineNumberTable:
        line 12: 0
}
SourceFile: "TestJvm.java"
```

### access_flag(u2) 

00 21这两个字节的数据表示这个变量的访问标志位，JVM对访问标示符的规范如下： 

| Flag Name      | Value  | Remarks                                  |
| -------------- | ------ | ---------------------------------------- |
| ACC_PUBLIC     | 0x0001 | public                                   |
| ACC_FINAL      | 0x0010 | final                                    |
| ACC_SUPER      | 0x0020 | 用于兼容早期编译器，新编译器都设置该标记，以在使用invokespecial指令时对子类方法做特定处理。 |
| ACC_INTERFACE  | 0x0200 | 接口，同时需要设置：ACC_ABSTRACT.不可同时设置：ACC_FINAL、ACC_SUPER、ACC_ENUM |
| ACC_ABSTRACT   | 0x0400 | 抽象类，无法实例化。不可与ACC_FINAL同时设置。              |
| ACC_SYNTHETIC  | 0x1000 | synthetic,有编译器产生，不存在与源代码中。               |
| ACC_ANNOTATION | 0x2000 | 注解类型（annotation），需同时设置：ACC_INTERFACE、ACC_ABSTRACT |
| ACC_ENUM       | 0x4000 | 枚举类型                                     |
| ACC_STATIC     | 0x0008 | static，静态。                               |

这个表里面无法直接查询到0021这个值，原因是0021=0020+0001，也就是表示当前class的access_flag是ACC_PUBLIC|ACC_SUPER。ACC_PUBLIC和代码里的public 关键字相对应。ACC_SUPER表示当用invokespecial指令来调用父类的方法时需要特殊处理

### this_class(u2) 00 05

this_class指向constant pool的索引值，该值必须是CONSTANT_Class_info类型，这里是5，即指向常量池中的第5项，即是“ club/virgilin/jvm/TestJvm”。

### super_class(u2)  00 0d

super_class存的是父类的名称在常量池里的索引，这里指向第13个常量，即是“java/lang/Object”。 

### interfaces(u2)

interfaces包含interfaces_count和interfaces[]两个字段。0002 ，，所以这里的interfaces_count为2（0002），所以后面的。 000e这里指向第14个常量, 即是“  java/io/Serializable”。000f 这里指向第15个常量，即是“ java/lang/Runnable”。

### fields(u2)

```java 
00 04 fields count        //表示成员变量的个数，此处为1个
0008 0010 0011 0000   static java.lang.String name
0008 0012 0013 0000   static int a
0009 0014 0015 0000   public static byte b
0001 0016 0017 0000   public long c//成员变量的结构
```

每个成员变量对应一个field_info结构：

```java
field_info {
    u2 access_flags; 0008
    u2 name_index; 0010
    u2 descriptor_index; 0011
    u2 attributes_count; 0000
    attribute_info attributes[attributes_count];
}
```

access_flags为0002，即是ACC_STATIC
name_index指向常量池的第16个常量，为“name” 
descriptor_index指向常量池的第17个常量为“ Ljava/lang/String” 
三个字段结合起来，说明这个变量是”static java.lang.String name”。 
接下来的是attribute字段，用来描述该变量的属性，因为这个变量没有附加属性，所以attributes_count为0，attribute_info为空。 

### methods

最前面的2个字节是method_count 
method_count：00 05，为什么会有两个方法呢？我们明明只写了四个方法，这是因为JVM 会自动生成一个<init>方法，这个是类的默认构造方法。

接下来的内容是两个**method_info**结构：

| 类型             | 名称               | 数量                           | 含义     |
| -------------- | ---------------- | ---------------------------- | ------ |
| u2             | access_flags     | 1                            | 访问控制符  |
| u2             | name_index       | 1                            | 方法名索引  |
| u2             | descriptor_index | 1                            | 返回值描述符 |
| u2             | attributes_count | 1                            | 属性长度   |
| attribute_info | attribute_info   | attributes[attributes_count] | 属性表    |



```java
method_info {
	u2 access_flags;
	u2 name_index;
	u2 descriptor_index;
	u2 attributes_count;
	attribute_info attributes[attributes_count];
}
```

即只需说明属性的名称以及占用位数的长度即可，属性表具体的结构可以去自定义

```java
第一个方法开始
0001 public
0018 <init>
0019 ()V
0001 表示有一个属性表
```

**attribute**属性表结构
属性表的结构比较灵活，各种不同的属性只要满足以下结构即可：

| 类型   | 名称                   | 数量               | 含义    |
| ---- | -------------------- | ---------------- | ----- |
| u2   | attribute_name_index | 1                | 属性名索引 |
| u4   | attribute_length     | 1                | 属性长度  |
| u1   | info                 | attribute_length | 属性表   |

```java
Code 开始
001a Code  attribute_name_index
0000 002f  47*info属性字段常 attribute_length
```



Code属性**：Java程序方法体中的代码经过Javac编译器处理后，最终变为字节码指令存储在Code属性中。当然不是所有的方法都必须有这个属性（接口中的方法或抽象方法就不存在Code属性），Code属性表结构如下：

| 类型             | 名称                     | 数量                     |
| -------------- | ---------------------- | ---------------------- |
| u2             | attribute_name_index   | 1                      |
| u4             | attribute_length       | 1                      |
| u2             | max_stack              | 1                      |
| u2             | max_locals             | 1                      |
| u4             | code_length            | 1                      |
| u1             | code                   | code_length            |
| u2             | exception_table_length | 1                      |
| exception_info | exception_table        | exception_table_length |
| u2             | attributes_count       | 1                      |
| attribute_info | attributes             | attributes_count       |

```java
0001   max_stack  最大栈深
0001   max_locals 最大本地变量 
0000 0005   code_length code长度
2a  aload_0  将第一个引用类型本地变量推送至栈顶
b7  invokespecial   调用超类构建方法, 实例初始化方法, 私有方法
0001  java/lang/Object."<init>":()V
b1  return 从当前方法返回void
0000     exception_table_length  无异常
0002     attributes_count  属性表
attribute开始
001b     attribute_name_index    #27   LineNumberTable
0000 0006   attribute_length
0001
0000 000b
LineNumberTable:
       line 0: 11
attribute结束
attribute开始
001c     attribute_name_index  #28   LocalVariableTable
0000 000c attribute_length
0001  line_number_table_length
00 00 start_pc
00 05 length
00 1d name_index  this 
00 1e descriptor_index  Lclub/virgilin/jvm/TestJvm
00 00 index
attribute结束
Code结束    
Method（<init>)–1个属性Code表-2个属性表（LineNumberTable ，LocalVariableTable）
```

**LineNumberTable** 属性结构

| 类型               | 名称                       | 数量               |
| ---------------- | ------------------------ | ---------------- |
| u2               | attribute_name_index     | 1                |
| u4               | attribute_length         | 1                |
| u2               | line_number_table_length | 1                |
| line_number_info | line_number_table        | line_number_info |

**line_number_info**属性结构

| 类型   | 名称          | 数量   | 描述       |
| ---- | ----------- | ---- | -------- |
| u2   | start_pc    | 1    | 字节码行号    |
| u2   | line_number | 1    | Java源码行号 |

**LocalVariableTable** 属性结构

| 类型                  | 名称                          | 数量                          |
| ------------------- | --------------------------- | --------------------------- |
| u2                  | attribute_name_index        | 1                           |
| u4                  | attribute_length            | 1                           |
| u2                  | local_variable_table_length | 1                           |
| local_variable_info | local_variable_table        | local_variable_table_length |

**local_variable_info**属性结构

| 类型   | 名称               | 数量   |
| ---- | ---------------- | ---- |
| u2   | start_pc         | 1    |
| u2   | length           | 1    |
| u2   | name_index       | 1    |
| u2   | descriptor_index | 1    |
| u2   | index            | 1    |

```java
第二个方法
0001      public
001f       run
0019      ()V
0001      表示有一个属性表
001a   Code  attribute_name_index
0000 0037    55*info属性字段常 attribute_length
0002    max_stack  最大栈深
0001    max_locals 最大本地变量 
0000 0009       code_length code长度
b2         getstatic  获取指定类的静态域, 并将其压入栈顶
0002   #2 = Fieldref #43.#44// java/lang/System.out:Ljava/io/PrintStream;
12         ldc  将int,float或String型常量值从常量池中推送至栈顶
03     #3 = String   #31    // run
b6         invokevirtual  调用实例方法
00 04  #4 = Methodref   #45.#46  // java/io/PrintStream.println:(Ljava/lang/String;)V
b1         return  从当前方法返回void
0000 exception_table_length  无异常
0002 attributes_count  属性表


001b attribute_name_index  #27   LineNumberTable
0000  000a attribute_length=10
0002 两个line_number_info表
0000 0012    0 ---------- 18
0008 0013    8 ---------- 19
LineNumberTable:
    line 18: 0
    line 19: 8
      
001c   attribute_name_index  #28   LocalVariableTable
0000 000c attribute_length=12
0001 local_variable_table_length=1
0000  start_pc
0009  length
001d  name_index   #29 = Utf8               this
001e  descriptor_index #30 = Utf8               Lclub/virgilin/jvm/TestJvm;
0000  index
LocalVariableTable:
       Start  Length  Slot  Name   Signature
           0       9     0  this   Lclub/virgilin/jvm/TestJvm;
```

```java
第三个方法
0009      public static
0020      #32=Utf8     main
0021      #33=Utf8     ([Ljava/lang/String;)V
0001      表示有一个属性表
001a     Code  attribute_name_indexx
0000 0050    80*info属性字段常 attribute_length
0003  max_stack  最大栈深
0002  max_locals 最大本地变量
0000 0014  code_length=20 code长度
bb   new  创建一个对象, 并将其引用引用值压入栈顶
0005  #5=Class   #47      // club/virgilin/jvm/TestJvm
59    dup  复制栈顶数值并将复制值压入栈顶
b7    invokespecial  调用超类构建方法, 实例初始化方法, 私有方法
0006  #6=Methodref    #5.#42         // club/virgilin/jvm/TestJvm."<init>":()V
4c    astore_1  将栈顶引用型数值存入第二个本地变量
bb    new  创建一个对象, 并将其引用引用值压入栈顶
0007  #7=Class   #48    // java/lang/Thread
59    dup  复制栈顶数值并将复制值压入栈顶
2b    aload_1 将第二个引用类型本地变量推送至栈顶
b7    invokespecial 调用超类构建方法, 实例初始化方法, 私有方法
0008  #8=Methodref #7.#49 // java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
b6    invokevirtual 调用实例方法
0009  #9 = Methodref          #7.#50         // java/lang/Thread.start:()V
b1    return 从当前方法返回void
0000 exception_table_length  无异常
0002 attributes_count  属性表 2 个
001b attribute_name_index  #27   LineNumberTable
0000000e attribute_length=14
0003   3个line_number_info表
0000 0016 0 ------- 22
0008 0017 8 ------- 23
0013 0018 19------- 24
LineNumberTable:
    line 22: 0
    line 23: 8
    line 24: 19                        
001c ttribute_name_index  #28   LocalVariableTabl
0000 0016  attribute_length=22
0002 local_variable_table_length=1
0000 start_pc=0
0014 length=20
0022 name_index   #34 = Utf8               args
0023 descriptor_index    #35 = Utf8               [Ljava/lang/String;
0000 index
0008 start_pc=8
000c length=12
0024 name_index   #36 = Utf8               testJvm
001e descriptor_index #30  Lclub/virgilin/jvm/TestJvm;
0001 index                  
LocalVariableTable:
       Start  Length  Slot  Name   Signature
           0       20     0  args   [Ljava/lang/String;
           0       12     1  testJvm   Lclub/virgilin/jvm/TestJvm;    
```

```java
第四个方法
0009 
0025
0026 
0001 
001a 
0000 0033
0001 0001 0000
0005 1ab3 000a b100 0000 0200 1b00 0000
0a00 0200 0000 1b00 0400 1c00 1c00 0000
0c00 0100 0000 0500 1200 1300 00
对应：
 public static void setA(int);
    descriptor: (I)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=1, args_size=1
         0: iload_0
         1: putstatic     #10                 // Field a:I
         4: return
      LineNumberTable:
        line 27: 0
        line 28: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0     a   I
```

```java
第五个方法{静态代码块}
0008 static
0027
0019
0001
001a
0000 001e
00 0100 0000
0000 0612 0bb3 000c b100 0000 0100 1b00
0000 0600 0100 00
00 0c
对应：
static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: ldc           #11                 // String virgilin
         2: putstatic     #12                 // Field name:Ljava/lang/String;
         5: return
      LineNumberTable:
        line 12: 0
```

### attribute_info

```java
类的属性表：
0001  attributes_count=1
0028   attribute_name_index    #40=Utf8    SourceFile
0000 0002  attribute_length
0029      #41 = Utf8    TestJvm.java
SourceFile: "TestJvm.java"
```



**字节码指令表** 

| 字节码  | 助记符             | 指令含义                                     |
| ---- | --------------- | ---------------------------------------- |
| 0x00 | nop             | None                                     |
| 0x01 | aconst_null     | 将null推送至栈顶                               |
| 0x02 | iconst_m1       | 将int型-1推送至栈顶                             |
| 0x03 | iconst_0        | 将int型0推送至栈顶                              |
| 0x04 | iconst_1        | 将int型1推送至栈顶                              |
| 0x05 | iconst_2        | 将int型2推送至栈顶                              |
| 0x06 | iconst_3        | 将int型3推送至栈顶                              |
| 0x07 | iconst_4        | 将int型4推送至栈顶                              |
| 0x08 | iconst_5        | 将int型5推送至栈顶                              |
| 0x09 | lconst_0        | 将long型0推送至栈顶                             |
| 0x0a | lconst_1        | 将long型1推送至栈顶                             |
| 0x0b | fconst_0        | 将float型0推送至栈顶                            |
| 0x0c | fconst_1        | 将float型1推送至栈顶                            |
| 0x0d | fconst_2        | 将float型2推送至栈顶                            |
| 0x0e | dconst_0        | 将double型0推送至栈顶                           |
| 0x0f | dconst_1        | 将double型1推送至栈顶                           |
| 0x10 | bipush          | 将单字节的常量值(-128~127)推送至栈顶                  |
| 0x11 | sipush          | 将一个短整型常量(-32768~32767)推送至栈顶              |
| 0x12 | ldc             | 将int,float或String型常量值从常量池中推送至栈顶          |
| 0x13 | ldc_w           | 将int,float或String型常量值从常量池中推送至栈顶(宽索引)     |
| 0x14 | ldc2_w          | 将long或double型常量值从常量池中推送至栈顶(宽索引)          |
| 0x15 | iload           | 将指定的int型本地变量推送至栈顶                        |
| 0x16 | lload           | 将指定的long型本地变量推送至栈顶                       |
| 0x17 | fload           | 将指定的float型本地变量推送至栈顶                      |
| 0x18 | dload           | 将指定的double型本地变量推送至栈顶                     |
| 0x19 | aload           | 将指定的引用类型本地变量推送至栈顶                        |
| 0x1a | iload_0         | 将第一个int型本地变量推送至栈顶                        |
| 0x1b | iload_1         | 将第二个int型本地变量推送至栈顶                        |
| 0x1c | iload_2         | 将第三个int型本地变量推送至栈顶                        |
| 0x1d | iload_3         | 将第四个int型本地变量推送至栈顶                        |
| 0x1e | lload_0         | 将第一个long型本地变量推送至栈顶                       |
| 0x1f | lload_1         | 将第二个long型本地变量推送至栈顶                       |
| 0x20 | lload_2         | 将第三个long型本地变量推送至栈顶                       |
| 0x21 | lload_3         | 将第四个long型本地变量推送至栈顶                       |
| 0x22 | fload_0         | 将第一个float型本地变量推送至栈顶                      |
| 0x23 | fload_1         | 将第二个float型本地变量推送至栈顶                      |
| 0x24 | fload_2         | 将第三个float型本地变量推送至栈顶                      |
| 0x25 | fload_3         | 将第四个float型本地变量推送至栈顶                      |
| 0x26 | dload_0         | 将第一个double型本地变量推送至栈顶                     |
| 0x27 | dload_1         | 将第二个double型本地变量推送至栈顶                     |
| 0x28 | dload_2         | 将第三个double型本地变量推送至栈顶                     |
| 0x29 | dload_3         | 将第四个double型本地变量推送至栈顶                     |
| 0x2a | aload_0         | 将第一个引用类型本地变量推送至栈顶                        |
| 0x2b | aload_1         | 将第二个引用类型本地变量推送至栈顶                        |
| 0x2c | aload_2         | 将第三个引用类型本地变量推送至栈顶                        |
| 0x2d | aload_3         | 将第四个引用类型本地变量推送至栈顶                        |
| 0x2e | iaload          | 将int型数组指定索引的值推送至栈顶                       |
| 0x2f | laload          | 将long型数组指定索引的值推送至栈顶                      |
| 0x30 | faload          | 将float型数组指定索引的值推送至栈顶                     |
| 0x31 | daload          | 将double型数组指定索引的值推送至栈顶                    |
| 0x32 | aaload          | 将引用类型数组指定索引的值推送至栈顶                       |
| 0x33 | baload          | 将boolean或byte型数组指定索引的值推送至栈顶              |
| 0x34 | caload          | 将char型数组指定索引的值推送至栈顶                      |
| 0x35 | saload          | 将short型数组指定索引的值推送至栈顶                     |
| 0x36 | istore          | 将栈顶int型数值存入指定本地变量                        |
| 0x37 | lstore          | 将栈顶long型数值存入指定本地变量                       |
| 0x38 | fstore          | 将栈顶float型数值存入指定本地变量                      |
| 0x39 | dstore          | 将栈顶double型数值存入指定本地变量                     |
| 0x3a | astore          | 将栈顶引用类型数值存入指定本地变量                        |
| 0x3b | istore_0        | 将栈顶int型数值存入第一个本地变量                       |
| 0x3c | istore_1        | 将栈顶int型数值存入第二个本地变量                       |
| 0x3d | istore_2        | 将栈顶int型数值存入第三个本地变量                       |
| 0x3e | istore_3        | 将栈顶int型数值存入第四个本地变量                       |
| 0x3f | lstore_0        | 将栈顶long型数值存入第一个本地变量                      |
| 0x40 | lstore_1        | 将栈顶long型数值存入第二个本地变量                      |
| 0x41 | lstore_2        | 将栈顶long型数值存入第三个本地变量                      |
| 0x42 | lstore_3        | 将栈顶long型数值存入第四个本地变量                      |
| 0x43 | fstore_0        | 将栈顶float型数值存入第一个本地变量                     |
| 0x44 | fstore_1        | 将栈顶float型数值存入第二个本地变量                     |
| 0x45 | fstore_2        | 将栈顶float型数值存入第三个本地变量                     |
| 0x46 | fstore_3        | 将栈顶float型数值存入第四个本地变量                     |
| 0x47 | dstore_0        | 将栈顶double型数值存入第一个本地变量                    |
| 0x48 | dstore_1        | 将栈顶double型数值存入第二个本地变量                    |
| 0x49 | dstore_2        | 将栈顶double型数值存入第三个本地变量                    |
| 0x4a | dstore_3        | 将栈顶double型数值存入第四个本地变量                    |
| 0x4b | astore_0        | 将栈顶引用型数值存入第一个本地变量                        |
| 0x4c | astore_1        | 将栈顶引用型数值存入第二个本地变量                        |
| 0x4d | astore_2        | 将栈顶引用型数值存入第三个本地变量                        |
| 0x4e | astore_3        | 将栈顶引用型数值存入第四个本地变量                        |
| 0x4f | iastore         | 将栈顶int型数值存入指定数组的指定索引位置                   |
| 0x50 | lastore         | 将栈顶long型数值存入指定数组的指定索引位置                  |
| 0x51 | fastore         | 将栈顶float型数值存入指定数组的指定索引位置                 |
| 0x52 | dastore         | 将栈顶double型数值存入指定数组的指定索引位置                |
| 0x53 | aastore         | 将栈顶引用型数值存入指定数组的指定索引位置                    |
| 0x54 | bastore         | 将栈顶boolean或byte型数值存入指定数组的指定索引位置          |
| 0x55 | castore         | 将栈顶char型数值存入指定数组的指定索引位置                  |
| 0x56 | sastore         | 将栈顶short型数值存入指定数组的指定索引位置                 |
| 0x57 | pop             | 将栈顶数值弹出(数值不能是long或double类型的)             |
| 0x58 | pop2            | 将栈顶的一个(对于非long或double类型)或两个数值(对于非long或double的其他类型)弹出 |
| 0x59 | dup             | 复制栈顶数值并将复制值压入栈顶                          |
| 0x5a | dup_x1          | 复制栈顶数值并将两个复制值压入栈顶                        |
| 0x5b | dup_x2          | 复制栈顶数值并将三个(或两个)复制值压入栈顶                   |
| 0x5c | dup2            | 复制栈顶一个(对于long或double类型)或两个(对于非long或double的其他类型)数值并将复制值压入栈顶 |
| 0x5d | dup2_x1         | dup_x1指令的双倍版本                            |
| 0x5e | dup2_x2         | dup_x2指令的双倍版本                            |
| 0x5f | swap            | 将栈顶最顶端的两个数值互换(数值不能是long或double类型)        |
| 0x60 | iadd            | 将栈顶两int型数值相加并将结果压入栈顶                     |
| 0x61 | ladd            | 将栈顶两long型数值相加并将结果压入栈顶                    |
| 0x62 | fadd            | 将栈顶两float型数值相加并将结果压入栈顶                   |
| 0x63 | dadd            | 将栈顶两double型数值相加并将结果压入栈顶                  |
| 0x64 | isub            | 将栈顶两int型数值相减并将结果压入栈顶                     |
| 0x65 | lsub            | 将栈顶两long型数值相减并将结果压入栈顶                    |
| 0x66 | fsub            | 将栈顶两float型数值相减并将结果压入栈顶                   |
| 0x67 | dsub            | 将栈顶两double型数值相减并将结果压入栈顶                  |
| 0x68 | imul            | 将栈顶两int型数值相乘并将结果压入栈顶                     |
| 0x69 | lmul            | 将栈顶两long型数值相乘并将结果压入栈顶                    |
| 0x6a | fmul            | 将栈顶两float型数值相乘并将结果压入栈顶                   |
| 0x6b | dmul            | 将栈顶两double型数值相乘并将结果压入栈顶                  |
| 0x6c | idiv            | 将栈顶两int型数值相除并将结果压入栈顶                     |
| 0x6d | ldiv            | 将栈顶两long型数值相除并将结果压入栈顶                    |
| 0x6e | fdiv            | 将栈顶两float型数值相除并将结果压入栈顶                   |
| 0x6f | ddiv            | 将栈顶两double型数值相除并将结果压入栈顶                  |
| 0x70 | irem            | 将栈顶两int型数值作取模运算并将结果压入栈顶                  |
| 0x71 | lrem            | 将栈顶两long型数值作取模运算并将结果压入栈顶                 |
| 0x72 | frem            | 将栈顶两float型数值作取模运算并将结果压入栈顶                |
| 0x73 | drem            | 将栈顶两double型数值作取模运算并将结果压入栈顶               |
| 0x74 | ineg            | 将栈顶int型数值取负并将结果压入栈顶                      |
| 0x75 | lneg            | 将栈顶long型数值取负并将结果压入栈顶                     |
| 0x76 | fneg            | 将栈顶float型数值取负并将结果压入栈顶                    |
| 0x77 | dneg            | 将栈顶double型数值取负并将结果压入栈顶                   |
| 0x78 | ishl            | 将int型数值左移指定位数并将结果压入栈顶                    |
| 0x79 | lshl            | 将long型数值左移指定位数并将结果压入栈顶                   |
| 0x7a | ishr            | 将int型数值右(带符号)移指定位数并将结果压入栈顶               |
| 0x7b | lshr            | 将long型数值右(带符号)移指定位数并将结果压入栈顶              |
| 0x7c | iushr           | 将int型数值右(无符号)移指定位数并将结果压入栈顶               |
| 0x7d | lushr           | 将long型数值右(无符号)移指定位数并将结果压入栈顶              |
| 0x7e | iand            | 将栈顶两int型数值"按位与"并将结果压入栈顶                  |
| 0x7f | land            | 将栈顶两long型数值"按位与"并将结果压入栈顶                 |
| 0x80 | ior             | 将栈顶两int型数值"按位或"并将结果压入栈顶                  |
| 0x81 | lor             | 将栈顶两long型数值"按位或"并将结果压入栈顶                 |
| 0x82 | ixor            | 将栈顶两int型数值"按位异或"并将结果压入栈顶                 |
| 0x83 | lxor            | 将栈顶两long型数值"按位异或"并将结果压入栈顶                |
| 0x84 | iinc            | 将指定int型变量增加指定值(如i++, i--, i+=2等)         |
| 0x85 | i2l             | 将栈顶int型数值强制转换为long型数值并将结果压入栈顶            |
| 0x86 | i2f             | 将栈顶int型数值强制转换为float型数值并将结果压入栈顶           |
| 0x87 | i2d             | 将栈顶int型数值强制转换为double型数值并将结果压入栈顶          |
| 0x88 | l2i             | 将栈顶long型数值强制转换为int型数值并将结果压入栈顶            |
| 0x89 | l2f             | 将栈顶long型数值强制转换为float型数值并将结果压入栈顶          |
| 0x8a | l2d             | 将栈顶long型数值强制转换为double型数值并将结果压入栈顶         |
| 0x8b | f2i             | 将栈顶float型数值强制转换为int型数值并将结果压入栈顶           |
| 0x8c | f2l             | 将栈顶float型数值强制转换为long型数值并将结果压入栈顶          |
| 0x8d | f2d             | 将栈顶float型数值强制转换为double型数值并将结果压入栈顶        |
| 0x8e | d2i             | 将栈顶double型数值强制转换为int型数值并将结果压入栈顶          |
| 0x8f | d2l             | 将栈顶double型数值强制转换为long型数值并将结果压入栈顶         |
| 0x90 | d2f             | 将栈顶double型数值强制转换为float型数值并将结果压入栈顶        |
| 0x91 | i2b             | 将栈顶int型数值强制转换为byte型数值并将结果压入栈顶            |
| 0x92 | i2c             | 将栈顶int型数值强制转换为char型数值并将结果压入栈顶            |
| 0x93 | i2s             | 将栈顶int型数值强制转换为short型数值并将结果压入栈顶           |
| 0x94 | lcmp            | 比较栈顶两long型数值大小, 并将结果(1, 0或-1)压入栈顶        |
| 0x95 | fcmpl           | 比较栈顶两float型数值大小, 并将结果(1, 0或-1)压入栈顶; 当其中一个数值为`NaN`时, 将-1压入栈顶 |
| 0x96 | fcmpg           | 比较栈顶两float型数值大小, 并将结果(1, 0或-1)压入栈顶; 当其中一个数值为`NaN`时, 将1压入栈顶 |
| 0x97 | dcmpl           | 比较栈顶两double型数值大小, 并将结果(1, 0或-1)压入栈顶; 当其中一个数值为`NaN`时, 将-1压入栈顶 |
| 0x98 | dcmpg           | 比较栈顶两double型数值大小, 并将结果(1, 0或-1)压入栈顶; 当其中一个数值为`NaN`时, 将1压入栈顶 |
| 0x99 | ifeq            | 当栈顶int型数值等于0时跳转                          |
| 0x9a | ifne            | 当栈顶int型数值不等于0时跳转                         |
| 0x9b | iflt            | 当栈顶int型数值小于0时跳转                          |
| 0x9c | ifge            | 当栈顶int型数值大于等于0时跳转                        |
| 0x9d | ifgt            | 当栈顶int型数值大于0时跳转                          |
| 0x9e | ifle            | 当栈顶int型数值小于等于0时跳转                        |
| 0x9f | if_icmpeq       | 比较栈顶两int型数值大小, 当结果等于0时跳转                 |
| 0xa0 | if_icmpne       | 比较栈顶两int型数值大小, 当结果不等于0时跳转                |
| 0xa1 | if_icmplt       | 比较栈顶两int型数值大小, 当结果小于0时跳转                 |
| 0xa2 | if_icmpge       | 比较栈顶两int型数值大小, 当结果大于等于0时跳转               |
| 0xa3 | if_icmpgt       | 比较栈顶两int型数值大小, 当结果大于0时跳转                 |
| 0xa4 | if_icmple       | 比较栈顶两int型数值大小, 当结果小于等于0时跳转               |
| 0xa5 | if_acmpeq       | 比较栈顶两引用型数值, 当结果相等时跳转                     |
| 0xa6 | if_acmpne       | 比较栈顶两引用型数值, 当结果不相等时跳转                    |
| 0xa7 | goto            | 无条件跳转                                    |
| 0xa8 | jsr             | 跳转至指定的16位offset位置, 并将jsr的下一条指令地址压入栈顶     |
| 0xa9 | ret             | 返回至本地变量指定的index的指令位置(一般与jsr或jsr_w联合使用)   |
| 0xaa | tableswitch     | 用于switch条件跳转, case值连续(可变长度指令)            |
| 0xab | lookupswitch    | 用于switch条件跳转, case值不连续(可变长度指令)           |
| 0xac | ireturn         | 从当前方法返回int                               |
| 0xad | lreturn         | 从当前方法返回long                              |
| 0xae | freturn         | 从当前方法返回float                             |
| 0xaf | dreturn         | 从当前方法返回double                            |
| 0xb0 | areturn         | 从当前方法返回对象引用                              |
| 0xb1 | return          | 从当前方法返回void                              |
| 0xb2 | getstatic       | 获取指定类的静态域, 并将其压入栈顶                       |
| 0xb3 | putstatic       | 为指定类的静态域赋值                               |
| 0xb4 | getfield        | 获取指定类的实例域, 并将其压入栈顶                       |
| 0xb5 | putfield        | 为指定类的实例域赋值                               |
| 0xb6 | invokevirtual   | 调用实例方法                                   |
| 0xb7 | invokespecial   | 调用超类构建方法, 实例初始化方法, 私有方法                  |
| 0xb8 | invokestatic    | 调用静态方法                                   |
| 0xb9 | invokeinterface | 调用接口方法                                   |
| 0xba | invokedynamic   | 调用动态方法                                   |
| 0xbb | new             | 创建一个对象, 并将其引用引用值压入栈顶                     |
| 0xbc | newarray        | 创建一个指定的原始类型(如int, float, char等)的数组, 并将其引用值压入栈顶 |
| 0xbd | anewarray       | 创建一个引用型(如类, 接口, 数组)的数组, 并将其引用值压入栈顶       |
| 0xbe | arraylength     | 获取数组的长度值并压入栈顶                            |
| 0xbf | athrow          | 将栈顶的异常抛出                                 |
| 0xc0 | checkcast       | 检验类型转换, 检验未通过将抛出 ClassCastException      |
| 0xc1 | instanceof      | 检验对象是否是指定类的实际, 如果是将1压入栈顶, 否则将0压入栈顶       |
| 0xc2 | monitorenter    | 获得对象的锁, 用于同步方法或同步块                       |
| 0xc3 | monitorexit     | 释放对象的锁, 用于同步方法或同步块                       |
| 0xc4 | wide            | 扩展本地变量的宽度                                |
| 0xc5 | multianewarray  | 创建指定类型和指定维度的多维数组(执行该指令时, 操作栈中必须包含各维度的长度值), 并将其引用压入栈顶 |
| 0xc6 | ifnull          | 为null时跳转                                 |
| 0xc7 | ifnonnull       | 不为null时跳转                                |
| 0xc8 | goto_w          | 无条件跳转(宽索引)                               |
| 0xc9 | jsr_w           | 跳转至指定的32位offset位置, 并将jsr_w的下一条指令地址压入栈顶   |