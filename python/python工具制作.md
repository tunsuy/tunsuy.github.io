## Python工具制作

### 一、	基于setuptool
1、增加setup.cfg文件，填写相关内容及命令行，如下  
```python
[entry_points]
console_scripts =
    test = test.main:main
```
2、增加setup.py文件，进行安装部署使用
```python
try:
    import multiprocessing  # noqa
except ImportError:
    pass

setuptools.setup(
    setup_requires=['pbr'],
    pbr=True)
```

### 二、	基于oslo_config的cli
1、增加parser参数信息
```python
command_opt = cfg.SubCommandOpt('command',
                                title='Commands',
                                help='Show available commands.',
                            handler=add_command_parsers)

comman_opt是oslo_config->cfg.py中 class SubCommandOpt(Opt) 的一个实例
```

2、将command opt预置到CONF中
```python
class ConfigOpts(collections.Mapping):
CONF = ConfigOpts()
CONF.register_cli_opt(command_opt)
```
进入oslo_config->cfg.py中register_cli_opt方法，部分代码如下
```python
if cli:
    self._add_cli_opt(opt, None)

if _is_opt_registered(self._opts, opt):
    return False

self._opts[opt.dest] = {'opt': opt, 'cli': cli}
self._track_deprecated_opts(opt)
return True
```
可以看出，该方法主要是将cli_opt参数存储在CONF实例的_opts和_cli_opts属性中

3、将命令行参数注册到CONF中  
CONF(sys.argv[1:], project='karbor')  
因为CONF是oslo_config->cfg.py中ConfigOpts的一个实例，这里对实例进行调用，也就是调用了ConfigOpts的__call__方法。代码如下：
```python
def __call__(self,
             args=None,
             project=None,
             prog=None,
             version=None,
             usage=None,
             default_config_files=None,
             default_config_dirs=None,
             validate_default_values=False,
             description=None,
             epilog=None):
        self.clear()

    self._validate_default_values = validate_default_values

    prog, default_config_files, default_config_dirs = self._pre_setup(
        project, prog, version, usage, description, epilog,
        default_config_files, default_config_dirs)

    self._setup(project, prog, version, usage, default_config_files,
                default_config_dirs)

    self._namespace = self._parse_cli_opts(args if args is not None
                                           else sys.argv[1:])
    if self._namespace._files_not_found:
        raise ConfigFilesNotFoundError(self._namespace._files_not_found)
    if self._namespace._files_permission_denied:
        raise ConfigFilesPermissionDeniedError(
            self._namespace._files_permission_denied)

    self._check_required_opts()
```
主要查看下._parse_cli_opts方法：
```python
def _parse_cli_opts(self, args):
    self._args = args
    for opt, group in self._all_cli_opts():
        opt._add_to_cli(self._oparser, group)

    return self._parse_config_files()
```
该方法主要是将命令行中的参数注册到argparse中，查看_add_to_cli方法如下：
```python
def _add_to_cli(self, parser, group=None):
    """Add argparse sub-parsers and invoke the handler method."""
    dest = self.dest
    if group is not None:
        dest = group.name + '_' + dest

    subparsers = parser.add_subparsers(dest=dest,
                                       title=self.title,
                                       description=self.description,
                                       help=self.help)
    # NOTE(jd) Set explicitly to True for Python 3
    # See http://bugs.python.org/issue9253 for context
    subparsers.required = True

    if self.handler is not None:
        self.handler(subparsers)
```
这里的self.handler就是之前第一步中comman_opt传递进来的

4、执行命令行方法  
CONF.command.func()  
因为CONF是oslo_config->cfg.py中ConfigOpts的一个实例，这里是对实例的command属性进行调用，也就是调用了ConfigOpts的_getattr__方法。代码如下：
```python
def __getattr__(self, name):
    try:
        return self._get(name)
    except ValueError:
        raise
    except Exception:
        raise NoSuchOptError(name)
```
