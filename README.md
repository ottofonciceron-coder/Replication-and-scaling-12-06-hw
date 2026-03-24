#  Домашнее задание к занятию "Репликация и масштабирование" - Марчук Кирилл



### Выполните конфигурацию master-slave репликации


1. `Настраиваем Master для репликации в файле /etc/mysql/mysql.conf.d/mysqld.cnf:`
2. `Создём пользователя для репликации`
3. `Делаем второй экземпляр MySQL в Docker:`
4. `Настраиваем server-id для Slave в контейнере:`
5. `Скопироваем данные с Master на Slave`
6. `Настраиваем подключение Slave к Master`
7. `Добавляем тестовую запись на Master`
8. `Проверяем наличие записи на Slave`


```
#1 Master для репликации
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = sakila
binlog_format = ROW
bind-address = 0.0.0.0

#2 Пользователя для репликации
CREATE USER 'repl'@'%' IDENTIFIED BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

#3 Развернул второй экземпляр MySQL в Docker:
docker run -d --name mysql-slave -p 3307:3306 \
  -e MYSQL_ROOT_PASSWORD=rootpass -e MYSQL_DATABASE=sakila mysql:8.0

#4 Настроил server-id для Slave в контейнере:
docker exec mysql-slave bash -c "echo '[mysqld]\nserver-id = 2' > /etc/mysql/conf.d/slave.cnf"
docker restart mysql-slave

#5 Скопироваем данные с Master на Slave
sudo mysqldump -u root --all-databases --source-data=2 > /tmp/master_dump.sql
docker cp /tmp/master_dump.sql mysql-slave:/tmp/
docker exec -i mysql-slave mysql -uroot -prootpass < /tmp/master_dump.sql

#6 Настраиваем подключение Slave к Master
CHANGE MASTER TO
  MASTER_HOST='172.17.0.1',
  MASTER_PORT=3306,
  MASTER_USER='repl',
  MASTER_PASSWORD='replpass',
  MASTER_LOG_FILE='mysql-bin.000002',
  MASTER_LOG_POS=1168;
START SLAVE;

#7 Добавляем тестовую запись на Master
INSERT INTO actor (first_name, last_name) VALUES ('REPLICA', 'TEST');

#8 Проверяем наличие записи на Slave
SELECT * FROM actor WHERE first_name = 'REPLICA';


```


![zadanie1](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/1%20Конфигурация%20master.png)`

![zadanie1](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/2%20Конфигурация%20slave%20(контейнер).png)`

![zadanie1](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/3%20Статус%20master.png)`

![zadanie1](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/5%20Полный%20статус%20slave%20.png)`

![zadanie1](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/8%20Проверка%20server-id%20на%20slave.png)`

![zadanie1](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/6%20Проверка%20репликации.png)`

![zadanie1](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/7%20Проверка%20репликации%20(данные%20на%20slave).png)`



---

### Задание 2 Разработайте план для выполнения горизонтального и вертикального шаринга базы данных. База данных состоит из трёх таблиц.Опишите принципы построения системы и их разграничение или разбивку между базами данных.

1. `Архитектура системы`
2. `Полная комбинированная схема`

![zadanie2](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/Архитектура%20системы.png)`

![zadanie2](https://github.com/ottofonciceron-coder/Replication-and-scaling-12-06-hw/blob/main/Полная%20%20схема.png)`

1. `Вертикальный шардинг - Разделение таблиц по функциональному назначению (users, books, stores).Плюсы - Простота масштабирования, изоляция нагрузки, безопасность.Минусы - Сложность JOIN-запросов между БД`
2. `Горизонтальный шардинг - Разделение таблиц по ключу: users > user_id, books > genre + book_id, stores > region.Плюсы - Равномерное распределение данных, линейное масштабирование.Минусы- Сложность транзакций между шардами`
3. `Комбинированный подход - Система использует двухуровневое разделение данных.Первый уровень — вертикальный шардинг (разделение по доменам).Второй уровень — горизонтальный шардинг внутри каждого домена`

### 3 В каких режимах будут работать сервера.
1. `Master (Read-Write) + Slave (Read-Only) внутри каждого шарда`
2. `Master - Read-Write - Принимает запросы на запись (INSERT, UPDATE, DELETE) и чтение`
3. `Slave	- Read-Only - Принимает только запросы на чтение (SELECT), получает данные с Master через репликацию`

---

