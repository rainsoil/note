## `max virtual memory areas vm.max_map_count [65530] is too low, increase to at least `



修改/etc/sysctl.conf文件，增加配置vm.max_map_count=262144

```
vi /etc/sysctl.conf
sysctl -p
```

　执行命令sysctl -p生效

## 创建用户

新建用户，每台服务器都要新建，ES不允许root用户运行



```shell
groupadd elsearch                                    新增elsearch用户组
useradd elsearch -g elsearch -p elasticsearch        创建elsearch用户
chown -R elsearch:elsearch ./elasticsearch-6.6.1     用户目录权限
```

3.切换到elsearch用户下，启动ES



```shell
su elsearch
cd /opt/dp/elasticsearch-6.6.1/bin
./elasticsearch &
```



