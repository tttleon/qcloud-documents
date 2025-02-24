Delete 不同于其他导入方式，它是一个同步过程。和 Insert into 相似，所有的 Delete 操作在 Doris 中是一个独立的导入作业，一般 Delete 语句需要指定表和分区以及删除的条件来筛选要删除的数据，并将会同时删除 base 表和 rollup 表的数据。

## 语法
主要的 Delete 语法如下：
```
DELETE FROM table_name [PARTITION partition_name]
WHERE
column_name1 op value[ AND column_name2 op value ...];
```

示例1：
```
DELETE FROM my_table PARTITION p1 WHERE k1 = 3;
```

示例2:
```
DELETE FROM my_table PARTITION p1 WHERE k1 < 3 AND k2 = "abc";
```

下面介绍删除语句中使用到的参数：
- PARTITION
Delete 语句的目标分区，若未指定，则此表必须为单分区表，否则无法 delete。

- WHERE
Delete 语句的条件语句，所有删除语句都必须指定 WHERE 语句。

>?
>- `Where` 语句中的 op 的类型可包括`=, >, <, >=, <=, !=, in, not in`。
>- `Where` 语句中的列只能是 `key` 列。
>-  当选定的 `key` 列不存在某个 rollup 表内时，无法进行 delete。
>-  条件语句中各个条件只能是 `and` 关系，如希望达成 `or` 可将条件分别写入两个 delete 语句中。
>- 如果指定表为 RANGE 或者 LIST 分区表，则必须指定 `PARTITION`。如果是单分区表，可以不指定。
>- 不同于 Insert into 命令，delete 不能手动指定`label`，有关 label 的概念可以查看 [Insert Into文档](./insert-into-manual.md)。

## 返回结果

Delete 命令是一个 SQL 命令，返回结果是同步的，分为以下几种：
1. 执行成功。
如果Delete顺利执行完成并可见，将返回下列结果，`Query OK`表示成功
```
	mysql> delete from test_tbl PARTITION p1 where k1 = 1;
    Query OK, 0 rows affected (0.04 sec)
    {'label':'delete_e7830c72-eb14-4cb9-bbb6-eebd4511d251', 'status':'VISIBLE', 'txnId':'4005'}
```
2. 提交成功，但未可见。
Doris 的事务提交分为两步：提交和发布版本，只有完成了发布版本步骤，结果才对用户是可见的。若已经提交成功了，那么就可以认为最终一定会发布成功，Doris 会尝试在提交完后等待发布一段时间，如果超时后即使发布版本还未完成也会优先返回给用户，提示用户提交已经完成。若如果 Delete 已经提交并执行，但是仍未发布版本和可见，将返回下列结果：
```
	mysql> delete from test_tbl PARTITION p1 where k1 = 1;
    Query OK, 0 rows affected (0.04 sec)
    {'label':'delete_e7830c72-eb14-4cb9-bbb6-eebd4511d251', 'status':'COMMITTED', 'txnId':'4005', 'err':'delete job is committed but may be taking effect later' }
```
结果会同时返回一个 json 字符串：`affected rows`，表示此次删除影响的行，由于 Doris 的删除目前是逻辑删除，因此对于这个值是恒为0。
`label`为自动生成的 label，是该导入作业的标识。每个导入作业，都有一个在单 database 内部唯一的 Label。
`status`表示数据删除是否可见，如果可见，显示`VISIBLE`，如果不可见,显示`COMMITTED`。
`txnId`为这个 Delete job 对应的事务 id。
`err`字段会显示一些本次删除的详细信息。
	
3. 提交失败，事务取消。
如果 Delete 语句没有提交成功，将会被 Doris 自动中止，返回下列结果：
```
	mysql> delete from test_tbl partition p1 where k1 > 80;
    ERROR 1064 (HY000): errCode = 2, detailMessage = {错误原因}
```
示例：
例如说一个超时的删除，将会返回 timeout 时间和未完成的`(tablet=replica)`。
```
	mysql> delete from test_tbl partition p1 where k1 > 80;
    ERROR 1064 (HY000): errCode = 2, detailMessage = failed to delete replicas from job: 4005, Unfinished replicas:10000=60000, 10001=60000, 10002=60000
```
**综上，对于Delete操作返回结果的正确处理逻辑为：**
	1. 如果返回结果为 `ERROR 1064 (HY000)`，则表示删除失败。
	2. 如果返回结果为 `Query OK`，则表示删除执行成功。
	3. 如果 `status` 为 `COMMITTED`，表示数据仍不可见，用户可以稍等一段时间再用 `show delete` 命令查看结果。
	4. 如果 `status` 为 `VISIBLE`，表示数据删除成功。

## 可配置项

### FE配置
**TIMEOUT 配置**
总体来说，Doris 的删除作业的超时时间限制在30秒到5分钟时间内，具体时间可通过下面配置项调整。
- `tablet_delete_timeout_second`
delete 自身的超时时间是可受指定分区下 tablet 的数量弹性改变的，此项配置为平均一个 tablet 所贡献的 timeout 时间，默认值为2。
假设此次删除所指定分区下有5个 tablet，那么可提供给 delete 的 timeout 时间为10秒，由于低于最低超时时间30秒，因此最终超时时间为30秒。
   
- `load_straggler_wait_second`
如果用户预估的数据量确实比较大，使得5分钟的上限不足时，用户可以通过此项调整 timeout 上限，默认值为300。
  
**TIMEOUT 的具体计算规则为(秒)**
`TIMEOUT = MIN(load_straggler_wait_second, MAX(30, tablet_delete_timeout_second * tablet_num))`
  
- `query_timeout`
因为 delete 本身是一个 SQL 命令，因此删除语句也会受 session 限制，timeout 还受 Session 中的 `query_timeout` 值影响，可以通过 `SET query_timeout = xxx` 来增加超时时间，单位是秒。
  
**IN 谓词配置**
- `max_allowed_in_element_num_of_delete`
如果用户在使用 in 谓词时需要占用的元素比较多，用户可以通过此项调整允许携带的元素上限，默认值为1024。
   
## 查看历史记录
1. 用户可以通过 show delete 语句查看历史上已执行完成的删除记录。
语法：
```
	SHOW DELETE [FROM db_name]
```
示例：
```
	mysql> show delete from test_db;
	+-----------+---------------+---------------------+-----------------+----------+
	| TableName | PartitionName | CreateTime          | DeleteCondition | State    |
	+-----------+---------------+---------------------+-----------------+----------+
	| empty_tbl | p3            | 2020-04-15 23:09:35 | k1 EQ "1"       | FINISHED |
	| test_tbl  | p4            | 2020-04-15 23:09:53 | k1 GT "80"      | FINISHED |
	+-----------+---------------+---------------------+-----------------+----------+
	2 rows in set (0.00 sec)
```
	
