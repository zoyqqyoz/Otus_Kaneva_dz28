Vagrant-стенд c Postgres

Цель домашнего задания
Научиться настраивать репликацию и создавать резервные копии в СУБД PostgreSQL

Описание домашнего задания
1) Настроить hot_standby репликацию с использованием слотов
2) Настроить правильное резервное копирование

Создадим Vagrantfile, в котором будут указаны параметры наших ВМ:

```
# Описание параметров ВМ
MACHINES = {
  # Имя DV "pam"
  :node1 => {
        # VM box
        :box_name => "centos/stream8",
        # Имя VM
        :vm_name => "node1",
        # Количество ядер CPU
        :cpus => 2,
        # Указываем количество ОЗУ (В Мегабайтах)
        :memory => 1024,
        # Указываем IP-адрес для ВМ
        :ip => "192.168.57.11",
  },
  :node2 => {
        :box_name => "centos/stream8",
        :vm_name => "node2",
        :cpus => 2,
        :memory => 1024,
        :ip => "192.168.57.12",

  },
  :barman => {
        :box_name => "centos/stream8",
        :vm_name => "barman",
        :cpus => 1,
        :memory => 1024,
        :ip => "192.168.57.13",

  },

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
    
    config.vm.define boxname do |box|
   
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      box.vm.network "private_network", ip: boxconfig[:ip]
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end

      # Запуск ansible-playbook
      if boxconfig[:vm_name] == "barman"
       box.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/provision.yml"
        ansible.inventory_path = "ansible/hosts"
        ansible.host_key_checking = "false"
        ansible.limit = "all"
       end
      end
    end
  end
end
```

После создания Vagrantfile запустим наши ВМ командой vagrant up. Будет создано три виртуальных машины.

```
neva@Uneva:~/Otus_Kaneva_dz28$ vagrant status
Current machine states:
neva@Uneva:~/Otus_Kaneva_dz28$ vagrant up
node1                     running (virtualbox)
node2                     running (virtualbox)
barman                    running (virtualbox)
```

На всех хостах предварительно должны быть выключены firewalld и SElinux:
Отключаем службу firewalld:  systemctl stop firewalld
Удаляем службу из автозагрузки: systemctl disable firewalld
Отключаем SElinux: setenforce 0
Правим параметр SELINUX=disabled в файле /etc/selinux/config 

```
neva@Uneva:~/Otus_Kaneva_dz28$ vagrant ssh node1
[vagrant@node1 ~]$ sudo -i

[root@node1 ~]# systemctl stop firewalld
[root@node1 ~]# systemctl disable firewalld
[root@node1 ~]# setenforce 0
[root@node1 ~]# vi  /etc/selinux/config

SELINUX=disabled


[root@node1 ~]# sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
[root@node1 ~]# sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
[root@node1 ~]# yum install -y epel-release
[root@node1 ~]# yum install -y vim telnet

```

Настройка hot_standby репликации с использованием слотов

Перед настройкой репликации необходимо установить postgres-server на хосты node1 и node2:

Добавим postgres репозиторий: 

```
[root@node1 ~]# dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm 
```

Исключаем старый postgresql модуль: 

```
[root@node1 ~]# yum -qy module disable postgresq
```

Устанавливаем postgresql-server 14:

```
[root@node1 ~]# yum install -y postgresql14-server
```

Выполняем инициализацию кластера:

```
[root@node1 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
```

Запускаем postgresql-server и добавляем его в автозагрузку:

```
[root@node1 ~]# systemctl start postgresql-14
[root@node1 ~]# systemctl enable postgresql-14
```

Далее приступаем к настройке репликации:

На node1 подключаемся к psql и создаём пользователя replicator c правами репликации и паролем «Otus2022!»

```
[root@node1 ~]# sudo -u postgres psql

postgres=# CREATE USER replicator WITH REPLICATION Encrypted PASSWORD 'Otus2022!';
CREATE ROLE
```

В файле /var/lib/pgsql/14/data/postgresql.conf указываем следующие параметры:

```
#Указываем ip-адреса, на которых postgres будет слушать трафик на порту 5432 (параметр port)
listen_addresses = 'localhost, 192.168.57.11'
#Указываем порт порт postgres
port = 5432 
#Устанавливаем максимально 100 одновременных подключений
max_connections = 100
log_directory = 'log' 
log_filename = 'postgresql-%a.log' 
log_rotation_age = 1d 
log_rotation_size = 0 
log_truncate_on_rotation = on 
max_wal_size = 1GB
min_wal_size = 80MB
log_line_prefix = '%m [%p] ' 
#Указываем часовой пояс для Москвы
log_timezone = 'UTC+3'
timezone = 'UTC+3'
datestyle = 'iso, mdy'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8' 
lc_numeric = 'en_US.UTF-8' 
lc_time = 'en_US.UTF-8' 
default_text_search_config = 'pg_catalog.english'
#можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления; 
hot_standby = on
#Включаем репликацию
wal_level = replica
#Количество планируемых слейвов
max_wal_senders = 3
#Максимальное количество слотов репликации
max_replication_slots = 3
#будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.
hot_standby_feedback = on
#Включаем использование зашифрованных паролей
password_encryption = scram-sha-256
```

Настраиваем параметры подключения в файле /var/lib/pgsql/14/data/pg_hba.conf: 

```
[root@node1 ~]# vi /var/lib/pgsql/16/data/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
host    replication replicator    192.168.57.11/24      scram-sha-256
host    replication replicator    192.168.57.12/24      scram-sha-256
```

Перезапускаем postgresql-server: 

```
[root@node1 ~]# systemctl restart postgresql-16.service
```

На хосте node2 останавливаем postgresql-server:

```
[root@node2 ~]# systemctl stop postgresql-16.service
```

С помощью утилиты pg_basebackup копируем данные с node1:

```
[root@node2 ~]# pg_basebackup -h 192.168.57.11 -U  replicator -p 5432 -D /var/lib/pgsql/16/data/ -R -P
```

В файле var/lib/pgsql/14/data/postgresql.conf меняем параметр listen_addresses = 'localhost, 192.168.57.12':

```
[root@node2 ~]# vi /var/lib/pgsql/16/data/postgresql.conf
```

Запускаем службу postgresql-server: 

```
[root@node2 ~]# systemctl start postgresql-16.service
```

Проверка репликации: 
На хосте node1 в psql создадим базу otus_test и выведем список БД: 

```
postgres=# CREATE DATABASE otus_test;
CREATE DATABASE
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 otus_test | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

На хосте node2 также в psql также проверим список БД (команда \l), в списке БД должна появится БД otus_test. 

```
[root@node2 ~]# sudo -u postgres psql
psql (16.0)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 otus_test | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

Также можно проверить репликацию другим способом:
На хосте node1 в psql вводим команду:

```
postgres=# select * from pg_stat_replication;
  pid  | usesysid |   usename   | application_name |  client_addr  | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | repl
ay_lag | sync_priority | sync_state |          reply_time
-------+----------+-------------+------------------+---------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+-----
-------+---------------+------------+-------------------------------
 28629 |    16385 | replication | walreceiver      | 192.168.57.12 |                 |       43468 | 2023-10-17 11:46:57.254467-03 |          737 | streaming | 0/3000148 | 0/3000148 | 0/3000148 | 0/3000148  |           |           |
       |             0 | async      | 2023-10-17 11:50:18.783283-03
(1 row)
```

На хосте node2 в psql вводим команду: 

```
postgres=# select * from pg_stat_wal_receiver;
pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time        | slot_name |  sender_
host  | sender_port |                                                                                                                                         conninfo

-------+-----------+-------------------+-------------------+-------------+-------------+--------------+-------------------------------+-------------------------------+----------------+-------------------------------+-----------+---------
------+-------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------
 26157 | streaming | 0/3000000         |                 1 | 0/3000148   | 0/3000148   |            1 | 2023-10-17 11:50:27.852086-03 | 2023-10-17 11:50:27.852364-03 | 0/3000148      | 2023-10-17 11:46:57.382516-03 |           | 192.168.
57.11 |        5432 | user=replication password=******** channel_binding=prefer dbname=replication host=192.168.57.11 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1
.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
(1 row)
```

Вывод обеих команд должен быть не пустым. 

На этом настройка репликации завершена. 


В случае выхода из строя master-хоста (node1), на slave-сервере (node2) в psql необхоимо выполнить команду select pg_promote();
Также можно создать триггер-файл. Если в дальнейшем хост node1 заработает корректно, то для восстановления его работы (как master-сервера) необходимо: 
Настроить сервер node1 как slave-сервер
Также с помощью команды select pg_promote(); перевести режим его работы в master





Настройка резервного копирования

Настраивать резервное копирование мы будем с помощью утилиты Barman. В документации Barman рекомендуется разворачивать Barman на отдельном сервере. В этом случае потребуется настроить доступы между серверами по SSH-ключам. В данном руководстве мы будем разворачивать Barman на отдельном хосте, если Вам удобнее, для теста можно будет развернуть Barman на хосте node1. 

На хостах node1 и node2 необходимо установить утилиту barman-cli, для этого: 

```
neva@Uneva:~/Otus_Kaneva_dz28$ vagrant ssh node1
Last login: Wed Oct 18 12:56:03 2023 from 192.168.57.1
[vagrant@node1 ~]$ sudo -i
[root@node1 ~]# dnf install epel-release -y
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:03:45 ago on Wed Oct 18 12:55:51 2023.
Dependencies resolved.
=============================================================================================================================================================================================================================================
 Package                                                      Architecture                                           Version                                                    Repository                                              Size
=============================================================================================================================================================================================================================================
Installing:
 epel-release                                                 noarch                                                 8-11.el8                                                   extras                                                  24 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package

Total download size: 24 k
Installed size: 35 k
Downloading Packages:
epel-release-8-11.el8.noarch.rpm                                                                                                                                                                             137 kB/s |  24 kB     00:00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                                                         53 kB/s |  24 kB     00:00
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                                                     1/1
  Installing       : epel-release-8-11.el8.noarch                                                                                                                                                                                        1/1
  Running scriptlet: epel-release-8-11.el8.noarch                                                                                                                                                                                        1/1
  Verifying        : epel-release-8-11.el8.noarch                                                                                                                                                                                        1/1

Installed:
  epel-release-8-11.el8.noarch

Complete!
[root@node1 ~]# dnf install barman-cli
Failed to set locale, defaulting to C.UTF-8
Extra Packages for Enterprise Linux Modular 8 - x86_64                                                                                                                                                       526 kB/s | 733 kB     00:01
Extra Packages for Enterprise Linux 8 - x86_64                                                                                                                                                               3.3 MB/s |  16 MB     00:04
Last metadata expiration check: 0:00:01 ago on Wed Oct 18 12:59:53 2023.
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Dependencies resolved.
=============================================================================================================================================================================================================================================
 Package                                                        Architecture                                      Version                                                       Repository                                              Size
=============================================================================================================================================================================================================================================
Installing:
 barman-cli                                                     noarch                                            3.9.0-1PGDG.rhel8                                             pgdg-common                                             63 k
Installing dependencies:
 python3-argcomplete                                            noarch                                            1.9.3-6.el8                                                   appstream                                               60 k
 python3-barman                                                 noarch                                            3.9.0-1PGDG.rhel8                                             pgdg-common                                            515 k
 python3-setuptools                                             noarch                                            39.2.0-6.el8                                                  baseos                                                 163 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  4 Packages

Total download size: 801 k
Installed size: 3.4 M
Is this ok [y/N]: y
Downloading Packages:
(1/4): barman-cli-3.9.0-1PGDG.rhel8.noarch.rpm                                                                                                                                                               130 kB/s |  63 kB     00:00
(2/4): python3-argcomplete-1.9.3-6.el8.noarch.rpm                                                                                                                                                            123 kB/s |  60 kB     00:00
(3/4): python3-setuptools-39.2.0-6.el8.noarch.rpm                                                                                                                                                            284 kB/s | 163 kB     00:00
(4/4): python3-barman-3.9.0-1PGDG.rhel8.noarch.rpm                                                                                                                                                           1.3 MB/s | 515 kB     00:00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                                                        914 kB/s | 801 kB     00:00
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                                                     1/1
  Installing       : python3-setuptools-39.2.0-6.el8.noarch                                                                                                                                                                              1/4
  Installing       : python3-argcomplete-1.9.3-6.el8.noarch                                                                                                                                                                              2/4
  Installing       : python3-barman-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                             3/4
  Installing       : barman-cli-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                                 4/4
  Running scriptlet: barman-cli-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                                 4/4
  Verifying        : python3-argcomplete-1.9.3-6.el8.noarch                                                                                                                                                                              1/4
  Verifying        : python3-setuptools-39.2.0-6.el8.noarch                                                                                                                                                                              2/4
  Verifying        : barman-cli-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                                 3/4
  Verifying        : python3-barman-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                             4/4

Installed:
  barman-cli-3.9.0-1PGDG.rhel8.noarch                     python3-argcomplete-1.9.3-6.el8.noarch                     python3-barman-3.9.0-1PGDG.rhel8.noarch                     python3-setuptools-39.2.0-6.el8.noarch

Complete!
```


На хосте barman выполняем следующие настройки: 
отключаем firewalld и SElinux

```
[root@barman ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@barman ~]# setenforce 0
```

Устанавливаем epel-release: 

```
[root@barman ~]# dnf install epel-release -y
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:03:57 ago on Wed Oct 18 13:03:38 2023.
Dependencies resolved.
=============================================================================================================================================================================================================================================
 Package                                                      Architecture                                           Version                                                    Repository                                              Size
=============================================================================================================================================================================================================================================
Installing:
 epel-release                                                 noarch                                                 8-11.el8                                                   extras                                                  24 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package

Total download size: 24 k
Installed size: 35 k
Downloading Packages:
epel-release-8-11.el8.noarch.rpm                                                                                                                                                                              50 kB/s |  24 kB     00:00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                                                         34 kB/s |  24 kB     00:00
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                                                     1/1
  Installing       : epel-release-8-11.el8.noarch                                                                                                                                                                                        1/1
  Running scriptlet: epel-release-8-11.el8.noarch                                                                                                                                                                                        1/1
  Verifying        : epel-release-8-11.el8.noarch                                                                                                                                                                                        1/1

Installed:
  epel-release-8-11.el8.noarch

Complete!
```

Добавим postgres репозиторий: 

```
[root@barman ~]# dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
Failed to set locale, defaulting to C.UTF-8
Extra Packages for Enterprise Linux Modular 8 - x86_64                                                                                                                                                       584 kB/s | 733 kB     00:01
Extra Packages for Enterprise Linux 8 - x86_64                                                                                                                                                               3.4 MB/s |  16 MB     00:04
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
pgdg-redhat-repo-latest.noarch.rpm                                                                                                                                                                            15 kB/s |  13 kB     00:00
Dependencies resolved.
=============================================================================================================================================================================================================================================
 Package                                                       Architecture                                        Version                                                   Repository                                                 Size
=============================================================================================================================================================================================================================================
Installing:
 pgdg-redhat-repo                                              noarch                                              42.0-35PGDG                                               @commandline                                               13 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package

Total size: 13 k
Installed size: 15 k
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                                                     1/1
  Installing       : pgdg-redhat-repo-42.0-35PGDG.noarch                                                                                                                                                                                 1/1
  Verifying        : pgdg-redhat-repo-42.0-35PGDG.noarch                                                                                                                                                                                 1/1

Installed:
  pgdg-redhat-repo-42.0-35PGDG.noarch

Complete!
```

Исключаем старый postgresql модуль: 

```
[root@barman ~]# yum -qy module disable postgresql
Failed to set locale, defaulting to C.UTF-8
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
````

Устанавливаем пакеты barman и postgresql-client:

```
[root@barman ~]#  dnf install barman-cli barman postgresql14
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:00:40 ago on Wed Oct 18 13:09:36 2023.
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Module yaml error: Unexpected key in data: static_context [line 9 col 3]
Dependencies resolved.
=============================================================================================================================================================================================================================================
 Package                                                   Architecture                                 Version                                                                      Repository                                         Size
=============================================================================================================================================================================================================================================
Installing:
 barman                                                    noarch                                       3.9.0-1PGDG.rhel8                                                            pgdg-common                                        54 k
 barman-cli                                                noarch                                       3.9.0-1PGDG.rhel8                                                            pgdg-common                                        63 k
 postgresql14                                              x86_64                                       14.9-2PGDG.rhel8                                                             pgdg14                                            1.5 M
Upgrading:
 chkconfig                                                 x86_64                                       1.19.1-1.el8                                                                 baseos                                            198 k
 platform-python-pip                                       noarch                                       9.0.3-20.el8                                                                 baseos                                            1.7 M
Installing dependencies:
 lz4                                                       x86_64                                       1.8.3-3.el8_4                                                                baseos                                            103 k
 postgresql14-libs                                         x86_64                                       14.9-2PGDG.rhel8                                                             pgdg14                                            280 k
 python3-argcomplete                                       noarch                                       1.9.3-6.el8                                                                  appstream                                          60 k
 python3-barman                                            noarch                                       3.9.0-1PGDG.rhel8                                                            pgdg-common                                       515 k
 python3-pip                                               noarch                                       9.0.3-20.el8                                                                 appstream                                          20 k
 python3-psycopg2                                          x86_64                                       2.9.5-3.rhel8                                                                pgdg-common                                       188 k
 python3-setuptools                                        noarch                                       39.2.0-6.el8                                                                 baseos                                            163 k
 python36                                                  x86_64                                       3.6.8-38.module_el8.5.0+895+a459eca8                                         appstream                                          19 k
Enabling module streams:
 python36                                                                                               3.6

Transaction Summary
=============================================================================================================================================================================================================================================
Install  11 Packages
Upgrade   2 Packages

Total download size: 4.9 M
Is this ok [y/N]: y
Downloading Packages:
(1/13): python3-pip-9.0.3-20.el8.noarch.rpm                                                                                                                                                                   46 kB/s |  20 kB     00:00
(2/13): python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_64.rpm                                                                                                                                              44 kB/s |  19 kB     00:00
(3/13): python3-argcomplete-1.9.3-6.el8.noarch.rpm                                                                                                                                                           125 kB/s |  60 kB     00:00
(4/13): lz4-1.8.3-3.el8_4.x86_64.rpm                                                                                                                                                                         621 kB/s | 103 kB     00:00
(5/13): python3-setuptools-39.2.0-6.el8.noarch.rpm                                                                                                                                                           706 kB/s | 163 kB     00:00
(6/13): barman-3.9.0-1PGDG.rhel8.noarch.rpm                                                                                                                                                                   48 kB/s |  54 kB     00:01
(7/13): barman-cli-3.9.0-1PGDG.rhel8.noarch.rpm                                                                                                                                                               51 kB/s |  63 kB     00:01
(8/13): python3-psycopg2-2.9.5-3.rhel8.x86_64.rpm                                                                                                                                                            425 kB/s | 188 kB     00:00
(9/13): postgresql14-libs-14.9-2PGDG.rhel8.x86_64.rpm                                                                                                                                                        720 kB/s | 280 kB     00:00
(10/13): python3-barman-3.9.0-1PGDG.rhel8.noarch.rpm                                                                                                                                                         277 kB/s | 515 kB     00:01
(11/13): chkconfig-1.19.1-1.el8.x86_64.rpm                                                                                                                                                                   1.1 MB/s | 198 kB     00:00
(12/13): platform-python-pip-9.0.3-20.el8.noarch.rpm                                                                                                                                                         3.2 MB/s | 1.7 MB     00:00
(13/13): postgresql14-14.9-2PGDG.rhel8.x86_64.rpm                                                                                                                                                            623 kB/s | 1.5 MB     00:02
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                                                        1.1 MB/s | 4.9 MB     00:04
warning: /var/cache/dnf/pgdg-common-38b6c5045045f841/packages/barman-3.9.0-1PGDG.rhel8.noarch.rpm: Header V4 DSA/SHA1 Signature, key ID 442df0f8: NOKEY
PostgreSQL common RPMs for RHEL / Rocky 8 - x86_64                                                                                                                                                           1.6 MB/s | 1.7 kB     00:00
Importing GPG key 0x442DF0F8:
 Userid     : "PostgreSQL RPM Building Project <pgsql-pkg-yum@postgresql.org>"
 Fingerprint: 68C9 E2B9 1A37 D136 FE74 D176 1F16 D2E1 442D F0F8
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
Is this ok [y/N]: y
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                                                     1/1
  Upgrading        : chkconfig-1.19.1-1.el8.x86_64                                                                                                                                                                                      1/15
  Installing       : postgresql14-libs-14.9-2PGDG.rhel8.x86_64                                                                                                                                                                          2/15
  Running scriptlet: postgresql14-libs-14.9-2PGDG.rhel8.x86_64                                                                                                                                                                          2/15
  Installing       : python3-setuptools-39.2.0-6.el8.noarch                                                                                                                                                                             3/15
  Installing       : python3-psycopg2-2.9.5-3.rhel8.x86_64                                                                                                                                                                              4/15
  Upgrading        : platform-python-pip-9.0.3-20.el8.noarch                                                                                                                                                                            5/15
  Installing       : python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_64                                                                                                                                                               6/15
  Running scriptlet: python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_64                                                                                                                                                               6/15
  Installing       : python3-pip-9.0.3-20.el8.noarch                                                                                                                                                                                    7/15
  Installing       : lz4-1.8.3-3.el8_4.x86_64                                                                                                                                                                                           8/15
  Installing       : python3-argcomplete-1.9.3-6.el8.noarch                                                                                                                                                                             9/15
  Installing       : python3-barman-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                           10/15
  Running scriptlet: barman-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                                   11/15
  Installing       : barman-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                                   11/15
  Installing       : barman-cli-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                               12/15
  Installing       : postgresql14-14.9-2PGDG.rhel8.x86_64                                                                                                                                                                              13/15
  Running scriptlet: postgresql14-14.9-2PGDG.rhel8.x86_64                                                                                                                                                                              13/15
  Cleanup          : platform-python-pip-9.0.3-18.el8.noarch                                                                                                                                                                           14/15
  Cleanup          : chkconfig-1.13-2.el8.x86_64                                                                                                                                                                                       15/15
  Running scriptlet: chkconfig-1.13-2.el8.x86_64                                                                                                                                                                                       15/15
  Verifying        : python3-argcomplete-1.9.3-6.el8.noarch                                                                                                                                                                             1/15
  Verifying        : python3-pip-9.0.3-20.el8.noarch                                                                                                                                                                                    2/15
  Verifying        : python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_64                                                                                                                                                               3/15
  Verifying        : lz4-1.8.3-3.el8_4.x86_64                                                                                                                                                                                           4/15
  Verifying        : python3-setuptools-39.2.0-6.el8.noarch                                                                                                                                                                             5/15
  Verifying        : barman-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                                    6/15
  Verifying        : barman-cli-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                                7/15
  Verifying        : python3-barman-3.9.0-1PGDG.rhel8.noarch                                                                                                                                                                            8/15
  Verifying        : python3-psycopg2-2.9.5-3.rhel8.x86_64                                                                                                                                                                              9/15
  Verifying        : postgresql14-14.9-2PGDG.rhel8.x86_64                                                                                                                                                                              10/15
  Verifying        : postgresql14-libs-14.9-2PGDG.rhel8.x86_64                                                                                                                                                                         11/15
  Verifying        : chkconfig-1.19.1-1.el8.x86_64                                                                                                                                                                                     12/15
  Verifying        : chkconfig-1.13-2.el8.x86_64                                                                                                                                                                                       13/15
  Verifying        : platform-python-pip-9.0.3-20.el8.noarch                                                                                                                                                                           14/15
  Verifying        : platform-python-pip-9.0.3-18.el8.noarch                                                                                                                                                                           15/15

Upgraded:
  chkconfig-1.19.1-1.el8.x86_64                                                                                    platform-python-pip-9.0.3-20.el8.noarch

Installed:
  barman-3.9.0-1PGDG.rhel8.noarch                            barman-cli-3.9.0-1PGDG.rhel8.noarch           lz4-1.8.3-3.el8_4.x86_64              postgresql14-14.9-2PGDG.rhel8.x86_64        postgresql14-libs-14.9-2PGDG.rhel8.x86_64
  python3-argcomplete-1.9.3-6.el8.noarch                     python3-barman-3.9.0-1PGDG.rhel8.noarch       python3-pip-9.0.3-20.el8.noarch       python3-psycopg2-2.9.5-3.rhel8.x86_64       python3-setuptools-39.2.0-6.el8.noarch
  python36-3.6.8-38.module_el8.5.0+895+a459eca8.x86_64

Complete!

```

Переходим в пользователя barman и генерируем ssh-ключ: 

```
[root@barman barman]# su barman
bash-4.4$ cd
bash-4.4$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/barman/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /var/lib/barman/.ssh/id_rsa.
Your public key has been saved in /var/lib/barman/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:rUxXdpvQvwZVj7N76vPQJYQRDdBUuge8cH4dFTJYE7I barman@barman
The key's randomart image is:
+---[RSA 4096]----+
|          .=B@o.+|
|           o+==.+|
|          .EX *o.|
|         . * B.B.|
|        S o +.B +|
|       o o   o.+o|
|        o     ooo|
|              o+ |
|             .oo.|
+----[SHA256]-----+
```

На хосте node1: 
Переходим в пользователя postgres и генерируем ssh-ключ: 

```
[root@node1 ~]# su postgres
bash-4.4$ cd
bash-4.4$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/pgsql/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /var/lib/pgsql/.ssh/id_rsa.
Your public key has been saved in /var/lib/pgsql/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:Hc9cwdgAltZeg0iab6KRnk69tIX2C+F2UhQaeGuryyc postgres@node1
The key's randomart image is:
+---[RSA 4096]----+
|       ...==.*.  |
|      . .*+.+ =. |
|       .+oo. ... |
|       .o+ =..   |
|      o.S.= +    |
|     . *.*       |
|      =.X o      |
|     +E+.O       |
|      ++o o.     |
+----[SHA256]-----+
```

После генерации ключа, выводим содержимое файла ~/.ssh/id_rsa.pub: 

```
bash-4.4$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDpLBoLlgIx/mufQWgjbWp98D09pAbrfKEEa9Gi1qydXOvJk7/hlTKccSSxwfLp68e2Ymm0A928RADqqNpG4bvW9GodyP8Nm2hIp8cdly0BzQHvbAjY9uKYN7r6IqF1P9YFpa9wpyp+H4R0UAZzQgR7WcizyWktYBjI4ohV32sgW3pyFAWofVq8XZmbz5bRNnl5EXpsYXLf0wh5r+h0szqGhyzPPZ1pALb7rMMbzRmmxkigeIPjjIu+UTR3/jWkTfGSOGEsqVR6ogD94PJVkDcZK2rdKBUnGn5gGQPos97OBTonl2O72d+uV0OFvqEIy6A74gI+ON/SPhTy2cNo/n2vcEdDt4SSOT6pui3tiNlxjfXCaWXiLlWwNPoKw9tqUgDgncQ4LtO8Chd8L+H4aO/xN+1G64PmKeLxJs9D193eVTR4LsHSxNLgSUMMJjJlVrZRDECSQwsIbr07RIQ3IxjOUu802eQHPqusm4k/0hMBOwRmsifcX8jVblCJoDtM+270NktlxEm/oYgcreXm5x9h9fUINAEOzV19APxutqg0hUOpCSoPRnaZ+ZBHCRxfOdUOuz5PRsTkgGvSxXh8xB1iZ/N8xm6Xn0t5vnswFSHl6zuOodQtGaThx3kd+8pOerUdydxieFjLRlqEZibtIT0QN6mmCULfxWpNd7iSiTlTWQ== postgres@node1
```

Копируем содержимое файла на сервер barman в файл /var/lib/barman/.ssh/authorized_keys


```
bash-4.4$ vi /var/lib/barman/.ssh/authorized_keys
bash-4.4$ cat /var/lib/barman/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDpLBoLlgIx/mufQWgjbWp98D09pAbrfKEEa9Gi1qydXOvJk7/hlTKccSSxwfLp68e2Ymm0A928RADqqNpG4bvW9GodyP8Nm2hIp8cdly0BzQHvbAjY9uKYN7r6IqF1P9YFpa9wpyp+H4R0UAZzQgR7WcizyWktYBjI4ohV32sgW3pyFAWofVq8XZmbz5bRNnl5EXpsYXLf0wh5r+h0szqGhyzPPZ1pALb7rMMbzRmmxkigeIPjjIu+UTR3/jWkTfGSOGEsqVR6ogD94PJVkDcZK2rdKBUnGn5gGQPos97OBTonl2O72d+uV0OFvqEIy6A74gI+ON/SPhTy2cNo/n2vcEdDt4SSOT6pui3tiNlxjfXCaWXiLlWwNPoKw9tqUgDgncQ4LtO8Chd8L+H4aO/xN+1G64PmKeLxJs9D193eVTR4LsHSxNLgSUMMJjJlVrZRDECSQwsIbr07RIQ3IxjOUu802eQHPqusm4k/0hMBOwRmsifcX8jVblCJoDtM+270NktlxEm/oYgcreXm5x9h9fUINAEOzV19APxutqg0hUOpCSoPRnaZ+ZBHCRxfOdUOuz5PRsTkgGvSxXh8xB1iZ/N8xm6Xn0t5vnswFSHl6zuOodQtGaThx3kd+8pOerUdydxieFjLRlqEZibtIT0QN6mmCULfxWpNd7iSiTlTWQ== postgres@node1
```

На node1 в psql создаём пользователя barman c правами суперпользователя: 

```
[root@node1 ~]# sudo -u postgres psql
postgres=# CREATE USER barman WITH SUPERUSER Encrypted PASSWORD 'Otus2022!';
CREATE ROLE
```

В файл /var/lib/pgsql/14/data/pg_hba.conf добавляем разрешения для пользователя barman: 

```
[root@node1 pgsql]# vi /var/lib/pgsql/14/data/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
host    replication replicator         192.168.57.11/32         scram-sha-256
host    replication replicator         192.168.57.12/32          scram-sha-256
host    all         barman             192.168.57.13/32        scram-sha-256
host    replication barman           192.168.57.13/32      scram-sha-256
```

Перезапускаем службу postgresql-14

```
[root@node1 pgsql]#  systemctl restart postgresql-14
```

В psql создадим тестовую базу otus:

```
[root@node1 pgsql]# sudo -u postgres psql
psql (14.9)
Type "help" for help.

postgres=# CREATE DATABASE otus;
CREATE DATABASE
```

В базе создаём таблицу test в базе otus: 

```
postgres=# \c otus;
You are now connected to database "otus" as user "postgres".
otus=# CREATE TABLE test (id int, name varchar(30));
CREATE TABLE

otus=# INSERT INTO test VALUES (1, 'alex');
INSERT 0 1
otus=#
```

На хосте barman: 
После генерации ключа, выводим содержимое файла и копируем содержимое файла на сервер node1 в файл /var/lib/pgsql/.ssh/authorized_keys

```
bash-4.4$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCtyfbRo2caA8qMkRHSPZuSvZc4mtbvhZ3pODCRv1Tv/Uxae0JJ1VU+GazIpQV8iKJomDbSLY838szReBJzCFxmGCigY1/jaMXEt3MQWB7m0E0oecKWqHMs5YGVY7oZrlEQHiJiwz0BvW0gKc+q7Fl/BlCBGvGaXWrTgIhtdQRdAIpDiZaL+deNva4uOADdD5iaauQ5/zcoTi19LxcQ79SJX3GdiC3VxUVEsR+6Bu1x2m4/kMpMmBRap2hz/Gk2Rx7uYoxD/5LC0e3neQYE3gkOWQJAjMK1XOhyqBb3zv/yrpR4rnGZtcMl9x6SftZyHodvhdmHWAe0lY7QU56i5TVWWP5yAroLimMuwBlivcRlNyJ5xa9FSwFkxbKBRijm1100nn3MjkTX2A9KfFsD1JFiKJPgjv6ia1HVbamllCd3MhFwxTWplIumqS7cogjYtmpa6QGt/9K++FtczZgrgklBpNiY1/Rn1WgCYGKFIzvbm1AtEbqhDjbcIvf14nRkk4FiXtOlNkfb12m7h84SLBPkviZEcIykhN5D/IIPj8AZOMlgz/HHynxLGlFW4rfJuWsimWrZZKA4HF0GhVvbz+k/uWotK33hP4fRuE0gqWovUaXz9TiMfBr/eAG1MN2PRrDTKGxEhpbljv8fb1KTe2R2idyHjKEbvl+rYRgTanDKdQ== barman@barman

bash-4.4$ vi /var/lib/pgsql/.ssh/authorized_keys
bash-4.4$ cat /var/lib/pgsql/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCtyfbRo2caA8qMkRHSPZuSvZc4mtbvhZ3pODCRv1Tv/Uxae0JJ1VU+GazIpQV8iKJomDbSLY838szReBJzCFxmGCigY1/jaMXEt3MQWB7m0E0oecKWqHMs5YGVY7oZrlEQHiJiwz0BvW0gKc+q7Fl/BlCBGvGaXWrTgIhtdQRdAIpDiZaL+deNva4uOADdD5iaauQ5/zcoTi19LxcQ79SJX3GdiC3VxUVEsR+6Bu1x2m4/kMpMmBRap2hz/Gk2Rx7uYoxD/5LC0e3neQYE3gkOWQJAjMK1XOhyqBb3zv/yrpR4rnGZtcMl9x6SftZyHodvhdmHWAe0lY7QU56i5TVWWP5yAroLimMuwBlivcRlNyJ5xa9FSwFkxbKBRijm1100nn3MjkTX2A9KfFsD1JFiKJPgjv6ia1HVbamllCd3MhFwxTWplIumqS7cogjYtmpa6QGt/9K++FtczZgrgklBpNiY1/Rn1WgCYGKFIzvbm1AtEbqhDjbcIvf14nRkk4FiXtOlNkfb12m7h84SLBPkviZEcIykhN5D/IIPj8AZOMlgz/HHynxLGlFW4rfJuWsimWrZZKA4HF0GhVvbz+k/uWotK33hP4fRuE0gqWovUaXz9TiMfBr/eAG1MN2PRrDTKGxEhpbljv8fb1KTe2R2idyHjKEbvl+rYRgTanDKdQ== barman@barman
````

Находясь в пользователе barman создаём файл ~/.pgpass со следующим содержимым: 

```
192.168.57.11:5432:*:barman:Otus2022!
```

В данном файле указываются реквизиты доступа для postgres. Через знак двоеточия пишутся следующие параметры: 
ip-адрес
порт postgres
имя БД (* означает подключение к любой БД)
имя пользователя
пароль пользователя
Файл должен быть с правами 600, владелец файла barman. 


```
bash-4.4$ chmod 0600  ~/.pgpass
```

После создания postgres-пользователя barman необходимо проверить, что права для пользователя настроены корректно: 
Проверяем возможность подключения к postgres-серверу:

```
bash-4.4$  psql -h 192.168.57.11 -U barman -d postgres
psql (16.0)
Type "help" for help.
postgres=>
```

Проверяем репликацию: 

```
bash-4.4$  psql -h 192.168.57.11 -U barman -c "IDENTIFY_SYSTEM" replication=1
      systemid       | timeline |  xlogpos  | dbname
---------------------+----------+-----------+--------
 7291281386332005577 |        1 | 0/301A158 |
(1 row)

```

Создаём файл /etc/barman.conf со следующим содержимым (владельцем файла должен быть пользователь barman):
Владельцем файла должен быть пользователь barman.

```
; Barman, Backup and Recovery Manager for PostgreSQL
; http://www.pgbarman.org/ - http://www.enterprisedb.com/
;
; Main configuration file

[barman]
; System user
barman_user = barman

; Directory of configuration files. Place your sections in separate files with .conf extension
; For example place the 'main' server section in /etc/barman.d/main.conf
configuration_files_directory = /etc/barman.d

; Main directory
barman_home = /var/lib/barman

; Locks directory - default: %(barman_home)s
;barman_lock_directory = /var/run/barman

; Log location
log_file = /var/log/barman/barman.log

; Log level (see https://docs.python.org/3/library/logging.html#levels)
log_level = INFO

; Default compression level: possible values are None (default), bzip2, gzip, pigz, pygzip or pybzip2
compression = gzip

; Pre/post backup hook scripts
;pre_backup_script = env | grep ^BARMAN
;pre_backup_retry_script = env | grep ^BARMAN
;post_backup_retry_script = env | grep ^BARMAN
;post_backup_script = env | grep ^BARMAN

; Pre/post archive hook scripts
;pre_archive_script = env | grep ^BARMAN
;pre_archive_retry_script = env | grep ^BARMAN
;post_archive_retry_script = env | grep ^BARMAN
;post_archive_script = env | grep ^BARMAN

; Pre/post delete scripts
;pre_delete_script = env | grep ^BARMAN
;pre_delete_retry_script = env | grep ^BARMAN
;post_delete_retry_script = env | grep ^BARMAN
;post_delete_script = env | grep ^BARMAN

; Pre/post wal delete scripts
;pre_wal_delete_script = env | grep ^BARMAN
;pre_wal_delete_retry_script = env | grep ^BARMAN
;post_wal_delete_retry_script = env | grep ^BARMAN
;post_wal_delete_script = env | grep ^BARMAN

; Global bandwidth limit in kilobytes per second - default 0 (meaning no limit)
;bandwidth_limit = 4000

; Number of parallel jobs for backup and recovery via rsync (default 1)
;parallel_jobs = 1

; Immediate checkpoint for backup command - default false
;immediate_checkpoint = false

; Enable network compression for data transfers - default false
;network_compression = false

; Number of retries of data copy during base backup after an error - default 0
;basebackup_retry_times = 0

; Number of seconds of wait after a failed copy, before retrying - default 30
;basebackup_retry_sleep = 30

; Maximum execution time, in seconds, per server
; for a barman check command - default 30
;check_timeout = 30

; Time frame that must contain the latest backup date.
; If the latest backup is older than the time frame, barman check
; command will report an error to the user.
; If empty, the latest backup is always considered valid.
; Syntax for this option is: "i (DAYS | WEEKS | MONTHS | HOURS)" where i is an
; integer > 0 which identifies the number of days | weeks | months of
; validity of the latest backup for this check. Also known as 'smelly backup'.
;last_backup_maximum_age =

; Time frame that must contain the latest WAL file
; If the latest WAL file is older than the time frame, barman check
; command will report an error to the user.
; Syntax for this option is: "i (DAYS | WEEKS | MONTHS | HOURS)" where i is an
; integer > 0
;last_wal_maximum_age =

; Minimum number of required backups (redundancy)
minimum_redundancy = 1
last_backup_maximum_age = 4 DAYS

; Global retention policy (REDUNDANCY or RECOVERY WINDOW)
; Examples of retention policies
; Retention policy (disabled, default)
;retention_policy =
; Retention policy (based on redundancy)
backup_method = rsync
archiver = on
retention_policy = REDUNDANCY 3
immediate_checkpoint = true
; Retention policy (based on recovery window)
;retention_policy = RECOVERY WINDOW OF 4 WEEKS

[root@barman ~]# vi /etc/barman.d/node1.conf
[root@barman ~]# cat /etc/barman.d/node1.conf
[node1]
#Описание задания
description = "backup node1"
#Команда подключения к хосту node1
ssh_command = ssh postgres@192.168.57.11
#Команда для подключения к postgres-серверу
conninfo = host=192.168.57.11 user=barman port=5432 dbname=postgres
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 7 days
wal_retention_policy = main
streaming_archiver=on
#Указание префикса, который будет использоваться как $PATH на хосте node1
path_prefix = /usr/pgsql-16/bin/
#настройки слота
create_slot = auto
slot_name = node1
#Команда для потоковой передачи от postgres-сервера
streaming_conninfo = host=192.168.57.11 user=barman
#Тип выполняемого бекапа
backup_method = postgres
archiver = off


[root@barman etc]# chown barman:barman /etc/barman.conf
```

Создаём файл /etc/barman.d/node1.conf со следующим содержимым (владельцем файла должен быть пользователь barman):

```
[node1]
#Описание задания
description = "backup node1"
#Команда подключения к хосту node1
ssh_command = ssh postgres@192.168.57.11
#Команда для подключения к postgres-серверу
conninfo = host=192.168.57.11 user=barman port=5432 dbname=postgres
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 7 days
wal_retention_policy = main
streaming_archiver=on
#Указание префикса, который будет использоваться как $PATH на хосте node1
path_prefix = /usr/pgsql-16/bin/
#настройки слота
create_slot = auto
slot_name = node1
#Команда для потоковой передачи от postgres-сервера
streaming_conninfo = host=192.168.57.11 user=barman
#Тип выполняемого бекапа
backup_method = postgres
archiver = off

[root@barman etc]# chown barman:barman /etc/barman.d/node1.conf
```

На этом настройка бекапа завершена. Теперь проверим работу barman: 

```
[root@barman barman]# su barman
bash-4.4$ barman switch-wal node1
The WAL file 000000010000000000000004 has been closed on server 'node1'
bash-4.4$ barman cron
Starting WAL archiving for server node1
Server node1:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
        backup minimum size: OK (0 B)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK (no system Id stored on disk)
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK

```

После этого запускаем резервную копию:

```
bash-4.4$ barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20231019T074628
Backup start at LSN: 0/5000060 (000000010000000000000005, 00000060)
Starting backup copy via pg_basebackup for 20231019T074628
Copy done (time: 2 seconds)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
        000000010000000000000003 from server node1 has been removed
        000000010000000000000004 from server node1 has been removed
Backup size: 33.6 MiB
Backup end at LSN: 0/7000000 (000000010000000000000006, 00000000)
Backup completed (start time: 2023-10-19 07:46:28.898803, elapsed time: 3 seconds)
Processing xlog segments from streaming for node1
        000000010000000000000005
        000000010000000000000006
```

На этом процесс настройки бекапа закончен
Проверка восстановления из бекапов:
На хосте node1 в psql удаляем базы Otus: 

```
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# DROP DATABASE otus;
DROP DATABASE
```

Далее на хосте barman запустим восстановление: 

```
bash-4.4$ barman list-backup node1
node1 20231019T144630 - Thu Oct 19 11:46:32 2023 - Size: 25.3 MiB - WAL Size: 0 B
node1 20231019T144610 - Thu Oct 19 11:46:12 2023 - Size: 25.3 MiB - WAL Size: 32.2 KiB
node1 20231019T074628 - Thu Oct 19 04:46:31 2023 - Size: 33.6 MiB - WAL Size: 48.8 KiB
bash-4.4$ barman recover node1 20231019T144630 /var/lib/pgsql/14/data/ --remote-ssh-comman "ssh postgres@192.168.57.11"
Starting remote restore for server node1 using backup 20231019T144630
Destination directory: /var/lib/pgsql/14/data/
Remote command: ssh postgres@192.168.57.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2023-10-19 14:47:12.945679+00:00, elapsed time: 6 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```

На хосте node1 потребуется перезапустить postgresql-сервер и снова проверить список БД.

	

Результат выполнения команды vagrant up с ansible playbook:

```
neva@Uneva:~/Otus_Kaneva_dz28$ vagrant up
Bringing machine 'node1' up with 'virtualbox' provider...
Bringing machine 'node2' up with 'virtualbox' provider...
Bringing machine 'barman' up with 'virtualbox' provider...
==> node1: Importing base box 'centos/8'...
==> node1: Matching MAC address for NAT networking...
==> node1: Checking if box 'centos/8' version '2011.0' is up to date...
==> node1: Setting the name of the VM: Otus_Kaneva_dz28_node1_1697798791295_61933
==> node1: Clearing any previously set network interfaces...
==> node1: Preparing network interfaces based on configuration...
    node1: Adapter 1: nat
    node1: Adapter 2: hostonly
==> node1: Forwarding ports...
    node1: 22 (guest) => 2222 (host) (adapter 1)
==> node1: Running 'pre-boot' VM customizations...
==> node1: Booting VM...
==> node1: Waiting for machine to boot. This may take a few minutes...
    node1: SSH address: 127.0.0.1:2222
    node1: SSH username: vagrant
    node1: SSH auth method: private key
    node1:
    node1: Vagrant insecure key detected. Vagrant will automatically replace
    node1: this with a newly generated keypair for better security.
    node1:
    node1: Inserting generated public key within guest...
    node1: Removing insecure key from the guest if it's present...
    node1: Key inserted! Disconnecting and reconnecting using new SSH key...
==> node1: Machine booted and ready!
==> node1: Checking for guest additions in VM...
    node1: No guest additions were detected on the base box for this VM! Guest
    node1: additions are required for forwarded ports, shared folders, host only
    node1: networking, and more. If SSH fails on this machine, please install
    node1: the guest additions and repackage the box to continue.
    node1:
    node1: This is not an error message; everything may continue to work properly,
    node1: in which case you may ignore this message.
==> node1: Setting hostname...
==> node1: Configuring and enabling network interfaces...
==> node1: Rsyncing folder: /home/neva/Otus_Kaneva_dz28/ => /vagrant
==> node2: Importing base box 'centos/8'...
==> node2: Matching MAC address for NAT networking...
==> node2: Checking if box 'centos/8' version '2011.0' is up to date...
==> node2: Setting the name of the VM: Otus_Kaneva_dz28_node2_1697798858849_71726
==> node2: Fixed port collision for 22 => 2222. Now on port 2200.
==> node2: Clearing any previously set network interfaces...
==> node2: Preparing network interfaces based on configuration...
    node2: Adapter 1: nat
    node2: Adapter 2: hostonly
==> node2: Forwarding ports...
    node2: 22 (guest) => 2200 (host) (adapter 1)
==> node2: Running 'pre-boot' VM customizations...
==> node2: Booting VM...
==> node2: Waiting for machine to boot. This may take a few minutes...
    node2: SSH address: 127.0.0.1:2200
    node2: SSH username: vagrant
    node2: SSH auth method: private key
    node2:
    node2: Vagrant insecure key detected. Vagrant will automatically replace
    node2: this with a newly generated keypair for better security.
    node2:
    node2: Inserting generated public key within guest...
    node2: Removing insecure key from the guest if it's present...
    node2: Key inserted! Disconnecting and reconnecting using new SSH key...
==> node2: Machine booted and ready!
==> node2: Checking for guest additions in VM...
    node2: No guest additions were detected on the base box for this VM! Guest
    node2: additions are required for forwarded ports, shared folders, host only
    node2: networking, and more. If SSH fails on this machine, please install
    node2: the guest additions and repackage the box to continue.
    node2:
    node2: This is not an error message; everything may continue to work properly,
    node2: in which case you may ignore this message.
==> node2: Setting hostname...
==> node2: Configuring and enabling network interfaces...
==> node2: Rsyncing folder: /home/neva/Otus_Kaneva_dz28/ => /vagrant
==> barman: Importing base box 'centos/8'...
==> barman: Matching MAC address for NAT networking...
==> barman: Checking if box 'centos/8' version '2011.0' is up to date...
==> barman: Setting the name of the VM: Otus_Kaneva_dz28_barman_1697798925874_59905
==> barman: Fixed port collision for 22 => 2222. Now on port 2201.
==> barman: Clearing any previously set network interfaces...
==> barman: Preparing network interfaces based on configuration...
    barman: Adapter 1: nat
    barman: Adapter 2: hostonly
==> barman: Forwarding ports...
    barman: 22 (guest) => 2201 (host) (adapter 1)
==> barman: Running 'pre-boot' VM customizations...
==> barman: Booting VM...
==> barman: Waiting for machine to boot. This may take a few minutes...
    barman: SSH address: 127.0.0.1:2201
    barman: SSH username: vagrant
    barman: SSH auth method: private key
    barman:
    barman: Vagrant insecure key detected. Vagrant will automatically replace
    barman: this with a newly generated keypair for better security.
    barman:
    barman: Inserting generated public key within guest...
    barman: Removing insecure key from the guest if it's present...
    barman: Key inserted! Disconnecting and reconnecting using new SSH key...
==> barman: Machine booted and ready!
==> barman: Checking for guest additions in VM...
    barman: No guest additions were detected on the base box for this VM! Guest
    barman: additions are required for forwarded ports, shared folders, host only
    barman: networking, and more. If SSH fails on this machine, please install
    barman: the guest additions and repackage the box to continue.
    barman:
    barman: This is not an error message; everything may continue to work properly,
    barman: in which case you may ignore this message.
==> barman: Setting hostname...
==> barman: Configuring and enabling network interfaces...
==> barman: Rsyncing folder: /home/neva/Otus_Kaneva_dz28/ => /vagrant
==> barman: Running provisioner: ansible...
Vagrant gathered an unknown Ansible version:


and falls back on the compatibility mode '1.8'.

Alternatively, the compatibility mode can be specified in your Vagrantfile:
https://www.vagrantup.com/docs/provisioning/ansible_common.html#compatibility_mode

    barman: Running ansible-playbook...

PLAY [Postgres] ****************************************************************

TASK [Gathering Facts] *********************************************************
ok: [barman]
ok: [node2]
ok: [node1]

TASK [replace repos] ***********************************************************
changed: [barman] => (item=/etc/yum.repos.d/CentOS-Linux-AppStream.repo)
changed: [node2] => (item=/etc/yum.repos.d/CentOS-Linux-AppStream.repo)
changed: [node1] => (item=/etc/yum.repos.d/CentOS-Linux-AppStream.repo)
changed: [barman] => (item=/etc/yum.repos.d/CentOS-Linux-BaseOS.repo)
changed: [node1] => (item=/etc/yum.repos.d/CentOS-Linux-BaseOS.repo)
changed: [node2] => (item=/etc/yum.repos.d/CentOS-Linux-BaseOS.repo)

TASK [replace repos2] **********************************************************
changed: [barman] => (item=/etc/yum.repos.d/CentOS-Linux-AppStream.repo)
changed: [node2] => (item=/etc/yum.repos.d/CentOS-Linux-AppStream.repo)
changed: [barman] => (item=/etc/yum.repos.d/CentOS-Linux-BaseOS.repo)
changed: [node2] => (item=/etc/yum.repos.d/CentOS-Linux-BaseOS.repo)
changed: [node1] => (item=/etc/yum.repos.d/CentOS-Linux-AppStream.repo)
changed: [node1] => (item=/etc/yum.repos.d/CentOS-Linux-BaseOS.repo)

TASK [install base tools] ******************************************************
changed: [node2]
changed: [barman]
changed: [node1]

PLAY [install postgres 14 and set up replication] ******************************

TASK [Gathering Facts] *********************************************************
ok: [node1]
ok: [node2]

TASK [install_postgres : disable firewalld service] ****************************
ok: [node1]
ok: [node2]

TASK [install_postgres : Disable SELinux] **************************************
[WARNING]: SELinux state temporarily changed from 'enforcing' to 'permissive'.
State change will take effect next reboot.
changed: [node1]
changed: [node2]

TASK [install_postgres : Ensure SELinux is set to disable mode] ****************
ok: [node2]
ok: [node1]

TASK [install_postgres : install repo] *****************************************
changed: [node1]
changed: [node2]

TASK [install_postgres : disable old postgresql module] ************************
changed: [node2]
changed: [node1]

TASK [install_postgres : install postgresql-server 14] *************************
changed: [node2]
changed: [node1]

TASK [install_postgres : check init] *******************************************
ok: [node1]
ok: [node2]

TASK [install_postgres : initialization setup] *********************************
changed: [node1]
changed: [node2]

TASK [install_postgres : enable and start service] *****************************
changed: [node2]
changed: [node1]

TASK [postgres_replication : install base tools] *******************************
changed: [node2]
changed: [node1]

TASK [postgres_replication : Create replicator user] ***************************
skipping: [node2]
[WARNING]: Module remote_tmp /var/lib/pgsql/.ansible/tmp did not exist and was
created with a mode of 0700, this may cause issues when running as another
user. To avoid this, create the remote_tmp dir with the correct permissions
manually
changed: [node1]

TASK [postgres_replication : stop postgresql-server on node2] ******************
skipping: [node1]
changed: [node2]

TASK [postgres_replication : copy postgresql.conf] *****************************
skipping: [node2]
changed: [node1]

TASK [postgres_replication : copy pg_hba.conf] *********************************
skipping: [node2]
changed: [node1]

TASK [postgres_replication : restart postgresql-server on node1] ***************
skipping: [node2]
changed: [node1]

TASK [postgres_replication : Remove files from data catalog] *******************
skipping: [node1]
changed: [node2]

TASK [postgres_replication : copy files from master to slave] ******************
skipping: [node1]
changed: [node2]

TASK [postgres_replication : copy postgresql.conf] *****************************
skipping: [node1]
ok: [node2]

TASK [postgres_replication : copy pg_hba.conf] *********************************
skipping: [node1]
ok: [node2]

TASK [postgres_replication : start postgresql-server on node2] *****************
skipping: [node1]
changed: [node2]

PLAY [set up backup] ***********************************************************

TASK [Gathering Facts] *********************************************************
ok: [barman]
ok: [node1]
ok: [node2]

TASK [install_barman : install base tools] *************************************
changed: [barman]
changed: [node1]
changed: [node2]

TASK [install_barman : disable firewalld service] ******************************
skipping: [node1]
skipping: [node2]
ok: [barman]

TASK [install_barman : Disable SELinux] ****************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : Ensure SELinux is set to disable mode] ******************
skipping: [node1]
skipping: [node2]
ok: [barman]

TASK [install_barman : install repo] *******************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : disable old postgresql module] **************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : install epel-release] ***********************************
changed: [node2]
changed: [node1]
changed: [barman]

TASK [install_barman : install barman and postgresql packages on barman] *******
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : install barman-cli and postgresql packages on nodes] ****
skipping: [barman]
changed: [node1]
changed: [node2]

TASK [install_barman : generate SSH key for postgres] **************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : generate SSH key for barman] ****************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : fetch all public ssh keys node1] ************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : transfer public key to barman] **************************
skipping: [node2]
skipping: [barman]
changed: [node1 -> barman(192.168.57.13)]

TASK [install_barman : fetch all public ssh keys barman] ***********************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : transfer public key to barman] **************************
skipping: [node1]
skipping: [node2]
changed: [barman -> node1(192.168.57.11)]

TASK [install_barman : Create barman user] *************************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : Add permission for barman] ******************************
skipping: [barman]
changed: [node1]
changed: [node2]

TASK [install_barman : Add permission for barman] ******************************
skipping: [barman]
changed: [node1]
changed: [node2]

TASK [install_barman : restart postgresql-server on node1] *********************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : Create DB for backup] ***********************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : Add tables to otus_backup] ******************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : copy .pgpass] *******************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : Creates directory] **************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : copy barman.conf] ***************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : copy node1.conf] ****************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : barman switch-wal node1] ********************************
skipping: [node1]
skipping: [node2]
changed: [barman]
[WARNING]: Module remote_tmp /var/lib/barman/.ansible/tmp did not exist and was
created with a mode of 0700, this may cause issues when running as another
user. To avoid this, create the remote_tmp dir with the correct permissions
manually

TASK [install_barman : barman cron] ********************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

PLAY RECAP *********************************************************************
barman                     : ok=22   changed=18   unreachable=0    failed=0    skipped=10   rescued=0    ignored=0
node1                      : ok=32   changed=26   unreachable=0    failed=0    skipped=21   rescued=0    ignored=0
node2                      : ok=27   changed=19   unreachable=0    failed=0    skipped=26   rescued=0    ignored=0

```




