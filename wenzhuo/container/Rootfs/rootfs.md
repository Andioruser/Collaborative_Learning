# 详解Linux RootFS  

## 什么是文件系统？
< 文件系统是操作系统用于明确存储设备（常见的是磁盘，也有基于NAND Flash的固态硬盘）或分区上的文件的方法和数据结构；
< 即在存储设备上组织文件的方法。操作系统中负责管理和存储文件信息的软件机构称为文件管理系统，简称文件系统。
< 文件系统由三部分组成：文件系统的接口，对对象操纵和管理的软件集合，对象及属性。
< 从系统角度来看，文件系统是对文件存储设备的空间进行组织和分配，负责文件存储并对存入的文件进行保护和检索的系统。
< 具体地说，它负责为用户建立文件，存入、读出、修改、转储文件，控制文件的存取，当用户不再使用时撤销文件等

**简单理解文件系统是用来组织管理文件的，可以对文件进行读写等操作。**

## 什么是rootfs
