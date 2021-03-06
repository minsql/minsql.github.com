---
title: performace_schema 를 이용하여 최근에 실행된 쿼리 확인하기
author: min_cho
created: 2016/02/01 18:49:30
modified:
layout: post
tags: mysql mysql_tips
image:
  feature: mysql.png
categories: MySQL
toc: true
toc_label: "My Table of Contents"
toc_icon: "cog"
---


# performace_schema 를 이용하여 최근에 실행된 쿼리 확인하기

show processlist 와 show engine processlist 를 확인하면, 어떤 쿼리인지는 알 수 없으나 트랜잭션을 잡고 있는것을 확인할 수 있습니다. commit 되지 않은 트랜잭션이 존재한다면, MySQL InnoDB는 old image를 읽어오기 위해 시간이 지날수록 느려지고 다른 쿼리또한 lock 으로 인해 대기될 수 있습니다. 해당 쿼리를 tracking 하기 위해서는 기본적으로 performace_schema 가 ON (1) 되어 있어야 합니다. MySQL 5.6.6 이상에서는 은 기본적으로 해당값이 ON 되어 있으므로, 추가적인 설정이 필요없습니다. <http://dev.mysql.com/doc/refman/5.6/en/performance-schema-system-variables.html#sysvar_performance_schema>


        mysql> show global variables like 'performance_schema';
        +--------------------+-------+
        | Variable_name      | Value |
        +--------------------+-------+
        | performance_schema | ON    |
        +--------------------+-------+
        1 row in set (0.00 sec)


먼저 예제를 하나 만들도록 하겠습니다. session 1 에서 테이블을 만들고, 해당 테이블에 값을 넣습니다.


    Session 1>
        mysql> create table p ( a int);
        Query OK, 0 rows affected (0.03 sec)

        mysql> begin;
        Query OK, 0 rows affected (0.00 sec)

        mysql> insert into p values (1);
        Query OK, 1 row affected (0.00 sec)


다른 세션에서 show engine innodb status, show processlist;, information_schema.INNODB_TRX 를 조회해 보겠습니다.


    Session 2>
        mysql> show engine innodb statusG
        ------------
        TRANSACTIONS
        ------------
        Trx id counter 632685
        Purge done for trx's n:o < 632684 undo n:o < 0 state: running but idle
        History list length 443
        LIST OF TRANSACTIONS FOR EACH SESSION:
        ---TRANSACTION 0, not started
        MySQL thread id 53587, OS thread handle 0x7f6f0c0a2700, query id 498268 localhost root init
        show engine innodb status
        ---TRANSACTION 0, not started
        MySQL thread id 15918, OS thread handle 0x7f6eec0ed700, query id 498176 localhost gu cleaning up
        ---TRANSACTION 632684, ACTIVE 32 sec
    /* 해당 transaction 이 Active 상태로 32초동안 active 상태지만, 어떤 쿼리인지는 알 수 없습니다. 단순히 test.p 테이블에 실행된 쿼리로만 짐작이 됩니다.
    현재는 아무런 쿼리도 돌고 있지 않기 때문입니다. 물론 undo log entires 를 통해 DML이 실행되었을것이라고 짐작할 수 있습니다.
    undo log entries 가 없는 경우도 있는데, 이는 begin; select .... 구문을 실행했기 때문입니다.
    commit 이나 rollback 이 오랬동안 이루어지지 않았다면, 이또한 성능정하를 일이키는 요소입니다.
    **/
        1 lock struct(s), heap size 360, 0 row lock(s), undo log entries 1
        sMySQL thread id 56604, OS thread handle 0x7f6f0c061700, query id 497985 localhost root cleaning up
        TABLE LOCK table `test`.`p` trx id 632684 lock mode IX



        mysql> show processlist;
        +-------+------+-----------+-------+---------+------+-------+------------------+
        | Id    | User | Host      | db    | Command | Time | State | Info             |
        +-------+------+-----------+-------+---------+------+-------+------------------+
        | 15918 | gu   | localhost | mysql | Sleep   |   24 |       | NULL             |
        | 53587 | root | localhost | NULL  | Query   |    0 | init  | show processlist |
        | 56604 | root | localhost | test  | Sleep   |   42 |       | NULL             |
        +-------+------+-----------+-------+---------+------+-------+------------------+
        5 rows in set (0.00 sec)

    /* show processlist; 에서는 현재 해당 thread가 아무런 작업을 하고 있지 않기 때문에 Sleep 상태만을 보여줍니다.**/

        mysql> select * from information_schema.INNODB_TRXG
        *************************** 1. row ***************************
                    trx_id: 632684
                 trx_state: RUNNING
                   trx_started: 2016-01-26 04:22:10
             trx_requested_lock_id: NULL
              trx_wait_started: NULL
                trx_weight: 2
               trx_mysql_thread_id: 56604
                 trx_query: NULL
               trx_operation_state: NULL
             trx_tables_in_use: 0
             trx_tables_locked: 0
              trx_lock_structs: 1
             trx_lock_memory_bytes: 360
               trx_rows_locked: 0
             trx_rows_modified: 1
           trx_concurrency_tickets: 0
               trx_isolation_level: REPEATABLE READ
             trx_unique_checks: 1
            trx_foreign_key_checks: 1
        trx_last_foreign_key_error: NULL
         trx_adaptive_hash_latched: 0
         trx_adaptive_hash_timeout: 10000
              trx_is_read_only: 0
        trx_autocommit_non_locking: 0
        1 row in set (0.00 sec)

    /* information_schema.INNODB_TRX 에서는 현재 해당 thread가 아무런 작업을 하고 있지 않기 때문에 trx_rows_modified 를 통해 어떤 DML 이 실행되었고, 1개의 record를 수정한채 commit 하지 않았다는것만을 알 수 있습니다. **/


performance_schema.events_statements_current 의 데이터를 조회해 보겠습니다. 해당 테이블은 마지막에 해당 thread에서 실행된 쿼리를 저장하고 있습니다. 해당 테이블의 값을 조회하기 위해서는 performance_schema.threads 와 조인이 필요합니다. processlist 에서 보여주는 mysql 이 만든 thread와 performance_schema 에서 만든 thread 값이 다르기 때문입니다. (performance_schema.threads 는 background thread의 값도 가지고 있기 때문입니다.)


    Session 2>
        mysql> SELECT esc.THREAD_ID, t.processlist_id, esc.SQL_TEXT FROM performance_schema.events_statements_current esc JOIN performance_schema.threads t ON t.thread_id = esc.thread_id WHERE t.processlist_id = 56604;
        +-----------+----------------+--------------------------+
        | THREAD_ID | processlist_id | SQL_TEXT                 |
        +-----------+----------------+--------------------------+
        |     56624 |          56604 | insert into p values (1) |
        +-----------+----------------+--------------------------+

    /* process_id 가 56604 의 thread 가 실행한 마지막 쿼리가 나타납니다. 56604 값은 show engine innodb statusG 의 transaction 값 중 thread id 56604 값을 통해 알아낼 수 있습니다. **/


위는 default 로 동작되는 방법입니다. 그렇다면, 지나간 쿼리에 대해서는 어떻게 로그인을 하는지 확인해보겠습니다. \- https://dev.mysql.com/doc/refman/5.6/en/performance-schema-statement-tables.html 먼저 performance_schema.setup_consumers 테이블의 events_statements_history, events_statements_history_long 의 값을 ON 시켜주어야 합니다. (default 로 events_statements_current 는 ON이 되어 있는것을 확인할 수 있습니다.)


        mysql> select * from performance_schema.setup_consumers where name like 'events_statements_%';
        +--------------------------------+---------+
        | NAME                           | ENABLED |
        +--------------------------------+---------+
        | events_statements_current      | YES     |
        | events_statements_history      | NO      |
        | events_statements_history_long | NO      |
        +--------------------------------+---------+
        3 rows in set (0.00 sec)


        mysql> UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE name like 'events_statements_%';
        Query OK, 2 row affected (0.06 sec)
        Rows matched: 3  Changed: 2  Warnings: 0


다시 session 1에서 쿼리를 실행해 보겠습니다.

```sql
Session 1>
mysql> select * from p;
+------+
| a    |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

mysql> select * from p where a=1;
+------+
| a    |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

mysql> select * from p where a=0;
Empty set (0.00 sec)
```

다른 세션에서 events_statements_history_long 의 값을 조회하여 해당세션이 실행한 쿼리들을 확인할 수 있습니다.

```sql
SELECT
    esh.THREAD_ID, t.processlist_id, esh.SQL_TEXT
FROM
    performance_schema.events_statements_history_long esh
        JOIN
    performance_schema.threads t ON t.thread_id = esh.thread_id
WHERE
    t.processlist_id = 56604 order by TIMER_END ASC;

+-----------+----------------+---------------------------+
| THREAD_ID | processlist_id | SQL_TEXT                  |
+-----------+----------------+---------------------------+
|     56624 |          56604 | select * from p           |
|     56624 |          56604 | select * from p where a=1 |
|     56624 |          56604 | select * from p where a=0 |
+-----------+----------------+---------------------------+
3 rows in set (0.00 sec)
```

위와 같이 시간순으로 조회될 수 있습니다. 전체 쿼리를 저장할 수 있는 갯수는 아래의 설정을 통해 가능합니다. (performance_schema_events_statements_history_size, performance_schema_events_statements_history_long_size)
5.6.6 이상부터는 default 값으로 MySQL 의 현재상태를 고려하여 값이 자동으로 늘어나거나 줄어들게 됩니다.
물론 아래의 값으로 고정시킬 수도 있습니다.

https://dev.mysql.com/doc/refman/5.6/en/performance-schema-system-variables.html#sysvar_performance_schema_events_statements_history_size
https://dev.mysql.com/doc/refman/5.6/en/performance-schema-system-variables.html#sysvar_performance_schema_events_statements_history_long_size
