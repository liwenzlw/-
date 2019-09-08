
## 部署canal服务器
### 1. Windows环境下开启mysql的binlog功能，并配置binlog模式为row

修改mysql的my-default.ini文件

![1](./canalQuickStart.asserts/1.png)

```shell
[mysqld]
log-bin=mysql-bin #添加这一行就ok
binlog-format=ROW #选择row模式
server_id=1 #配置mysql replaction需要定义，不能和canal的slaveId重复
```

注意：针对阿里云 RDS for MySQL , 默认打开了 binlog , 并且账号默认具有 binlog dump 权限 , 不需要任何权限或者 binlog 设置,可以直接跳过这一步

### 2. 授权 MySQL canal 账号具有作为 MySQL slave 的权限, 如果已有账户可直接 grant

```sql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ;
FLUSH PRIVILEGES;
```

### 3. 下载 canal, 访问 release 页面 , 选择需要的包下载, 如以 1.0.17 版本为例