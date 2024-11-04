# Домашнее задание #1
## Работа с уровнями изоляции транзакции в PostgreSQL

- Создать ВМ в YandexCloud:
```
yc compute instance create \
  --name pg-db1 \
  --hostname pg-db1 \
  --network-interface subnet-name=pg-subnet-b,nat-ip-version=ipv4,address=192.168.2.11 \
  --memory 4G \
  --cores 2 \
  --create-boot-disk image-folder-id=standard-images,image-family=centos-7,size=20,auto-delete=true \
  --zone ru-central1-b \
  --ssh-key ~/.ssh/id_ed25519.pub
```

- Подключиться к созданной ВМ:
```
ssh yc-user@89.169.172.119
```

- Установить PostgreSQL
```
sudo yum install epel-release
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum install -y postgresql15-server
```

- Инициализировать и запустить кластер:
```
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable postgresql-15
sudo systemctl start postgresql-15
```
- Запустить везде psql из-под пользователя postgres:
```
sudo su postgres
psql
```

- Выключить autocommit
```
\set AUTOCOMMIT off
```

- Создать в первой сессии таблицу и заполнить её данными:
```
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```

- Посмотреть текущий уровень изоляции транзакции:
```
postgres=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
(1 row)
```

- Начать новую транзакцию в обеих сессиях с дефолтным (не меняя) уровнем изоляции:
```
postgres=# begin;
BEGIN
postgres=*# 
```

- в первой сессии добавить новую запись
```
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```

- сделать select * from persons во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
- видите ли вы новую запись и если да то почему?

__Во второй сессии новой записи не видно, так как изменения в первой сессии не были подтверждены. На уровне изоляции read commited грязное чтение не допускается. Также следует отметить, что в PostgreSQL грязное чтение невозможно на любом уровне изоляции.__

- завершить первую транзакцию - commit;


- сделать select * from persons во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
- завершите транзакцию во второй сессии - commit

- видите ли вы новую запись и если да то почему?

__Во второй сессии видно новую запись, потому что изменения в первой сессии были подтверждены на момент выполнения чтения во второй сессии, а на уровне изоляции Read Commited допускается аномалия неповторяемого чтения.__

- начать новые но уже repeatable read транзакции - set transaction isolation level repeatable read;

```
postgres=*# set transaction isolation level repeatable read;
SET
```

- в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');

```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```

- сделать select * from persons во второй сессии
```
postgres=*# select * from persons;                          
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
- видите ли вы новую запись и если да то почему?

__Во второй сессии новой записи не видно, так как изменения в первой сессии не были подтверждены. На уровне изоляции repeatable read  грязное чтение не допускается.__


- завершить первую транзакцию - commit;

сделать select * from persons во второй сессии
```
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

- видите ли вы новую запись и если да то почему?

__Во второй сессии новой записи не видно, несмотря на то, что в первой сессии изменения были подтверждены. Это происходит, потому что на уровне repeatable read неповторяемое чтение не допускается.__

- завершить вторую транзакцию - commit;

- сделать select * from persons во второй сессии
```
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  5 | sveta      | svetova
(4 rows)
```

- видите ли вы новую запись и если да то почему?

__Теперь во второй сессии новую запись видно, потому что предыдущая транзакция во второй сессии завершилась и началась новая, а на момент начала новой транзакции эта запись уже существовала__.
