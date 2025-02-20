## 简介

DWARF 是一种广泛使用的标准调试信息格式, DWARF使用DIE（Debugging Information Entry）来描述变量、数据类型、代码等，DIE中包含了标签（Tag）和一系列属性（Attributes）.DWARF还定义了一些关键的数据结构，如行号表（Line Number Table）、调用栈信息（Call Frame Information）等，有了这些关键数据结构之后，开发者就可以在源码级别动态添加断点、显示完整的调用栈信息、查看调用栈中指定栈帧的信息.

ELF 文件格式包含了DWARF调试信息对应的section，一般以".debug”或”.zdebug”开头。.debug前缀开头的section表示数据未压缩，.zdebug前缀开头的section表示数据经过了压缩.

## DIE

DWARF使用一系列的调试信息条目（DIEs）来对源程序进行描述，对于源程序中的某个程序构造进行描述，有的可能需要一个调试信息条目（DIE）就够了，有的则可能需要一组调试信息条目（DIEs）共同描述才可以.

### DIE（调试信息条目）

每个`DIE`由一个`tag`和一系列`attribute`组成. 调试信息条目多将其存储在.debug_info和.debug_types中，如果涉及到压缩会存储到.zdebug_info和.zdebug_types中.

### Tag

其名称以DW_TAG开头，它指明了DIE描述的程序构造所属的类型.

### Attribute

其名称以DW_AT开头. 它进一步补充了DIE要描述的程序构造的信息.

attribute取值可以划分为如下几种类型：

1. Address, 引用被描述程序的地址空间的某个位置；

2. Block, 未被解释的任意数量的字节数据块；

3. Constant, 1、2、4、8字节未被解释的数据，或者以LEB128形式编码的数据；

4. Flag, 指示属性存在与否的小常数；

5. lineptr, 引用存储着行号信息的DWARF section中的某个位置；

6. loclistptr, 引用存储着位置列表的DWARF section中的某个位置，某些对象的内存地址在其生命周期内会发生移动，需要通过位置列表来进行描述；

7. macptr, 引用存储着macro信息的DWARF section中的某个位置；

8. rangelistptr, 引用存储着非相邻地址区间信息的DWARF section中的某个位置；

9. Reference, 引用某个描述program的DIE；

10. String, 以'\0'结尾的字符序列，字符串可能会在DIE中直接表示，也可能通过一个独立的字符串表中的偏移量（索引）来引用。

### DIE的类型

1. 数据和类型: https://www.hitzhangjie.pro/debugger101.io/8-dwarf/3-die-desc-datatype.html

2. 可执行代码: https://www.hitzhangjie.pro/debugger101.io/8-dwarf/3-die-desc-code.html

### CFI（调用栈）

https://www.hitzhangjie.pro/debugger101.io/8-dwarf/4-other-callframe-info.html

## 在ELF文件中相关的SECTIONS

- .debug_abbrev, 存储.debug_info中使用的缩写信息；
- .debug_arranges, 存储一个加速访问的查询表，通过内存地址查询对应编译单元信息；
- .debug_frame, 存储调用栈帧信息；
- .debug_info, 存储核心DWARF数据，包含了描述变量、代码等的DIEs；
- .debug_line, 存储行号表程序 (程序指令由行号表状态机执行，执行后构建出完整的行号表)
- .debug_loc, 存储location描述信息；
- .debug_macinfo, 存储宏相关描述信息；
- .debug_pubnames, 存储一个加速访问的查询表，通过名称查询全局对象和函数；
- .debug_pubtypes, 存储一个加速访问的查询表，通过名称查询全局类型；
- .debug_ranges, 存储DIEs中引用的address ranges；
- .debug_str, 存储.debug_info中引用的字符串表，也是通过偏移量来引用；
- .debug_types, 存储描述数据类型相关的DIEs；

## References

1. DWARF标准: https://dwarfstd.org/

2. 文档: https://www.hitzhangjie.pro/debugger101.io/8-dwarf/

3. CSDN: ttps://blog.csdn.net/pwl999/article/details/107569603

4. eh_frame: https://refspecs.linuxfoundation.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html