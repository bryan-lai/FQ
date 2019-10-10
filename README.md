# 小米路由器R1D V2Ray安装指南

由于众所周知的原因，SS基本已经不能用了，目前使用V2ray效果不错。家里小米路由器1代SS插件也无效了，苦于网上资料太少。研究了一天才搞定，特此分享给需要的人。

## 原理
- copy了几个类似的安装步骤，都不起作用。不明白原理，稍有问题就无效。明白原理才能根据自己情况调整。 另外理解原理其他智能路由器也都一样可用。
- 原理如下：
  终端访问 --> 小米路由器 -->1.透明代理--> 2.V2ray客户端 --> V2Ray服务器 --> 自由世界
- 本文只介绍1和2部分, 另外解决大陆地区无法直接上网问题
1. 透明代理通过iptables端口转发完成
2. V2ray客户端安装小米路由器支持的openwrt版本
3. 用iptables+ipset配合配置中大陆地区ip地址库直接上网

## 步骤
### 清除冲突
需要删除ss插件及相关的透明代理设置，防止干扰

### V2ray客户端安装
- 下载v2ray
- 下载地址：https://github.com/v2ray/v2ray-core/releases/
- r2d选择arm：
- https://github.com/v2ray/v2ray-core/releases/download/v4.20.0/v2ray-linux-arm.zip
1. 解压上传客户端文件
2. 上传客户端配置文件，建议使用本机电脑上V2ray客户端可用的配置文件

### ipset配合配置中大陆地区直接上网
- 我的路由器上v2ray各种geo设置都无法解决国内直接访问问题，例如网易云音乐等受限，最后发现通过ipset配合效果最好
- 首先执行ipset --version看看是不是有装ipset
- 然后下载 https://github.com/17mon/china_ip_list/raw/master/china_ip_list.txt
- 下载完执行下面的命令
```
ipset -N china hash:net
ipset flush china
for iprange in `cat /path/to/china_ip_list.txt`; do
    ipset -A china $iprange
done
```
- 说明：在iptables 转发端口前面加一条 iptables -t nat -A V2RAY -p tcp -m set --match-set china dst -j RETURN 绕过透明代理
### 透明代理
执行下列脚本设置iptables（包括上面ipset配合配置中大陆地区直接上网）
```
iptables -t nat -N V2RAY
iptables -t nat -A V2RAY -d 0.0.0.0 -j RETURN
iptables -t nat -A V2RAY -d 127.0.0.0 -j RETURN
iptables -t nat -A V2RAY -d 0/8 -j RETURN
iptables -t nat -A V2RAY -d 127/8 -j RETURN
iptables -t nat -A V2RAY -d 10/8 -j RETURN
iptables -t nat -A V2RAY -d 169.254/16 -j RETURN
iptables -t nat -A V2RAY -d 172.16/12 -j RETURN
iptables -t nat -A V2RAY -d 192.168/16 -j RETURN
iptables -t nat -A V2RAY -d 224/4 -j RETURN
iptables -t nat -A V2RAY -d 240/4 -j RETURN
iptables -t nat -A V2RAY -d 192.168.31.0/24 -j RETURN
iptables -t nat -A V2RAY -p tcp -m set --match-set china dst -j RETURN
# From lans redirect to Dokodemo-door's local port
iptables -t nat -A V2RAY -s 192.168.31.0/24 -p tcp -j REDIRECT --to-ports 1099
iptables -t nat -A PREROUTING -p tcp -j V2RAY
```
- 其中1099是我配置的v2ray监听端口，可用根据你自己情况修改
### 启动V2ray
`nohup <v2ray path>/v2ray --config=<v2ray配置文件path>/v2ray_mi.json &`
- 其中< v2ray path > 和<v2ray配置文件path> 根据自己情况修改
  
### 重启后失效
将脚本加入启动脚本，防止路由器重启后失效
请将ipset、iptables和v2ray启动脚本依次加入启动脚本/etc/rc.local中
