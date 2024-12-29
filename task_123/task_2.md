# Инструкция для задания 2.  

## Шаг 1. Подключение через внешний адрес к jn, вход в пользователя hadoop, переход на nn, переход в папку hadoop.  
```bash
ssh team@ВНЕШНИЙ_АДРЕС
sudo -i -u hadoop
ssh team-22-nn
cd hadoop-3.4.0/etc/hadoop
```
 
## Шаг 2. Настройка MapReduce и YARN
Необходимо настроить конфиг mapreduce:  
```bash
vim mapred-site.xml
```
Содержимое конфига нужно заполнить следующим вместо имеющегося:  
```bash
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
        <property>
                <name>mapreduce.application.classpath</name>
                <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
        </property>
</configuration>
```
Затем конфиг yarn:  
```bash
vim yarn-site.xml
```
Содержимое конфига нужно заполнить следующим вместо имеющегося:  
```bash
<?xml version="1.0"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
<configuration>

<!-- Site specific YARN configuration properties -->

        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
        <property>
                <name>yarn.nodemanager.env-whitelist</name>
                <value>JAVA_HOME, HADOOP_COMMON_HOME, HADOOP_HDFS_HOME, HADOOP_CONF_DIR, CLASSPATH_PREPEND_DISTCACHE, HADOOP_YARN_HOME, HADOOP_HOME, PATH, LANG, TZ, HADOOP_MAPRED_HOME</value>
        </property>
</configuration>
```
Копируем конфиги mapred-site.xml и yarn-site.xml на data nodes командами:  
```bash
scp mapred-site.xml team-22-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp mapred-site.xml team-22-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp yarn-site.xml team-22-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp yarn-site.xml team-22-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop
```

## Шаг 3. Запуск yarn и historyserver.  
Контроль: в данный момент вы должны находиться по пути hadoop-3.4.0/etc/hadoop на узле name node. Вы можете проверить это командой:  
```bash
pwd
```
Возвращаемся на два уровня выше в папку hadoop-3.4.0 и запускаем  yarn и historyserver командами:  
```bash
cd ../../
sbin/start-yarn.sh
mapred --daemon start historyserver
```

## Шаг 4. Проверка UI веб-интерфейсов yarn и historyserver с помощью nginx.  

```bash
exit # выходим на jn
exit # выходим из пользователя hadoop
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/ya # копируем конфиг nginx для yarn
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/dh # копируем конфиг nginx для historyserver
```
Открываем конфиг yarn и указываем порт в listen и proxy_pass на порт 8088:  
```bash
 sudo vim /etc/nginx/sites-available/ya
```
Открываем конфиг historyserver и заменяем порт в listen и proxy_pass на порт 19888:  
```bash
 sudo vim /etc/nginx/sites-available/dh
```
Включаем хосты и запускаем nginx командами:  
```bash
 sudo ln -s /etc/nginx/sites-available/ya /etc/nginx/sites-enabled/ya
 sudo ln -s /etc/nginx/sites-available/dh /etc/nginx/sites-enabled/dh
 sudo systemctl reload nginx
 exit # выходим с серверов
```

Проверяем работу через UI интерфейс yarn, перейдя в браузере по ссылке:  
```bash
ВНЕШНИЙ_АДРЕС:8088
```
Вероятно, что это не сработает, значит порты закрыты. Тогда используем проброс портов, находясь в терминале в своем компьютере (не на удаленном сервере):  
```bash
ssh -L 8088:team-22-nn:8088 team@ВНЕШНИЙ_АДРЕС
```
Переходим в браузере по ссылке:  
```bash
localhost:8088/
```

Проверяем работу через UI интерфейс historyserver, перейдя в браузере по ссылке:  
```bash
ВНЕШНИЙ_АДРЕС:19888
```
Вероятно, что это не сработает, значит порты закрыты. Тогда используем проброс портов, находясь в терминале в своем компьютере (не на удаленном сервере):  
```bash
ssh -L 19888:team-22-nn:19888 team@ВНЕШНИЙ_АДРЕС
```
Переходим в браузере по ссылке:  
```bash
localhost:19888/
```

## Шаг 4. Проверка UI веб-интерфейсов NodeManager с помощью nginx.  

```bash
ssh team@ВНЕШНИЙ_АДРЕС
sudo -i -u hadoop
ssh team-22-dn-00 # идем на dn-00
vim /home/hadoop/hadoop-3.4.0/etc/hadoop/yarn-site.xml # открываем yarn-site.xml
```
Добавляем к имеющемуся содрежимому внутрь конфига:  
```xml
<property>
    <name>yarn.nodemanager.webapp.address</name>
    <value>0.0.0.0:8042</value>
</property>
```
Затем:  
```bash
exit
ssh team-22-dn-01 # идем на dn-01
vim /home/hadoop/hadoop-3.4.0/etc/hadoop/yarn-site.xml # открываем yarn-site.xml
```
Добавляем к имеющемуся содержимому внутрь конфига:  
```xml
<property>
    <name>yarn.nodemanager.webapp.address</name>
    <value>0.0.0.0:8042</value>
</property>
```
Затем:  
```bash
exit # вышли с dn-01 на jn
exit # вышли из пользователя hadoop
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/nm-0 # Копируем конфигурации nginx
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/nm-1 # Копируем конфигурации nginx
```
Теперь необходимо настроить конфиги:  
Первый:  
```bash
sudo vim /etc/nginx/sites-available/nm-0
```
Заменяем строку с listen на:
```bash
listen 8043 default_server;
```
Заменяем строку с proxy_pass на:
```bash
proxy_pass http://team-22-nn:8042;
```
Второй:  
```bash
sudo vim /etc/nginx/sites-available/nm-1
```
Заменяем строку с listen на:
```bash
listen 8044 default_server;
```
Заменяем строку с proxy_pass на:
```bash
proxy_pass http://team-22-nn:8042;
```
Добавляем nm в nginx и перезагружаем nginx:  
```bash
sudo ln -s /etc/nginx/sites-available/nm-0 /etc/nginx/sites-enabled/nm-0
sudo ln -s /etc/nginx/sites-available/nm-1 /etc/nginx/sites-enabled/nm-1
sudo systemctl reload nginx
exit # выходим с серверов
```
Проверяем работу через UI интерфейс nm-0, перейдя в браузере по ссылке:  
```bash
ВНЕШНИЙ_АДРЕС:8043
```
Вероятно, что это не сработает, значит порты закрыты. Тогда используем проброс портов, находясь в терминале в своем компьютере (не на удаленном сервере):  
```bash
ssh -L 8043:team-22-nn:8042 team@ВНЕШНИЙ_АДРЕС
```
Переходим в браузере по ссылке:  
```bash
localhost:8043/
```

Проверяем работу через UI интерфейс nm-1, перейдя в браузере по ссылке:  
```bash
ВНЕШНИЙ_АДРЕС:8044
```
Вероятно, что это не сработает, значит порты закрыты. Тогда используем проброс портов, находясь в терминале в своем компьютере (не на удаленном сервере):  
```bash
ssh -L 8044:team-22-nn:8042 team@ВНЕШНИЙ_АДРЕС
```
Переходим в браузере по ссылке:  
```bash
localhost:8044/
```


# Поздравляю, вы успешно выполнили второе задание!

##  Выключение
Для выключения кластера выполняем следующую последовательность команд:  
```bash
ssh team@ВНЕШНИЙ_АДРЕС
sudo -i -u hadoop
ssh team-22-nn
cd hadoop-3.4.0/
mapred --daemon stop historyserver # выключение historyserver
sbin/stop-yarn.sh # выключение yarn
sbin/stop-dfs.sh # выключение dfs
```
Проверка на всех узлах с помощью команды:  
```bash
jps
```