# TCPdump操作方法

## 树莓派环境准备：

```shell
sudo apt-get install tcpdump
pip install scapy
```



## 记录通信指令指令：

```shell
# 方法1：
sudo tcpdump -w capture.pcap -i eth0 tcp port 20002 
# 方法2：（常用方法）
sudo tcpdump -tttt -w capture.pcap -i eth0 tcp 
# 方法3：
sudo tcpdump -w capture.pcap -i eth0 tcp   #根据具体实施
```

##  查看文件时间戳：

``` shell
tcpdump -r capture.pcap -tttt
```

