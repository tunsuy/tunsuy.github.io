# zookeeper认证机制

## 1、client

* 将认证data和hmac值通过kmc加密写入配置文件；
* 请求的时候，将认证data和hmac值解密后，并将认证data加盐值配上hmac值通过python的hmac加密加入到zk的请求auth中；

注: 也可以采用其他加密方式；
    盐值可以是时间戳，便于进行访问频率的限制

伪代码：
```python
from kazoo.client import KazooClient
from kazoo.security import make_acl
from base64 import b64encode
import hashlib

def zk_client():
	ZKClient.zkClient = KazooClient(hosts=self.options.servers,
									timeout=ZKClient.TIMEOUT,
									connection_retry=KazooRetry(
										max_delay=1800))
	ZKClient.zkClient.add_listener(zk_listener)
	ZKClient.zkClient.start()
	auth = build_auth_info()
	acl = get_set_acl_val()
	ZKClient.zkClient.add_auth(self.scheme, auth)
	ZKClient.zkClient.default_acl = (acl,)
	bConnectedZk = True
	
def zk_listener():
	pass
	
def build_auth_info():
	return "认证信息" + ":" + "时间戳"
	
def get_set_acl_val():
    scheme = "test"

    name_hash = b64encode(
        hashlib.sha256("认证信息").digest()
    ).strip()
    cred_hash = b64encode(
        hashlib.sha256("时间戳").digest()
    ).strip()
    auth = name_hash.decode('utf-8') + ":" + cred_hash.decode('utf-8')
    acl = make_acl(scheme, auth, all=True)
    return acl
```

## 2、server

* 认证信息配置文件：包含wcc加密后的data、hmac值。
* 获取请求过来的认证信息，解析出摘要和盐值；
* 将配置文件中的认证data、hmac值通过wcc解密；
* 将解密后的认证data加盐值配上hmac值通过hmac加密；
* 将加密出来的值跟请求中的数字摘要比对是否一致。

注：还可以增加请求ip锁定机制
    通过请求中的时间戳盐值，检查是否访问频繁。

伪代码：
```java
public class XXXAuthenticationProvider implements AuthenticationProvider
{
    
    private ConfigurationProvider config;


    public EnhancedAuthenticationProvider()
    {
        pass
    }

    @Override
    public String getScheme()
    {
        return "test";
    }

    @Override
    public Code handleAuthentication(ServerCnxn cnxn, byte[] authData)
    {
        String id = new String(authData);

        if (!isDigestEqual(id))
        {
            return Code.AUTHFAILED;
        }
        
        try
        {
            cnxn.addAuthInfo(new Id(getScheme(), id + ":" + clientIp));
        }
        catch (Exception e)
        {
            pass
        }

        return Code.OK;
    }

    /**
     *
     * @param id
     * @return
     */
    private boolean isDigestEqual(String id)
    {
        int offset = id.lastIndexOf(":");
        if (-1 == offset)
        {
            // 没有找到的话，不存在随机盐值
            todo
        }

        // 获取盐值（时间戳）
        String salt = id.substring(offset + 1);

        // 获取认证信息
        String digest = id.substring(0, offset);

        // 是否检查时间间隔
        if (config.isCheckAuthTime())
        {
            int authTime = Integer.parseInt(salt);
            int curTime = (int) (System.currentTimeMillis() / 1000);

            // 时间间隔大于最小时间间隔时，鉴权报错
            if (config.getDifferTime() < Math.abs(authTime - curTime))
            {
                return false;
            }
        }

        String authData = config.getAuthData();
        if (null == authData)
        {
            return false;
        }

        String hmacData = config.getHmacData();
        if (null == hmacData)
        {
            return false;
        }

        String decodeHmacData = HMACSHA256.decode(hmacData);
        String decodeData = HMACSHA256.decode(authData);
        String authKeyDigest = HMACSHA256.reEncode(decodeData, salt, decodeHmacData);
        if (authKeyDigest == null)
        {
            return false;
        }

        return authKeyDigest.equals(digest);

    }

    @Override
    public boolean matches(String id, String aclExpr)
    {
        return true;
    }

    @Override
    public boolean isAuthenticated()
    {
        return true;
    }

    @Override
    public boolean isValid(String id)
    {
        return true;
    }

}
```


acl：http://jm.taobao.org/2011/05/30/947/