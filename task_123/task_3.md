# Инструкция для задания 3.  

## Шаг 1. PostgreSQL
Подключение через внешний адрес к jn, переход на nn. 
```bash
ssh team@ВНЕШНИЙ_АДРЕС
ssh team-22-nn
sudo apt install postgresql # установка postgresql
sudo -i -u postgres # активация пользователя postgres
psql # подключение консоли postgresql
```
Затем:
### 5. Создание бд
```sql
CREATE DATABASE metastore; -- создание базы данных
CREATE USER hive WITH PASSWORD 'your_password'; -- создание пользователя hive 
GRANT ALL PRIVILEGES ON DATABASE "metastore" TO hive; -- необходимо выдать права на бд
ALTER DATABASE metastore OWNER TO hive; -- необходимо передать права владения базой пользователю
\q -- выход из консоли
```
Выйдем из пользователя postgres обратно на nn:  
 ```bash
 exit
 ```

## Шаг 2. Настройка конфигов PostgreSQL
Откроем конфиг postgresql.conf и добавим адрес name node в секции "CONNECTIONS AND AUTHENTICATION" в разделе Connection settings:  
```bash
sudo vim /etc/postgresql/16/main/postgresql.conf
```
Необходимо добавить адрес:  
```bash
listen_addresses = 'team-22-nn'
```
Откроем конфиг pg_hba.conf:  
```bash
sudo vim /etc/postgresql/16/main/pg_hba.conf
```
И добавим в начало в секции "IPv4 local connections":
```bash
host    metastore       hive            <ip_jumpnode>/32         password
```

Затем:  
```bash
sudo systemctl restart postgresql # перезапускаем postgresql
sudo systemctl status postgresql # проверяем статус
exit # возвращаемся на jn
```

## Шаг 3. Установка PostgreSQL Client
```bash
sudo apt install postgresql-client-16
psql -h team-22-nn -p 5432 -U hive -W -d metastore # проверка возможности подключения к таблице
```
Если успешно, выходим из консоли командой:  
```sql
\q
```

## Шаг 4. Настройка Hadoop на Jump Node
Контроль: в данный момент вы должны находиться на jn в пользователе team.  
Переключаемся в пользователя hadoop и разархивируем архив для hadoop (он уже должен лежать в каталоге после 1 задания)
```bash
sudo -i -u hadoop
tar -xvzf hadoop-3.4.0.tar.gz
```
Добавление переменных окружения в ".profile"  
```bash
vim ~/.profile
```
Добавляем следующее содержимое:  
```
export HADOOP_HOME=/home/hadoop/hadoop-3.4.0
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
Применяем изменения  
```bash
source ~/.profile
```

Корректировка конфигурационных файлов  
```bash
cd hadoop-3.4.0/etc/hadoop
```
core-site.xml:
```bash
vim core-site.xml
```
Добавляем следующее содержимое в файл:  
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://team-22-nn:9000</value>
    </property>
</configuration>
```

hdfs-site.xml: 
```bash
vim hdfs-site.xml
```
Добавляем следующее содержимое в файл:  
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```

## Шаг 5. Настройка Apache Hive

```bash
cd ~ # Возврат в конверую директорию
wget https://dlcdn.apache.org/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz # Скачивание дистрибутива Apache Hive
tar -xvzf apache-hive-4.0.1-bin.tar.gz # Разархивация архива
cd apache-hive-4.0.1-bin/ # Переход в папку дистрибутива
cd lib # Переход в папку драйверов
ls -l | grep postgres # Ищем нужный драйвер, обнаруживаем, что его нет
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar # Скачиваем нужный драйвер
```

Для настройки конфигурационных файлов Hive возвращаемся в директорию выше и заходим в папку conf:  
```bash
cd ../conf/
vim hive-site.xml # Создание нового файла hive-site.xml
```
Добавляем следующее содержимое в файл:
```xml
<configuration>
   <property>
       <name>hive.server2.authentication</name>
       <value>NONE</value>
   </property>
   <property>
       <name>hive.metastore.warehouse.dir</name>
       <value>/user/hive/warehouse</value>
   </property>
   <property>
       <name>hive.server2.thrift.port</name>
       <value>5433</value>
   </property>
   <property>
       <name>javax.jdo.option.ConnectionURL</name>
       <value>jdbc:postgresql://team-22-nn:5432/metastore</value>
   </property>
   <property>
       <name>javax.jdo.option.ConnectionDriverName</name>
       <value>org.postgresql.Driver</value>
   </property>
   <property>
       <name>javax.jdo.option.ConnectionUserName</name>
       <value>hive</value>
   </property>
   <property>
       <name>javax.jdo.option.ConnectionPassword</name>
       <value><postgre password></value>
   </property>
</configuration>
```

Добавление переменных окружения в ".profile"  
```bash
vim ~/.profile
```
Добавляем следующее содержимое:  
```plaintext
export HIVE_HOME=/home/hadoop/apache-hive-4.0.1-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin
```

Применяем изменения  
```bash
source ~/.profile
```

Проверяем версии следующими командами:  
```bash
hive --version
hadoop version
```

Создание директории HDFS
```bash
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
```

Инициализация схемы Hive
```bash
cd ../
bin/schematool -dbType postgres -initSchema
```

Запуск Hive Server
```bash
hive --hiveconf hive.server2.enable.doAs=false --hiveconf hive.security.authorization.enabled=false --service hiveserver2 1>> /tmp/hs2.log 2>> /tmp/hs2.log &
```
Проверка запуска hive командой:  
```bash
jps # должны увидеть RunJar
```

## Шаг 6. Загрузка датасета в Hive

Переходим в корневую директорию на Jump mode и скачиваем архив с данными из репозитория.  
Демонстрирую пример в виде данных о продажах авто "vehicles_team_22.csv", который я предварительно архивировал и положил в репозиторий на github.  
```bash
cd
wget https://github.com/vechenn/bigdata/raw/main/vehicles_dataset/vehicles_team_22.csv.zip
```

Для распаковки архива скачаем unzip, затем распакуем архив
```bash
exit # выходим из пользователя hadoop, так как у него нет sudo прав
sudo apt install unzip # вводим пароль sudo
sudo -i -u hadoop # возвращаемся обратно в пользователя hadoop
unzip vehicles_team_22.csv.zip # распаковываем архив
rm vehicles_team_22.csv.zip # удаляем архив
tail -n +2 vehicles_team_22.csv > vehicles_team_22_no_header.csv # удаляем заголовок
hdfs dfs -mkdir /input # Создадим папку на hdfs
hdfs dfs -chmod g+w /input # Установим права
hdfs dfs -put vehicles_team_22_no_header.csv /input # Кладем данные для последующей загрузки в hive
hdfs fsck /input/vehicles_team_22_no_header.csv # Можно просмотреть информацию о загруженных данных
```

##  Шаг 6. Подключение к Hive через Beeline

```bash
source ~/.profile
beeline -u jdbc:hive2://team-22-jn:5433
```
С этого момента вы подключили к hive с помощью beeline. Здесь можно просматривать таблицы, делать запросы и т.д.  

Создадим тестовую БД:  
```
SHOW DATABASES;
CREATE DATABASE test;
SHOW DATABASES;
DESCRIBE DATABASE test;
```

Создание непартицированной таблицы в hive
```
use test;
drop table if exists test.cars_no_partition;
CREATE TABLE IF NOT EXISTS test.cars_no_partition(
    id STRING,
    url STRING,
    region STRING,
    price STRING,
    year STRING,
    manufacturer STRING,
    model STRING,
    condition STRING,
    cylinders STRING,
    fuel STRING,
    odometer STRING,
    title_status STRING,
    transmission STRING,
    VIN STRING,
    drive STRING,
    size STRING,
    type STRING,
    paint_color STRING,
    state STRING,
    posting_date STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ';';

LOAD DATA INPATH '/input/vehicles_team_22_no_header.csv' INTO TABLE test.cars_no_partition;
```
Включение динамического партиционирования
```
SET hive.exec.dynamic.partition=true;
```
Создание в Hive партиционированной таблицы и загрузка в нее данных из непартиционированной
```
drop table if exists test.cars_with_partition;
CREATE TABLE IF NOT EXISTS test.cars_with_partition (
    id STRING,
    url STRING,
    region STRING,
    price STRING,
    year STRING,
    manufacturer STRING,
    model STRING,
    condition STRING,
    cylinders STRING,
    fuel STRING,
    odometer STRING,
    title_status STRING,
    transmission STRING,
    VIN STRING,
    drive STRING,
    size STRING,
    type STRING,
    paint_color STRING,
    state STRING)
PARTITIONED BY (posting_date STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ';';

INSERT OVERWRITE TABLE cars_with_partition
PARTITION(posting_date)
SELECT id, url, region, price, year, manufacturer, model, condition, cylinders, fuel, odometer, title_status, transmission, VIN, drive, size, type, paint_color, state, posting_date
FROM cars_no_partition;
```

Проверим данные в новой таблице
```
SELECT COUNT(*) FROM test.cars_with_partition;
```

Выход из Beeline осуществляется с помощью команды:
```
Ctrl+C
```
или
```
!quit
```