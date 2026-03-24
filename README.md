#  # Домашнее задание к занятию "Репликация и масштабирование" - Марчук Кирилл



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

### Принципы построения:


1. `Вертикальный шардинг - Разделение таблиц по функциональному назначению (users, books, stores).Плюсы - Простота масштабирования, изоляция нагрузки, безопасность.Минусы - Сложность JOIN-запросов между БД`
2. `Горизонтальный шардинг - Разделение таблиц по ключу: users > user_id, books > genre + book_id, stores > region.Плюсы - Равномерное распределение данных, линейное масштабирование.Минусы- Сложность транзакций между шардами`
3. `Выполним запрос на продолжительность которых больше средней продолжительности всех фильмов`

### План горизонтального и вертикального шардинга.

```
graph TB
    subgraph Clients["Клиенты / Приложения"]
        API[API Gateway]
    end

    subgraph Router["Shard Router Layer"]
        RouterLogic[Маршрутизатор запросов<br/>Определение шарда по ключу]
    end

    subgraph Vertical1["Вертикальный шард: DB_Auth"]
        direction TB
        AuthMaster[Master Server<br/>port 3306<br/>Read-Write]
        AuthSlave1[Slave Replica 1<br/>Read-Only]
        AuthSlave2[Slave Replica 2<br/>Read-Only]
        AuthMaster --> AuthSlave1
        AuthMaster --> AuthSlave2
    end

    subgraph Vertical2["Вертикальный шард: DB_Catalog"]
        direction TB
        CatalogMaster[Master Server<br/>port 3306<br/>Read-Write]
        CatalogSlave1[Slave Replica 1<br/>Read-Only]
        CatalogSlave2[Slave Replica 2<br/>Read-Only]
        CatalogMaster --> CatalogSlave1
        CatalogMaster --> CatalogSlave2
    end

    subgraph Vertical3["Вертикальный шард: DB_Commerce"]
        direction TB
        CommerceMaster[Master Server<br/>port 3306<br/>Read-Write]
        CommerceSlave1[Slave Replica 1<br/>Read-Only]
        CommerceSlave2[Slave Replica 2<br/>Read-Only]
        CommerceMaster --> CommerceSlave1
        CommerceMaster --> CommerceSlave2
    end

    Clients --> Router
    Router --> Vertical1
    Router --> Vertical2
    Router --> Vertical3

```


```
graph LR
    subgraph Users_Shards["users - горизонтальные шарды"]
        Shard1[Shard 1<br/>user_id: 1 – 1,000,000<br/>Master + Slave]
        Shard2[Shard 2<br/>user_id: 1,000,001 – 2,000,000<br/>Master + Slave]
        Shard3[Shard 3<br/>user_id: 2,000,001 – 3,000,000<br/>Master + Slave]
        ShardN[Shard N<br/>user_id: > 3,000,000<br/>Master + Slave]
    end
    
    RouterUsers[Router: hash(user_id) % N] --> Shard1
    RouterUsers --> Shard2
    RouterUsers --> Shard3
    RouterUsers --> ShardN

```

```
#Таблица books

graph LR
    subgraph Books_Shards["books - горизонтальные шарды"]
        Fiction[Shard Fiction<br/>genre: Fiction, Fantasy<br/>book_id: 1-500K]
        Mystery[Shard Mystery<br/>genre: Mystery, Thriller<br/>book_id: 1-500K]
        Science[Shard Science<br/>genre: Science, Technical<br/>book_id: 1-500K]
        Other[Shard Other<br/>other genres<br/>book_id: all ranges]
    end
    
    RouterBooks[Router: genre + book_id] --> Fiction
    RouterBooks --> Mystery
    RouterBooks --> Science
    RouterBooks --> Other

```

```
#Таблица stores

graph LR
    subgraph Stores_Shards["stores - горизонтальные шарды"]
        NA[Shard NA<br/>region: North America<br/>Master + Slave]
        EU[Shard EU<br/>region: Europe<br/>Master + Slave]
        Asia[Shard Asia<br/>region: Asia<br/>Master + Slave]
        Other[Shard Other<br/>region: other<br/>Master + Slave]
    end
    
    RouterStores[Router: region] --> NA
    RouterStores --> EU
    RouterStores --> Asia
    RouterStores --> Other

```

```
#Полная комбинированная схема

flowchart TB
    subgraph API["Application Layer"]
        LB[Load Balancer]
        Router[Shard Router<br/>Middleware]
    end

    subgraph Auth["DB_Auth (users)"]
        direction LR
        A1[users_shard_1<br/>ID: 1-1M<br/>Master+Slave]
        A2[users_shard_2<br/>ID: 1M-2M<br/>Master+Slave]
        A3[users_shard_3<br/>ID: 2M-3M<br/>Master+Slave]
    end

    subgraph Catalog["DB_Catalog (books)"]
        direction LR
        B1[books_shard_1<br/>Fiction<br/>Master+Slave]
        B2[books_shard_2<br/>Mystery<br/>Master+Slave]
        B3[books_shard_3<br/>Science<br/>Master+Slave]
    end

    subgraph Commerce["DB_Commerce (stores)"]
        direction LR
        S1[stores_shard_1<br/>North America<br/>Master+Slave]
        S2[stores_shard_2<br/>Europe<br/>Master+Slave]
        S3[stores_shard_3<br/>Asia<br/>Master+Slave]
    end

    LB --> Router
    Router -->|user_id| Auth
    Router -->|genre + book_id| Catalog
    Router -->|region| Commerce

```

---

