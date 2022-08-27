
@[TOC](MySQL解析binlog日志文件)

解析binlog并输出到屏幕的命令为：
```bash
[root@mysql-node1 ~]# mysqlbinlog /mysql/log/mysql-bin.000007
...
#220827  0:35:51 server id 142  end_log_pos 219 CRC32 0xa5c70dd2        Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#220827  0:35:51 server id 142  end_log_pos 397 CRC32 0xb368c4c6        Query   thread_id=3     exec_time=0     error_code=0
use `apptest`/*!*/;
SET TIMESTAMP=1661574951/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1073741824/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create table app01(ID int not null,
name varchar(20),
country varchar(30),
primary key pk_id(id)
)
/*!*/;
# at 397
#220827  0:37:49 server id 142  end_log_pos 462 CRC32 0xc86968fb        Anonymous_GTID  last_committed=1        sequence_number=2       rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 462
#220827  0:37:42 server id 142  end_log_pos 537 CRC32 0xf6af139b        Query   thread_id=3     exec_time=0     error_code=0
SET TIMESTAMP=1661575062/*!*/;
BEGIN
/*!*/;
# at 537
#220827  0:37:42 server id 142  end_log_pos 594 CRC32 0xebf02dab        Table_map: `apptest`.`app01` mapped to number 109
# at 594
#220827  0:37:42 server id 142  end_log_pos 652 CRC32 0x2b101abd        Write_rows: table id 109 flags: STMT_END_F

BINLOG '
lp8JYxOOAAAAOQAAAFICAAAAAG0AAAAAAAEAB2FwcHRlc3QABWFwcDAxAAMDDw8EPABaAAarLfDr
lp8JYx6OAAAAOgAAAIwCAAAAAG0AAAAAAAEAAgAD//gBAAAADVN0ZXBoZW4gQ3VycnkDVVNBvRoQ
Kw==
'/*!*/;
# at 652
#220827  0:37:49 server id 142  end_log_pos 683 CRC32 0x00b32b18        Xid = 12
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

上面Binlog的格式为**ROW**。可以看到，解析出的DML语句被加密了。


# -v：以伪SQL显示DML语句
使用`-v`来显示DML语句对应的伪SQL（以`###`开头）。
```bash
[root@mysql-node1 ~]# mysqlbinlog /mysql/log/mysql-bin.000007 -v
...
# at 594
#220827  0:37:42 server id 142  end_log_pos 652 CRC32 0x2b101abd        Write_rows: table id 109 flags: STMT_END_F

BINLOG '
lp8JYxOOAAAAOQAAAFICAAAAAG0AAAAAAAEAB2FwcHRlc3QABWFwcDAxAAMDDw8EPABaAAarLfDr
lp8JYx6OAAAAOgAAAIwCAAAAAG0AAAAAAAEAAgAD//gBAAAADVN0ZXBoZW4gQ3VycnkDVVNBvRoQ
Kw==
'/*!*/;
### INSERT INTO `apptest`.`app01`
### SET
###   @1=1
###   @2='Stephen Curry'
###   @3='USA'
# at 652
#220827  0:37:49 server id 142  end_log_pos 683 CRC32 0x00b32b18        Xid = 12
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

如果不想看到加密的内容，可以加上`--base64-output=DECODE-ROWS`：
```bash
[root@mysql-node1 ~]# mysqlbinlog /mysql/log/mysql-bin.000007 -v --base64-output=DECODE-ROWS
...
# at 594
#220827  0:37:42 server id 142  end_log_pos 652 CRC32 0x2b101abd        Write_rows: table id 109 flags: STMT_END_F
### INSERT INTO `apptest`.`app01`
### SET
###   @1=1
###   @2='Stephen Curry'
###   @3='USA'
# at 652
#220827  0:37:49 server id 142  end_log_pos 683 CRC32 0x00b32b18        Xid = 12
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

# -vv：显示伪SQL和数据类型
使用`-vv`来显示DML语句对应的伪SQL和列的数据类型等元数据信息。
```bash
[root@mysql-node1 ~]# mysqlbinlog /mysql/log/mysql-bin.000007 -vv --base64-output=decode-rows
...
# at 594
#220827  0:37:42 server id 142  end_log_pos 652 CRC32 0x2b101abd        Write_rows: table id 109 flags: STMT_END_F
### INSERT INTO `apptest`.`app01`
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='Stephen Curry' /* VARSTRING(60) meta=60 nullable=1 is_null=0 */
###   @3='USA' /* VARSTRING(90) meta=90 nullable=1 is_null=0 */
# at 652
#220827  0:37:49 server id 142  end_log_pos 683 CRC32 0x00b32b18        Xid = 12
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

# 解析binlog中特定数据库的内容
使用`--databse`指定数据库。 
```bash
mysqlbinlog --no-defaults --database=app --base64-output=decode-rows \
-vv  mysql-bin.00001 > /tmp/binlog-000.txt
 ```
 
# 按起止时间解析binlog文件
使用`--start-datetime`和`--stop-datetime`指定起止时间。
```bash
mysqlbinlog --no-defaults --database=app --base64-output=decode-rows \
-vv  --start-datetime='2022-05-13 17:30:00' \
--stop-datetime='2022-05-13 19:55:00' \
mysql-bin.00001 > /tmp/binlog-001.txt
```

**References**
【1】https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog.html
【2】https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog-row-events.html
