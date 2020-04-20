# rabbitmq加密套件详解

## TLS协议
TLS协议有三个作用：验证，防篡改，加密。这三个作用也基本上是密码学相关的三个应用。

验证是可以同时支持客户端验证服务端和服务端验证客户端两个需求的，只是在大部分的cs应用场景下，都是客户端验证服务端即可，主要目的是为了防止网站伪造，防钓鱼网站的目的。

防篡改的主要密码学方法是哈希算法，各个版本的SSL/TLS握手应用了大量的不同的哈希算法。

加密在TLS中有两个主要的体现，一个是握手的过程中的非对称加密用以建立信道，一个信道建立之后的对称加密用于实际的通信。之所以分为两个是因为对称加密在功能上不同完成非对称加密的密钥协商的功能，而非对称加密在性能上达不到对称加密的数据要求。通常一个加密信道建立之后，这个信道都有一个存活的时间，如果存活的时间太长，理论上就会有被破解的风险，因为对称加密的信道相关的参数在协商确定后是基本不变的。所以即使是长连接也需要隔一段时间重新进行TLS握手，这个过程叫做重协商。
通常一个TLS握手之后的连接工程上可以持续的时间远比理论值要长很多，因为不是所有的流量都有必要要防止攻击成本非常高的攻击的。可能只有支付等极其敏感的业务需要高严格的安全性参数考虑。所以重协商功能在工程上通常是不启用的，nginx就默认关闭了这个功能（还有其他安全上的考虑）。即使我们需要重协商，在客户端编程的时候，也可以直接的采用重写发起一个连接来完成，业务简单清爽，带来的性能开销也并不会多太多。因为信道过期并不是一个非常频繁的操作，更多的是信道持续不到信道过期的时间。TLS1.3已经取消了重协商的机制。

## 密码学套件
密码学套件是TLS发展了一段时间积累了很多密码学使用的经验之后提出的一整套的解决方案。一个套件中包含了应用于整个握手和传输使用到的所有非对称加密，对称加密和哈希算法，甚至包括证书的类型。最早期的SSL虽然也许要一系列的加密算法，但是这些算法并不是像现在的称为密码学套件（Cipher Suite），而是被称作密码选择（Cipher Choice）。密码学套件是SSLv3开始提出的概念。后续的版本在升级的时候会产生新的安全强度更高的密码学套件，同时抛弃比较弱的密码学套件。

一个密码学套件是完成整个TLS握手的关键。在TLS握手的时候ClientHello里面携带了客户端支持的密码学套件列表，ServerHello中携带了Server根据Client提供的密码学套件列表中选择的本地也支持的密码学套件。也就是说选择使用什么密码学套件的选择权在Server的手里（在使用nginx的时候把ssl_prefer_server_ciphers配置项打开），而Server通常可以通过配置文件指定Server要支持的密码学套件的列表和顺序。为此需求，OpenSSL专门定义了一套比较难懂的定义方法，Nginx的密码学套件的配置方法也只是对OpenSSL定义的定义形式的透传。也就是说使用Nginx定义的ssl_prefer_server_ciphers配置项是原封不动的传输给OpenSSL的。
使用OpenSSL的工具程序就可以验证分析一个密码学套件字符串：openssl -v cipher 'RC4:HIGH:!aNULL:!MD5' ，就可以看到RC4:HIGH:!aNULL:!MD5 这个密码学套件配置的详细内容。这个命令是专门用于解析OpenSSL的密码学套件的配置的工具，由于字符串格式的配置比较难学习，使用这个命令可以起到有效的帮助。

密码套件分为三大部分：密钥交换算法，数据加密算法，消息验证算法（MAC,message authentication code）。
* 密钥交换算法用于握手过程中建立信道，一般采用非对称加密算法。
* 数据加密算法用于信道建立之后的加密传输数据，一般采用对称加密算法。
* MAC顾名思义是一种哈希，用于验证消息的完整性，包括整个握手流程的完整性（例如TLS握手的最后一步就是一个对已有的握手消息的全盘哈希计算的过程）。

TLS_DHE_RSA_WITH_AES_256_CBC_SHA是一个密码学套件的标准名字。
* 这里的TLS代表的是TLS协议，如果未来TLS改名，这个名字可能会变，否则会一直是这个名字。
* WITH是一个分隔单次，WITH前面的表示的是握手过程所使用的非对称加密方法，WITH后面的表示的是加密信道的对称加密方法和用于数据完整性检查的哈希方法。WITH前面通常有两个单词，第一个单词是约定密钥交换的协议，第二个单词是约定证书的验证算法。要区别这两个域，必须要首先明白，两个节点之间交换信息和证书本身是两个不同的独立的功能。两个功能都需要使用非对称加密算法。交换信息使用的非对称加密算法是第一个单词，证书使用的非对称加密算法是第二个。有的证书套件，例如TLS_RSA_WITH_AES_256_CBC_SHA，WITH单词前面只有一个RSA单词，这时就表示交换算法和证书算法都是使用的RSA，所以只指定一次即可。可选的主要的密钥交换算法包括: RSA, DH, ECDH, ECDHE。可选的主要的证书算法包括：RSA, DSA, ECDSA。两者可以独立选择，并不冲突。
* AES_256_CBC指的是AES这种对称加密算法的256位算法的CBC模式，AES本身是一类对称加密算法的统称，实际的使用时要指定位数和计算模式，CBC就是一种基于块的计算模式。
* 最后一个SHA就是代码计算一个消息完整性的哈希算法。

## rabbitmq加密套件
### 列出rabbitmq支持的加密套件
要列出正在运行的节点的Erlang运行时支持的密码套件，请使用`Rabbitmq-diagnostics cipher_suites --openssl-format`
```sh
rabbitmq-diagnostics cipher_suites --openssl-format -q
```
这将生成OpenSSL格式的密码套件列表。

请注意，如果`--openssl-format`设置为`false`
```sh
rabbitmq-diagnostics cipher_suites -q --openssl-format=false
```
这个命令`rabbitmq-diagnostics cipher_suites`将以仅在经典样式配置文件中接受的格式列出密码套件。

两种rabbitmq配置样式文件中均接受OpenSSL格式，但是如果是在经典样式配置文件中，需要加双引号。

上述命令列出的密码套件采用的格式适用于客户端TLS连接的socket加密。它们与配置值加密使用的密码不一样。

使用经典样式配置文件时，可以使用以下命令，因为它将生成可用于该文件格式的密码套件列表：
```sh
rabbitmq-diagnostics cipher_suites --openssl-format=false --formatter=erlang -q
```

### 配置加密套件
在新样式配置文件中，使用ssl_options.ciphers配置选项配置密码套件。
以下示例演示了如何使用该选项。
```sh
listeners.ssl.1 = 5671

ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile   = /path/to/server_certificate.pem
ssl_options.keyfile    = /path/to/server_key.pem
ssl_options.versions.1 = tlsv1.2
ssl_options.versions.2 = tlsv1.1

ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = false

ssl_options.ciphers.1  = ECDHE-ECDSA-AES256-GCM-SHA384
ssl_options.ciphers.2  = ECDHE-RSA-AES256-GCM-SHA384
ssl_options.ciphers.3  = ECDHE-ECDSA-AES256-SHA384
ssl_options.ciphers.4  = ECDHE-RSA-AES256-SHA384
ssl_options.ciphers.5  = ECDH-ECDSA-AES256-GCM-SHA384
ssl_options.ciphers.6  = ECDH-RSA-AES256-GCM-SHA384
ssl_options.ciphers.7  = ECDH-ECDSA-AES256-SHA384
ssl_options.ciphers.8  = ECDH-RSA-AES256-SHA384
ssl_options.ciphers.9  = DHE-RSA-AES256-GCM-SHA384
ssl_options.ciphers.10 = DHE-DSS-AES256-GCM-SHA384
ssl_options.ciphers.11 = DHE-RSA-AES256-SHA256
ssl_options.ciphers.12 = DHE-DSS-AES256-SHA256
ssl_options.ciphers.13 = ECDHE-ECDSA-AES128-GCM-SHA256
ssl_options.ciphers.14 = ECDHE-RSA-AES128-GCM-SHA256
ssl_options.ciphers.15 = ECDHE-ECDSA-AES128-SHA256
ssl_options.ciphers.16 = ECDHE-RSA-AES128-SHA256
ssl_options.ciphers.17 = ECDH-ECDSA-AES128-GCM-SHA256
ssl_options.ciphers.18 = ECDH-RSA-AES128-GCM-SHA256
ssl_options.ciphers.19 = ECDH-ECDSA-AES128-SHA256
ssl_options.ciphers.20 = ECDH-RSA-AES128-SHA256
ssl_options.ciphers.21 = DHE-RSA-AES128-GCM-SHA256
ssl_options.ciphers.22 = DHE-DSS-AES128-GCM-SHA256
ssl_options.ciphers.23 = DHE-RSA-AES128-SHA256
ssl_options.ciphers.24 = DHE-DSS-AES128-SHA256
ssl_options.ciphers.25 = ECDHE-ECDSA-AES256-SHA
ssl_options.ciphers.26 = ECDHE-RSA-AES256-SHA
ssl_options.ciphers.27 = DHE-RSA-AES256-SHA
ssl_options.ciphers.28 = DHE-DSS-AES256-SHA
ssl_options.ciphers.29 = ECDH-ECDSA-AES256-SHA
ssl_options.ciphers.30 = ECDH-RSA-AES256-SHA
ssl_options.ciphers.31 = ECDHE-ECDSA-AES128-SHA
ssl_options.ciphers.32 = ECDHE-RSA-AES128-SHA
ssl_options.ciphers.33 = DHE-RSA-AES128-SHA
ssl_options.ciphers.34 = DHE-DSS-AES128-SHA
ssl_options.ciphers.35 = ECDH-ECDSA-AES128-SHA
ssl_options.ciphers.36 = ECDH-RSA-AES128-SHA

ssl_options.honor_cipher_order = true
ssl_options.honor_ecc_order    = true
```

经典配置格式：
```sh
%% list allowed ciphers
[
 {ssl, [{versions, ['tlsv1.2', 'tlsv1.1']}]},
 {rabbit, [
           {ssl_listeners, [5671]},
           {ssl_options, [{cacertfile,"/path/to/ca_certificate.pem"},
                          {certfile,  "/path/to/server_certificate.pem"},
                          {keyfile,   "/path/to/server_key.pem"},
                          {versions, ['tlsv1.2', 'tlsv1.1']},
                          %% This list is just an example!
                          %% Not all cipher suites are available on all machines.
                          %% Cipher suite order is important: preferred suites
                          %% should be listed first.
                          %% Different suites have different security and CPU load characteristics.
                          {ciphers,  [
                            "ECDHE-ECDSA-AES256-GCM-SHA384",
                            "ECDHE-RSA-AES256-GCM-SHA384",
                            "ECDHE-ECDSA-AES256-SHA384",
                            "ECDHE-RSA-AES256-SHA384",
                            "ECDH-ECDSA-AES256-GCM-SHA384",
                            "ECDH-RSA-AES256-GCM-SHA384",
                            "ECDH-ECDSA-AES256-SHA384",
                            "ECDH-RSA-AES256-SHA384",
                            "DHE-RSA-AES256-GCM-SHA384",
                            "DHE-DSS-AES256-GCM-SHA384",
                            "DHE-RSA-AES256-SHA256",
                            "DHE-DSS-AES256-SHA256",
                            "ECDHE-ECDSA-AES128-GCM-SHA256",
                            "ECDHE-RSA-AES128-GCM-SHA256",
                            "ECDHE-ECDSA-AES128-SHA256",
                            "ECDHE-RSA-AES128-SHA256",
                            "ECDH-ECDSA-AES128-GCM-SHA256",
                            "ECDH-RSA-AES128-GCM-SHA256",
                            "ECDH-ECDSA-AES128-SHA256",
                            "ECDH-RSA-AES128-SHA256",
                            "DHE-RSA-AES128-GCM-SHA256",
                            "DHE-DSS-AES128-GCM-SHA256",
                            "DHE-RSA-AES128-SHA256",
                            "DHE-DSS-AES128-SHA256",
                            "ECDHE-ECDSA-AES256-SHA",
                            "ECDHE-RSA-AES256-SHA",
                            "DHE-RSA-AES256-SHA",
                            "DHE-DSS-AES256-SHA",
                            "ECDH-ECDSA-AES256-SHA",
                            "ECDH-RSA-AES256-SHA",
                            "ECDHE-ECDSA-AES128-SHA",
                            "ECDHE-RSA-AES128-SHA",
                            "DHE-RSA-AES128-SHA",
                            "DHE-DSS-AES128-SHA",
                            "ECDH-ECDSA-AES128-SHA",
                            "ECDH-RSA-AES128-SHA"
                            ]}
                         ]}
          ]}
].
```

### 加密套件顺序
在TLS连接协商期间，服务器和客户端将协商使用哪种密码套件。可以强制服务器的TLS指示其首选项（根据密码套件顺序），以避免恶意客户端故意对弱密码套件进行协商进而对其进行攻击。为此，请将`honor_cipher_order和honor_ecc_order`配置为`true`：
```sh
listeners.ssl.1        = 5671
ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile   = /path/to/server_certificate.pem
ssl_options.keyfile    = /path/to/server_key.pem
ssl_options.versions.1 = tlsv1.2
ssl_options.versions.2 = tlsv1.1

ssl_options.honor_cipher_order = true
ssl_options.honor_ecc_order    = true
```

经典格式：
```sh
%% Enforce server-provided cipher suite order (preference)
[
 {ssl, [{versions, ['tlsv1.2', 'tlsv1.1']}]},
 {rabbit, [
           {ssl_listeners, [5671]},
           {ssl_options, [{cacertfile, "/path/to/ca_certificate.pem"},
                          {certfile,   "/path/to/server_certificate.pem"},
                          {keyfile,    "/path/to/server_key.pem"},
                          {versions,   ['tlsv1.2', 'tlsv1.1']},

                          %% ...


                          {honor_cipher_order,   true},
                          {honor_ecc_order,      true},
                         ]}
          ]}
].
```

### 评估TLS设置
由于TLS具有许多可配置的参数，并且出于历史原因，其中一些参数的默认设置不理想，因此建议使用TLS设置评估。存在多种工具，这些工具可以在启用TLS的服务器端点上执行各种测试，例如，测试它是否易于受到已知攻击（如POODLE，BEAST等）的攻击。

testssl.sh是成熟且扩展的TLS端点测试工具，可与不为HTTP服务的协议端点一起使用。请注意，该工具执行许多测试（例如，在某些计算机上，它运行超过350个密码套件测试），并且通过每个测试对于每个环境都可能有意义，也可能没有意义。例如，许多生产部署不使用CRL（证书吊销列表）。大多数开发环境使用自签名证书，而不必担心启用的最佳密码套件集；等等。

脚本地址：https://testssl.sh/