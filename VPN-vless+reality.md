## 1.注册 DMIT 账号

浏览器打开 https://www.dmit.io
右上角 Register 注册账号
填邮箱、密码、姓名（可以用英文拼音）
验证邮箱

## 2.选购 VPS

登录后点 Services → Order New Services
选择 Tokyo 区域
套餐推荐优先级：

TYO.Pro.TINY（CN2 GIA，月付约 $21.9）— 如果有货首选
TYO.EB（CMI 线路）— TYO.Pro 缺货时的替代
LAX.Pro（洛杉矶 CN2 GIA）— 如果东京全线缺货


操作系统选 Ubuntu 22.04 或 Ubuntu 24.04
付款周期：月付或季付都行，先试一个月
结算页面看看有没有优惠码可用（搜索 "DMIT 优惠码 2026"）
付款方式选加密货币（或支付宝，看你方便）

## 3.获取 VPS 信息
购买成功后：

进入 Services → 点击你的新 VPS
记录下：

IP 地址（IPv4）
SSH 密钥或root 密码

**⚠️ 重要：**DMIT 默认使用 SSH 密钥登录，不是密码。购买时它会让你上传公钥或者生成一对密钥。如果你不熟悉：
在你 Mac 上先检查有没有现有密钥：
ls ~/.ssh/id_*.pub
如果有输出（比如 id_ed25519.pub），复制这个公钥内容：
cat ~/.ssh/id_ed25519.pub
在 DMIT 购买/设置页面粘贴进去。
如果没有密钥，先生成一个：
ssh-keygen -t ed25519 -C "kevin@dmit"
一路回车（密码可以留空），然后 cat ~/.ssh/id_ed25519.pub 复制公钥上传到 DMIT。

## 4.先测延迟
拿到新 IP 后，先别急着部署，在 Mac 上测一下：
ping -c 10 新VPS的IP
如果延迟在 50-100ms，没有丢包，说明线路好，继续部署。如果延迟还是 200ms+（不太可能，CN2 GIA 线路一般不会），联系 DMIT 客服换 IP。

## 5.SSH 连接新 VPS
bashssh root@新VPS的IP
如果用的是 SSH 密钥登录，它应该直接进去不问密码。

## 6.系统更新 + 安装 Xray
bash# 更新系统
`apt update && apt upgrade -y`

### 安装 Xray
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

#### 验证
`xray version`


## 7.生成密钥和 UUID：

```bash
# 生成密钥对
xray x25519

# 生成 UUID
xray uuid

# 生成 shortId
openssl rand -hex 8
```

三个命令的输出都保存到 Mac 本地文本文件里。记住对应关系：

- `PrivateKey` → 服务端配置
- `Password` → 客户端的 Public Key
- `UUID` → 两端都要
- `shortId` → 两端都要
注意：UUID，会显示两行，第一行是hash32，不用管。

## 8.写服务端配置文件
```
cat > /usr/local/etc/xray/config.json << 'XRAYEOF'
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "你的uuid",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "addons.mozilla.org:443",
          "xver": 0,
          "serverNames": [
            "addons.mozilla.org"
          ],
          "privateKey": "你的PrivateKey",
          "shortIds": [
            "",
            "你的shortId"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
XRAYEOF
```
如果不能写文件，就用sudo，管理员权限。

验证'xray run -test -config /usr/local/etc/xray/config.json'
看到 Configuration OK 就继续。

## 9.启动服务
sudo xray run -test -config /usr/local/etc/xray/config.json
sudo systemctl start xray
sudo systemctl enable xray

## 10.启动防火墙
```sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo systemctl start xray
sudo systemctl enable xray
sudo systemctl status xray
sudo ss -tlnp | grep 443
```

## 问题参考
- `curl ifconfig.me;`
vps上读取IP本机测试是否接通vps，接通需返回vps公网ip

- DEST（伪装站点）选择
`openssl s_client -connect www.microsoft.com:443 -tls1_3 -alpn h2 < /dev/null 2>/dev/null | grep -E "Protocol|ALPN"`
www.microsoft.com:443
addons.mozilla.org:443
哪个同时输出 Protocol: TLSv1.3 和 ALPN protocol: h2 

- yaml节点格式
```
proxies:
  - name: "节点1"
    type: vless
    server: IP1
    port: 443
    uuid: xxx
    network: tcp
    udp: true
    tls: true
    flow: xtls-rprx-vision
    servername: addons.mozilla.org
    client-fingerprint: chrome
    reality-opts:
      public-key: xxx
      short-id: xxx

  - name: "节点2"
    type: vless
    server: IP2
    port: 443
    uuid: yyy
    network: tcp
    udp: true
    tls: true
    flow: xtls-rprx-vision
    servername: addons.mozilla.org
    client-fingerprint: chrome
    reality-opts:
      public-key: yyy
      short-id: yyy

proxy-groups:
  - name: "Proxy"
    type: select
    proxies:
      - "节点1"
      - "节点2"
      - "DIRECT"

rules:
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```
- google GCP 控制面板操作：

进入 VPC network → Firewall rules（防火墙规则）
点 Create Firewall Rule
填写：

Name: allow-443
Direction: Ingress
Targets: All instances in the network
Source IP ranges: 0.0.0.0/0
Protocols and ports: 选 Specified protocols and ports → 勾选 TCP → 填 443
点 Create 保存

- 检测xray服务，端口监听。
**Xray 还在跑吗**
sudo systemctl status xray

**443 在监听吗**
sudo ss -tlnp | grep 443

**最近的日志**
sudo journalctl -u xray -n 20 --no-pager

客户端nc -zv GCP的IP 443，测试是否能连通服务端口，排除端口问题。

- 端口不监听的问题
ss -tlnp | grep 443 没有输出，说明 Xray 没在监听 443 端口。
日志显示 Xray 启动了但没有报错，这通常是因为 非 root 用户无法绑定 443 端口（低于 1024 的端口需要 root 权限），而 Xray 的 service 文件里配置了以 nobody 用户运行。
两种解决方式：
方式 A：给 Xray 绑定低端口的权限（推荐）
bashsudo setcap cap_net_bind_service=+ep /usr/local/bin/xray
sudo systemctl restart xray
sudo ss -tlnp | grep 443
方式 B：如果 A 不生效，改用高端口
把配置里的端口改成 8443，然后防火墙放行：
bashsudo sed -i 's/"port": 443/"port": 8443/' /usr/local/etc/xray/config.json
sudo ufw allow 8443/tcp
sudo systemctl restart xray
sudo ss -tlnp | grep 8443
GCP 防火墙也要加一条放行 8443 的规则。客户端 yaml 里 port 也改成 8443。
先试方式 A，贴结果。
## 成功之后
测网速`curl -o /dev/null -s -w "连接时间: %{time_connect}s\n总时间: %{time_total}s\n" https://www.google.com`
