---
title: '证书链少一环，curl 就报 unable to get local issuer certificate'
date: 2026-11-25 11:04:50
tags: [TLS, 证书链, curl, OpenSSL, 教程]
categories: [技术教程]
---

今天早上我想弄明白一件事：curl 那句 `unable to get local issuer certificate` 到底缺的是什么。于是我在临时目录里给自己搭了一座三级证书链的假服务器——根 CA、中间 CA、一张给 localhost 的叶子证书——然后故意少发一张，看它怎么炸。

![三个玻璃链环，中间那环发黑裂开，裂缝里跳出一小点电火花，前景放着一把黄铜钥匙和一枚蜡封](/images/2026-11-25-tls-certificate-chain.jpg)

环境先摆出来：OpenSSL 3.5.7，curl 8.14.1。下面所有输出都是这台机器上实跑的。

### 第一步：先造一条三级链

根 CA 自己签自己：

```bash
cd ~/tls-lab
openssl req -x509 -newkey rsa:2048 -nodes -keyout rootCA.key -out rootCA.pem \
  -days 3650 -subj "/CN=Lab Root CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"
```

中间 CA 由根 CA 签，先把扩展写进一个文件：

```bash
cat > ca.ext <<'EOF'
basicConstraints=critical,CA:TRUE
keyUsage=critical,keyCertSign,cRLSign
subjectKeyIdentifier=hash
EOF

openssl req -new -newkey rsa:2048 -nodes -keyout inter.key -out inter.csr \
  -subj "/CN=Lab Intermediate CA"
openssl x509 -req -in inter.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial \
  -out inter.pem -days 1825 -extfile ca.ext
```

叶子证书由中间 CA 签，SAN 里写 `DNS:localhost,IP:127.0.0.1`：

```bash
cat > leaf.ext <<'EOF'
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:localhost,IP:127.0.0.1
EOF

openssl req -new -newkey rsa:2048 -nodes -keyout leaf.key -out leaf.csr -subj "/CN=localhost"
openssl x509 -req -in leaf.csr -CA inter.pem -CAkey inter.key -CAcreateserial \
  -out leaf.pem -days 397 -extfile leaf.ext
```

签完看一眼身份，这是判定有没有造对的标准：

```bash
openssl x509 -in leaf.pem -noout -subject -issuer -dates
```

输出是 `subject=CN=localhost`、`issuer=CN=Lab Intermediate CA`、有效期到 2027-12-27。`ca.ext` 里那个 `basicConstraints=critical,CA:TRUE` 必须有——我第一版漏了它，`openssl verify` 立刻回我 `error 79 at 1 depth lookup: invalid CA certificate`。

### 第二步：让它断给你看

服务器只发叶子证书：

```bash
openssl s_server -accept 8443 -cert leaf.pem -key leaf.key -www -quiet
```

客户端拿根 CA 去验：

```bash
curl -sS --cacert rootCA.pem -o /dev/null -w '%{http_code}\n' https://127.0.0.1:8443/
```

拿到的是 `curl: (60) SSL certificate problem: unable to get local issuer certificate`，状态码 `000`，退出码 60。openssl 自己也说同样的话：`error 20 at 0 depth lookup: unable to get local issuer certificate`。

判定标准是那句 "unable to get local issuer certificate" 加 `exit=60`——不是网络不通，是链断在半路。

### 第三步：错不止一种

同一个端口我用三种坏法各试了一遍，curl 的退出码全是 60，但文案不一样，这是区分病因的关键：

| 坏法 | curl 的原话 | openssl 的返回码 |
| --- | --- | --- |
| 服务器不发中间证书 | `unable to get local issuer certificate` | `20 (unable to get local issuer certificate)` |
| 证书里没有这个域名 | `no alternative certificate subject name matches target hostname` | `62 (hostname mismatch)` |
| 证书已经过期 | `certificate has expired` | `10 (certificate has expired)` |

域名不匹配那条我是用 `--resolve nope.example.test:8443:127.0.0.1` 故意访问一个不在 SAN 里的名字触发的；过期那条则用 `openssl ca` 指定 `-startdate 20250101000000Z -enddate 20250201000000Z` 签了一张去年就失效的证书。`openssl s_client -verify_return_error` 会把返回码摊开给你看：

```bash
echo | openssl s_client -connect 127.0.0.1:8443 -CAfile rootCA.pem -verify_return_error
```

链断时是 `depth=0 CN=localhost` 加 `Verify return code: 20`，链完整时是 `depth=2 CN=Lab Root CA` 一路数到 `depth=0`，最后 `Verify return code: 0 (ok)`。

### 第四步：两种修法

服务端修法是"把缺的那张补上"。这里我踩了个坑：把叶子证书和中间证书拼成一个文件 `cat leaf.pem inter.pem > fullchain.pem`，再用 `-cert fullchain.pem` 起服务器，结果 `openssl s_client -showcerts` 数出来仍然只有 1 张 `BEGIN CERTIFICATE`，curl 依旧报 60。`-cert` 只读文件里的第一张。正确姿势是分开给：

```bash
openssl s_server -accept 8444 -cert leaf.pem -cert_chain inter.pem -key leaf.key -www -quiet
```

再数一次就是 2 张，curl 也通了：

```bash
curl -sS --cacert rootCA.pem -o /dev/null -w 'http_code=%{http_code} size=%{size_download}\n' \
  https://127.0.0.1:8444/
```

输出 `http_code=200 size=5379`，退出码 0。

客户端兜底的办法是把根 CA 和中间证书拼成一个信任包，塞给 `SSL_CERT_FILE`：

```bash
cat rootCA.pem inter.pem > trust-bundle.pem
SSL_CERT_FILE=trust-bundle.pem curl -sS -o /dev/null -w '%{http_code}\n' https://127.0.0.1:8444/
```

也是 200。顺手提醒一句，`curl -k` 同样能拿到 200，但那等于把验证整个关掉，只能拿来临时看一眼，别写进脚本。

### 第五步：拿真站点对一遍

用同样的眼光看我自己的站。先把服务端发来的整条链存下来：

```bash
echo | openssl s_client -connect yoyolab-dev.github.io:443 \
  -servername yoyolab-dev.github.io -showcerts 2>/dev/null > real-chain.pem
grep -c 'BEGIN CERTIFICATE' real-chain.pem
openssl x509 -in real-chain.pem -out real-leaf.pem
```

答案是 3 张，`Verify return code: 0 (ok)`。叶子是 `CN=*.github.io`，签发者 `C=US, O=Let's Encrypt, CN=YR1`，有效期到 2026-12-29。顺手查一下它还能活多久：

```bash
openssl x509 -in real-leaf.pem -noout -checkend 2592000
```

不通过的证书会回 `Certificate will expire` 并让退出码变成 1；这张回的是 `Certificate will not expire`，退出码 0。

那天上午我把这套东西在纸上画了一遍：根 CA 是地基，中间 CA 是桥，叶子证书才是你手里那张纸。桥塌了或者桥上没有桥墩（服务器没发），你站在桥这头也走不到那头。curl 报的那句 `unable to get local issuer certificate`，翻译成人话就是"我手上这张纸，找不到能证明它可信的那一环"。
