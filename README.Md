Xray 科学上网小结
-
基础设置引用 Project X 的小小白白话文 https://xtls.github.io/document/level-0/ ，其中增加了利用回落功能将流量分别分流到 Websocket 和 Nginx 中，网站使用 Cloudreve 网盘作伪装  
### 配置 Cloudreve 网盘
版本号参照 https://github.com/cloudreve/Cloudreve 中的最新版本，这里以3.5.3为例  
```shell
mkdir /usr/local/bin/cloudreve && cd /usr/local/bin/cloudreve
wget https://github.com/cloudreve/Cloudreve/releases/download/3.5.3/cloudreve_3.5.3_linux_amd64.tar.gz
tar -zxvf cloudreve_3.5.3_linux_amd64.tar.gz
chmod +x ./cloudreve
./cloudreve
```
Cloudreve 在首次启动时，会创建初始管理员账号，请注意保管管理员密码，此密码只会在首次启动时出现。如果您忘记初始管理员密码，需要删除同级目录下的 cloudreve.db，重新启动主程序以初始化新的管理员账户。Cloudreve 默认会监听5212端口。你可以在浏览器中访问 http://服务器IP:5212 进入 Cloudreve。  
接下来按 Ctrl+C 退出 Cloudreve，设置守护进程。  
```shell
nano /usr/lib/systemd/system/cloudreve.service

# 输入以下代码
[Unit]
Description=Cloudreve
Documentation=https://docs.cloudreve.org
After=network.target
After=mysqld.service
Wants=network.target

[Service]
WorkingDirectory=/usr/local/bin/cloudreve
ExecStart=/usr/local/bin/cloudreve/cloudreve
Restart=on-abnormal
RestartSec=5s
KillMode=mixed

StandardOutput=null
StandardError=syslog

[Install]
WantedBy=multi-user.target
```
设置开机启动
```shell
systemctl start cloudreve
systemctl enable cloudreve
```
设置 Cloudreve 配置文件并重启，注意如果不是 root 用户的话需要对 cloudreve.sock 进行权限设置以让它能被其他用户读取和写入
```shell
nano /usr/local/bin/cloudreve/conf.ini

# 编辑以下代码
[System]
Mode = master
Debug = false
Listen = :5212

[UnixSocket]
Listen = /run/cloudreve.sock

# 若不为root用户则需对cloudreve.sock进行权限设置
sudo chmod a+rwx /run/cloudreve.sock

# 重启 CLoudreve 服务
systemctl restart cloudreve
```
### 申请和安装域名证书
域名证书用 DNS-api 验证，这里以 Cloudflare 的 api 进行说明  
首先登录到 Cloudflare 账户，然后查看 My Profile ，点击 Global API Key 的 View API Key ，输入密码，保存 KEY ，然后设置 Cloudflare 的 Global API key  
```shell
export CF_Key="xxxxxx"
export CF_Email="xxxxxx"
```
其余过程参考小小白白话文，申请证书的命令步骤稍有变动
```shell
acme.sh --issue --dns dns_cf -d domain.com -k ec-256 --force
```
### 配置Nginx
若不为 root 用户，则需查找 Nginx 配置文件 /etc/nginx/nginx.conf 中的 User ，将其改为自己的用户名
在 Nginx 站点配置文件 /etc/nginx/conf.d/default.conf 中增加Server段
```shell
# 禁止IP或错误的域名访问网站
server {
    listen unix:/dev/shm/h1.sock proxy_protocol default_server;
    listen unix:/dev/shm/h2c.sock http2 proxy_protocol default_server;
    set_real_ip_from unix:;
    real_ip_header proxy_protocol;
    server_name _;
    return 400;
}

server {
    listen unix:/dev/shm/h1.sock proxy_protocol;
    listen unix:/dev/shm/h2c.sock http2 proxy_protocol;
    set_real_ip_from unix:;
    real_ip_header proxy_protocol;
    server_name domain.com;

    location / {
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        proxy_pass http://unix:/run/cloudreve.sock;
		    proxy_redirect off;
    }
}
```
### 配置Xray
Config.json 中的 fallbacks 改成如下内容
```json
"fallbacks": [
  {
    "dest": "/dev/shm/h1.sock",
    "xver": 2
  },
  {
    "alpn": "h2",
    "dest": "/dev/shm/h2c.sock",
    "xver": 2
  },
  {
    "path": "/websocket",
    "dest": "@vless-ws",
    "xver": 2
  }
]
```
inbound增加一段用来监听回落到Websocket端口的流量
```json
{
  "listen": "@vless-ws",
  "protocol": "vless",
  "settings": {
  "clients": [
    {
      "id": "", // 填写你的 UUID
      "level": 0,
      "email": "vpsadmin@domain.com"
    }
  ],
  "decryption": "none"
  },
  "streamSettings": {
    "network": "ws",
    "security": "none",
    "wsSettings": {
      "acceptProxyProtocol": true,
      "path": "/websocket"
    }
  },
  "sniffing": {
    "enabled": true,
    "destOverride": [
      "http",
      "tls"
    ]
  }
}
```
### 修改设置
以后若要继续修改设置需要先清除sock文件，避免重新启动后 Nginx 报出端口被占用的错误
```shell
systemctl stop nginx
systemctl stop xray
rm -rf /dev/shm/*.sock
```
