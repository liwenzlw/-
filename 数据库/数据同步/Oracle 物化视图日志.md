
原文地址：https://blog.51cto.com/hbxztc/1955101

## 一、物化视图日志是什么

oracle 的物化视图的快速刷新要求必须建立物化视图日志，通过物化视图日志可以实现增量刷新功能。

官方文档给出的对物化视图日志的释义：

物化视图日志在建立时有多种选项：可以指定为ROWID、PRIMARY KEY和OBJECTID几种类型，同时还可以指定SEQUENCE或明确指定列名。不过上面这些情况产生的物化视图日志的结构都不相同。这里要注意，当发生DML 操作时，内部的触发器会把变化记录到物化视图日志里，也就是说物化视图不支持DDL的同步，所以在物化视图的编写过程中不可使用select * from 的形式，因为这样当基表发生变化时，物化视图就会失效。

物化视图日志的名称为MLOG$_后面跟基表的名称，如果表名的长度超过20位，则只取前20位，当截短后出现名称重复时，Oracle会自动在物化视图日志名称后面加上数字作为序号。

虽然物化视图格式会有不同，但任何物化视图都会包括如下列：

下面是一个primarykey的物化视图日志：
```shell
zx@ORA11G>desc mlog$_employees
 Name			 Null?	  Type
 ----------------------- -------- ----------------
 EMPLOYEE_ID			  NUMBER(6)
 SNAPTIME$$			  DATE
 DMLTYPE$$			  VARCHAR2(1)
 OLD_NEW$$			  VARCHAR2(1)
 CHANGE_VECTOR$$		  RAW(255)
 XID$$				  NUMBER
```

相关解释如下：

* SNAPTIME$$：用于表示刷新时间。

* DMLTYPE$$：用于表示DML操作类型，I表示INSERT，D表示DELETE，U表示UPDATE。

* OLD_NEW$$：用于表示这个值是新值还是旧值。N（EW）表示新值，O（LD）表示旧值，U表示UPDATE操作。

* CHANGE_VECTOR$$：表示修改矢量，用来表示被修改的是哪个或哪几个字段。

INSERT和DELETE操作都是记录集的，即INSERT和DELETE会影响整条记录。而UPDATE操作是字段集的，UPDATE操作可能会更新整条记录的所有字段，也可能只更新个别字段。

无论从性能上考虑还是从数据的一致性上考虑，物化视图刷新时都应该是基于字段集。Oracle就是通过CHANGE_VECTOR$$列来记录每条记录发生变化的字段包括哪些。

基于主键、ROWID和OBJECT ID的物化视图日志在CHANGE_VECTOR$$上略有不同，但是总体设计的思路是一致的。

CHANGE_VECTOR$$列是RAW类型，其实Oracle采用的方式就是用每个BIT位去映射一个列。

比如：第一列被更新设置为02，即00000010。第二列设置为04，即00000100，第三列设置为08，即00001000。当第一列和第二列同时被更新，则设置为06，00000110。如果三列都被更新，设置为0E，00001110。

依此类推，第4列被更新时为10，第5列20，第6列40，第7列80，第8列0001。当第1000列被更新时，CHANGE_VECTOR$$的长度为1000/4+2为252。

除了可以表示UPDATE的字段，还可以表示INSERT和DELETE。DELETE操作CHANGE_VECTOR列为全0，具体个数由基表的列数决定。INSERT操作的最低位为FE如果基表列数较多，而存在高位的话，所有的高位都为FF。如果INSERT操作是前面讨论过的由UPDATE操作更新了主键造成的，则这个INSERT操作对应的CHANGEVECTOR列为全FF。

如果WITH后面跟了ROWID，则物化视图日志中会包含：M_ROW$$：用来存储发生变化的记录的ROWID。

如果WITH后面跟了PRIMARY KEY，则物化视图日志中会包含主键列。

如果WITH后面跟了OBJECT ID，则物化视图日志中会包含：SYS_NC_OID$：用来记录每个变化对象的对象ID。

如果WITH后面跟了SEQUENCE，则物化视图日子中会包含：SEQUENCE$$：给每个操作一个SEQUENCE号，从而保证刷新时按照顺序进行刷新。

如果WITH后面跟了一个或多个COLUMN名称，则物化视图日志中会包含这些列。

## 二、根据物化视图日志来快速刷新数据过程

### 2.1 创建测试环境

```shell
zx@ORA11G>create table t (id number,name varchar2(10),address varchar2(10));

Table created.

zx@ORA11G>create materialized view log on t with rowid,sequence (id,name) including new values;

Materialized view log created.

zx@ORA11G>desc mlog$_t
 Name			 Null?	  Type
 ----------------------- -------- ----------------
 ID				  NUMBER
 NAME				  VARCHAR2(10)
 M_ROW$$			  VARCHAR2(255)
 SEQUENCE$$			  NUMBER
 SNAPTIME$$			  DATE
 DMLTYPE$$			  VARCHAR2(1)
 OLD_NEW$$			  VARCHAR2(1)
 CHANGE_VECTOR$$		  RAW(255)
 XID$$				  NUMBER

```

* ID和NAME是建立物化视图日志时指定的基表中的列，它们记录每次DML操作对应的ID和NAME的值。

* M_ROW$$：保存基表的ROWID信息，根据M_ROW$$中的信息可以定位到发生DML操作的记录。

* SEQUENCE$$：根据DML操作发生的顺序记录序列的编号，当刷新时，根据SEQUENCE中的顺序就可以和基表中的执行顺序保持一致。

* SNAPTIME$$：列记录了刷新操作的时间。

* DMLTYPE$$：的记录值I、U和D，表示操作是INSERT、UPDATE还是DELETE。

* OLD_NEW$$：表示物化视图日志中保存的信息是DML操作之前的值（旧值）还是DML操作之后的值（新值）。除了O和N这两种类型外，对于UPDATE操作，还可能表示为U。

* CHANGE_VECTOR$$：记录DML操作发生在那个或那几个字段上

当刷新物化视图时，只需要根据SEQUENCE列给出的顺序，通过M_ROW$$定位到基表的记录，如果是UPDATE操作，通过CHANGE_VECTOR$$定位到字段，然后根据基表中的数据重复执行DML操作。

如果物化视图日志只针对一个物化视图，那么刷新过程就是这么简单，还需要做的不过是在刷新之后将物化视图日志清除掉。但是，Oracle的物化视图日志是可以同时支持多个物化视图的快速刷新的，也就是说，物化视图在刷新时还必须判断哪些物化视图日志记录是当前物化视图刷新需要的，哪些是不需要的。而且，物化视图还必须确定，在刷新物化视图后，物化视图日志中哪些记录是需要清除的，哪些是不需要清除的。

回顾一下物化视图日志的结构，发现只剩下一个SHAPTIME$$列，那么Oracle如何仅通过这一列就完成了对多个物化视图的支持呢？

### 2.2 下面建立一个小例子，通过例子来进行说明：

使用上文中建立的表和物化视图日志，下面对这个表建立三个快速刷新的物化视图，并对t表执行DML操作：

```shell
zx@ORA11G>create materialized view mv_t_id 
  2  refresh fast
  3  as select id,count(*) 
  4  from t
  5  group by id;

Materialized view created.

zx@ORA11G>create materialized view mv_t_name 
  2  refresh fast
  3  as select name,count(*) 
  4  from t
  5  group by name;

Materialized view created.

zx@ORA11G>create materialized view mv_t_both 
  2  refresh fast
  3  as select id,name,count(*) 
  4  from t
  5  group by id,name;

Materialized view created.

zx@ORA11G>insert into t values (1, 'zx', 'hb');

1 row created.

zx@ORA11G>insert into t values (2, 'wl', 'sd');

1 row created.

insert into t values (3, 'yc', 'bj');

1 row created.

zx@ORA11G>update t set address = 'bj_cp' where id = 3;

1 row updated.

zx@ORA11G>delete from t where id = 2;

1 row deleted.

zx@ORA11G>commit;

Commit complete.
```

查询物化视图日志，可以查看每次dml操作都有对应的日志

```shell
zx@ORA11G>col M_ROW$$ for a30
zx@ORA11G>col change_vector$$ for a30
zx@ORA11G>set num 20
zx@ORA11G>select * from mlog$_t;

                  ID NAME       M_ROW$$                                  SEQUENCE$$ SNAPTIME$$        D O CHANGE_VECTOR$$                               XID$$
-------------------- ---------- ------------------------------ -------------------- ----------------- - - ------------------------------ --------------------
                   1 zx         AAAVs6AAEAAAAJVAAA                                8 40000101 00:00:00 I N FE                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                                9 40000101 00:00:00 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               10 40000101 00:00:00 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               11 40000101 00:00:00 U U 08                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               12 40000101 00:00:00 U N 08                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                               13 40000101 00:00:00 D O 00                                 2814882911093459

6 rows selected.
```

当发生了DML操作后，物化视图日志中的SNAPTIME$$列保持的值是40000101 00:00:00。这个值表示这条记录还没有被任何物化视图刷新过。第一个刷新这些记录的物化视图会将SNAPTIME$$的值更新为物化视图当前的刷新时间。

刷新一个物化视图，并再次查看物化视图日志：

```shell
zx@ORA11G>exec dbms_mview.refresh('MV_T_ID');

PL/SQL procedure successfully completed.

zx@ORA11G>select * from mlog$_t;

                  ID NAME       M_ROW$$                                  SEQUENCE$$ SNAPTIME$$        D O CHANGE_VECTOR$$                               XID$$
-------------------- ---------- ------------------------------ -------------------- ----------------- - - ------------------------------ --------------------
                   1 zx         AAAVs6AAEAAAAJVAAA                                8 20170809 15:58:30 I N FE                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                                9 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               10 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               11 20170809 15:58:30 U U 08                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               12 20170809 15:58:30 U N 08                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                               13 20170809 15:58:30 D O 00                                 2814882911093459

6 rows selected.
```

Oracle根据数据字典中的信息可以知道表T上建立了三个物化视图，因此，MV_T_ID刷新完之后，不会删除物化视图记录。但SNAPTIME$$列对应的时候修改为MV_T_ID物化视图刷新时的时间

Oracle的数据字典中还保存着每个物化视图上次刷新的时间和当前的刷新状态。

```shell
zx@ORA11G>select name,master,last_refresh from user_mview_refresh_times;

NAME                           MASTER                         LAST_REFRESH
------------------------------ ------------------------------ -----------------
MV_T_BOTH                      T                              20170809 15:45:10
MV_T_ID                        T                              20170809 15:58:30
MV_T_NAME                      T                              20170809 15:45:05

zx@ORA11G>select mview_name,last_refresh_date, staleness from user_mviews;

MVIEW_NAME                     LAST_REFRESH_DATE STALENESS
------------------------------ ----------------- -------------------
MV_T_BOTH                      20170809 15:45:10 NEEDS_COMPILE
MV_T_ID                        20170809 15:58:30 FRESH
MV_T_NAME                      20170809 15:45:05 NEEDS_COMPILE
```

这些视图中记录了每个物化视图上次执行刷新操作的时间，并且给出每个物化视图中的数据是否是和基表同步的。由于MV_T_ID刚刚进行了刷新，因此状态是FRESH，而另外两个由于在刷新（建立）之后，基表又进行了DML操作，因此状态为NEEDS_COMPILE。如果这时对基表进行DML操作，则MV_T_ID的状态也会变为NEEDS_COMPILE。

```shell
zx@ORA11G>insert into t values (4, 'zf', 'sd');

1 row created.

zx@ORA11G>commit;

Commit complete.

zx@ORA11G>select mview_name,last_refresh_date, staleness from user_mviews;

MVIEW_NAME                     LAST_REFRESH_DATE STALENESS
------------------------------ ----------------- -------------------
MV_T_BOTH                      20170809 15:45:10 NEEDS_COMPILE
MV_T_ID                        20170809 15:58:30 NEEDS_COMPILE
MV_T_NAME                      20170809 15:45:05 NEEDS_COMPILE

zx@ORA11G>select * from mlog$_t;

                  ID NAME       M_ROW$$                                  SEQUENCE$$ SNAPTIME$$        D O CHANGE_VECTOR$$                               XID$$
-------------------- ---------- ------------------------------ -------------------- ----------------- - - ------------------------------ --------------------
                   1 zx         AAAVs6AAEAAAAJVAAA                                8 20170809 15:58:30 I N FE                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                                9 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               10 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               11 20170809 15:58:30 U U 08                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               12 20170809 15:58:30 U N 08                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                               13 20170809 15:58:30 D O 00                                 2814882911093459
                   4 zf         AAAVs6AAEAAAAJVAAD                               14 40000101 00:00:00 I N FE                                  844463584838593
```

下面刷新物化视图MV_T_NAME，刷新操作的判断依据是，只刷新SNAPTIME$$列大于当前物化视图的LAST_REFRESH_DATE的记录，由于物化视图日志中所有记录的SNAPTIME$$的值都比物化视图MV_T_ID_NAME上次刷新的时间点大，因此会刷新所有记录。对于SNAPTIME$$列的值是40000101 00:00:00的记录，物化视图会把SNAPTIME$$列的值更新为当前刷新时间，对于那些已经被更新过的SNAPTIME$$列，则保持原值。

```shell
zx@ORA11G>exec dbms_mview.refresh('MV_T_NAME');

PL/SQL procedure successfully completed.

zx@ORA11G>select mview_name,last_refresh_date, staleness from user_mviews;

MVIEW_NAME                     LAST_REFRESH_DATE STALENESS
------------------------------ ----------------- -------------------
MV_T_BOTH                      20170809 15:45:10 NEEDS_COMPILE
MV_T_ID                        20170809 15:58:30 NEEDS_COMPILE
MV_T_NAME                      20170809 16:16:01 FRESH

zx@ORA11G>select * from mlog$_t;

                  ID NAME       M_ROW$$                                  SEQUENCE$$ SNAPTIME$$        D O CHANGE_VECTOR$$                               XID$$
-------------------- ---------- ------------------------------ -------------------- ----------------- - - ------------------------------ --------------------
                   1 zx         AAAVs6AAEAAAAJVAAA                                8 20170809 15:58:30 I N FE                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                                9 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               10 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               11 20170809 15:58:30 U U 08                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               12 20170809 15:58:30 U N 08                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                               13 20170809 15:58:30 D O 00                                 2814882911093459
                   4 zf         AAAVs6AAEAAAAJVAAD                               14 20170809 16:16:01 I N FE                                  844463584838593

7 rows selected.
```

如果这时再次刷新物化视图MV_T_ID，则只有ID=4的这条记录的SNAPTIME$$的时间点大于MV_T_ID上次刷新的时间点，因此，只刷新这一条记录，且不会改变SNAPTIME$$的值

```shell
zx@ORA11G>exec dbms_mview.refresh('MV_T_ID');

PL/SQL procedure successfully completed.

zx@ORA11G>select mview_name,last_refresh_date, staleness from user_mviews;

MVIEW_NAME                     LAST_REFRESH_DATE STALENESS
------------------------------ ----------------- -------------------
MV_T_BOTH                      20170809 15:45:10 NEEDS_COMPILE
MV_T_ID                        20170809 16:17:43 FRESH
MV_T_NAME                      20170809 16:16:01 FRESH

zx@ORA11G>select * from mlog$_t;

                  ID NAME       M_ROW$$                                  SEQUENCE$$ SNAPTIME$$        D O CHANGE_VECTOR$$                               XID$$
-------------------- ---------- ------------------------------ -------------------- ----------------- - - ------------------------------ --------------------
                   1 zx         AAAVs6AAEAAAAJVAAA                                8 20170809 15:58:30 I N FE                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                                9 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               10 20170809 15:58:30 I N FE                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               11 20170809 15:58:30 U U 08                                 2814882911093459
                   3 yc         AAAVs6AAEAAAAJVAAC                               12 20170809 15:58:30 U N 08                                 2814882911093459
                   2 wl         AAAVs6AAEAAAAJVAAB                               13 20170809 15:58:30 D O 00                                 2814882911093459
                   4 zf         AAAVs6AAEAAAAJVAAD                               14 20170809 16:16:01 I N FE                                  844463584838593
```

到目前为止，还没有看到过物化视图日志的清除，其实每次进行完刷新，物化视图日志都会试图删除没有用的物化视图日志记录。物化视图日志记录的删除条件是删除那些SNAPTIME$$列小于等于基表所有物化视图的上次刷新时间。在上面的例子中，由于MV_T_BOTH一直没有刷新，因此它的LAST_REFRESH_DATE比物化视图日志中所有记录的值都小，因此，一直没有发生物化视图日志记录清除的现象。

```shell
zx@ORA11G>insert into t values (5, 'zq', 'jx');

1 row created.

zx@ORA11G>commit;

Commit complete.

zx@ORA11G>exec dbms_mview.refresh('MV_T_BOTH');

PL/SQL procedure successfully completed.

zx@ORA11G>select mview_name,last_refresh_date, staleness from user_mviews;

MVIEW_NAME                     LAST_REFRESH_DATE STALENESS
------------------------------ ----------------- -------------------
MV_T_BOTH                      20170809 16:19:51 FRESH
MV_T_ID                        20170809 16:17:43 NEEDS_COMPILE
MV_T_NAME                      20170809 16:16:01 NEEDS_COMPILE

zx@ORA11G>select * from mlog$_t;

                  ID NAME       M_ROW$$                                  SEQUENCE$$ SNAPTIME$$        D O CHANGE_VECTOR$$                               XID$$
-------------------- ---------- ------------------------------ -------------------- ----------------- - - ------------------------------ --------------------
                   5 zq         AAAVs6AAEAAAAJVAAE                               15 20170809 16:19:51 I N FE                                 2251898597934032
```

物化视图MV_T_BOTH刷新了物化视图中的每条记录，更新了ID=5的记录的SNAPTIME$$时间，并清除了其它所有物化视图日志记录。

总结：

    物化视图在刷新时，会刷新所有SNAPTIME$$大于本物化视图上次刷新时间的记录，并将所有是40000101 00:00:00的记录更新为当前刷新时间。对于其他大于上次刷新时间的记录，只刷新不更改。这样，当刷新执行完以后，数据字典中记录当前物化视图的上次刷新时间为当前时刻，这保证了物化视图日志中目前所有的记录都小于或等于刷新时间。因此，每个物化视图只要刷新大于上次刷新时间的记录，且保证每次刷新后，所有记录的时间都小于等于上次刷新时间，那么无论有多少个物化视图，就可以互不影响的使用同一个物化视图日志进行快速刷新了。当物化视图刷新完之后，会清除那些SNAPTIME$$列小于所有物化视图的上次刷新时间的记录，而这些记录已经被所有的物化视图都刷新过了，保存在物化视图日志中已经没有意义了。



参考：http://blog.csdn.net/tianlesoftware/article/details/7720580

http://docs.oracle.com/cd/E11882_01/server.112/e10706/repmview.htm#i30732

http://docs.oracle.com/cd/E11882_01/server.112/e41084/statements_6003.htm#SQLRF01303
