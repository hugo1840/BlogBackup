Oracle数据库收到类似以下ORA-4031共享内存无法分配告警时
```bash
DATABASE:"ORA-04031: unable to allocate 2456 bytes of shared memory ..."
```

一般有两种应急处置办法：一是flush共享池，二是重启数据库。

Flush共享池的方法如下。

查看当前共享池中的可用内存（**free momory**）：
```sql
select bb.* from (select pool,name,sum(bytes)/1024/1024 size_mb
from v$sgastat where pool='shared pool'
group by pool,name order by sum(bytes)/1024/1024 desc) bb
where rownum < 21 order by size_mb;
```
 
在RAC俩节点flush共享池：
```sql
alter system flush shared_pool;
```
然后查看free memory对比之前是否增加。

Flush共享池后缓存的执行计划被清理，短时间内SQL解析的时间会增加，并且能够释放的内存有限，有条件的话最好还是重启数据库。
