# mysql存储引擎InnoDB详解，从底层看清InnoDB数据结构

InnoDB一个支持事务安全的存储引擎，同时也是mysql的默认存储引擎。

## InnoDB简介

**InnoDB将数据划分为若干页，以页作为磁盘与内存交互的基本单位，一般页的大小为16KB**

