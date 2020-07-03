## python插件加载机制

### 前言：stevedore库简介
stevedore是用来实现动态加载代码的开源模块。它是在OpenStack中用来加载插件的公共模块。可以独立于OpenStack而安装使用。  https://docs.openstack.org/stevedore/latest/  
stevedore使用setuptools的entry points来定义并加载插件。entry point引用的是定义在模块中的对象，比如类、函数、实例等，只要在import模块时能够被创建的对象都可以。

### 一：插件的名字和命名空间
一般来讲，entry point的名字是公开的，用户可见的，经常出现在配置文件中。而命名空间，也就是entry point组名却是一种实现细节，一般是面向开发者而非最终用户的。可以用Python的包名作为entry  point命名空间，以保证唯一性，但这不是必须的。  
entry points的主要特征就是，它可以是独立注册的，也就是说插件的开发和安装可以完全独立于使用它的应用，只要开发者和使用者在命名空间和API上达成一致即可。  
命名空间被用来搜索entry points。entry points的名字在给定的发布包中必须是唯一的，但在一个命名空间中可以不唯一。也就是说，同一个发布包内不允许出现同名的entry point，但是如果是两个独立的发布包，却可以使用完全相同的entrypoint组名和entry point名来注册插件。
    
### 二：插件的使用方式
在stevedore中，有三种使用插件的方式：Drivers、Hooks、Extensions
#### 1：Drivers        
一个名字对应一个entry point。使用时根据插件的命名空间和名字，定位到单独的插件  

#### 2：Hooks  
一个名字对应多个entry point。允许同一个命名空间中的插件具有相同的名字，根据给定的命名空间和名字，加载该名字对应的多个插件。  

#### 3：Extensions  
多个名字，多个entry point。给定命名空间，加载该命名空间中所有的插件，当然也允许同一个命名空间中的插件具有相同的名字。

### 三、插件加载流程
1、	客户端插件加载（protection为例）  
入口文件： Client_factory工厂文件  
将该文件目录下的clients目录下的所有客户端文件import进项目中  
注：以后想增加一个客户端，则只需将编写的客户端文件放在clients目录下即可  

2、	加载配置文件中定义的provider_registry 
使用了stevedore库的driver加载方式来加载插件  
以这个例子来说：
```python
def load_class(namespace, plugin_name):
    try:
        LOG.debug('Start load plugin %s. ', plugin_name)
        # Try to resolve plugin by name
        mgr = driver.DriverManager(namespace, plugin_name)
        return mgr.driver
    except RuntimeError as e1:
        # fallback to class name
        try:
            return importutils.import_class(plugin_name)
        except ImportError as e2:
            LOG.error("Error loading plugin by name, %s", e1)
            LOG.error("Error loading plugin by class, %s", e2)
            raise ImportError(_("Class not found."))
```
它加载了如图名字空间中的所有插件，这个名字空间在哪里定义的呢？  
是在相应项目的setup.py文件中定义的（如果项目采用的是py文件跟配置分离的方式，那么就是在setup.cfg中定义的）。  
Protection的所有插件就是定义在karbor项目的setup.cfg文件中的。  
Provider插件的配置信息如下：
```python
karbor.provider =
    provider-registry = karbor.services.protection.provider:ProviderRegistry
```
在加载provider_registry插件时，会实例化一个ProviderRegistry的实例，在实例化的过程中，会加载所有的providers。
```python
class ProviderRegistry(object):
    def __init__(self):
        super(ProviderRegistry, self).__init__()
        self.providers = {}
        self._load_providers()
```
加载providers：  
获取到dj.providers.d目录下所有.conf的配置文件     
打开其中volume文件，内容如下：  
```python
[provider]
name = Huawei Volume Backup Provider
description = This provider uses Huawei eBackup's services
id = 1dc03359-1301-467b-9972-1a7599ae791d

plugin=cbs-volume-protection-plugin
bank=cbs-bank-plugin
```
对这些每一个配置文件都分别实例化一个PluggableProtectionProvider。实例化过程中，同样的通过driver方式加载配置文件setup.cfg中定义的bank插件和plugin插件
```python
[entry_points]
console_scripts =
    karbor-api = kangaroo.cmd.api:main
    karbor-operationengine = kangaroo.cmd.operationengine:main
    karbor-protection = kangaroo.cmd.protection:main
    kangaroo-manage = kangaroo.cmd.manage:main
karbor.protections =
    cbs-volume-protection-plugin = kangaroo.plugin.volume.main:VolumeProtection
    cbs-bank-plugin = kangaroo.protection.bank_plugins.bank_plugin:DBBankPlugin
    cbs-vm-protection-plugin = kangaroo.plugin.server.karbor_plugin:karbor_nova_plugin
karbor.provider =
    dj-provider = kangaroo.protection.dj_provider:ProviderRegistry
```
这样就找到了卷备份插件的入口。  

3、	加载protection_registry  
使用了stevedore库的extention加载方式来加载插件  
以这个例子来说：
```python
    def load_plugins(self):
        """Load all protectable plugins configured and register them.

        """
        mgr = extension.ExtensionManager(
            namespace='karbor.protectables',
            invoke_on_load=True,
            on_load_failure_callback=_warn_missing_protectable)
```
protection配置的entry_points信息如下：
```python
[entry_points]
...
karbor.protections =
    karbor-swift-bank-plugin = karbor.services.protection.bank_plugins.swift_bank_plugin:SwiftBankPlugin
    karbor-fs-bank-plugin = karbor.services.protection.bank_plugins.file_system_bank_plugin:FileSystemBankPlugin
    karbor-s3-bank-plugin = karbor.services.protection.bank_plugins.s3_bank_plugin:S3BankPlugin
```

