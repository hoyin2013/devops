# gtid模式下mysql主从出现1032错误的处理方法

## 1.记录故障时的日志号和日志点

```mysql
Master_Log_File: mybinlog.001639
Read_Master_Log_Pos: 630371711
```

## 从主库查看对应事务的gtid号，记录`SESSION.GTID_NEXT`

```bash
mysqlbinlog --start-position=630371711  -vv --base64-output=DECODE-ROWS  mybinlog.001639 >binlog-001639.sql

# echo
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 630371711
#200408  9:21:05 server id 1  end_log_pos 630371776 CRC32 0xef7ca8f2    GTID    last_committed=1030     sequence_number=1031    rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= '4b401c5f-10bb-11ea-b500-b02628f11b92:8420166'/*!*/;
# at 630371776
```

## 从库跳过该事务并重新进行同步

```mysql
stop slave;
SET @@SESSION.GTID_NEXT= '4b401c5f-10bb-11ea-b500-b02628f11b92:8420166';
BEGIN;COMMIT;
SET GTID_NEXT='AUTOMATIC';
start slave;
```
