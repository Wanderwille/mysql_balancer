# Настройка HaProxy для балансировки MySQL Bitrix

### Для теста использовал стенды:
- BitrixVM машина с IP адресом 192.168.90.93 - Master (На ней же установлен HaProxy)
- BitrixVM машина с IP адресом 192.168.90.94 - Slave

1. Для начала работы нужно установить HaProxy 
```bash
yum install haproxy 
```
Команда используется для RHEL систем (В Deb подобных системах) ```apt install haproxy```

2. Скопировать конфигурационный файл haproxy.cfg
```bash
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy_backup.cfg
```
3. Начинаем настройку HaProxy
```bash
global
    log /dev/log local0 debug
    maxconn 4096
    daemon

defaults
    log global
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend mysql_front
    bind *:3307
    mode tcp
    default_backend mysql_backend

backend mysql_backend
    balance roundrobin
    mode tcp
    option mysql-check user haproxy_check
    server master 192.168.90.93:3306 check inter 2s rise 2 fall 3 on-marked-down shutdown-sessions
    server slave  192.168.90.94:3306 check inter 2s rise 2 fall 3 backup

listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats auth admin:password
```

Это настройки для балансировки нагрузки HaProxy

4. Теперь нужно настроить репликацию MySQL 

- Для начала на Master машине редактируем конфигурационный файл my.cnf, добавляя туда следующие строки
```bash
[mysqld]
bind-address = 0.0.0.0
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog-format=ROW
binlog-do-db=db_name (Имя БД)
```

Перезагружаем БД на Master ```systemctl restart mysql```

5. Теперь нужно настроить Slave
```bash
[mysqld]
bind-address = 0.0.0.0
server-id = 2
relay-log=relay-bin
log-bin=/var/log/mysql/mysql-bin.log
binlog-format=ROW
```

Так же перезагрузить БД ```systemctl restart mysql```

6. Теперь приступим к созданию пользователя для haproxy
```bash
 CREATE USER 'haproxy_check'@'192.168.90.%' IDENTIFIED BY ''; (Пароль не задается, так как HaProxy не авторизуется с паролем)
 GRANT ALL PRIVILEGES ON *.* TO 'haproxy_check'@'192.168.90.%';
 FLUSH PRIVILEGES;
```

Для того что бы повысить безопасность, можно разрешить доступ этому пользователю только с одного хоста:
```bash
CREATE USER 'haproxy_check'@'192.168.90.93' IDENTIFIED BY ''; (Пароль не задается, так как HaProxy не авторизуется с паролем)
CREATE USER 'haproxy_check'@'192.168.90.94' IDENTIFIED BY '';
GRANT ALL PRIVILEGES ON *.* TO 'haproxy_check'@'192.168.90.93';
GRANT ALL PRIVILEGES ON *.* TO 'haproxy_check'@'192.168.90.94';
```

## Делается и на Master и на Slave

7. Далее меняется метод аунтификации на:
```bash
ALTER USER 'haproxy_check'@'192.168.90.%' IDENTIFIED WITH mysql_native_password BY '';
FLUSH PRIVILEGES;
```
Либо:
```bash
ALTER USER 'haproxy_check'@'192.168.90.93' IDENTIFIED WITH mysql_native_password BY '';
ALTER USER 'haproxy_check'@'192.168.90.94' IDENTIFIED WITH mysql_native_password BY '';
FLUSH PRIVILEGES;
```
Так же на Master нужно создать пользователя replica
```Bash
CREATE USER 'replica'@'192.168.90.94' IDENTIFIED BY '';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'192.168.90.94';
FLUSH PRIVILEGES;
```

8. Проверяем:
```bash
SELECT user, host, plugin FROM mysql.user WHERE user = 'haproxy_check';
```

```sql
+---------------+---------------+-----------------------+
| user          | host          | plugin                |
+---------------+---------------+-----------------------+
| haproxy_check | 192.168.90.%  | mysql_native_password |
+---------------+---------------+-----------------------+
```

9. Бывают проблемы, что он просто всегда запрашивает пароль, либо возникает такая ошибка:

```bash
[root@Bitrix-master haproxy]# mysql -u haproxy_check -h 192.168.90.94
ERROR 1045 (28000): Access denied for user 'haproxy_check'@'192.168.90.93' (using password: YES)
```
Как исправить:
В каталоге /root, есть файл .my.cnf(если нет, можно создать, либо добавить данные строки в файл /etc/my.cnf)
```bash
[client]
user=haproxy_check
password=''
socket=/var/lib/mysqld/mysqld.sock
```

И попробовать соединение

Еще нужно настроить firewall (На CentOS 9 это firewalld)
```bash
firewall-cmd --add-port=3306/tcp --permanent
firewall-cmd --add-port=3307/tcp --permanent
firewall-cmd --add-port=8404/tcp --permanent
firewall-cmd --reload
```
Порт 8404 используется для статистики на HaProxy

10. Теперь идем на Master и проверяем что все работает:
```sql
SHOW MASTER STATUS;
```

```bash
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |  1498450 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
Тут надо запомнить 2 параметра: Position и File

11. Теперь настраиваем Slave:

```bash
HANGE MASTER TO
MASTER_HOST='192.168.90.93',
MASTER_USER='haproxy_check',
MASTER_PASSWORD='',
MASTER_LOG_FILE='mysql-bin.000001',  (указываем File из Master)
MASTER_LOG_POS=12345;  (указываем Position из Master)
```
Далее 
```bash
START SLAVE;
```

Перед стартом нужно восстановить ту базу, которую вы хотите реплицировать

Далее:
```bash
SHOW SLAVE STATUS\G;
```
Если все четко, то будет выглядить так:

```bash
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 192.168.90.93
                  Master_User: replica
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000014
          Read_Master_Log_Pos: 74481
               Relay_Log_File: relay-bin.000002
                Relay_Log_Pos: 27380
        Relay_Master_Log_File: mysql-bin.000014
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 74481
              Relay_Log_Space: 27584
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: 886518b1-f8ee-11ef-b643-bc24114a0919
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
1 row in set, 1 warning (0.00 sec)
```
Обращаем внимание на два параметра:
```bash
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

# Известные проблемы:

1. Иногда реплика некорректно работает
2. При создании таблиц на мастере не репликация слетает и нужно настраивать все заново
3. При изменении в таблицах реплика также слетает

## На этом настройку репликации можно закончить

Теперь нужно проверить как работает HaProxy

В Bitrix файл .settings.php изменяем подключение к БД

```php
array (
        'className' => '\\Bitrix\\Main\\DB\\MysqliConnection',
        'host' => '192.168.90.93:3307',
        'database' => 'dbsentry',
        'login'    => 'haproxy_check',
        'password' => '',
        'options' => 2,
      ),
```
Порт ставим такой, как в конфиге HaProxy 
```bash
frontend mysql_front
    bind *:3307
    mode tcp
    default_backend mysql_backend
```
И IP адрес HaProxy
Далее он будет балансировать трафик и если Master выйдет из строя, его работу на себя возьмет Slave

Можно попробовать с Slave подключиться:
```bash
mysql -u haproxy_check -h 192.168.90.93 -P 3307
```
И проверить к какой БД мы подключились:
```bash
mysql> SHOW VARIABLES LIKE 'hostname';
+---------------+--------------+
| Variable_name | Value        |
+---------------+--------------+
| hostname      | Bitrix-Slave |
+---------------+--------------+
1 row in set (0.00 sec)
```
Перед нормальной проверкой Master БД лучше остановить!
