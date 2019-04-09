三方库：keystonemiddleware  
入口：auth_token/__init__.py#filter_factory()方法  

通过之前的wsgi知识可以知道，消息在中间件中传递的时候是调用的中间件的call方法；  
这里是调用了父类BaseAuthProtocol的call方法，然后调用到AuthProtocol的process_request方法；  
如果token非法则抛出异常返回401；合法则业务继续向下流动。  

### 大体流程
1.去除header中已有的认证信息，防止token伪造
2.初始化_token_cache，后续用来缓存token
3.调用父类BaseAuthProtocol的process_request方法
4.后续处理，认证不通过抛出异常，通过在请求体中塞入认证信息等

### 配置文件
```python
[keystone_authtoken]
token_cache_time = 300
keystone_offline_time = 0
check_revocations_for_cached = false
revocation_cache_time = 86400
insecure = true
cafile = xxx
auth_section = trustee

[trustee]
auth_type = password
auth_url = xxxx
username = ts
password = xxx
user_domain_name = op_service
insecure = true
```
注：上述流程都需要使用到配置文件中的相关配置
这些字段都是在keystonemiddleware里面定义好了的，配置文件中不能变更


### BaseAuthProtocol的process_request方法
根据header中塞入的内容不同分别走走不同的流程：service_token和user_token；
两者大体流程一致。

#### 判断token是否已经缓存
1.如果缓存了，判断当前token是否被撤销了（通过revoked.pem）
2.如果没有缓存，则先使用离线方式校验

注：keystonemiddleware支持mamcache做缓存，只需要部署一套mamcache集群，对接相关配置即可。


##### 离线校验方式
1.先根据token类型不同进行解压，然后_cms_verify进行处理；
2.通过本地ca、cert证书文件采用openssl命令来验证CMS签名的合法性。
3.如果是证书配置相关的异常（证书不存在、格式不对等），则通过keystoneclient调用接口去请求证书并保存到本地，以便后续直接本地离线校验。然后再次通过openssl命令进行校验，再次失败则抛出异常。
4.如果是其他异常，则直接抛出token不合法的异常。

注：1、本地的证书文件是否有过期删除机制（暂时没找到）
2、有可能出现这样的问题：本地的证书文件还是旧的，但是api请求中的token已经是更新之后的证书生成的，这样就会导致请求会报401.
解决方法：对该三方库打补丁，在_cms_verify方法中修改如下代码：
```python
    931         try:
    932             return verify()
    933         except ksc_exceptions.CertificateConfigError:
    934             # the certs might be missing; unconditionally fetch to avoid racing
    935             self._fetch_signing_cert()
    936             self._fetch_ca_cert()
    937 
    938             try:
    939                 # retry with certs in place
    940                 return verify()
    941             except ksc_exceptions.CertificateConfigError as err:
    942                 # if this is still occurring, something else is wrong and we
    943                 # need err.output to identify the problem
    944                 self.log.error(_LE('CMS Verify output: %s'), err.output)
    945                 raise
    946         except ksm_exceptions.InvalidToken:
    947             # Used for mo_pki certificate updates,
    948             # while local authentication files are still old.
    949             self._fetch_signing_cert()
    950             self._fetch_ca_cert()
    951 
    952             # retry with certs in place
    953             return verify()
```
其中
```python
except ksm_exceptions.InvalidToken:
	# Used for mo_pki certificate updates,
	# while local authentication files are still old.
	self._fetch_signing_cert()
	self._fetch_ca_cert()

	# retry with certs in place
	return verify()
```
为新增的代码片段。

##### 请求证书内容
keystonemiddleware中使用keystoneClient客户端去请求证书内容，
其中keystoneClient就是访问的配置文件中定义的auth_url地址（auth_url节点服务器就是获取token的地方）。


#### _validate_token
主要验证token是否快超期

#### _confirm_token_bind
根据下面的策略
class _BIND_MODE(object): DISABLED = ‘disabled’ PERMISSIVE = ‘permissive’ STRICT = ‘strict’ REQUIRED = ‘required’ KERBEROS = ‘kerberos’
然后判断绑定的是否合理



参考文档：<https://www.cnblogs.com/menkeyi/p/6995372.html>

