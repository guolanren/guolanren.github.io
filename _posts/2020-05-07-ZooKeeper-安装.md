---
title: ZooKeeper-安装
description: ZooKeeper 快速安装...
categories: 
 - code
tags:
 - zookeeper
---

------

## 安装

```shell
# 1. 下载
$ mkdir /opt/zookeeper
$ cd /opt/zookeeper
$ tar -zx -f apache-zookeeper-3.6.1-bin.tar.gz

# 2. 复制 zoo_sample.cfg
$ cd /opt/zookeeper/apache-zookeeper-3.6.1-bin/conf
$ cp zoo_sample.cfg zoo.cfg

# 3. 设置 dataDir 目录，配置"host:通信port:选举port"
$ vi zoo.cfg
	...
	dataDir=/var/lib/zookeeper
	...
	server.1=192.168.225.128:2888:3888
	# server.2=192.168.225.129:2888:3888
	# server.3=192.168.225.130:2888:3888
	
# 4. 配置环境变量
shell> vi /etc/profile.d/zookeeper.sh
	#!/bin/bash
	export ZK_HOME=/opt/zookeeper/zookeeper-3.6.1-bin
	export PATH=${PATH}:${ZK_HOME}/bin
shell> source /etc/profile.d/zookeeper.sh

# 5. 创建 myid 文件
shell> mkdir /var/lib/zookeeper
shell> vi /var/lib/zookeeper/myid
	1
	
# 6. 启动 zookeeper
shell> zkServer.sh [start | start-foreground]
```

## 自启服务

```shell
# 创建文件
$ vi /etc/init.d/zookeeper
	#!/bin/sh
    export JAVA_HOME=/usr/java/jdk
    EXEC=/opt/zookeeper/apache-zookeeper-3.6.1-bin/bin/zkServer.sh
    PIDFILE=/var/lib/zookeeper/zookeeper_server.pid
    CONF_DIR="/opt/zookeeper/apache-zookeeper-3.6.1-bin/conf"
    ###############
    # SysV Init Information
    # chkconfig: - 58 74
    # description: zookeeper.
    ### BEGIN INIT INFO
    # Provides: zookeeper
    # Required-Start: $network $local_fs $remote_fs
    # Required-Stop: $network $local_fs $remote_fs
    # Default-Start: 2 3 4 5
    # Default-Stop: 0 1 6
    # Should-Start: $syslog $named
    # Should-Stop: $syslog $named
    # Short-Description: start and stop zookeeper
    # Description: zookeeper daemon
    ### END INIT INFO

    case "$1" in
        start)
            if [ -f $PIDFILE ]
            then
                echo "$PIDFILE exists, process is already running or crashed"
            else
                echo "Starting ZooKeeper server..."
                $EXEC --config $CONF_DIR start
            fi
            ;;
        stop)
            if [ ! -f $PIDFILE ]
            then
                echo "$PIDFILE does not exist, process is not running"
            else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $EXEC stop
                while [ -x /proc/${PID} ]
                do
                    echo "Waiting for ZooKeeper to shutdown ..."
                    sleep 1
                done
                echo "ZooKeeper stopped"
            fi
            ;;
        status)
            if [ ! -f $PIDFILE ]
            then
                echo "$PIDFILE does not exist, process is not running"
            else
                PID=$(cat $PIDFILE)
                if [ ! -x /proc/${PID} ]
                then
                    echo 'ZooKeeper is not running'
                else
                    echo "ZooKeeper is running ($PID)"
                fi
            fi
            ;;
        restart)
            $0 stop
            $0 start
            ;;
        *)
            echo "Please use start, stop, restart or status as first argument"
            ;;
    esac
 
# 2. 安装服务
$ chkconfig --add zookeeper
$ chkconfig zookeeper on
```