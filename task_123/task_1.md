# Инструкция для задания 1.  

## Шаг 1. Подключение через внешний адрес к jn (jump node)  
Для работы выделено 4 узла с локальными адресами и одним внешним адресом с паролем, по которому можно попасть на один из узлов, а именно на узел jump node.

```bash
ssh team@ВНЕШНИЙ_АДРЕС
```
После успешного введения пароля вы попадете на прыжковую ноду, в моем примере она будет называться team@team-22-jn. 
Примечение: все узлы будут носить в своем названиее номер 22, так как это номер моей команды, в вашем случае могут быть иные названия.   
Необходимо добавить свой ssh ключ в список авторизованных ключей.  

```bash
vim .ssh/authorized_keys
``` 
Аналогичную процедуру необходимо сделать всем участникам команды, либо одному из участников собрать у всех ключи и добавить в список авторизованных ключей.  

## Шаг 2. Создание пользователя hadoop  

```bash
sudo adduser hadoop # команда для создания нового пользователя
```
При создании пользователя потребуется: ввести пароль, выданный с внешним адресом из шаг 1; установить пароль для нового пользователя, ввести его повторно; установить Full name: hadoop; оставшиеся поля можно оставить пустыми.  

## Шаг 3. Генерация ssh ключей для нового пользователя hadoop на всех узлах и установление связи между ними.

Следующую последовательность действий необходимо сделать на каждом из узлов (team-22-jn, team-22-nn, team-22-dn-00, team-22-dn-01). Перемещаться по узлам вы можете с помощью локальных адресов, начиная с jn, на которой вы оказались после входа через внешний адрес.    

```bash
sudo -i -u hadoop # вход в пользователя
ssh-keygen # генерация ключа
cat .ssh/id_ed25519.pub # вывод содержимого ключа, сохранение куда-нибудь к себе (скоро понадобится)
exit # выход из пользователя hadoop
```

```bash
sudo vim /etc/hosts # открываем файл
```
Копируем и вносим локальные адреса узлов в файл, остальную информацию комментируем.  
В моем примере локальные адреса следующие, что я и добавляю в информацию:  
```bash
192.168.1.90    team-22-jn
192.168.1.91    team-22-nn
192.168.1.92    team-22-dn-00
192.168.1.93    team-22-dn-01
```

Не забудьте повторить вышеперечисленные действия на каждом узле!!!  
В итоге вы также должны собрать 4 публичных ключа с каждого узла (команда cat .ssh/id_ed25519.pub).  

## Шаг 4. Добавление публичных ключей каждого узла в авторизованные ключи и распространение их на каждый узел.  

```bash
sudo -i -u hadoop # активация пользователя hadoop
vim .ssh/authorized_keys # добавляем в список 4 сохраненных раннее публичных ключа
scp .ssh/authorized_keys team-22-nn:/home/hadoop/.ssh/ # копирование файла на узел team-22-nn
scp .ssh/authorized_keys team-22-dn-00:/home/hadoop/.ssh/ # копирование файла на узел team-22-dn-00
scp .ssh/authorized_keys team-22-dn-01:/home/hadoop/.ssh/ # копирование файла на узел team-22-dn-01
```
Возможно, потребуется пароль пользователя hadoop.  

## Шаг 5. Скачивание архива hadoop и распространение архива на каждый узел.  
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz # скачивание архива
scp hadoop-3.4.0.tar.gz team-22-nn:/home/hadoop # копирование файла на узел team-22-nn
scp hadoop-3.4.0.tar.gz team-22-dn-00:/home/hadoop # копирование файла на узел team-22-dn-00
scp hadoop-3.4.0.tar.gz team-22-dn-01:/home/hadoop # копирование файла на узел team-22-dn-01
```

## Шаг 6. На каждом узле необходимо распаковать архив.  
Контроль: в данный момент вы должны находиться на jn в user-е hadoop. Выполняем следующую последовательность команд, чтобы войти на каждый узел, распаковать архив и выйти обратно на jn.  
```bash
ssh team-22-nn 
tar -xvzf hadoop-3.4.0.tar.gz 
exit
ssh team-22-dn-00
tar -xvzf hadoop-3.4.0.tar.gz
exit
ssh team-22-dn-01
tar -xvzf hadoop-3.4.0.tar.gz
exit
```


## Шаг 7. Начало настройки кластера на узле nn и распространение настроек на другие узлы.  
Контроль: в данный момент вы должны находиться на jn в user-е hadoop.
```bash
ssh team-22-nn # переходим на nn
java -version # проверка 11-ой версии java
readlink -f /usr/bin/java # проверка полного пути, где лежит java
```
В качестве вывода последней команды должны получить:
```bash
/usr/lib/jvm/java-11-openjdk-amd64/bin/java
```
Данный путь необходимо сохранить для следующей команды добавления переменных окружения.  
```bash
vim ~/.profile # открываем файл
```
Добавляем следующие переменные окружения
```bash
export HADOOP_HOME=/home/hadoop/hadoop-3.4.0
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
Для валидации того, что вы сделали все правильно можно попробовать команду:  
```bash
source ~/.profile # в случае правильных действий, вы не получите ошибок
```
Проверим версию hadoop командой:  
```bash
hadoop version
```

Копируем конфиг profile на data nodes командами:  
```bash
scp ~/.profile team-22-dn-00:/home/hadoop
scp ~/.profile team-22-dn-01:/home/hadoop
```

Переходим в папку дистрибутива командой:  
```bash
cd hadoop-3.4.0/etc/hadoop
```
Добавляем путь JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 в файл hadoop-env.sh
```bash
vim hadoop-env.sh # открыть файл
```

Копируем конфиг hadoop-env.sh на data nodes командами:  
```bash
scp hadoop-env.sh team-22-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp hadoop-env.sh team-22-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop
```

Теперь необходимо настроить файловую систему с помощью конфига core-site.xml, hdfs-site.xml и workers  
Начнем с файла core-site.xml:  
```bash
vim core-site.xml
```
Добавляем в файл следующее:  
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://team-22-nn:9000</value>
    </property>
</configuration>
```
Затем файл hdfs-site.xml:  
```bash
nano hdfs-site.xml
```
Добавляем в файл следующее (3 - так как задаем фактор репликации на 3 data nod-ы): 
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```
Затем файл workers:  
```bash
vim workers
```
Добавляем в файл следующее вместо localhost:  
```bash
- team-22-nn
- team-22-dn-00
- team-22-dn-01
```

Осталось все настройки перенести на data nod-ы:  
```bash
scp core-site.xml team-22-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp core-site.xml team-22-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp hdfs-site.xml team-22-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp hdfs-site.xml team-22-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp workers team-22-dn-00:/home/hadoop/hadoop-3.4.0/etc/hadoop
scp workers team-22-dn-01:/home/hadoop/hadoop-3.4.0/etc/hadoop
```

## Шаг 8. Запуск кластера.  
Контроль: в данный момент вы должны находиться по пути hadoop-3.4.0/etc/hadoop на узле name node. Вы можете проверить это командой:  
```bash
pwd
```
Необходимо вернуться на уровень папки hadoop-3.4.0:  
```bash
cd ../../
```
Теперь выполним следующую последовательность команд:  
```bash
bin/hdfs namenode -format # форматируем файловую систему
sbin/start-dfs.sh # запустим кластер hadoop
jps # проверим, что все поднялось
```
После последней команды вы должны увидеть список, в котором будут (NameNode, SecondaryNameNode, DataNode)


## Шаг 9. Проверка UI интерфейса кластера с помощью nginx.  
Контроль: в данный момент вы должны находиться по пути hadoop-3.4.0/etc/hadoop на узле name node. Необходимо вернуться на jump node и выйти из пользователя hadoop последовательностью:  
```bash
exit
exit
```
Далее необходимо настроить конфиг nginx:  
```bash
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/nn # копируем конфиг nginx
sudo vim /etc/nginx/sites-available/nn # открываем конфиг
```
Содержимое конфига нужно заполнить следующим вместо имеющегося:  
```bash
##
# You should look at the following URL's in order to grasp a solid understanding
# of Nginx configuration files in order to fully unleash the power of Nginx.
# https://www.nginx.com/resources/wiki/start/
# https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/
# https://wiki.debian.org/Nginx/DirectoryStructure
#
# In most cases, administrators will remove this file from sites-enabled/ and
# leave it as reference inside of sites-available where it will continue to be
# updated by the nginx packaging team.
#
# This file will automatically load configuration files provided by other
# applications, such as Drupal or Wordpress. These applications will be made
# available underneath a path with that package name, such as /drupal8.
#
# Please see /usr/share/doc/nginx-doc/examples/ for more detailed examples.
##

# Default server configuration
#
server {
        listen 9870 default_server;
        # listen [::]:80 default_server;

        # SSL configuration
        #
        # listen 443 ssl default_server;
        # listen [::]:443 ssl default_server;
        #
        # Note: You should disable gzip for SSL traffic.
        # See: https://bugs.debian.org/773332
        #
        # Read up on ssl_ciphers to ensure a secure configuration.
        # See: https://bugs.debian.org/765782
        #
        # Self signed certs generated by the ssl-cert package
        # Don't use them in a production server!
        #
        # include snippets/snakeoil.conf;

        root /var/www/html;

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                # try_files $uri $uri/ =404;
                proxy_pass http://team-22-nn:9870;
        }

        # pass PHP scripts to FastCGI server
        #
        #location ~ \.php$ {
        #       include snippets/fastcgi-php.conf;
        #
        #       # With php-fpm (or other unix sockets):
        #       fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        #       # With php-cgi (or other tcp sockets):
        #       fastcgi_pass 127.0.0.1:9000;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #       deny all;
        #}
}


# Virtual Host configuration for example.com
#
# You can move that to a different file under sites-available/ and symlink that
# to sites-enabled/ to enable it.
#
#server {
#       listen 80;
#       listen [::]:80;
#
#       server_name example.com;
#
#       root /var/www/example.com;
#       index index.html;
#
#       location / {
#               try_files $uri $uri/ =404;
#       }
#}

```
Затем создаем ссылку и перезапускаем nginx последовательностью команд и отключаемся от сервера:  
```bash
sudo ln -s /etc/nginx/sites-available/nn /etc/nginx/sites-enabled/nn
sudo systemctl reload nginx
exit
```

Проверяем работу через UI интерфейс, перейдя в браузере по ссылке:  
```bash
ВНЕШНИЙ_АДРЕС:9870
```
Вероятно, что это не сработает, значит порты закрыты. Тогда используем проброс портов, находясь в терминале в своем компьютере (не на удаленном сервере):  
```bash
ssh -L 9870:team-22-nn:9870 team@ВНЕШНИЙ_АДРЕС
```
Переходим в браузере по ссылке:  
```bash
localhost:9870/
```


# Поздравляю, вы успешно выполнили первое задание!

##  Выключение
Для выключения кластера выполняем следующую последовательность команд:  
```bash
ssh team@ВНЕШНИЙ_АДРЕС
sudo -i -u hadoop
ssh team-22-nn
cd hadoop-3.4.0/
sbin/stop-dfs.sh
```
Проверка на всех узлах с помощью команды:  
```bash
jps
```