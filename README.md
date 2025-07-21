1. 安装  https://playlab.eu.org/archives/hysteria2
执行下面的一键安装脚本（官方）安装 Hysteria 2

https://v2.hysteria.network/docs/getting-started/Installation/

bash <(curl -fsSL https://get.hy2.sh/)

2执行下面的命令先将 Hysteria 设置为开机自启

systemctl enable hysteria-server.service

3. 服务端配置
修改服务端配置文件

nano /etc/hysteria/config.yaml

将配置文件中的内容全部删除，填入以下配置。根据自己的需要选择使用 CA 证书，还是使用自签证书，将对应的注释取消即可
listen: :443 #默认端口443，可以修改为其他端口

#使用CA证书
#acme:
#  domains:
#    - your.domain.net #已经解析到服务器的域名
#  email: your@email.com #你的邮箱

#使用自签证书
#tls:
#  cert: /etc/hysteria/server.crt 
#  key: /etc/hysteria/server.key 

auth:
  type: password
  password: 123456 #认证密码，使用一个强密码进行替换

resolver:
  type: udp
  tcp:
    addr: 8.8.8.8:53 
    timeout: 4s 
  udp:
    addr: 8.8.4.4:53 
    timeout: 4s
  tls:
    addr: 1.1.1.1:853 
    timeout: 10s
    sni: cloudflare-dns.com 
    insecure: false 
  https:
    addr: 1.1.1.1:443 
    timeout: 10s
    sni: cloudflare-dns.com
    insecure: false

masquerade:
  type: proxy
  proxy:
    url: https://cn.bing.com/ #伪装网址
    rewriteHost: true



伪装网址推荐使用个人网盘的网址，个人网盘比较符合单节点大流量的特征，可以通过谷歌搜索 intext:登录 cloudreve 来查找别人搭建好的网盘网址

可以使用以下命令生成自签证书

openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) -keyout /etc/hysteria/server.key -out /etc/hysteria/server.crt -subj "/CN=bing.com" -days 3650 && sudo chown hysteria /etc/hysteria/server.key && sudo chown hysteria /etc/hysteria/server.crt


启动 Hysteria

systemctl start hysteria-server.service
查看 Hysteria 启动状态

systemctl status hysteria-server.service
重新启动 Hysteria

systemctl restart hysteria-server.service


如果显示：{"error": "invalid config: tls: open /etc/hysteria/server.crt: permission denied"} 或者 failed to load server conf 的错误，则说明 Hysteria 没有访问证书文件的权限，需要执行下面的命令将 Hysteria 切换到 root 用户运行

chmod 777 /etc/hysteria/server.crt 
/etc/hysteria/server.key



4. UFW 防火墙
查看防火墙状态

ufw status
开放 80 和 443 端口

ufw allow http && ufw allow https



5.1 V2rayN 客户端配置
下载安装 6.30 以上版本的 V2rayN 客户端，注意需要下载 v2rayN-With-Core.zip 或者 zz_v2rayN-With-Core-SelfContained.7z 的文件

点击 服务器 -> 添加[hysteria2]服务器 ，填写服务器的配置信息就可以了

如果是使用 CA 证书搭建的，SNI 填写你的域名，跳过证书验证选择 false，使用自签证书搭建的，SNI 就填写伪装网址，跳过证书验证选择 true

5.2 sing-box 客户端配置
很多人可能不知道 sing-box 这个客户端，它是一个通用代理平台，1.5.0-beta.2 后支持 Hysteria 2，可以在 Android, iOS 设备上使用

点击 New Profile，“Name”随便填，点击 Create，之后选择这个配置，点击 Edit Content，修改并复制下面的配置替换掉里面的内容就可以了。也可以点击 Check 验证配置的格式是否正确

{
  "log": {
    "disabled": false,
    "level": "error"
  },
  "dns": {
    "servers": [
      {
        "tag": "cloudflare",
        "address": "https://1.1.1.1/dns-query",
		"detour": "proxy"
      },
      {
        "tag": "local",
        "address": "223.5.5.5",
        "detour": "direct"
      },
      {
        "tag": "block",
        "address": "rcode://success"
      }
    ],
    "rules": [
		{
		  "geosite": [
			"cn"
		  ],
		  "server": "local",
		  "disable_cache": true
		},
		{
		  "geosite": [
			"category-ads-all"
		  ],
		  "server": "block",
		  "disable_cache": true
		}
    ],
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "tun",
	  "tag": "tun-in",
      "inet4_address": "172.19.0.1/30",
	  "inet6_address": "fdfe:dcba:9876::1/126",
      "auto_route": true,
      "strict_route": false,
      "sniff": true
    }
  ],
  "outbounds": [
    {
      "type": "hysteria2",
      "tag": "proxy",
      "server": "111.111.111.111", #服务器地址
      "server_port": 443, #服务器端口
      "up_mbps": 20, #最大上传速率
      "down_mbps": 50, #最大下载速率
      "password": "123456", #密码和服务端一致
      "tls": {
        "enabled": true,
        "server_name": "your.domain.net", #没有域名的填伪装网址
        "insecure": false #使用自签证书需要改成true
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    },
    {
      "type": "dns",
      "tag": "dns-out"
    }
  ],
  "route": {
    "rules": [
      {
        "protocol": "dns",
        "outbound": "dns-out"
      },
      {
        "geosite": "cn",
        "geoip": [
          "private",
          "cn"
        ],
        "outbound": "direct"
      },
      {
        "geosite": "category-ads-all",
        "outbound": "block"
      }
    ],
    "auto_detect_interface": true
  }
}



5.3 NekoBox 客户端配置
NekoBox 相对于 sing-box 配置更简单，功能更强大，但是目前只支持 Android 设备

点击右上角的添加按钮，找到 手动输入 -> Hysteria，然后在图形界面中填写服务器的配置信息就可以了

6. 性能优化
将发送、接收的两个缓冲区都设置为 16 MB：

sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216


