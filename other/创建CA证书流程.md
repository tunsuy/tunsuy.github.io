# 创建CA证书流程

本文介绍了如何创建自己的证书颁发机构以及如何创建由该证书颁发机构签名的SSL证书。  
尽管有许多文章讨论如何创建自己的SSL证书，但在大多数情况下，它们描述了如何创建自签名证书。这比较简单，但是无法验证或跟踪那些证书。  
我个人更喜欢先创建个人证书颁发机构（CA），然后再从该证书颁发机构颁发证书。这种方法的主要优点是，你可以将CA的证书导入浏览器或手机中，并且当你访问自己的网站或连接到SMTP/IMAP服务器时，不会再收到任何警告。现在被认为是值得信赖的。如果你为自己的项目创建证书层次结构，并且希望成为唯一可以为用户颁发证书的人员，则这也是必要的。

## 文件用途
```sh
ca
├── certs               证书目录
├── crl                 证书吊销目录
├── index.txt           CA 签发证书列表
├── index.txt.attr      CA 签发证书列表配置
├── newcerts            CA 签发的证书备份
├── openssl.cnf         openssl 配置文件，-config 参数用
├── private             私钥目录
└── serial              CA 下一次签发证书时使用的序列号
```

## 自签发证书

### 创建证书颁发机构配置
```sh
[ ca ]
default_ca = mypersonalca

[ mypersonalca ]
#
# WARNING: if you change that, change the default_keyfile in the [req] section below too
# Where everything is kept
dir = ./mypersonalca

# Where the issued certs are kept
certs = $dir/certs

# Where the issued crl are kept
crl_dir = $dir/crl

# database index file
database = $dir/index.txt

# default place for new certs
new_certs_dir = $dir/certs

#
# The CA certificate
certificate = $dir/certs/ca.pem

# The current serial number
serial = $dir/serial

# The current CRL
crl = $dir/crl/crl.pem

# WARNING: if you change that, change the default_keyfile in the [req] section below too
# The private key
private_key = $dir/private/ca.key

# private random number file
RANDFILE = $dir/private/.rand

# The extentions to add to the cert
x509_extensions = usr_cert

# how long to certify for
default_days = 365

# how long before next CRL
default_crl_days= 30

# which message digest to use
default_md = sha256

# keep passed DN ordering
preserve = no

# Section names
policy = mypolicy
x509_extensions = certificate_extensions

[ mypolicy ]
# Use the supplied information
commonName = supplied
stateOrProvinceName = supplied
countryName = supplied
emailAddress = supplied
organizationName = supplied
organizationalUnitName = optional

[ certificate_extensions ]
# The signed certificate cannot be used as CA
basicConstraints = CA:false

[ req ]
# same as private_key
default_keyfile = ./mypersonalca/private/ca.key

# Which hash to use
default_md = sha256

# No prompts
prompt = no

# This is for CA
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
string_mask = utf8only
basicConstraints = CA:true
distinguished_name = root_ca_distinguished_name
x509_extensions = root_ca_extensions

[ root_ca_distinguished_name ]
# EDIT THOSE
commonName = My Personal CA
stateOrProvinceName = California
countryName = US
emailAddress = certs@example.com
organizationName = My Personal Certification Authority

[ root_ca_extensions ]
basicConstraints = CA:true
```

在磁盘上创建目录，然后保存以上配置文件命名为ca.cnf。你可以编辑标记为“ EDIT THOSE”的参数，还可以更改一些参数（例如，如果你希望证书的有效期超过一年，则可以更改default_days），但是默认值应该足够绝大多数用户
```sh
# mkdir -p mypersonalca/certs
# mkdir -p mypersonalca/private
# mkdir -p mypersonalca/crl
# echo "01" > mypersonalca/serial
# touch mypersonalca/index.txt
```

### 生成证书颁发机构的私钥和证书
要生成具有2048位私钥和有效期为十年（3650天）的证书的证书颁发机构，请执行以下命令：
```sh
OPENSSL=ca.cnf openssl req -x509 -nodes -days 3650 \
    -newkey rsa:2048 -out mypersonalca/certs/ca.pem \
    -outform PEM -keyout ./mypersonalca/private/ca.key
```
它将询问你输入有关CA证书中的信息问题。做出有意义的回答，这样当你在浏览器中看到此证书时，就不会怀疑它的含义。
完成上述命令后，会得到两个文件：`mypersonalca/certs/ca.pem`和`mypersonalca/private/ca.key`。密钥文件必须保密。任何获得ca.key的人都可以为你的CA签署证书。 ca.pem文件是你的公共CA证书，可以将其导入到浏览器或移动平台中，以使设备可以识别您的根CA。
现在有了CA密钥，可以开始生成和签名证书了。

获得有效证书的过程分为两个阶段。
* 生成证书
* 使用CA对其进行签名

### 生成证书
首先将生成证书。以下命令用于生成证书和1024位私钥：
```sh
openssl req -newkey rsa:1024 -nodes -sha256 \
   -keyout cert.key -keyform PEM -out cert.req -outform PEM
```
该命令将询问有关证书颁发者的一些问题。连接到你的服务器的任何人都可以看到此信息。如果你为Web或邮件服务器创建SSL证书，请特别注意“公用名”字段。你必须在此处输入此证书将使用的标准域名。例如，如果你的Web服务器是https://mymail.example.com，则你的通用名称必须是mymail.example.com。你也可以在域名的第一部分使用通配符：example.com的所有子域都可以使用具有通用名称（例如* .example.com）的证书。尽管不一定要提供电子邮件地址，但也必须提供。

### 由CA签署证书
证书一旦创建，就需要由你的CA签名，以便可识别：
```sh
OPENSSL_CONF=ca.cnf openssl ca -batch -notext -in cert.req -out cert.pem
```
现在，你获得了一对：签名证书cert.pem和相应的私钥cert.key，您可以根据需要使用它们。

### 打印证书参数
有时你可能需要查看特定的证书，以查看其中的内容。以下命令将打印cert.pem证书的内容：
```sh
openssl x509 -in cert.pem -noout -text
```

## 多级CA

OpenSSL的常规设置信息。
本示例创建一个根证书颁发机构，一个中间签名颁发机构，然后创建一个示例证书。在实践中，Root和Intermediate将位于不同的服务器上，而Root甚至可能被锁定在不错且安全的地方。
一切都在一个基本目录中创建，每个CA都有一些子目录，然后是证书和配置等。
私钥应受密码保护，在这些示例中，我使用“ rootpass”和“ intermediatepass”。
```sh
.
├── certificate-configs
├── certs
├── intermediate-ca
└── root-ca
```
我们将在任何地方都将基本目录称为$ BASE，只是为了清楚地说明每一步的位置。

### 先决条件
```sh
mkdir -p $BASE/certs
mkdir -p $BASE/root-ca
cd $BASE/root-ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo unique_subject = yes > index.txt.attr
echo 1000 > serial
echo 1000 > crlnumber
```

### 设置主要配置
使用以下命令创建一个名为$ BASE / root-ca / openssl.cnf的文件：
```sh
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = .
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/root.key.pem
certificate       = $dir/certs/root.cert.pem
crlnumber         = $dir/crlnumber
crl               = $dir/crl/root.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 730

default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict

[ policy_strict ]
countryName            = match
stateOrProvinceName    = match
localityName           = optional
organizationName       = match
organizationalUnitName = optional
commonName             = supplied
emailAddress           = optional

[ req ]
prompt                 = no
string_mask            = utf8only
default_bits           = 2048
default_md             = sha256
distinguished_name     = req_distinguished_name
x509_extensions        = v3_ca

[ req_distinguished_name ]
countryName            = GB
stateOrProvinceName    = Surrey
localityName           = Camberley
0.organizationName     = CylCorp
organizationalUnitName = Security
commonName             = CylCorp Root CA

[ v3_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign
extendedKeyUsage       = serverAuth
crlDistributionPoints  = @crl_section
authorityInfoAccess    = @ocsp_section

[ v3_intermediate_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign
authorityInfoAccess    = @ocsp_section
crlDistributionPoints  = @crl_section

[ crl_ext ]
authorityKeyIdentifier = keyid:always
issuerAltName          = issuer:copy 

[ ocsp ]
basicConstraints       = CA:FALSE
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid, issuer
keyUsage               = critical, digitalSignature
extendedKeyUsage       = critical, OCSPSigning

[alt_names]
DNS.0 = Sandbox Intermidiate CA 1
DNS.1 = Sandbox CA Intermidiate 1

[crl_section]
URI.0 = http://pki.cylindric.net/CylCorpRoot.crl
URI.1 = http://pki.cylindric.net/SandboxRoot.crl

[ocsp_section]
caIssuers;URI.0 = http://pki.cylindric.net/CylCorpRoot.crt
caIssuers;URI.1 = http://pki.backup.cylindric.net/CylCorpRoot.crt
OCSP;URI.0 = http://pki.cylindric.net/ocsp/
OCSP;URI.1 = http://pki.backup.cylindric.net/ocsp/
```

### 创建根密钥
```sh
cd $BASE/root-ca

openssl genrsa \
        -aes256 \
        -out private/root.key.pem \
        -passout pass:rootpass 4096

chmod 400 private/root.key.pem
```

### 创建CA证书
```sh
cd $BASE/root-ca

openssl req \
        -config openssl.cnf \
        -key private/root.key.pem \
        -passin pass:rootpass \
        -new \
        -x509 \
        -days 7300 \
        -extensions v3_ca \
        -out certs/root.cert.pem

chmod 444 certs/root.cert.pem
```

### SSL中级CA

#### 配置中级CA
```sh
cd $BASE
mkdir -p $BASE/intermediate-ca
cd $BASE/intermediate-ca
mkdir certs crl csr newcerts private
chmod 700 private
echo 1000 > serial
echo 1000 > crlnumber
touch index.txt
echo unique_subject = yes > index.txt.attr
```

```sh
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = .
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/intermediate.key.pem
certificate       = $dir/certs/intermediate.cert.pem
crlnumber         = $dir/crlnumber
crl               = $dir/crl/intermediate.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 365
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_loose

[ policy_loose ]
countryName            = optional
stateOrProvinceName    = optional
localityName           = optional
organizationName       = optional
organizationalUnitName = optional
commonName             = supplied
emailAddress           = optional

[ req ]
prompt                 = no
string_mask            = utf8only
default_bits           = 2048
default_md             = sha256
distinguished_name     = req_distinguished_name
x509_extensions        = v3_ca

[ req_distinguished_name ]
countryName            = GB
stateOrProvinceName    = Surrey
localityName           = Camberley
0.organizationName     = CylCorp
organizationalUnitName = Security
commonName             = CylCorp Intermediate CA

[ v3_ca ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints       = critical, CA:true, pathlen:0
keyUsage               = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
authorityKeyIdentifier = keyid:always

[crl_section]
URI.0 = http://pki.cylindric.net/CylCorpIntermediate.crl
URI.1 = http://pki.cylindric.net/CylCorpIntermediate.crl

[ocsp_section]
caIssuers;URI.0 = http://pki.cylindric.net/CylCorpIntermediate.crt
caIssuers;URI.1 = http://pki.backup.cylindric.net/CylCorpIntermediate.crt
OCSP;URI.0 = http://pki.cylindric.net/ocsp/
OCSP;URI.1 = http://pki.backup.cylindric.net/ocsp/
```

#### 创建中级证书
```sh
cd $BASE/intermediate-ca

openssl genrsa \
        -aes256 \
        -out private/intermediate.key.pem \
        -passout pass:intermediatepass 4096

chmod 400 private/intermediate.key.pem
```

#### 创建中级CA请求
```sh
cd $BASE/intermediate-ca

openssl req \
        -config openssl.cnf \
        -new -sha256 \
        -key private/intermediate.key.pem \
        -passin pass:intermediatepass \
        -out csr/intermediate.csr.pem
```

#### 创建中级CA证书
该证书是由根CA签署的，因此请注意，我们现在位于root-ca中...
```sh
cd $BASE/root-ca

openssl ca \
        -batch \
        -config openssl.cnf \
        -extensions v3_intermediate_ca \
        -days 3650 \
        -notext \
        -in ../intermediate-ca/csr/intermediate.csr.pem \
        -passin pass:rootpass \
        -out ../intermediate-ca/certs/intermediate.cert.pem

rm -f ../intermediate-ca/csr/intermediate.csr.pem

chmod 444 ../intermediate-ca/certs/intermediate.cert.pem
```

#### 更新CRL

在每次添加或撤销证书后，请确保CRL是最新的，这一点很重要。
```sh
cd $BASE/root-ca

openssl ca \
        -config openssl.cnf \
        -gencrl \
        -passin pass:rootpass \
        -cert certs/root.cert.pem \
        -out crl/root.crl.pem

openssl crl \
        -inform PEM \
        -in crl/root.crl.pem \
        -outform DER \
        -out crl/root.crl

cd $BASE/intermediate-ca

openssl ca \
        -config openssl.cnf \
        -gencrl \
        -passin pass:intermediatepass \
        -cert certs/intermediate.cert.pem \
        -out crl/intermediate.crl.pem

openssl crl \
        -inform PEM \
        -in crl/intermediate.crl.pem \
        -outform DER \
        -out crl/intermediate.crl

```

### 创建证书链
```sh
cd $BASE

cat intermediate-ca/certs/intermediate.cert.pem root-ca/certs/root.cert.pem > certs/ca-chain.cert.pem
chmod 444 certs/ca-chain.cert.pem
```
