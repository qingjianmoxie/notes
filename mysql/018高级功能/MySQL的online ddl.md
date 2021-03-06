MySQL 5.6开始支持Online DDL

## 一、锁步骤

Online DDL的过程中，对于锁的获取分为五步（具体online DDL过程比较复杂，本文不展开说明）：

1）拿到MDL写锁

2）降级成MDL读锁

3）真正做DDL

4）升级成MDL写锁

5）释放MDL锁

1、2、4、5如果没有锁冲突，执行时间都是非常短的。绝大部分时间是第三步占用了，而这个期间，表可用正常读写，所以被称为Online DDL



## 二、日志步骤

在 MySQL 5.5 版本之前，重建MySQL表的过程是自动完成转存数据、交换表名、删除旧表。如果在这个过程中，有新的数据要写入到表 A 的话，就会造成数据丢失。因此，在整个 DDL 过程中，表 A 中不能有更新。也就是说，这个 DDL 不是 Online 的。

MySQL 5.6 版本开始引入的 Online DDL，对这个操作流程做了优化。

- 建立一个临时文件，扫描表 A 主键的所有数据页；
- 用数据页中表 A 的记录生成 B+ 树，存储到临时文件中；
- 生成临时文件的过程中，将所有对 A 的操作记录在一个日志文件（row log）中；
- 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表 A 相同的数据文件；(**应用row log的过程可能又回有页分裂**)
- 用临时文件替换表 A 的数据文件。