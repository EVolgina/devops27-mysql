# Задача 1
- Используя Docker, поднимите инстанс MySQL (версию 8). Данные БД сохраните в volume.
- Изучите бэкап БД и восстановитесь из него.
- Перейдите в управляющую консоль mysql внутри контейнера.
- Используя команду \h, получите список управляющих команд.
- Найдите команду для выдачи статуса БД и приведите в ответе из её вывода версию сервера БД.
- Подключитесь к восстановленной БД и получите список таблиц из этой БД.
- Приведите в ответе количество записей с price > 300.\
В следующих заданиях мы будем продолжать работу с этим контейнером.
### Ответ: 
Запускаем контейнер при помощи команды docker run и далее по схеме
```
sudo docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=admin -e MYSQL_DATABASE=test_db -v mysql_data:/var/lib/mysql
-p 3306:3306 mysql:8
curl -o test_dump.sql https://raw.githubusercontent.com/netology-code/virt-homeworks/virt-11/06-db-03-mysql/test_data/test_dump.sql
sudo docker cp test_dump.sql mysql:/var/tmp/test_dump.sql
Successfully copied 4.1kB to mysql:/var/tmp/test_dump.sql
vagrant@server1:~/mysql$ sudo docker exec -it c0eb718e2f4f bash
bash-4.4# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.33 MySQL Community Server - GPL
Copyright (c) 2000, 2023, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql> \h
```
![z1](https://github.com/EVolgina/devops27-mysql/blob/main/zd-1.PNG)
![sh](https://github.com/EVolgina/devops27-mysql/blob/main/status.PNG)
Подключаемся к базе данных,которую создали
```
mysql> USE test_db;
Database changed
mysql> source /var/tmp/test_dump.sql; - после данной команды идет восстановление из бэкапа
mysql> SHOW TABLES;
+-------------------+
| Tables_in_test_db |
+-------------------+
| orders            |
+-------------------+
1 row in set (0.01 sec)
mysql> select count(*) from orders where price > 300;
+----------+
| count(*) |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)
```
# Задача 2
- Создайте пользователя test в БД c паролем test-pass, используя:
- плагин авторизации mysql_native_password
- срок истечения пароля — 180 дней
- количество попыток авторизации — 3
- максимальное количество запросов в час — 100
- аттрибуты пользователя:
- Фамилия "Pretty"
- Имя "James".
- Предоставьте привелегии пользователю test на операции SELECT базы test_db.
- Используя таблицу INFORMATION_SCHEMA.USER_ATTRIBUTES, получите данные по пользователю test и приведите в ответе к задаче.
### Ответ:
```
mysql> CREATE USER 'test'@'localhost' -                создаем пользователя с данными из задачи
    ->     IDENTIFIED WITH mysql_native_password BY 'test-pass'
    ->     WITH MAX_CONNECTIONS_PER_HOUR 100
    ->     PASSWORD EXPIRE INTERVAL 180 DAY
    ->     FAILED_LOGIN_ATTEMPTS 3 PASSWORD_LOCK_TIME 2
    ->     ATTRIBUTE '{"first_name":"James", "last_name":"Pretty"}';
Query OK, 0 rows affected (0.03 sec)
mysql> GRANT SELECT ON test_db.* to 'test'@'localhost'; - предостаим привелегии пользователю
Query OK, 0 rows affected, 1 warning (0.02 sec)
mysql> SELECT * from INFORMATION_SCHEMA.USER_ATTRIBUTES where USER = 'test'; - получим данные по пользователю
+------+-----------+------------------------------------------------+
| USER | HOST      | ATTRIBUTE                                      |
+------+-----------+------------------------------------------------+
| test | localhost | {"last_name": "Pretty", "first_name": "James"} |
+------+-----------+------------------------------------------------+
1 row in set (0.00 sec)
```
# Задача 3
- Установите профилирование SET profiling = 1. Изучите вывод профилирования команд SHOW PROFILES;.
- Исследуйте, какой engine используется в таблице БД test_db и приведите в ответе.
- Измените engine и приведите время выполнения и запрос на изменения из профайлера в ответе:
### Ответ:
```
mysql> SET profiling = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> SELECT * FROM orders; - выполним запрос
mysql> SHOW PROFILES;
+----------+------------+----------------------+
| Query_ID | Duration   | Query                |
+----------+------------+----------------------+
|        1 | 0.00050425 | SELECT * FROM orders |
+----------+------------+----------------------+
1 row in set, 1 warning (0.00 sec)
```
- определяем механизм хранения, используемый в таблице базы данных test_db, запросом базу данных information_schema
```
mysql> SELECT table_name, engine       
    -> FROM information_schema.tables                               
    -> WHERE table_schema = 'test_db';
+------------+--------+
| TABLE_NAME | ENGINE |
+------------+--------+
| orders     | InnoDB |
+------------+--------+
1 row in set (0.00 sec)
```
- на InnoDB
```
mysql> SET profiling = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> ALTER TABLE orders ENGINE = InnoDB;
Query OK, 0 rows affected (0.18 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> SHOW PROFILES;
+----------+------------+-----------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                   |
+----------+------------+-----------------------------------------------------------------------------------------+
|        1 | 0.00050425 | SELECT * FROM orders                                                                    |
|        2 | 0.00179200 | SELECT table_name, engine
FROM information_schema.tables
WHERE table_schema = 'test_db' |
|        3 | 0.00020800 | SET profiling = 1                                                                       |
|        4 | 0.18831750 | ALTER TABLE orders ENGINE = InnoDB                                                      |
+----------+------------+-----------------------------------------------------------------------------------------+
4 rows in set, 1 warning (0.00 sec)
```
- на MyISAM
```
mysql> SET profiling = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> ALTER TABLE orders ENGINE = MyISAM;
Query OK, 5 rows affected (0.13 sec)
Records: 5  Duplicates: 0  Warnings: 0

mysql> SHOW PROFILES;
+----------+------------+-----------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                   |
+----------+------------+-----------------------------------------------------------------------------------------+
|        1 | 0.00050425 | SELECT * FROM orders                                                                    |
|        2 | 0.00179200 | SELECT table_name, engine
FROM information_schema.tables
WHERE table_schema = 'test_db' |
|        3 | 0.00020800 | SET profiling = 1                                                                       |
|        4 | 0.18831750 | ALTER TABLE orders ENGINE = InnoDB                                                      |
|        5 | 0.00016850 | SET profiling = 1                                                                       |
|        6 | 0.14243575 | ALTER TABLE orders ENGINE = MyISAM                                                      |
+----------+------------+-----------------------------------------------------------------------------------------+
6 rows in set, 1 warning (0.00 sec)
```
# Задача 4
- Изучите файл my.cnf в директории /etc/mysql.
- Измените его согласно ТЗ (движок InnoDB):
- скорость IO важнее сохранности данных;
- нужна компрессия таблиц для экономии места на диске;
- размер буффера с незакомиченными транзакциями 1 Мб;
- буффер кеширования 30% от ОЗУ;
- размер файла логов операций 100 Мб.\
Приведите в ответе изменённый файл my.cnf
```
vagrant@server1:~/mysql$ sudo docker exec -it c0eb718e2f4f bash
bash-4.4# mysql -u root -p
Enter password:
bash-4.4# cat /etc/my.cnf
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html
[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M

# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
secure-file-priv=/var/lib/mysql-files
user=mysql

pid-file=/var/run/mysqld/mysqld.pid
[client]
socket=/var/run/mysqld/mysqld.sock
!includedir /etc/mysql/conf.d/
innodb_flush_log_at_trx_commit = 0
innodb_file_format=Barracuda
innodb_log_buffer_size= 1M
key_buffer_size = 300M
max_binlog_size= 100M
```
