# CentOS7.2安装redis6.0.7

1. 准备gcc
    1. 查看当前gcc版本：`gcc -v`，可以看到当前版本4.8.5。
    2. 由于redis6.0.7对gcc版本有要求，所以要先升级gcc版本：
        1. `yum -y install centos-release-scl && yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils && scl enable devtoolset-9 bash`
        2. 再次查看gcc版本，可以发现是9.3.1
2. 下载redis文件：
    
    `wget [http://download.redis.io/releases/redis-6.0.7.tar.gz](http://download.redis.io/releases/redis-6.0.7.tar.gz)`
    
3. 安装redis
    1. 加压redis文件：`tar -zxvf redis-6.0.7.tar.gz`
    2. 进入redis文件目录：`cd redis-6.0.7`
    3. 执行简单的`make`命令
    4. 执行`src/redis-server`，可以看到redis已经可以运行起来了，但此时是前台运行的，所以还需要更改一下redis的配置
    5. 更改一下redis安装路径，`make install PREFIX=/usr/local/redis`(我是先执行了make，又执行的该命令，应该可以直接执行该命令的)
4. 更改redis配置
    1. 将src下的redis.config文件拷贝一份到`/usr/local/redis/bin`下：`cp ~/redis-6.0.7/redis.conf /usr/local/redis/bin/`
    2. 更改配置
        1. 将守护进程模式改为后台启动：`daemonize yes`(225行左右，可以用`set nu`显示行号)
        2. 更改密码：`requirepass 12345678`(790行左右，如果不需要密码，则注释掉本行)
        3. 若允许远程连接，注释掉：`bind 127.0.0.1 ::1`（56行左右）
5. 设置开机自启动
    1. 配置redis服务文件：`vi /etc/systemd/system/redis.service`
    2. 填充内容（ExecStart要设置为自己的路径）：
        
        ```java
        [Unit]
        #服务描述
        Description=Redis Server Manager
        #服务类别
        After=syslog.target network.target
        [Service]
        #后台运行的形式
        Type=forking
        #服务命令
        ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/bin/redis.conf
        #给服务分配独立的临时空间
        PrivateTmp=true
        [Install]
        #运行级别下服务安装的相关设置，可设置为多用户，即系统运行级别为3
        WantedBy=multi-user.target
        ```
        
    3. 设置开启自启动：`systemctl enable redis.service`
    4. 启动redis服务：`systemctl start redis.service`
    5. 查看redis状态：`systemctl status redis.service`
        
        可以看到启动成功：`Active: active (running) since 一 2022-02-07 21:17:12 CST; 8s ago`
        
6. 尝试开启redis客户端
    1. 执行`redis-cli`，可以看到正常进入redis（也可以执行host和port: `redis-cli -h localhost -p 6379`）
    2. 执行`ping`命令，输出了`PONG`
        
        ```
        [root@caychan ~]# redis-cli
        127.0.0.1:6379> ping
        PONG
        ```
        
    3. 如果没有输出`PONG`，而是输出了`(error) NOAUTH Authentication required`，说明需要输出密码：
        1. `auth 12345678`（12345678为密码）
        2. 再次执行`ping`，可以正常输出`PONG`
        3. 如果不想每次输入密码，可以修改`redis.config`，将密码行注释掉`requirepass 12345678`（上面有提到）
        4. 重启redis服务使配置生效：`systemctl restart redis.service`