# ucoreOS课程lab8文件系统

[TOC]

## 概念

文件系统：操作系统当中负责持久数据保存的子系统，提供数据存储和访问的功能。功能：组织、检索、读写访问数据。

虚拟文件系统：为多种不同的物理文件系统提供统一的接口。

文件：具有符号名，由字节序列构成的数据项集合。文件是文件系统的基本数据单位。文件名是文件的标识符号。

- 文件属性：名称、类型、位置、大小、保护、创建者、创建时间、最近修改时间、……；
- 文件头：文件系统元数据中的信息，存储文件属性、位置和顺序。

文件系统的功能：

- 分配文件磁盘空间
  - 管理文件块（位置和顺序）；
  - 管理空闲空间（位置）；
  - 分配算法（策略）；
- 管理文件集合
  - 定位：给出文件名，能得到文件的位置和内容；
  - 命名：根据名字找到文件，通过认为命名的文件名找到系统中用数字命名的相应文件；
  - 文件系统结构：文件的组织方式；
- 数据可靠和安全
  - 安全：多层次保护数据安全；
  - 可靠：持久保存文件，避免系统崩溃、媒体错误、攻击等；

文件描述符：打开的文件在内存中所维护的相关信息；操作系统在打开文件表中维护的打开文件状态和信息。

- 内核跟踪进程打开的所有文件，为每个进程维护一个打开文件表，文件描述符是打开文件的标识。
- 文件描述符中信息内容：
  - 文件指针：最近一次读写位置(fseek(EOF))，每个进程分别维护自己的打开文件指针；
  - 文件打开计数：当前打开文件的次数，最后一个进程关闭文件时将其从打开文件表中删除；
  - 文件的磁盘位置：缓存数据访问信息，类似于快表的存在；
  - 访问权限：每个进程的文件访问模式信息（只读、可读可写）

文件的用户视图和系统视图

- 文件的用户试图：持久的数据结构；
- 系统访问接口：(UNIX等中）文件是字节序列的集合，系统不关心存储在磁盘上的数据结构。操作系统认为文件就是数据块的集合（数据块和扇区的区别：数据块是逻辑存储单元，扇区是物理存储单元）；
- 用户视图和系统视图的转换：
  - 进程读文件：获取字节所在的数据块，返回数据块内对应部分；
  - 进程写文件：获取数据块到内存，修改对应部分，写回数据块；
  - 文件系统中的基本操作单位是数据块，之访问1Byte也要缓存整个数据块。
- 访问模式：
  - 顺序访问：按字节依次读取，大多数文件访问的方式；
  - 随机访问：不常用，虚拟内存置换会用到；
  - 索引访问：依据数据特征索引，通常操作系统不完整提供索引访问，通常由数据库等提供，数据库是建立在索引内容上的磁盘访问；

文件内部结构：

- 无结构：单词、字节序列；
- 简单记录结构：分列、固定长度、可变长度；
- 复杂结构：格式化文档、可执行文件，这些操作系统只提供简单的识别（chmod +x 可执行），主要靠用户程序识别；

文件共享和访问控制

- 多用户系统中文件共享很必要
- 访问控制：常用访问模式：读、写、执行、删除、列表等；
- 文件访问控制列表(ACL)：<文件实体，权限>
  - UNIX做法：<用户|组|所有人，读|写|可执行>, chmod 777；
  - 用户标识ID，标识用户，表明每个用户所允许的权限及保护模式；
  - 组标识ID，允许用户组成组，并指定组访问权限；

语义一致性

- 规定多进程如何同时访问共享文件；
- UNIX中设计极其简单
- Unix文件系统（UFS）语义（完全没有协调控制，交给应用程序处理同步问题）：
  - 对打开文件的写入内容立即对其他打开该文件用户可见；
  - 共享文件指针允许多用户同时读取和写入文件；
  - 会话语义：写入内容只有当文件关闭时可见；
  - 读写锁：一些操作系统提供和文件系统提供该功能；