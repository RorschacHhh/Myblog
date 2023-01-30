---
title: Postgresql安装
date: 2023-01-30 10:37:58
tags: [postgresql,install]
categories: 数据库
---
PostgreSQL

##### 安装

    #安装支持
    yum -y install gcc automake autoconf libtool make readline-devel zlib-devel flex
    
    #设置账号
    groupadd postgres
    useradd -g postgres postgres
    passwd postgres
    
    #设置数据文件夹
    mkdir -p /data/pgsql/{data,log,archive}
    chown -R postgres.postgres /data/pgsql
    
    #解压
    tar -xzvf postgresql-12.4.tar.gz -C /usr/local/
    cd /usr/local/postgresql-xxxx
    
    #编译
    ./configure
    make & make install
    
    #修改参数
    vi /etc/security/limits.conf
    
    postgres soft  core unlimited
    postgres hard  nproc unlimited
    postgres soft  nproc unlimited
    postgres hard  memlock unlimited
    postgres hard  nofile 1024000
    postgres soft  memlock unlimited
    postgres soft  nofile 1024000
    postgres hard  stack  65536
    postgres soft  stack  65536
    
    #初始化数据库
    su - postgres
    cd /usr/local/pgsql/bin
    ./initdb -E utf8 -D /data/pgsql/data
    
    #添加环境变量
    vim /etc/bashrc
    
    export PGHOME=/usr/local/pgsql/
    export PGUSER=postgres
    export PGPORT=5432
    export PGDATA=/data/pgsql/data
    export PGLOG=/data/pgsql/log/postgres.log
    export PATH=$PGHOME/bin:$PATH:$HOME/bin
    export LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH
    PATH=/usr/local/pgsql/bin:$PATH
    export PATH
    
    source /etc/bashrc
    
    #修改数据库访问限制
    vim /data/pgsql/data/postgresql.conf 
    listen_addresses = '*'
    
    vim /data/pgsql/data/pg_hba.conf
    在文件的最下方追加如下命令（生产环境需要将其替换为远程访问数据库的IP地址或地址段） 
    host all all 0.0.0.0/0 trust
    
    #创建测试数据库
    pg_ctl -D $PGDATA -l $PGLOG start
    /usr/local/pgsql/bin/createdb test
    /usr/local/pgsql/bin/psql test
    
    #修改参数密码
    alter user postgres with password 'password';
    alter system set max_connections = 500;
    alter system set shared_buffers = '1GB'; #25%内存
    alter system set checkpoint_completion_target = 0.8; #数值越高IO越平滑
    alter system set work_mem = '16MB'; #每个会话的缓存
    alter system set maintenance_work_mem = '1024MB'; #8G一般512-1024
    alter system set wal_buffers = '4MB'; #
    
    #开机自启
    cp /usr/local/postgresql-xxxx/contrib/start-scripts/linux /etc/init.d/postgresql
    chkconfig --add postgresql
    chmod 755 /etc/init.d/postgresql
    
    #修改参数
    prefix="/usr/local/pgsql"
    PGDATA="/data/postgresql/data"
    PGLOG="/data/pgsql/log/postgres.log"
    PGUSER=postgres
    
    service postgresql start
    

##### 主从设置

    #安装支持
    yum install flex -y
    
    #下载安装repmgr root
    wget -c https://repmgr.org/download/repmgr-5.3.0.tar.gz
    mkdir /usr/local/pgsql/contrib
    tar -zxvf repmgr-5.3.0.tar.gz -C /usr/local/pgsql/contrib
    cd /usr/local/pgsql/contrib
    mv repmgr-5.3.0/ repmgr
    vim ~/.bash_profile
    
    export PGHOME=/usr/local/pgsql
    export PGUSER=postgres
    export PGPORT=5432
    export PGDATA=/data/pgsql/data
    export PGLOG=/data/pgsql/log/postgres.log
    export PATH=$PGHOME/bin:$PATH:$HOME/bin
    export LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH
    PATH=/usr/local/pgsql/bin:$PATH:$HOME/bin
    export PATH
    
    source ~/.bash_profile
    ./configure&&make install
    
    #设置repmgr
    vim /etc/repmgr.conf
    
    node_id=2
    node_name='node2'
    conninfo='host=10.29.0.4 user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/data/pgsql/data'
    
    log_level=INFO
    log_facility=STDERR
    log_file='/data/pgsql/log/repmgr.log'
    log_status_interval=10
    
    pg_bindir='/usr/local/pgsql/bin'
    
    #命令设置全路径
    failover='automatic'
    promote_command='/usr/local/pgsql/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file' 
    follow_command='/usr/local/pgsql/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'log_file='/data/pgsql/log/repmgr.log'
    
    service_start_command='sudo service postgresql start'
    service_stop_command='sudo service postgresql stop'
    service_restart_command='sudo service postgresql restart'
    service_reload_command='su - postgres -c \'pg_ctl reload\' '
    
    repmgrd_pid_file='/tmp/repmgrd.pid'
    repmgrd_service_start_command='/usr/local/pgsql/bin/repmgrd -f /etc/repmgr.conf start'
    repmgrd_service_stop_command='kill -9 `cat /tmp/repmgrd.pid`'
    
    replication_user='repmgr'
    use_replication_slots=true
    repmgr_bindir='/usr/local/pgsql/bin'
    passfile='/home/postgres/.pgpass'
    ssh_options='-q -o ConnectTimeout=10'
    monitoring_history=yes
    monitor_interval_secs=2
    reconnect_attempts=3
    reconnect_interval=5
    #修改postgresql.conf
    vim /data/pgsql/data/postgresql.conf
    
    max_wal_senders = 10
    max_replication_slots = 10
    wal_level = replica
    hot_standby = on
    archive_mode = on   # repmgr 本身不需要 WAL 文件归档。
    archive_command = '/bin/true'
    
    #修改pg_hba.conf
    vim /data/pgsql/data/pg_hba.conf
    local   replication   repmgr                              trust
    host    replication   repmgr      127.0.0.1/32            trust
    host    replication   repmgr      10.40.16.0/23           trust
    
    local   repmgr        repmgr                              trust
    host    repmgr        repmgr      127.0.0.1/32            trust
    host    repmgr        repmgr      10.40.16.0/23           trust
    
    #在master上设置repmgr的用户和数据库
    su - postgres
    pg_ctl start
    psql postgres
    
    create database repmgr;
    create user repmgr with password 'repmgr' superuser login;
    alter database repmgr owner to repmgr;
    
    #设置主数据库
    repmgr -f /etc/repmgr.conf primary register
    repmgr -f /etc/repmgr.conf cluster show
    
    #设置从库
    repmgr -h 主库IP -U repmgr -d repmgr -f /etc/repmgr.conf standby clone -F --dry-run
    repmgr -h 主库IP -U repmgr -d repmgr -f /etc/repmgr.conf standby clone -F
    
    su - postgres
    pg_ctl start -l $PGLOG
    --slave1节点，将slave1数据库注册到集群，并查看状态
    su - postgres -c "/usr/local/pgsql/bin/repmgr -f /etc/repmgr.conf standby register"
    su - postgres -c "/usr/local/pgsql/bin/repmgr -f /etc/repmgr.conf cluster show"
    
    #设置repmgrd.service
    [Unit]
    Description=A replication manager, and failover management tool for PostgreSQL
    After=syslog.target
    After=network.target
    After=postgresql.service   #posgres服务名
    
    [Service]
    Type=forking
    
    User=postgres
    Group=postgres
    
    # PID file
    PIDFile=/tmp/repmgrd.pid
    
    # Location of repmgr conf file:
    Environment=REPMGRDCONF=/etc/repmgr.conf
    Environment=PIDFILE=/tmp/repmgrd.pid
    
    # Where to send early-startup messages from the server
    # This is normally controlled by the global default set by systemd
    # StandardOutput=syslog
    ExecStart=/usr/local/pgsql/bin/repmgrd -f ${REPMGRDCONF} -p ${PIDFILE} -d --verbose
    ExecStop=/usr/bin/kill -TERM $MAINPID
    ExecReload=/usr/bin/kill -HUP $MAINPID
    
    # Give a reasonable amount of time for the server to start up/shut down
    TimeoutSec=300
    
    [Install]
    WantedBy=multi-user.target
    
    #主从恢复
    service postgresql stop
    su - postgres -c "repmgr -h 10.29.0.3 -U repmgr -d repmgr -f /etc/repmgr.conf standby clone -F"
    service postgresql start
    su - postgres -c "repmgr -f /etc/repmgr.conf standby register -F"
    su - postgres -c "repmgr service status"

##### 故障排除

    #主备无法切换
    检查是否有sudo权限并设置免密
    vim /etc/sudoers
    查看配置文件命令是否是全路径
    vim /etc/repmgr.conf
    查看节点状态是否是Paused状态
    repmgr service status

#### 8.keepalived

##### VIP设置

    #防火墙
     firewall-cmd --zone=public --add-protocol=vrrp --permanent
     firewall-cmd --reload
    
    #设置keepalived.conf
        ! Configuration File for keepalived
    
        global_defs {
           router_id 节点名
        }
    
        vrrp_script pg_check {
            script "/etc/keepalived/pg_check.sh" #设置脚本，不是主权重 -3
            weight -3
        }
    
        vrrp_instance VI_1 {
            interface ens192 #绑定网卡
            virtual_router_id 56
            priority 92     #主备权重可以一样也可以不一样
            advert_int 1  #每秒check
            authentication {
                auth_type PASS
                auth_pass 39hgoDQP
            }
            virtual_ipaddress {
                10.29.0.7  #虚拟ip
            }
           track_script {
                pg_check   #执行脚本
           }
        }
    
    #执行脚本
    #!/bin/bash
    A=`ps -ef | grep "postgres:" | grep "walwriter" | wc -l` 
    B=`ps -ef | grep "postgres:" | grep "archiver last was" | wc -l`
    if [ $A -ne "0" ] && [ $B -ne "0" ]; then
    #       /usr/bin/sudo systemctl start keepalived.service
            echo "is master"
            exit 0
    else
    #       /usr/bin/sudo killall keepalived
            echo "not master"
            exit 1
    fi