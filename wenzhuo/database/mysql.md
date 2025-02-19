**事务**    
将多条语句作为一个整体进行操作的功能，被称为数据库事务。    

数据库事务可以确保该事物范围内的所有操作可以全部成功或者全部失败。  

如果事务失败，那么效果就和没有执行这些SQL一样，不会对数据库数据有任何改动。 

可见，数据库事务具有ACID这4个特性： 

A : Atomicity, 原子性，将所有SQL作为原子工作单元执行，要么全部执行，要么全部不执行；
C : Consistency, 一致性，事务完成后，所有数据的状态都是一致的，即A账户只要减去了100，B账户则必定加上了100；
I : Isolation, 隔离性，如果多个事务并发执行，每个事务作出的修改必须与其他事务隔离；
D : Durability, 持久性，即事务完成后，对数据库数据的修改被持久化存储。

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
ROLLBACK;
```