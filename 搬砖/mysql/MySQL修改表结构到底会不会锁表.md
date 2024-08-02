### 〇、关于DDL、DML和DCL

**DDL**（Data Definition Languages）语句：数据定义语言，这些语句定义了不同的数据段、数据库、表、列、索引等数据库对象的定义。  
常用的语句关键字主要包括 create、drop、alter 等。

**DML**（Data Manipulation Language）语句：数据操纵语句，用于添加、删除、更新和查询数据库记录，并检查数据完整性。  
常用的语句关键字主要包括 insert、delete、udpate 和 select 等。(增删改查）

**DCL**（Data Control Language）语句：数据控制语句，用于控制不同数据段直接的许可和访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别。  
主要的语句关键字包括 grant、revoke 等。

___

### 一、DDL 实现方式

在 **MySQL5.6** 版本以前，执行 DDL 主要有两种方式：**Copy 方式** 和 **In-place 方式**

#### Copy 方式执行DDL操作

1.  创建与原表结构定义完全相同的临时表
2.  为原表加MDL（meta data lock，元数据锁）锁，禁止对表中数据进行增删改，允许查询
3.  在临时表上执行DDL语句
4.  按照主键 ID 递增的顺序，把数据一行一行地从原表里读出来再插入到临时表中
5.  升级原表上的锁，禁止对原表中数据进行读写操作
6.  将原表删除，将临时表重命名为原表名，DDL操作完成

#### In-place 方式执行DDL操作

**In-place 方式** 又称为 `Fast Index Creation` 。与 Copy 方式相比，In-place方式不复制数据，因此大大加快了执行速度。但是这种方式仅支持**对二级索引进行添加、删除操作**，而且与Copy方式一样需要全程锁表。下面以添加索引为例，简单介绍In-place方式的实现流程：

1.  创建新索引的数据字典
2.  为原表加MDL（meta data lock，元数据锁）锁，禁止对表中数据进行增删改，允许查询
3.  按照聚簇索引的顺序，查询数据，找到需要的索引列数据，排序后插入到新的索引页中
4.  等待打开当前表的所有只读事务提交
5.  创建索引结束

### 二、Online DDL

**MySQL5.6** 版本之后加入了 **Online DDL** 新特性，用于支持DDL执行期间DML语句的并行操作，提高数据库的吞吐量。  
与 Copy 方式和 In-place 方式相比，Online 方式在执行DDL的时候可以对表中数据进行读写操作。  
Online DDL可以有效改善DDL期间对数据库的影响：

-   Online DDL期间，查询和DML操作在多数情况下可以正常执行，对表格的锁时间也会大大减少，尽可能的保证数据库的可扩展性；
-   允许 In-place 操作的 DDL，避免重建表格占用过多磁盘IO及CPU资源，减少对数据库的整体负荷，使得在DDL期间，能够维持数据库的高性能及高吞吐量；
-   允许 In-place 操作的 DDL，比需要COPY到临时文件的操作要更少占用buffer pool，避免以往DDL过程中性能的临时下降，因为以前需要拷贝数据到临时表，这个过程会占用到buffer pool ，导致内存中的部分频繁访问的数据会被清理出去。

Online DDL 实现实质上也可以分为2种方式：**Copy** 方式和 **In-place** 方式：

-   对于不支持Online DDL的 SQL，则采用 **Copy** 方式，比如**删除主键**，**修改列数据类型**，**变更表字符集**等。这些操作都会导致记录格式发生变化，无法通过简单的全量+增量的方式实现Online DDL。
-   对于支持Online DDL的 SQL，则采用 **In-place** 方式，MySQL 内部以“**是否修改行记录格式**”为标准，又将 **In-place** 方式分为两类：
    
    -   如果修改了行记录格式，则需要重建表，比如 **添加主键**，**添加、删除字段**，**修行格式ROW\_FORMAT**，**OPTIMIZE优化表** 等操作，这种方式被称为 **rebuild方式**。
    -   如果没有修改行记录格式，仅修改表的元数据，则不需要重建表，比如 **添加、删除、重命名二级索引**，**设置、删除字段的默认值**，**重命名字段**，**重命名表** 等操作，这种方式被称为 **no-rebuild方式**。
![[1.jpg]]

___

### 三、Online DDL 实现流程

**Online DDL** 主要包括3个阶段，Prepare阶段，Execute阶段，Commit阶段。  
下面将主要介绍Online DDL执行过程中三个阶段的流程。

Prepare 阶段

-   持有 EXCLUSIVE-MDL 锁，禁止DML语句读写
-   根据DDL类型，确定执行方式(Copy,Online-rebuild,Online-no-rebuild)
-   创建新的 frm 和 ibd 临时文件（ibd临时文件仅rebuild类型需要）
-   分配 row\_log 空间，用来记录 DDL Execute 阶段产生的DML操作（仅rebuild类型需要）

Execute 阶段

-   降级 EXCLUSIVE-MDL 锁，允许DML语句读写
-   扫描原表主键以及二级索引的所有数据页，生成B+树，存储到临时文件中
-   将 DDL Execute 阶段产生的DML操作记录到 row\_log（仅rebuild类型需要）

Commit 阶段

-   升级到 EXCLUSIVE-MDL 锁，禁止DML语句读写
-   将 row\_log 中记录的DML操作应用到临时文件，得到一个逻辑数据上与原表相同的数据文件（仅rebuild类型需要）
-   重命名 frm 和 idb 临时文件，替换原表，将原表文件删除
-   提交事务（刷事务的redo日志），变更完成

由上面的流程可知，Prepare阶段和Commit阶段都禁止读写，只有Execute允许读写，那为什么说**Online DDL 方式在执行过程中可以对表中数据进行读写操作**？  
其实是因为Prepare和Commit阶段相对于Execute阶段时间特别短，因此基本可以认为是全程允许读写的。  
Prepare阶段和Commit阶段都禁止读写，主要是为了保证数据一致性。

___

### 四、Online DDL 的语法与可选参数

```pgsql
ALTER TABLE …. , ALGORITHM[=]{DEFAULT|INPLACE|COPY}, LOCK[=]{DEFAULT|NONE|SHARED|EXCLUSIVE}
```

```pgsql
ALTER TABLE tablename DROP COLUMN age,ALGORITHM=INPLACE,LOCK=DEFAULT;
```

**COPY**：使用 **Copy 方式** 执行DDL操作，DDL 执行过程中，不允许DML操作。  
**INPLACE**：使用 **In-place 方式** 执行DDL操作，DDL 执行过程中，允许DML操作。  
**DEFAULT**：默认选项，根据DDL的操作类型，自动选择DDL执行方式，优先选择 **In-place 方式**，不满足条件时选择 **Copy 方式**。

#### LOCK 选项

**EXCLUSIVE**：对整个表添加排他锁（X锁），不允许DML操作  
**SHARED**：对整个表添加共享锁（S锁），允许查询操作，但是不允许数据变更操作  
**NONE**：不对表加锁，既允许查询操作，也支持数据变更操作，即允许所有的 DML 操作，该模式下并发最好  
**DEFAULT**：默认选项，根据DDL的操作类型，最小程度的加锁，尽可能支持DML操作