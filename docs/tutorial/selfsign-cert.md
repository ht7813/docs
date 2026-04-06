# 制作自签SSL/Windows代码签名证书

!!! info "声明"
    本篇文章的部分内容来自于**AI生成**或**网络查找**，可能未经证实

!!! warning "注意"
    自签证书仅供本地/内部测试使用，生产环境请使用正式证书（如Let's Encrypt, Digicert等）

在本篇文章中，我将教大家如何创建自签 SSL/Windows代码签名 证书

## 基本概念

### 证书主体（Subject）

证书主体（Subject）用于标识数字证书的持有者，包括个人、组织或服务器的详细信息，是证书验证和身份识别的核心字段

在OpenSSL命令行参数填写证书主体时，命令格式一般是这样的：

```shell
-subj "/C=国家/ST=省份/L=城市/O=组织/OU=部门/CN=通用名称/emailAddress=邮箱"
```

#### 各字段含义

| 字段 | 全称 | 含义 | 示例 | 是否必需 |
|------|------|------|------|---------|
| **C** | Country | 国家代码（2字母） | `CN`（中国）、`US`（美国） | 可选 |
| **ST** | State/Province | 州/省 | `Beijing`、`Guangdong` | 可选 |
| **L** | Location/City | 城市 | `Shanghai`、`Shenzhen` | 可选 |
| **O** | Organization | 组织/公司名 | `Google Inc.`、`MyCompany` | 可选 |
| **OU** | Organizational Unit | 部门/单位 | `IT Department`、`Security Team` | 可选 |
| **CN** | Common Name | **通用名称**（最重要） | `example.com`、`localhost` | **强烈建议** |
| **emailAddress** | Email | 电子邮箱 | `admin@example.com` | 可选 |

#### 各字段的作用

##### **CN (Common Name) - 最关键**

- **作用**：指定证书要保护的域名或IP
- **浏览器验证**：访问的域名必须与CN匹配

!!! warning "注意"
    现代浏览器（Chrome 58+）会优先检查Subject Alternative Name (SAN) ，没有SAN时会检查CN

```bash
# 示例：保护特定域名
-subj "/CN=mysite.com"

# 示例：保护IP地址
-subj "/CN=192.168.1.100"

# 示例：本地开发
-subj "/CN=localhost"
```

##### **C (Country)**
- 2字母国家代码，遵循ISO 3166-1标准
- 常见代码：`CN`中国、`US`美国、`JP`日本、`KR`韩国、`DE`德国

##### **ST (State/Province)**
- 省份或州名称
- 建议使用英文或拼音：`Beijing`、`Shanghai`、`Guangdong`

##### **L (Location)**
- 城市名称
- 示例：`Haidian District`、`Pudong`

##### **O (Organization)**
- 公司或组织全称
- 如果是个人证书，可以写个人姓名

##### **OU (Organizational Unit)**
- 部门名称
- 示例：`Technical Support`、`Sales Department`

## 准备工作

### OpenSSL

#### Linux

一般情况下，OpenSSL可以从各发行版的包管理器下载到

```shell
# Ubuntu/Debian
sudo apt install openssl

# CentOS/RHEL/Fedora (New)
sudo dnf install openssl

# CentOS/RHEL (Old)
sudo yum install openssl

# Arch Linux
sudo pacman -S openssl

# Alpine
sudo apk add openssl
```

#### MacOS

你可以用Homebrew下载OpenSSL
```shell
brew install openssl
```

#### Windows

##### MSYS2 安装

如果你有MSYS2环境，那么可以直接安装（一般情况下已安装）

```shell
pacman -S openssl
```

##### 二进制文件安装

你可以从[此网站](https://slproweb.com/products/Win32OpenSSL.html)上下载到OpenSSL安装包，并进行环境变量(PATH)配置

!!! note "提示"
    如果只需要二进制文件 (openssl.exe)，不使用开发文件 (include, lib...)，一般只要下Light版就够了

### Signtool（代码签名用）

!!! danger "注意"
    该工具仅支持Windows

Signtool可以从[Windows SDK](https://learn.microsoft.com/zh-cn/windows/apps/windows-sdk/downloads)获取（安装组件里必须包含Signing Tools）。

之后进行环境变量(PATH)配置 （建议但不是必须）

## SSL证书

### 生成

!!! danger "注意"
    请务必将证书主体和SAN中的Alt Names替换成自己的！！！

#### 仅生成一个根证书

!!! warning "不推荐此方法"
    我们并不推荐这种生成方法，因为你要依次安装每一个应用程序的CA证书，不便管理

##### 一键生成（最简单）

```shell
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout server.key \
  -out server.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=MyOrg/OU=IT/CN=localhost"
```

##### 分步生成（更灵活）

```shell
# 1. 生成私钥
openssl genrsa -out server.key 2048

# 2. 生成证书签名请求(CSR)
openssl req -new -key server.key -out server.csr -subj "/C=CN/ST=Beijing/L=Beijing/O=MyOrg/OU=IT/CN=localhost"

# 3. 生成自签名证书
openssl x509 -req -in server.csr -signkey server.key -out server.crt -days 365
```

##### 生成带SAN的证书

!!! info "提示"
    推荐给证书加上Subject Alternative Name (SAN)，这样一个证书可以验证多个IP和DNS名

```shell
# 1. 创建配置文件 san.conf
cat > san.conf <<EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[dn]
C=CN
ST=Beijing
L=Beijing
O=MyOrg
OU=IT
CN=localhost

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = *.localhost
DNS.3 = example.com
IP.1 = 127.0.0.1
IP.2 = 192.168.1.100
EOF

# 2. 生成带SAN的证书
openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt \
  -days 365 -config san.conf -extensions v3_req

# 3. 查看证书信息验证SAN
openssl x509 -in server.crt -text -noout | grep -A1 "Subject Alternative Name"
```

#### 生成证书链（CA->Server证书）（推荐）

##### 流程图

```
根证书 (Root CA)
    ↓ 签名
中间证书 (Intermediate CA) [可选]
    ↓ 签名
服务器证书 (Server Certificate)
    ↓ 使用
你的网站/服务
```

##### 创建根证书（Root CA）

```shell
# 1.1 生成根证书私钥
openssl genrsa -out rootCA.key 4096

# 1.2 生成根证书（有效期10年，自签名）
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 \
  -out rootCA.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=MyRootCA/OU=CA/CN=My Root CA"

# 1.3 查看根证书
openssl x509 -in rootCA.crt -text -noout
```

##### 创建中间证书（Intermediate CA）[可选但推荐]

```shell
# 2.1 生成中间证书私钥
openssl genrsa -out intermediateCA.key 4096

# 2.2 生成中间证书签名请求(CSR)
openssl req -new -key intermediateCA.key -sha256 \
  -out intermediateCA.csr \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=MyIntermediateCA/OU=CA/CN=My Intermediate CA"

# 2.3 创建中间证书扩展配置文件
cat > intermediate.ext <<EOF
basicConstraints = critical, CA:TRUE, pathlen:0
keyUsage = critical, digitalSignature, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
EOF

# 2.4 用根证书签发中间证书（有效期5年）
openssl x509 -req -in intermediateCA.csr \
  -CA rootCA.crt -CAkey rootCA.key \
  -CAcreateserial -out intermediateCA.crt \
  -days 1825 -sha256 -extfile intermediate.ext
```

##### 创建服务器证书（由中间证书签发）

```shell
# 3.1 生成服务器私钥
openssl genrsa -out server.key 2048

# 3.2 生成服务器证书CSR
openssl req -new -key server.key -sha256 \
  -out server.csr \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=MyCompany/OU=IT/CN=myserver.local"

# 3.3 创建服务器证书扩展配置（包含SAN）
cat > server.ext <<EOF
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = myserver.local
DNS.2 = www.myserver.local
DNS.3 = *.myserver.local
IP.1 = 127.0.0.1
IP.2 = 192.168.1.100
EOF

# 3.4 用中间证书签发服务器证书（有效期1年）
openssl x509 -req -in server.csr \
  -CA intermediateCA.crt -CAkey intermediateCA.key \
  -CAcreateserial -out server.crt \
  -days 365 -sha256 -extfile server.ext
```

### 使用
!!! danger "注意"
    在使用自签证书前，请务必安装对应的根证书，否则系统不会识别！！！

    ```shell
    # Windows
    certutil -addstore "Root" server.crt # 或使用“证书导入向导”，导入到"受信任的根证书颁发机构"

    # MacOS
    sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain server.crt

    # Linux (Debian/Ubuntu)
    sudo cp server.crt /usr/local/share/ca-certificates/
    sudo update-ca-certificates

    # Linux (CentOS/RHEL)
    sudo cp server.crt /etc/pki/ca-trust/source/anchors/
    sudo update-ca-trust
    ```

在Web服务器中，需要配置证书才能开启SSL (HTTPS)功能

下面以Nginx为示例

```nginx
server {
    # 开启HTTPS
    listen 443 ssl;
    # 要和前面的CN或SAN里的值相同！！！
    server_name myserver.local;
    
    # 服务器证书
    ssl_certificate /path/to/server.crt;
    ssl_certificate_key /path/to/server.key;

    # ...
}
```

## Windows 代码签名证书

### 生成

#### Windows SDK 直接生成

##### 创建证书和私钥

```shell
makecert -sv codesign.pvk -r -n "CN=Test" codesign.cer
```

##### 创建SPC文件

```shell
cert2spc codesign.cer codesign.spc
```

##### 创建PFX文件

```shell
pvk2pfx -pvk codesign.pvk -pi [PASSWORD] -spc codesign.spc -pfx codesign.pfx -f
```

#### 基于CA证书生成 (OpenSSL)

##### 创建代码签名证书配置文件

```shell
# 创建 codesign.conf
cat > codesign.conf <<EOF
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[dn]
C=CN
ST=Beijing
L=Beijing
O=MyCompany Inc.
OU=Software Development
CN=MyCompany Code Signing Certificate
emailAddress=security@mycompany.com

[v3_req]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, codeSigning
subjectKeyIdentifier = hash
EOF
```

##### 生成代码签名证书

```shell
# 1. 生成代码签名私钥
openssl genrsa -out codesign.key 2048

# 2. 生成CSR
openssl req -new -key codesign.key -out codesign.csr -config codesign.conf

# 3. 创建扩展文件（代码签名专用）
cat > codesign.ext <<EOF
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, codeSigning
subjectAltName = otherName:1.3.6.1.4.1.311.20.2.3;UTF8:MyCompany Software
authorityKeyIdentifier = keyid,issuer
EOF

# 4. 用CA签发代码签名证书
openssl x509 -req -in codesign.csr \
  -CA intermediateCA.crt -CAkey intermediateCA.key \ 
  -CAcreateserial -out codesign.crt \
  -days 365 -sha256 -extfile codesign.ext # 如果没有中间CA则用rootCA.(crt/key)

# 5. 验证证书
openssl x509 -in codesign.crt -text -noout | grep -A5 "X509v3 Extended Key Usage"
# 应该输出: codeSigning
```

##### 转换为PKCS#12格式

```shell
# 合并证书链（代码签名证书 + 中间证书 + 根证书）
cat codesign.crt intermediateCA.crt rootCA.crt > codesign-chain.crt

# 转换为PFX格式
openssl pkcs12 -export \
  -out codesign.pfx \
  -inkey codesign.key \
  -in codesign-chain.crt \
  -passout pass:[PASSWORD]  # 设置密码，或留空使用 -passout pass:

# 如果需要无密码（不推荐）
openssl pkcs12 -export -out codesign.pfx -inkey codesign.key -in codesign-chain.crt -passout pass:
```

### 使用

!!! danger "注意"
    在使用自签证书前，请务必安装对应的根证书，否则系统不会识别！！！

    ```shell
    # 以管理员身份运行PowerShell/CMD

    # 安装根证书到"受信任的根证书颁发机构"
    certutil -addstore "Root" rootCA.crt

    # 安装中间证书到"中间证书颁发机构"（如果有）
    certutil -addstore "CA" intermediateCA.crt

    # 导入PFX到个人证书存储
    certutil -user -p [PASSWORD] -importpfx codesign.pfx

    # 或使用证书导入向导
    ```

使用SignTool可以签名Windows应用程序

```shell
# SHA-1
signtool sign /f codesign.pfx /p [PASSWORD] /t http://timestamp.sectigo.com/authenticode /v /fd sha1 app.exe

# SHA-256
signtool sign /f codesign.pfx /p [PASSWORD] /fd sha256 /tr http://timestamp.sectigo.com/?td=sha256 /td sha256 /as /v app.exe
```

## 结语

至此，我们已经完整地走过了自签证书从概念到实践的整个旅程。从理解证书主体的各个字段含义，到动手生成SSL/TLS证书，再到构建完整的CA证书链，最后深入到Windows代码签名证书的制作与使用——这一路走来，相信你已经掌握了自签证书的核心知识。

### 回顾与总结

**SSL证书部分**，我们学习了：

- 使用OpenSSL一键生成或分步生成自签证书
- 通过SAN扩展让单个证书支持多个域名和IP
- 构建根CA→中间CA→服务器证书的完整证书链体系
- 在Nginx等Web服务器中正确配置证书

**代码签名部分**，我们掌握了：

- 使用Windows SDK工具链快速生成测试证书
- 基于OpenSSL和已有CA体系生成更规范的代码签名证书
- 将证书转换为PFX格式并导入Windows证书存储
- 使用SignTool对应用程序进行SHA-1/SHA-256双签名

### 重要提醒

!!! warning "请记住"
    自签证书的定位始终是**开发测试**工具。它非常适合：
    - 本地开发环境（localhost）
    - 企业内部测试网络
    - 学习SSL/TLS工作原理
    - 验证代码签名流程
    
    但**永远不要**在生产环境、面向公网的服务或对外分发的软件中使用自签证书。对于生产环境，请选择Let's Encrypt（免费）、DigiCert、Sectigo等受信任的CA机构签发的正式证书。

### 后续学习建议

如果你希望进一步深入，可以探索以下方向：
- **证书自动化管理**：使用Certbot、acme.sh等工具自动续期Let's Encrypt证书
- **证书透明度（CT）**：了解现代证书生态的公开审计机制
- **硬件安全模块（HSM）**：企业级私钥保护方案
- **mTLS（双向TLS）**：服务间身份认证的高级应用

### 最后的叮嘱

证书和加密技术是网络安全的重要基石。掌握自签证书的制作与使用，不仅能帮助你解决开发测试中的实际问题，更能让你深入理解PKI（公钥基础设施）的工作原理。希望这份指南能成为你探索网络安全世界的起点，在实践过程中遇到问题时，请随时查阅官方文档或社区资源。

祝你在加密与安全的道路上一帆风顺！🔐

---

*本指南会持续更新，如发现错误或有改进建议，欢迎反馈。*