
## redash 구성

  [visualization tools](https://www.blendo.co/blog/the-ultimate-list-of-custom-dashboards-and-bi-tools-to-track-your-metrics-and-gather-insights/)

  [redash docker 설치](http://www.kwangsiklee.com/ko/2017/10/redash-docker-%EC%84%A4%EC%B9%98/
  )

# test on mac with original docker image
  mkdir redash
  cd redash
  git clone https://github.com/getredash/redash
  docker-compose up
  docker-compose down

  * 동작안하여 우분투 이미지로 직접 설치함

  > docker commands
  삭제
  `docker images | egrep -v ubuntu | egrep -v IMAGE | awk '{system ("docker rmi -f " $3)}'
  `
  중지
  `docker kill <container id>
  `
  네트워크 확인
  `docker network ls
  docker network inspect bridge
  `
  실행
  `docker run -d -p 8080:80 --name myredash ubuntu:16.04
  docker run -it -p 8080:80 --name myredash myredash:0.1
  `

# docker 우분투 이미지로 셋업

  [docker netwoking](http://bluese05.tistory.com/53?category=559611)
  [docker url](https://docs.docker.com/develop/develop-images/baseimages/#create-a-simple-parent-image-using-scratch)
  [redash install](https://redash.io/help-onpremise/setup/setting-up-redash-instance.html)
  [redash userguide](https://help.redash.io/collection/18-user-guide)
  [postgresql tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
  [docker 네트워크 아키텍처](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)

  > 도커 다운로드
   `docker run -it -v $PWD:/build ubuntu:16.04
   `  
  > 컨테이너 셋업

   `apt-get update
   apt-get install vim
   apt-get install sudo
   apt-get install wget
   `

  > 설치 부트스트랩 스크립트 작업
  [bootstrap script down](https://raw.githubusercontent.com/getredash/redash/master/setup/ubuntu/bootstrap.sh)


   다운로드 받은 스크립트 중 create_database()함수에 아래내용 추가

   `service postgresql restart
    service redis-server restart

    sudo -u postgres dropdb redash --if-exists
    sudo -u postgres dropuser redash --if-exists`

    vi bootstrap.sh

  `#!/bin/bash
   #
   # This script setups Redash along with supervisor, nginx, PostgreSQL and Redis. It was written to be used on
   # Ubuntu 16.04. Technically it can work with other Ubuntu versions, but you might get non compatible versions
   # of PostgreSQL, Redis and maybe some other dependencies.
   #
   # This script is not idempotent and if it stops in the middle, you can't just run it again. You should either
   # understand what parts of it to exclude or just start over on a new VM (assuming you're using a VM).

   set -eu

   REDASH_BASE_PATH=/opt/redash
   REDASH_BRANCH="${REDASH_BRANCH:-master}" # Default branch/version to master if not specified in REDASH_BRANCH env var
   REDASH_VERSION=${REDASH_VERSION-3.0.0.b3134} # Install latest version if not specified in REDASH_VERSION env var
   LATEST_URL="https://s3.amazonaws.com/redash-releases/redash.${REDASH_VERSION}.tar.gz"
   VERSION_DIR="$REDASH_BASE_PATH/redash.${REDASH_VERSION}"
   REDASH_TARBALL=/tmp/redash.tar.gz
   FILES_BASE_URL=https://raw.githubusercontent.com/getredash/redash/${REDASH_BRANCH}/setup/ubuntu/files

   cd /tmp/

   verify_root() {
       # Verify running as root:
       if [ "$(id -u)" != "0" ]; then
           if [ $# -ne 0 ]; then
               echo "Failed running with sudo. Exiting." 1>&2
               exit 1
           fi
           echo "This script must be run as root. Trying to run with sudo."
           sudo bash "$0" --with-sudo
           exit 0
       fi
   }

   create_redash_user() {
       adduser --system --no-create-home --disabled-login --gecos "" redash
   }

   install_system_packages() {
       apt-get -y update
       # Base packages
       apt install -y python-pip python-dev nginx curl build-essential pwgen
       # Data sources dependencies:
       apt install -y libffi-dev libssl-dev libmysqlclient-dev libpq-dev freetds-dev libsasl2-dev
       # SAML dependency
       apt install -y xmlsec1
       # Storage servers
       apt install -y postgresql redis-server
       apt install -y supervisor
   }

   create_directories() {
       mkdir -p $REDASH_BASE_PATH
       chown redash $REDASH_BASE_PATH

       # Default config file
       if [ ! -f "$REDASH_BASE_PATH/.env" ]; then
           sudo -u redash wget "$FILES_BASE_URL/env" -O $REDASH_BASE_PATH/.env
       fi

       COOKIE_SECRET=$(pwgen -1s 32)
       echo "export REDASH_COOKIE_SECRET=$COOKIE_SECRET" >> $REDASH_BASE_PATH/.env
   }

   extract_redash_sources() {
       rm -rf $REDASH_BASE_PATH/*
       sudo -u redash wget "$LATEST_URL" -O "$REDASH_TARBALL"
       sudo -u redash mkdir "$VERSION_DIR"
       sudo -u redash tar -C "$VERSION_DIR" -xvf "$REDASH_TARBALL"
       ln -nfs "$VERSION_DIR" $REDASH_BASE_PATH/current
       ln -nfs $REDASH_BASE_PATH/.env $REDASH_BASE_PATH/current/.env
   }

   install_python_packages() {
       pip install --upgrade pip
       # TODO: venv?
       pip install setproctitle # setproctitle is used by Celery for "pretty" process titles
       pip install -r $REDASH_BASE_PATH/current/requirements.txt
       pip install -r $REDASH_BASE_PATH/current/requirements_all_ds.txt
   }

   create_database() {
       service postgresql restart
       service redis-server restart

       sudo -u postgres dropdb redash --if-exists
       sudo -u postgres dropuser redash --if-exists

       # Create user and database
       sudo -u postgres createuser redash --no-superuser --no-createdb --no-createrole
       sudo -u postgres createdb redash --owner=redash

       cd $REDASH_BASE_PATH/current
       sudo -u redash bin/run ./manage.py database create_tables
   }

   setup_supervisor() {
       wget -O /etc/supervisor/conf.d/redash.conf "$FILES_BASE_URL/supervisord.conf"
       service supervisor restart
   }

   setup_nginx() {
       rm /etc/nginx/sites-enabled/default
       wget -O /etc/nginx/sites-available/redash "$FILES_BASE_URL/nginx_redash_site"
       ln -nfs /etc/nginx/sites-available/redash /etc/nginx/sites-enabled/redash
       service nginx restart
   }

   verify_root
   install_system_packages
   create_redash_user
   create_directories
   extract_redash_sources
   install_python_packages
   create_database
   setup_supervisor
   setup_nginx
  `
  > 설치 후 네트워크 포트 확인
    netstat -nl

  > 컨테이너 재기동시 수행 스크립트 -> 컨테이너에 자동 적용 필요, docker yml 고려.
  `service postgresql start
   service redis-server start
   service supervisor start
   service nginx start`

   > docker 수행내용 이미지 저장
   YOOJUNGHOONui-MacBook-Air:redash yoojunghoon$ docker ps
   CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
   0cdbe141839f        ubuntu:16.04        "/bin/bash"         About an hour ago   Up About an hour    0.0.0.0:8080->80/tcp   confident_carson
   YOOJUNGHOONui-MacBook-Air:redash yoojunghoon$ docker commit confident_carson myredash:0.1

  > 로컬 호스트에서 접속
   http://localhost:8080

  > postgreSQL DB설정
   테이블 생성
    `postgres=# CREATE TABLE playground (
    postgres(#     equip_id serial PRIMARY KEY,
    postgres(#     type varchar (50) NOT NULL,
    postgres(#     color varchar (25) NOT NULL,
    postgres(#     location varchar(25) check (location in ('north', 'south', 'west', 'east', 'northeast', 'southeast', 'southwest', 'northwest')),
    postgres(#     install_date date
    postgres(# );
    postgres=# \d
                       List of relations
     Schema |          Name           |   Type   |  Owner   
    --------+-------------------------+----------+----------
     public | playground              | table    | postgres
     public | playground_equip_id_seq | sequence | postgres
    (2 rows)

    postgres=# \dt
               List of relations
     Schema |    Name    | Type  |  Owner   
    --------+------------+-------+----------
     public | playground | table | postgres
    (1 row)

    postgres=# INSERT INTO playground (type, color, location, install_date) VALUES ('slide', 'blue', 'south', '2014-04-28');
    INSERT 0 1
    postgres=# INSERT INTO playground (type, color, location, install_date) VALUES ('swing', 'yellow', 'northwest', '2010-08-16');
    INSERT 0 1

    postgres=# select * from playground;
     equip_id | type  | color  | location  | install_date
    ----------+-------+--------+-----------+--------------
            2 | swing | yellow | northwest | 2010-08-16
    (1 row)
    `
    DB추가 생성
    `root@129391e4d36d:/# su - postgres
    postgres@129391e4d36d:~$ createdb mydb
    postgres@129391e4d36d:~$ psql -s mydb
    psql (9.5.12)
    Type "help" for help.

    mydb=# create user mydb password 'mydb';
    ***(Single step mode: verify command)*******************************************
    create user mydb password 'mydb';
    ***(press return to proceed or enter x and return to cancel)********************

    CREATE ROLE
    mydb=#
    mydb=# \l
                                 List of databases
       Name    |  Owner   | Encoding  | Collate | Ctype |   Access privileges   
    -----------+----------+-----------+---------+-------+-----------------------
     mydb      | postgres | SQL_ASCII | C       | C     |
     postgres  | postgres | SQL_ASCII | C       | C     |
     redash    | redash   | SQL_ASCII | C       | C     |
     template0 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
               |          |           |         |       | postgres=CTc/postgres
     template1 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
               |          |           |         |       | postgres=CTc/postgres
    (5 rows)

    mydb=# \du
                                       List of roles
     Role name |                         Attributes                         | Member of
    -----------+------------------------------------------------------------+-----------
     mydb      |                                                            | {}
     postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
     redash    |                                                            | {}

    mydb=# \q    
    `
    mydb 테이블 생성
    ` postgres@129391e4d36d:~$ psql -U mydb -h localhost -p 5432   
      Password for user mydb:
      psql (9.5.12)
      SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
      Type "help" for help.

      mydb=> CREATE TABLE playground (
      mydb(>     equip_id serial PRIMARY KEY,
      mydb(>     type varchar (50) NOT NULL,
      mydb(>     color varchar (25) NOT NULL,
      mydb(>     location varchar(25) check (location in ('north', 'south', 'west', 'east', 'northeast', 'southeast', 'southwest', 'northwest')),
      mydb(>     install_date date
      mydb(> );
      CREATE TABLE
      mydb=>
      mydb=>
      mydb=> INSERT INTO playground (type, color, location, install_date) VALUES ('slide', 'blue', 'south', '2014-04-28');
      INSERT 0 1
      mydb=> INSERT INTO playground (type, color, location, install_date) VALUES ('swing', 'yellow', 'northwest', '2010-08-16');
      INSERT 0 1
      mydb=>
      mydb=> select * from playgroud;
      ERROR:  relation "playgroud" does not exist
      LINE 1: select * from playgroud;
                            ^
      mydb=> select * from playground;
       equip_id | type  | color  | location  | install_date
      ----------+-------+--------+-----------+--------------
              1 | slide | blue   | south     | 2014-04-28
              2 | swing | yellow | northwest | 2010-08-16
      (2 rows)

      mydb=> \q
`
