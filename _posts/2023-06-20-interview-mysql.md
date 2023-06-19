---
layout:       post
title:        "MySQL 常见面试题"
author:       "matt"
header-style: text
catalog:      true
tags:
    - interview
    - mysql
---

## 一、基础篇  

### 1、MySQL 一条 select 语句执行流程是怎么样的？

MySQL 架构分为两层：Server 层和存储引擎层。Server 层负责建立连接、分析和执行 SQL；存储引擎层负责数据的存储和提取。

![MySQL 执行架构](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/sql%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B/mysql%E6%9F%A5%E8%AF%A2%E6%B5%81%E7%A8%8B.png)

#### 连接器

TCP 三次握手建立连接；校验客户端的用户名和密码；读取用户权限，后续权限操作逻辑都基于此刻读取的权限来判断。

```sql
# 连接客户端
mysql -h ${ip} -u ${user} -p ${port}

# 查看客户端连接数
> show processlist;

# 查看空闲连接的最大空闲时长
> show variables like 'wait_timeout';

# 断开空闲的连接
> kill connection +${connection_id}

# 查看最大连接数
> show variables like 'max_connections';

# 解决长连接的问题：
# 1、定期断开长连接；2、客户端主动重置连接。
```

#### 查询缓存

MySQL 服务收到 SQL 语句，解析 SQL 语句的第一个字段，如果是查询语句则会先去查询缓存（k-v 形式）。命中缓存则直接返回 value 给客户端，没有则继续执行，执行完成后的查询结果存入查询缓存。

`8.0 之后删除了查询缓存`。因为只要有一个表有更新，那么这个表的查询缓存就会被清空。

```sql
# 设置不缓存查询，/etc/mysql/my.cnf，0表示禁用查询缓存，1表示启动且仅对 SELECT 查询有效，2表示启用且对所有 SQL 查询都有效。
query_cache=0

# 单个操作不缓存
SELECT SQL_NO_CACHE * FROM ${table_name} WHERE ${condition};
```

#### 解析 SQL

在执行 SQL 查询语句之前，先有`解释器`来对 SQL 语句做`词法分析`和`语法分析`。

**词法分析：** 根据输入的字符串识别关键字，构建出`SQL 语法树`，方面后续模块获取 SQL 类型、表名、字段名、where 条件等等。

**语法分析：** 根据上面结果判断输入 SQL 语法是否`满足 MySQL 语法`（表、字段不存在不在这里执行）。

#### 执行 SQL

每条 select 查询语句的执行流程分为如下三个阶段：

- prepare：预处理器

检查 SQL 查询语句中的表或字段是否存在；将 select \* 中的 \* 符号，扩展为表上的所有列。

- optimize：优化器

将 SQL 查询语句的执行方案确定下来，例如多索引时决定使用哪个索引。

```sql
# 在查询语句前加上 explain 命令，就可以输出这条 SQL 语句的执行计划
explain select * from ${table_name} where ${condition}
```

- execute：执行器

确定了执行方案后，执行器就会与存储引擎交互，交互以`记录`为单位，读取记录并返回给客户端。常见的三种执行方式：`主键索引查询`、`全表扫描`、`索引下推`。

### 2、MySQL 一行记录是怎么存储的？（InnoDB）

#### 数据存放位置

MySQL 的数据都是保存在磁盘上，同时支持多种存储引擎，默认的存储引擎为 `InnoDB`。

```sql
# 查询 MySQL 数据库的文件存放目录
> show variables like 'datadir'

> ls ${sql_data_dir}
db.opt
${table_name}.frm
${table_name}.ibd
```

每创建一个 database 都会在 `${sql_data_dir}` 目录下创建一个以 database 为名的目录，保存表结构和表数据。其中，上述`db.opt`存储当前数据库的默认字符集和字符校验规则；`${table_name}.frm`存储表结构，保存每个表的元数据信息；`${table_name}.ibd`存储表数据，由参数`innodb_file_per_table`控制存储的数据、索引等信息单独存储在一个独占表空间，还是共享表空间文件。

#### 表空间文件的结构

表空间由`段（segment）、区（extent）、页（page）、行（row）`组成。

![InnoDB 存储引擎的逻辑存储结构](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.drawio.png)

- 行（row）
数据库中的记录以 row 的形式存放，每行记录根据不同的行格式，有不同的存储结构。

- 页（page）
InnoDB 的数据`以页为单位来读写`的，默认每个页大小为`16KB`。

- 区（extent）
InnoDB 存储引擎用`B+树`来组织数据，树的每一层通过双向链表连接起来，为了解决链表中相邻页的物理位置也相邻。当表中数据量大时，为索引分配空间时就`按区为单位`来分配，区的大小为`1MB`，连续64个页会被划为一个区。

- 段（segment）
段一般分为：`索引段`（B+树的非叶子节点的区集合）、`数据段`（B+树的叶子节点的区集合）、`回滚段`（回滚数据的区集合）。

#### 行格式

行格式是一条记录的存储结构，InnoDB 提供了4种行格式：`Redundant`（弃用）、`Compact`（紧凑型，5.0后引入）、`Dynamic`（5.7后默认）和`Compressed`。由于后两者都是基于 Compact 行格式改进的，这里主要介绍 Compact 行格式。

#### Compact 行格式

一条完整的记录分为`记录的额外信息`和`记录的真实信息`两个部分。
![Compact 行格式](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/COMPACT.drawio.png)

- 记录的额外信息

（1）变长字段长度列表（16进制）

`存储变长字段时`，把数据占用的大小存在变长字段长度列表里，读取数据的时候可以根据该值去读取对应的长度。在全是 int 类型的字段时就不会存储，节省空间。  

变长字段的真实数据占用的字节数按照`列的顺序倒序存放`。因为记录头信息中指向下一个记录的指针，指向下一条记录的`记录头信息和真实数据之间`的位置，向左读就是记录头信息，向右读就是真实数据。逆序存放使得`位置靠前的记录的真实数据和对应的字段长度信息可以同时在一个 CPU Cache Line中`，这样就可以[提高 CPU Cache 的命中率](https://xiaolincoding.com/os/1_hardware/how_to_make_cpu_run_faster.html)。

（2）NULL 值列表（16进制）

将 NULL 值记录到真实数据中浪费空间，将`每一列对应一个二进制位`（bit），二进制位按照列的顺序`逆序排列`，1表示为 NULL，二进制位个数不足整数个字节，则在高位补0。

当数据库表的字段都定义成 NOT NULL 时（`建表时建议将字段都设置为 NOT NULL`），表的行格式就不会有 NULL 值列表了。

（3）记录头信息

这里介绍一些重要的信息：

**delete_mask：** 标记此条数据是否被删除，删除标记为1。

**next_record：** 下一条记录的位置，记录之间通过链表组织的。

**record_type：** 表示当前记录的类型，0表示普通记录，1表示 B+ 树非叶子节点记录，2表示最小记录，3表示最大记录。

<br/>

- 记录的真实信息

row_id
、记trx_id录、真roll_pointer实数据部分除了我们定义的字段，还包括`row_id`、`trx_id`、`roll_pointer`三个字段。[MVCC 原理](https://xiaolincoding.com/mysql/transaction/mvcc.html)

![记录的真是数据结构](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%AE%B0%E5%BD%95%E7%9A%84%E7%9C%9F%E5%AE%9E%E6%95%B0%E6%8D%AE.png)

**row_id：** 建表时指定了主键或者唯一约束列，则没有 row_id 隐藏字段，占6个字节。

**trx_id:** 必须的，占6个字节，事务 ID，表示这个数据由哪个事务生成的。

**roll_pointer：** 必须的，占7个字节，记录这条记录上一个版本的指针。

- varchar(n) 中 n 最大取值为多少？

MySQL 规定除了 TEXT、BLOBs 这种大对象类型之外，其他所有的列（不包含隐藏列和记录头信息）占用的字符长度加起来不超过`65535 个字节`。需要包含`变长字段列表`和`NULL 值列表`所占用的字节数。

- 行溢出后，MySQL 是怎么操作的？

发生行溢出时，多的数据就会存到另外的`溢出页`中。

![Compact](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%8C%E6%BA%A2%E5%87%BA.png)

---


## 二、索引篇

---

## 三、事务篇

---

## 四、锁篇

---

## 五、日志篇

---

## 六、内存篇
++++++