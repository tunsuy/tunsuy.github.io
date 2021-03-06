## Postgresql访问控制

PostgreSQL对使用SSL连接加密客户端/服务器通信以提高安全性提供了本机支持。

### 客户端验证服务器证书
默认情况下，PostgreSQL将不对服务器证书进行任何验证。这意味着可以在客户端不知情的情况下假冒服务器身份（例如，通过修改DNS记录或接管服务器IP地址）。为了防止欺骗，客户端必须能够通过信任链来验证服务器的身份。通过在一台计算机上放置根（自签名）证书颁发机构（CA）证书，并在另一台计算机上放置由根证书签名的工作证书来建立信任链。也可以使用由根证书签名和签署工作证书的“中间”证书。

要允许客户端验证服务器的身份，请在客户端上放置一个根证书，并在服务器上放置一个由该根证书签名的工作证书。要允许服务器验证客户端的身份，请在服务器上放置根证书，并在客户端上放置由根证书签名的工作证书。一个或多个中间证书（通常与工作证书一起存储）也可以用于将工作证书链接到根证书。

建立信任链后，客户端可以通过两种方式来验证服务器发送的工作证书。如果参数`sslmode`设置为`verify-ca`，libpq将通过检查证书链（直到存储在客户端上的根证书）来验证服务器是否可信。如果sslmode设置为verify-full，libpq还将验证服务器主机名是否与存储在服务器证书中的名称匹配。如果无法验证服务器证书，则SSL连接将失败。建议在大多数对安全性敏感的环境中使用“完全验证”。

在`verify-full`模式下，主机名与证书的`Subject Alternative Name(SN)`属性匹配，或者如果不存在类型为dNSName的`Subject Alternative Name(SN)`，则与`Common Name(CN)`属性匹配。如果证书的`name`属性以星号`（*）`开头，则星号将被视为通配符，它将匹配除点`（.）`之外的所有字符。这意味着证书将不匹配子域。如果使用IP地址而不是主机名进行连接，则IP地址将被匹配（不进行任何DNS查找）。

要允许服务器证书验证，必须在用户主目录的`~/.postgresql/root.crt`文件中放置一个或多个根证书。如果需要中级证书以将服务器发送的证书链链接到客户端上存储的根证书，则还应将它们添加到文件中。
如果文件`~/.postgresql/root.crl`存在，还将检查证书吊销列表（CRL）条目。
可以通过设置连接参数`sslrootcert`和`sslcrl`或环境变量`PGSSLROOTCERT`和`PGSSLCRL`来更改根证书文件和CRL的位置。

### 提供不同模式的保护
sslmode参数的不同值提供不同级别的保护。 SSL可以针对三种类型的攻击提供保护：
##### 1、窃听
如果第三方可以监测客户端与服务器之间的网络流量，则它可以读取连接信息（包括用户名和密码）以及所传递的数据。 SSL使用加密来防止这种情况。

##### 2、中间人（MITM）
如果第三方可以在客户端和服务器之间传递数据时修改数据，则它可以伪装成服务器，因此即使加密了数据也可以查看和修改数据。然后，第三方可以将连接信息和数据转发到原始服务器，从而无法检测到此攻击。这样做的常见媒介包括DNS中毒和地址劫持，从而将客户端定向到与预期不同的服务器。还有其他几种攻击方法可以实现此目的。 SSL通过使客户端通过证书来验证服务器来防止这种情况。

##### 3、假冒
如果第三方可以伪装成授权客户，则可以简单地访问它不应该访问的数据。通常，这可能通过不安全的密码管理发生。 SSL通过确保只有有效证书的持有者才能访问服务器，从而使用客户端证书来防止这种情况。

所有SSL选项都以加密和密钥交换的形式带来开销，因此必须在性能和安全性之间进行权衡。下面说明了不同sslmode值可以防范的风险，以及它们对安全性和开销的说明。
* disable: 只尝试非SSL连接。
* allow: 首先尝试非SSL连接，如果连接失败，再尝试SSL连接。
* prefer: 首先尝试SSL连接，如果连接失败，将尝试一个非SSL连接。
* require: 可以防止窃听；只尝试SSL连接。默认信任网络，认为自己想要连接的服务器就是正确的服务器。
* verify_ca: 可以防止窃听，是否能够防止中间人攻击需要依赖ca证书策略；只尝试SSL连接，并且验证服务器证书是否由可信任的证书机构颁发。
* verify_full: 安全性最高，可以防止窃听和中间人攻击；只尝试SSL连接，并且验证服务器证书是否由可信任的证书机构颁发，以及验证服务器主机名是否与证书中的一致。
  默认值：prefer

verify-ca和verify-full之间的差异取决于根CA的策略。如果使用公共CA，则verify-ca允许连接到其他人可能已经在CA注册的服务器。在这种情况下，应始终使用“verify-full”。如果使用本地CA，甚至使用自签名证书，则使用verify-ca通常会提供足够的保护。
sslmode的默认值为`prefer`。如表中所示，从安全角度来看，这没有任何意义，并且仅在可能的情况下保证性能开销。仅将它作为向后兼容性的默认值提供，在安全部署中不建议使用。