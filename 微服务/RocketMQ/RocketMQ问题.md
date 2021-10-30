#  RocketMQ 问题

##  rocketmq 连接异常 sendDefaultImpl call timeout

需要将broker 的ip 换成本地的ip
在`conf/broker.conf` 里面加上 `brokerIP1=[本机ip]`

然后重启

```bash
nohup sh bin/mqbroker -n localhost:9876 -c conf/broker.conf autoCreateTopicEnable=true &
```

再运行程序, 就不在报错. 

