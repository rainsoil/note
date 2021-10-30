#   Window子系统ubuntu 开启2375 端口

```
1）查看配置文件
[root@docker-server ~]# systemctl show docker | grep FragmentPath=
FragmentPath=/usr/lib/systemd/system/docker.service

然后修改/lib/systemd/system/docker.service文件
[root@docker-server ~]# cp /lib/systemd/system/docker.service /lib/systemd/system/docker.service.bak
[root@docker-server ~]# vim /lib/systemd/system/docker.service
.......
ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://0.0.0.0:4243                  #添加这一行
#ExecStart=/usr/bin/dockerd-current \                                                            #注释掉默认的这一行
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \        
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          --seccomp-profile=/etc/docker/seccomp.json \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY \
    $REGISTRIES

2）修改/etc/default/docker文件
[root@docker-server ~]# cp /etc/default/docker /etc/default/docker.bak
[root@docker-server ~]# vim /etc/sysconfig/docker
......
DOCKER_OPTS="-H tcp://localhost:4243 -H unix:///var/run/docker.sock"          #添加这一行

3）DOCKER_HOST的环境变量设置
[root@docker-server ~]# vim ~/.bashrc 
........
export DOCKER_HOST=tcp://localhost:4243

4）重启docker服务
[root@docker-server ~]# systemctl daemon-reload
[root@docker-server ~]# systemctl restart docker

5）检查发现4243端口已启动
[root@docker-server ~]# netstat -ant
.......
tcp6       0      0 :::4243                 :::*                    LISTEN     
[root@docker-server ~]# lsof -i:4243
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dockerd-c 15400 root    6u  IPv6  59175      0t0  TCP *:4243 (LISTEN)
```