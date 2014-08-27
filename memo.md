docker
========

参考資料：[http://www.slideshare.net/enakai/docker-34668707](http://www.slideshare.net/enakai/docker-34668707)

1. Dockerとは
- Docker incが開発している仮想化技術
- VMより軽い
- コンテナ技術を利用した仮想化
− コンテナとは、
　ユーザ実行プロセスを独立した空間(コンテナ)に分離する技術
　カーネルは共有される
　コンテナごとに、ファイルシステム、ネットワーク、ユーザ、プロセステーブルを隔離
　コンテナごとに、CPU、メモリの割当量を制限

− 実体としては、コンテナという一つの技術があるわけではなく、ネットワークや
　ファイルシステムを分離するためのOSの個々の技術を組み合わせたもの。
　Dockerなどは、それらのOSの機能を組み合わせて使いやすくしたツール
　例えば、ネットワークの分離は、Network namespace
　ファイルシステムの分離は、
　その他のコンテナ技術
　　LXC(Linux Container)：Dockerが0.9まで採用していたコンテナ
　　OpenVZ：RHEL用のコンテナ
　　Warden：Cloud Foundaryが開発
　　Zone：Solarisのコンテナ
　
- コンテナだと、
　クリーンな環境がすぐにできる
　差分なく環境が作れる
　デプロイも簡単


2. コンテナ内でbashを実行
※VMではなく、bashが独立した空間で実行されるだけ
dockerは元となるOSのイメージをローカルにDLしてから実行する
- イメージの取得
sudo docker pull ubuntu:latest
- コンテナの起動
sudo docker run -i -t ubuntu:latest /bin/bash
-iは標準入力を受け付けるオプション
-tは仮想端末を使用するオプション

topとか

exitするとbashも終了する
そのままにする場合はデタッチ
Ctrl-p Ctrl-q
再度アタッチする場合は
sudo docker attach <CID>


3. コンテナでPostgreSQLを実行
参考資料：[http://docs.docker.com/examples/postgresql_service/](http://docs.docker.com/examples/postgresql_service/)

元となるイメージがない場合、Dockerfileというものを作って、イメージを作成する。

```
#
# example Dockerfile for http://docs.docker.com/examples/postgresql_service/
#

FROM ubuntu
MAINTAINER SvenDowideit@docker.com

# Add the PostgreSQL PGP key to verify their Debian packages.
# It should be the same key as https://www.postgresql.org/media/keys/ACCC4CF8.asc
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8

# Add PostgreSQL's repository. It contains the most recent stable release
#     of PostgreSQL, ``9.3``.
RUN echo "deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main" > /etc/apt/sources.list.d/pgdg.list

# Install ``python-software-properties``, ``software-properties-common`` and PostgreSQL 9.3
#  There are some warnings (in red) that show up during the build. You can hide
#  them by prefixing each apt-get statement with DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y python-software-properties software-properties-common postgresql-9.3 postgresql-client-9.3 postgresql-contrib-9.3

# Note: The official Debian and Ubuntu images automatically ``apt-get clean``
# after each ``apt-get``

# Run the rest of the commands as the ``postgres`` user created by the ``postgres-9.3`` package when it was ``apt-get installed``
USER postgres

# Create a PostgreSQL role named ``docker`` with ``docker`` as the password and
# then create a database `docker` owned by the ``docker`` role.
# Note: here we use ``&&\`` to run commands one after the other - the ``\``
#       allows the RUN command to span multiple lines.
RUN    /etc/init.d/postgresql start &&\
    psql --command "CREATE USER docker WITH SUPERUSER PASSWORD 'docker';" &&\
    createdb -O docker docker

# Adjust PostgreSQL configuration so that remote connections to the
# database are possible. 
RUN echo "host all  all    0.0.0.0/0  md5" >> /etc/postgresql/9.3/main/pg_hba.conf

# And add ``listen_addresses`` to ``/etc/postgresql/9.3/main/postgresql.conf``
RUN echo "listen_addresses='*'" >> /etc/postgresql/9.3/main/postgresql.conf

# Expose the PostgreSQL port
EXPOSE 5432

# Add VOLUMEs to allow backup of config, logs and databases
VOLUME  ["/etc/postgresql", "/var/log/postgresql", "/var/lib/postgresql"]

# Set the default command to run when starting the container
CMD ["/usr/lib/postgresql/9.3/bin/postgres", "-D", "/var/lib/postgresql/9.3/main", "-c", "config_file=/etc/postgresql/9.3/main/postgresql.conf"]
```

vi Dockerfile
sudo docker build -t eg_postgresql .

...
Removing intermediate container 9f30998297fc
Step 11 : CMD ["/usr/lib/postgresql/9.3/bin/postgres", "-D", "/var/lib/postgresql/9.3/main", "-c", "config_file=/etc/postgresql/9.3/main/postgresql.conf"]
 ---> Running in a1f1cb63014d
 ---> 09223d086e10
Removing intermediate container a1f1cb63014d
Successfully built 09223d086e10

- イメージを確認
docker images

- PostgreSQLの起動
sudo docker run --rm -P --name pg_test eg_postgresql

- 接続
psql -h localhost -p 49153 -d docker -U docker --password

接続してみる


■ファイルシステムの話
- ベースとなるイメージから起動されるが、ファイルはUnionFSにより
　差分管理される


