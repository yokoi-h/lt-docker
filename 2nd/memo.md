# 続Docker


##  1. ストリーミングレプリケーションをdockerで試す

### 構成
![Image of structure](https://github.com/yokoi-h/lt-docker/blob/master/images/postgres-replication-image.png)

* VMの外側から任意のIPアドレスで接続する
* レプリケーション用のセグメントを作成する

### 実現ステップ

1. PostgreSQL9.3入りイメージを作成する

  　1. sshdがインストールされたコンテナを用意し起動しておく

  　2. dockerが付与したIPアドレスを調べsshでログインする

  　3. PostgreSQL9.3をインストール

  　4. いったんコンテナを停止

  　5. イメージとして保存

2. イメージを使って2台のPostgreSQLコンテナを起動

3. コンテナのNICにIPアドレスを付与する

  　1. 事前のVM側のeth0をプロミスキャスモードにしておく

  　2. eth1に192.168.11.101, 192.168.11.102を付与する

  　3. eth2に10.10.0.1, 10.10.0.2を付与する

  　4. host OSからpingできるか確認

  　5. コンテナ間でpingが飛ぶか確認

4. レプリケーションの設定

　　1. プライマリの設定

　　2. セカンダリの設定

### コンテナにIPアドレスを付与するツール

* 任意のIPを付与するためのツールとしてpipeworkというツールがあります。
* あらかじめこれをdockerコンテナの母艦となるVMにインストールしておきます。

  ```shell
  root@vm-docker:~# apt-get install arping
  root@vm-docker:~# apt-get install bridge-utils
  root@vm-docker:~# git clone https://github.com/jpetazzo/pipework.git
  root@vm-docker:~# cd pipwork
  root@vm-docker:~# install -m 0755 pipework /usr/bin/
  ```

* 使い方
  ```
  pipework <ethデバイス or linux-bridge> -i <コンテナ内のNIC> コンテナID IPアドレス
  ```

### コマンドを粛々と実行

1. PostgreSQL9.3入りイメージを作成する

  　1. sshdがインストールされたコンテナを用意し起動しておく
  ```shell
  root@vm-docker:~# POSTGRES=$(docker run -P -d --name postgres yokoih/sshd)
  root@vm-docker:~# echo $POSTGRES
  03ff6b272a4f5d432e165778602c02c4df8484f849ed1180e897e96f50ff5370
  ```
  　2. dockerが付与したIPアドレスを調べsshでログインする
  ```shell
  root@vm-docker:~# docker inspect --format '{{ .NetworkSettings.IPAddress }}' $POSTGRES
  172.17.0.6
  root@vm-docker:~# ssh 172.17.0.6
The authenticity of host '172.17.0.6 (172.17.0.6)' can't be established.
ECDSA key fingerprint is 5b:c8:ca:2e:c1:11:c0:d0:d3:22:28:17:c6:09:96:28.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.17.0.6' (ECDSA) to the list of known hosts.
root@172.17.0.6's password: [docker]
root@03ff6b272a4f:~#
  ```
  　3. PostgreSQL9.3をインストール
  ```shell
  root@03ff6b272a4f:~# apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8
Executing: gpg --ignore-time-conflict --no-options --no-default-keyring --homedir /tmp/tmp.zsqiZx7f01 --no-auto-check-trustdb --trust-model always --keyring /etc/apt/trusted.gpg --primary-keyring /etc/apt/trusted.gpg --keyserver keyserver.ubuntu.com --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8
gpg: requesting key ACCC4CF8 from hkp server keyserver.ubuntu.com
gpg: key ACCC4CF8: public key "PostgreSQL Debian Repository" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
```
  ```shell
  root@03ff6b272a4f:~# echo "deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main" > /etc/apt/sources.list.d/pgdg.list
  root@03ff6b272a4f:~#
  root@03ff6b272a4f:~# apt-get update
  Reading package lists... Done
  root@03ff6b272a4f:~# apt-get install -y python-software-properties software-properties-common
  root@03ff6b272a4f:~# apt-get install -y postgresql-9.3 postgresql-client-9.3 postgresql-contrib-9.3
  Setting up postgresql-contrib-9.3 (9.3.5-1.pgdg12.4+1) ...
  Processing triggers for libc-bin (2.19-0ubuntu6.3) ...
  ```
  ```shell
  root@03ff6b272a4f:~# su - postgres
  postgres@03ff6b272a4f:~$ /etc/init.d/postgresql start
  Starting PostgreSQL 9.3 database server                                                                                         [ OK ]
  postgres@03ff6b272a4f:~$ psql --command "CREATE USER docker WITH SUPERUSER PASSWORD 'docker';"
  CREATE ROLE
  postgres@03ff6b272a4f:~$ createdb -O docker docker
  postgres@03ff6b272a4f:~$ echo "host all  all    0.0.0.0/0  md5" >> /etc/postgresql/9.3/main/pg_hba.conf
  postgres@03ff6b272a4f:~$ echo "listen_addresses='*'" >> /etc/postgresql/9.3/main/postgresql.conf
  postgres@03ff6b272a4f:~$ /etc/init.d/postgresql restart
  Restarting PostgreSQL 9.3 database server                                                                                       [ OK ]
  postgres@03ff6b272a4f:~$ psql docker
  psql (9.3.5)
  Type "help" for help.
  docker=#
  ```

  　4. いったん停止(停止しても更新したファイルの内容は残っています)
  ```shell
  postgres@03ff6b272a4f:~$ /etc/init.d/postgresql stop
  Stopping PostgreSQL 9.3 database server                                                                                         [ OK ]
  postgres@03ff6b272a4f:~$ exit
  logout
  root@03ff6b272a4f:~# exit
  logout
  Connection to 172.17.0.6 closed.
  root@vm-docker:~# docker stop $POSTGRES
  03ff6b272a4f5d432e165778602c02c4df8484f849ed1180e897e96f50ff5370
  ```

  　5. イメージとして保存
  ```shell
  root@vm-docker:~# sudo docker commit $POSTGRES yokoih/postgres_base
  ac4042982d7cbb3400f096b5aabc1cceab5dc6d935199a4acda776789ee7fd0b
  root@vm-docker:~# docker images
  REPOSITORY             TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
  yokoih/postgres_base   latest              ac4042982d7c        15 seconds ago      394.5 MB
  ubuntu                 latest              53bf7a53e890        7 days ago          221.3 MB
  yokoih/sshd            latest              0481b802b6c9        10 days ago         267.5 MB
  ```
2. イメージを使って2台のPostgreSQLコンテナを起動
  ```shell
  root@vm-docker:~# PG1=$(docker run -P -d --name primary yokoih/postgres_base)
  root@vm-docker:~# PG2=$(docker run -P -d --name secondary yokoih/postgres_base)
  root@vm-docker:~# echo $PG1
  1e6c2c87dacea40815fd9fed9b6a24b5f1d09f048412381bbdda3924293e4240
  root@vm-docker:~# echo $PG2
  2b116e57c0c1f3be13f54d245db1a016e468975f1e6d0bfe7801055fb49e075f
  root@vm-docker:~# docker ps
  CONTAINER ID        IMAGE                         COMMAND               CREATED             STATUS              PORTS                   NAMES
  2b116e57c0c1        yokoih/postgres_base:latest   "/usr/sbin/sshd -D"   22 seconds ago      Up 21 seconds       0.0.0.0:49158->22/tcp   secondary
  1e6c2c87dace        yokoih/postgres_base:latest   "/usr/sbin/sshd -D"   33 seconds ago      Up 31 seconds       0.0.0.0:49157->22/tcp   primary
  ```

3. コンテナのNICにIPアドレスを付与する

  　1. 事前にVM側のeth0をプロミスキャスモードにしておく
  ```shell
  root@vm-docker:~# ifconfig eth0 promisc
  ```
  　2. eth1に192.168.11.101, 192.168.11.102を付与する
  ```shell
  root@vm-docker:~# pipework eth0 -i eth1 $PG1 192.168.11.101/24
  root@vm-docker:~# pipework eth0 -i eth1 $PG2 192.168.11.102/24
  ```
  　3. eth2に10.10.0.1, 10.10.0.2を付与する
  ```shell
  root@vm-docker:~# pipework br1 -i eth2 $PG1 10.10.0.1/24
  root@vm-docker:~# pipework br1 -i eth2 $PG2 10.10.0.2/24
  root@vm-docker:~# brctl show
  ```
  　4. host OSからpingできるか確認
  ```shell
  $ ping -c 1 192.168.11.101
  PING 192.168.11.101 (192.168.11.101): 56 data bytes
  64 bytes from 192.168.11.101: icmp_seq=0 ttl=64 time=0.630 ms
  ...
  $ ping -c 1 192.168.11.102
  PING 192.168.11.102 (192.168.11.102): 56 data bytes
  64 bytes from 192.168.11.102: icmp_seq=0 ttl=64 time=0.588 ms
  ...
  ```
  　5. コンテナ間でpingが飛ぶか確認
  ```shell
  root@vm-docker:~# docker inspect --format '{{ .NetworkSettings.IPAddress }}' $PG1
  172.17.0.7
  root@vm-docker:~# docker inspect --format '{{ .NetworkSettings.IPAddress }}' $PG2
  172.17.0.8
  root@vm-docker:~# ssh 172.17.0.7
  The authenticity of host '172.17.0.7 (172.17.0.7)' can't be established.
  ECDSA key fingerprint is 5b:c8:ca:2e:c1:11:c0:d0:d3:22:28:17:c6:09:96:28.
  Are you sure you want to continue connecting (yes/no)? yes
  Warning: Permanently added '172.17.0.7' (ECDSA) to the list of known hosts.
  root@172.17.0.7's password:
  Last login: Tue Sep 30 23:20:07 2014 from 172.17.42.1
  root@1e6c2c87dace:~# ping 10.10.0.2
  PING 10.10.0.2 (10.10.0.2) 56(84) bytes of data.
  64 bytes from 10.10.0.2: icmp_seq=1 ttl=64 time=0.091 ms
  ...
  ```

4. レプリケーションの設定

 　1. プライマリの設定
  ```shell
  root@1e6c2c87dace:~# cd /etc/postgresql/9.3/main/
  root@1e6c2c87dace:/etc/postgresql/9.3/main# vi postgresql.conf
  root@1e6c2c87dace:/etc/postgresql/9.3/main# chown postgres:postgres postgresql.conf
  wal_level = 	hot_standby			# minimal, archive, or hot_standby
  archive_mode = on		# allows archiving to be done
  archive_command = 'cp %p /tmp/%f'		# command to use to archive a logfile segment
  max_wal_senders = 3		# max number of walsender processes
  synchronous_standby_names = 'hoge'	# standby servers that provide sync rep
  root@1e6c2c87dace:/etc/postgresql/9.3/main# vi pg_hba.conf
  host replication postgres 10.10.0.2/32 trust
  root@1e6c2c87dace:~# sudo su - postgres
  postgres@1e6c2c87dace:~$ /etc/init.d/postgresql start
  postgres@1e6c2c87dace:/etc/postgresql/9.3/main$ ps -ef | grep postgres
  root         51     23  0 00:22 pts/0    00:00:00 sudo su - postgres
  root         52     51  0 00:22 pts/0    00:00:00 su - postgres
  postgres     53     52  0 00:22 pts/0    00:00:00 -su
  postgres     77      1  0 00:22 ?        00:00:00 /usr/lib/postgresql/9.3/bin/postgres -D /var/lib/postgresql/9.3/main -c config_file=/etc/postgresql/9.3/main/postgresql.conf
  postgres     79     77  0 00:22 ?        00:00:00 postgres: checkpointer process
  postgres     80     77  0 00:22 ?        00:00:00 postgres: writer process
  postgres     81     77  0 00:22 ?        00:00:00 postgres: wal writer process
  postgres     82     77  0 00:22 ?        00:00:00 postgres: autovacuum launcher process
  postgres     83     77  0 00:22 ?        00:00:00 postgres: archiver process   last was 000000010000000000000002.00000028.backup
  postgres     84     77  0 00:22 ?        00:00:00 postgres: stats collector process
  postgres    133     77  0 00:33 ?        00:00:00 postgres: wal sender process postgres 10.10.0.2(43532) streaming 0/3000090
  postgres    138     53  0 00:35 pts/0    00:00:00 ps -ef
  postgres    139     53  0 00:35 pts/0    00:00:00 grep postgres
  ```
 　2. セカンダリの設定
  ```shell
  root@2b116e57c0c1:~# sudo su - postgres
  postgres@2b116e57c0c1:~$ /etc/init.d/postgresql start
  postgres@2b116e57c0c1:~$ export PGDATA=/var/lib/postgresql/9.3/standby
  postgres@2b116e57c0c1:~$ pg_basebackup -R -D ${PGDATA} -h 10.10.0.1 -p 5432
  NOTICE:  pg_stop_backup complete, all required WAL segments have been archived
  root@1e6c2c87dace:~# vi /etc/postgresql/9.3/main/postgresql.conf
  data_directory = '/var/lib/postgresql/9.3/standby'
  hot_standby = on
  wal_level = hot_standby
  root@1e6c2c87dace:~# vi /var/lib/postgresql/9.3/standby/recovery.conf
  standby_mode = 'on'
  primary_conninfo = 'user=postgres host=10.10.0.1 port=5432 sslmode=prefer sslcompression=1 krbsrvname=postgres application_name=hoge'
  # primary_conninfoの最後にapplication_name=hogeを追加
  postgres@2b116e57c0c1:~$ /etc/init.d/postgresql start
  postgres@2b116e57c0c1:~/9.3/standby$ ps -ef | grep postgres
  root         38     23  0 00:23 pts/0    00:00:00 sudo su - postgres
  root         39     38  0 00:23 pts/0    00:00:00 su - postgres
  postgres     40     39  0 00:23 pts/0    00:00:00 -su
  postgres    181      1  0 00:33 ?        00:00:00 /usr/lib/postgresql/9.3/bin/postgres -D /var/lib/postgresql/9.3/standby -c config_file=/etc/postgresql/9.3/main/postgresql.conf
  postgres    182    181  0 00:33 ?        00:00:00 postgres: startup process   recovering 000000010000000000000003
  postgres    183    181  0 00:33 ?        00:00:00 postgres: wal receiver process   streaming 0/3000090
  postgres    184    181  0 00:33 ?        00:00:00 postgres: checkpointer process
  postgres    185    181  0 00:33 ?        00:00:00 postgres: writer process
  postgres    264     40  0 00:35 pts/0    00:00:00 ps -ef
  postgres    265     40  0 00:35 pts/0    00:00:00 grep postgres
  ```

### 動作確認
primary側でテーブルを作成する。
```SQL
postgres@1e6c2c87dace:~$ psql docker
psql (9.3.5)
Type "help" for help.

docker=# create table test_table (id int, name text);
CREATE TABLE
docker=#
```

secondary側で参照する。
```SQL
postgres@2b116e57c0c1:~$ psql docker
psql (9.3.5)
Type "help" for help.

docker=# \d
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | test       | table | docker
 public | test10     | table | docker
 public | test2      | table | postgres
 public | test3      | table | postgres
 public | test4      | table | postgres
 public | test5      | table | postgres
 public | test6      | table | postgres
 public | test_table | table | postgres
(8 rows)
```



##  2. 母艦側のディレクトリをコンテナ内のPostgreSQLから使う

　1. 母艦側でディレクトリを作成
```bash
root@vm-docker:~# mkdir -p /volumes/postgres/9.3/conf
root@vm-docker:~# mkdir -p /volumes/postgres/9.3/data
root@vm-docker:~# ls -al /volumes/postgres/9.3
total 16
drwxr-xr-x 4 root root 4096 Oct  1 19:33 .
drwxr-xr-x 3 root root 4096 Oct  1 19:31 ..
drwxr-xr-x 2 root root 4096 Oct  1 19:33 conf
drwxr-xr-x 2 root root 4096 Oct  1 19:33 data
```

　2. Postgresコンテナを起動する
```bash
root@vm-docker:~# PG3=$(docker run -P -d --name volume_test -v /volumes/postgres/9.3/data:/var/lib/postgresql/9.3/ -v /volumes/postgres/9.3/conf:/etc/postgresql/9.3/ yokoih/postgres_base)
root@vm-docker:~# docker ps
CONTAINER ID        IMAGE                         COMMAND               CREATED             STATUS              PORTS                   NAMES
8ac3b4364619        yokoih/postgres_base:latest   "/usr/sbin/sshd -D"   7 seconds ago       Up 7 seconds        0.0.0.0:49160->22/tcp   volume_test
2b116e57c0c1        yokoih/postgres_base:latest   "/usr/sbin/sshd -D"   11 hours ago        Up 11 hours         0.0.0.0:49158->22/tcp   secondary
1e6c2c87dace        yokoih/postgres_base:latest   "/usr/sbin/sshd -D"   11 hours ago        Up 11 hours         0.0.0.0:49157->22/tcp   primary
0a83dcd9b851        yokoih/sshd:latest            "/usr/sbin/sshd -D"   27 hours ago        Up 27 hours         0.0.0.0:49154->22/tcp   postgres94
a78818203ab1        yokoih/sshd:latest            "/usr/sbin/sshd -D"   6 days ago          Up 6 days           0.0.0.0:49153->22/tcp   sshd
root@vm-docker:~#
```

　3. ログインしてパーミッションとかもろもろ設定
```bash
root@vm-docker:~# ssh 172.17.0.10
root@8ac3b4364619:~# chown -R postgres:postgres /etc/postgresql/9.3
root@8ac3b4364619:~# chown postgres:postgres /var/lib/postgresql/9.3/
root@8ac3b4364619:~#
```

　4. データベースクラスタを作成
```bash
root@8ac3b4364619:~# su - postgres
postgres@8ac3b4364619:~$ pg_createcluster 9.3 main --start
Creating new cluster 9.3/main ...
  config /etc/postgresql/9.3/main
  data   /var/lib/postgresql/9.3/main
  locale C
  port   5432
```

　5. ポスグレを起動
```bash
root@8ac3b4364619:~# su - postgres
postgres@8ac3b4364619:~$ /etc/init.d/postgresql start
 * Starting PostgreSQL 9.3 database server                                                                                                                                                                   [ OK ]
postgres@8ac3b4364619:~$ psql --command "CREATE USER docker WITH SUPERUSER PASSWORD 'docker';"
CREATE ROLE
postgres@8ac3b4364619:~$ createdb -O docker docker
postgres@8ac3b4364619:~$ psql docker
```

  ```SQL
  postgres@8ac3b4364619:~$ psql docker
  psql (9.3.5)
  Type "help" for help.

  docker=# create table test_table ( id int, name text);
  CREATE TABLE
  docker=# insert into test_table values ( 1, 'docker');
  INSERT 0 1
  docker=# select * from test_table;
   id |  name
  ----+--------
    1 | docker
  (1 row)

  docker=#
  ```

　6. VM側でもディレクトリが見えていますね
```bash
root@vm-docker:/volumes/postgres/9.3/data/main# ls -al
total 72
drwx------ 15 landscape mlocate 4096 Oct  1 19:40 .
drwxr-xr-x  3 landscape mlocate 4096 Oct  1 19:40 ..
drwx------  6 landscape mlocate 4096 Oct  1 19:41 base
drwx------  2 landscape mlocate 4096 Oct  1 19:42 global
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_clog
drwx------  4 landscape mlocate 4096 Oct  1 19:40 pg_multixact
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_notify
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_serial
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_snapshots
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_stat
drwx------  2 landscape mlocate 4096 Oct  1 19:44 pg_stat_tmp
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_subtrans
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_tblspc
drwx------  2 landscape mlocate 4096 Oct  1 19:40 pg_twophase
-rw-------  1 landscape mlocate    4 Oct  1 19:40 PG_VERSION
drwx------  3 landscape mlocate 4096 Oct  1 19:40 pg_xlog
-rw-------  1 landscape mlocate  133 Oct  1 19:40 postmaster.opts
-rw-------  1 landscape mlocate   99 Oct  1 19:40 postmaster.pid
```
```bash
root@vm-docker:/volumes/postgres/9.3/conf/main# ls -al
total 56
drwxr-xr-x 2 landscape mlocate  4096 Oct  1 19:40 .
drwxr-xr-x 3 landscape mlocate  4096 Oct  1 19:40 ..
-rw-r--r-- 1 landscape mlocate   315 Oct  1 19:40 environment
-rw-r--r-- 1 landscape mlocate   143 Oct  1 19:40 pg_ctl.conf
-rw-r----- 1 landscape mlocate  4649 Oct  1 19:40 pg_hba.conf
-rw-r----- 1 landscape mlocate  1636 Oct  1 19:40 pg_ident.conf
-rw-r--r-- 1 landscape mlocate 20595 Oct  1 19:40 postgresql.conf
-rw-r--r-- 1 landscape mlocate   378 Oct  1 19:40 start.conf
```
