

# 实验九 入侵检测

## 环境配置

### 拓扑结构

![topological_graph](./img/topological_graph.png)

### Snort安装

```python
# 禁止在apt安装时弹出交互式配置界面
export DEBIAN_FRONTEND=noninteractive

apt install snort
```

![welcomesnort](./img/welcomesnort.png)



## 实验流程

### 实验一：配置snort为嗅探模式

```python
# 显示IP/TCP/UDP/ICMP头
snort –v

# 显示应用层数据
snort -vd

# 显示数据链路层报文头
snort -vde

# -b 参数表示报文存储格式为 tcpdump 格式文件
# -q 静默操作，不显示版本欢迎信息和初始化信息
snort -q -v -b -i eth1 "port not 22"
```

![snort-](./img/snort-.jpeg)

```python
# 使用 CTRL-C 退出嗅探模式
# 嗅探到的数据包会保存在 /var/log/snort/snort.log.<epoch timestamp>
# 其中<epoch timestamp>为抓包开始时间的UNIX Epoch Time格式串
# 可以通过命令 date -d @<epoch timestamp> 转换时间为人类可读格式
# exampel: date -d @1511870195 转换时间为人类可读格式
# 上述命令用tshark等价实现如下：
tshark -i eth0 -f "port not 22" -w 1_tshark.pcap
```

![snort-log](./img/snort-log.jpeg)



### 实验二：配置并启用snort内置规则

```python
# /etc/snort/snort.conf 中的 HOME_NET 和 EXTERNAL_NET 需要正确定义
# 例如，学习实验目的，可以将上述两个变量值均设置为 any
snort -q -A console -b -i eth1 -c /etc/snort/snort.conf -l /var/log/snort/
```

![snort-conf](./img/snort-conf.jpeg)



### 实验三：自定义snort规则

```python
# 新建自定义 snort 规则文件
cat << EOF > /etc/snort/rules/cnss.rules

#INSERT
alert tcp \$EXTERNAL_NET any -> \$HTTP_SERVERS 80 (msg:"Access Violation has been detected on /etc/passwd ";flags: A+; content:"/etc/passwd"; nocase;sid:1000001; rev:1;)
alert tcp \$EXTERNAL_NET any -> \$HTTP_SERVERS 80 (msg:"Possible too many connections toward my http server"; threshold:type threshold, track by_src, count 100, seconds 2; classtype:attempted-dos; sid:1000002; rev:1;)
EOF

# 添加配置代码到 /etc/snort/snort.conf
include $RULE_PATH/cnss.rules

# 开启apache2
service apache2 start

#应用规则开启嗅探
snort -q -A fast -b -i eth0 -c /etc/snort/snort.conf -l /var/log/snort/

# 在attacker上使用ab命令进行压力测试
ab -c 100 -n 10000 http://$dst_ip/hello
```

![cnss_rules](./img/cnss_rules.jpeg)

![snort_rules](./img/snort_rules.png)



### 实验四：和防火墙联动

- 下载并解压[Guardian-1.7.tar.gz](https://c4pr1c3.gitee.io/cuc-ns/chap0x09/attach/guardian.tar.gz)

```python
# 解压缩 Guardian-1.7.tar.gz
tar zxf guardian.tar.gz

# 安装 Guardian 的依赖 lib
apt install libperl4-corelibs-perl
```

在Victim上先后开启**snort**和**guardian.pl**

```python
# 开启 snort
snort -q -A fast -b -i eth1 -c /etc/snort/snort.conf -l /var/log/snort/

```

```python
# 假设 guardian.tar.gz 解压缩后文件均在 /home/kali/Desktop/guardian 下
cd /home/kali/Desktop/guardian
```

编辑guardian.conf并保存，更改参数为

```python
HostIpAddr 172.16.111.124
Interface eth0
```

```python
# 启动 guardian.pl
perl guardian.pl -c guardian.conf
```

![guandian.conf](./img/guandian.conf.png)

在Attacker上用**nmap**暴力扫描Victim:

```python
nmap 172.16.111.124 -A -T4 -n -vv
```

guardian.conf 中默认的来源IP被屏蔽时间是 60 秒（屏蔽期间如果黑名单上的来源IP再次触发snort报警消息，则屏蔽时间会继续累加60秒）

![guardian](./img/guardian.jpeg)



## 课后问题：

1. IDS与防火墙的联动防御方式相比IPS方式防御存在哪些缺陷？是否存在相比较而言的优势？
   - 入侵检测系统（IDS）和入侵防御系统（IPS）都是网络基础架构的一部分。IDS / IPS将网络数据包与包含已知网络攻击签名的网络威胁数据库进行比较-并标记任何匹配的数据包。
   - IDS偏向于检测，属于管理系统，会检测出不安全的网络行为，但是不阻断任何网络行为。IPS偏向于防御，属于控制系统，可以阻止危险行为。



## 参考资料

- [第九章实验](https://c4pr1c3.gitee.io/cuc-ns/chap0x09/exp.html)
- [Snort 中文手册](http://www.kaiyuanba.cn/content/network/snort/Snortman.htm)
- [2019-NS-Public-chencwx](https://github.com/CUCCS/2019-NS-Public-chencwx/blob/ns_chap0x09)
- [2020-ns-public-LyuLumous](https://github.com/CUCCS/2020-ns-public-LyuLumos/blob/ch0x09/ch0x09/README.md)



