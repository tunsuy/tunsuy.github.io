# oslo_log详解



## context中的TLS

在oslo_context库的context.log入口定义了一个TLS线程变量
```python
_request_store = threading.local()
```
凡是引入该context的都会首先初始化这样一个变量

oslo_context库提供一个接口供调用方更新自己的context
oslo_context/context.log#RequestContext
```python
def update_store(self):
	"""Store the context in the current thread."""
	_request_store.context = self
```

同时提供这样一个接口获取当前的context实例
```python
def get_current():
    """Return this thread's current context

    If no context is set, returns None
    """
    return getattr(_request_store, 'context', None)
```

## 初始化log
oslo_log/log.py提供了一个外部接口`setup`
```python
def setup(conf, product_name, version='unknown'):
    """Setup logging for the current application."""
    if conf.log_config_append:
        _load_log_config(conf.log_config_append)
    else:
        _setup_logging_from_conf(conf, product_name, version)
    sys.excepthook = _create_logging_excepthook(product_name)
```
该方法配合oslo_conf库非常优雅的实现了让调用方初始化log。
调用方只需要在配置文件中配置相关的参数，然后在程序启动的时候，调用下该方法，后续就只需调用相应的日志等级方法打印日志`self.log.info("xxxx")`

下面来说下内部做了哪些初始化

进入`setup`中的`_setup_logging_from_conf`方法
```python
def _setup_logging_from_conf(conf, project, version):
    log_root = getLogger(None).logger

    # Remove all handlers
    for handler in list(log_root.handlers):
        log_root.removeHandler(handler)

    logpath = _get_log_file_path(conf)
    if logpath:
        # On Windows, in-use files cannot be moved or deleted.
        if conf.watch_log_file and platform.system() == 'Linux':
            from oslo_log import watchers
            file_handler = watchers.FastWatchedFileHandler
            filelog = file_handler(logpath)
        elif conf.log_rotation_type.lower() == "interval":
            file_handler = logging.handlers.TimedRotatingFileHandler
            when = conf.log_rotate_interval_type.lower()
            interval_type = LOG_ROTATE_INTERVAL_MAPPING[when]
            # When weekday is configured, "when" has to be a value between
            # 'w0'-'w6' (w0 for Monday, w1 for Tuesday, and so on)'
            if interval_type == 'w':
                interval_type = interval_type + str(conf.log_rotate_interval)
            filelog = file_handler(logpath,
                                   when=interval_type,
                                   interval=conf.log_rotate_interval,
                                   backupCount=conf.max_logfile_count)
        elif conf.log_rotation_type.lower() == "size":
            file_handler = logging.handlers.RotatingFileHandler
            maxBytes = conf.max_logfile_size_mb * units.Mi
            filelog = file_handler(logpath,
                                   maxBytes=maxBytes,
                                   backupCount=conf.max_logfile_count)
        else:
            file_handler = logging.handlers.WatchedFileHandler
            filelog = file_handler(logpath)

        log_root.addHandler(filelog)

    if conf.use_stderr:
        streamlog = handlers.ColorHandler()
        log_root.addHandler(streamlog)
```

### 获取logger
上面的方法中有这样一句`log_root = getLogger(None).logger`，获取logger
```python
def getLogger(name=None, project='unknown', version='unknown'):
    """Build a logger with the given name.

    :param name: The name for the logger. This is usually the module
                 name, ``__name__``.
    :type name: string
    :param project: The name of the project, to be injected into log
                    messages. For example, ``'nova'``.
    :type project: string
    :param version: The version of the project, to be injected into log
                    messages. For example, ``'2014.2'``.
    :type version: string
    """
    # NOTE(dhellmann): To maintain backwards compatibility with the
    # old oslo namespace package logger configurations, and to make it
    # possible to control all oslo logging with one logger node, we
    # replace "oslo_" with "oslo." so that modules under the new
    # non-namespaced packages get loggers as though they are.
    if name and name.startswith('oslo_'):
        name = 'oslo.' + name[5:]
    if name not in _loggers:
        _loggers[name] = KeywordArgumentAdapter(logging.getLogger(name),
                                                {'project': project,
                                                 'version': version})
    return _loggers[name]
```

该类`KeywordArgumentAdapter`继承自logging库的__init__.py中的`LoggerAdapter`类
并传递了这样一个logger类对`KeywordArgumentAdapter`进行实例化

进入logging库的__init__.py中的`getLogger`方法
```python
def getLogger(name=None):
    """
    Return a logger with the specified name, creating it if necessary.

    If no name is specified, return the root logger.
    """
    if name:
        return Logger.manager.getLogger(name)
    else:
        return root
```

进入`Manager`类的`getLogger`方法
```python
    def getLogger(self, name):
        """
        Get a logger with the specified name (channel name), creating it
        if it doesn't yet exist. This name is a dot-separated hierarchical
        name, such as "a", "a.b", "a.b.c" or similar.

        If a PlaceHolder existed for the specified name [i.e. the logger
        didn't exist but a child of it did], replace it with the created
        logger and fix up the parent/child references which pointed to the
        placeholder to now point to the logger.
        """
        rv = None
        if not isinstance(name, str):
            raise TypeError('A logger name must be a string')
        _acquireLock()
        try:
            if name in self.loggerDict:
                rv = self.loggerDict[name]
                if isinstance(rv, PlaceHolder):
                    ph = rv
                    rv = (self.loggerClass or _loggerClass)(name)
                    rv.manager = self
                    self.loggerDict[name] = rv
                    self._fixupChildren(ph, rv)
                    self._fixupParents(rv)
            else:
                rv = (self.loggerClass or _loggerClass)(name)
                rv.manager = self
                self.loggerDict[name] = rv
                self._fixupParents(rv)
        finally:
            _releaseLock()
        return rv
```
该方法中调用这样一句`rv = (self.loggerClass or _loggerClass)(name)`

到了这里我们就知道了上文中oslo_log中`_setup_logging_from_conf`方法的logger实例为logging库的__init__.py中的`Logger`类

### 增加handlers
我们回到`_setup_logging_from_conf`方法中，继续往下看
```python
log_root.addHandler(filelog)
```
这里调用了logging库的__init__.py中的`Logger`类的`addHandler`方法
```python
def addHandler(self, hdlr):
	"""
	Add the specified handler to this logger.
	"""
	_acquireLock()
	try:
		if not (hdlr in self.handlers):
			self.handlers.append(hdlr)
	finally:
		_releaseLock()
```

### 设置formatter
我们回到`_setup_logging_from_conf`方法中，继续往下看
```python
if not conf.use_json:
	for handler in log_root.handlers:
		handler.setFormatter(formatters.ContextFormatter(project=project,
														 version=version,
														 datefmt=datefmt,
														 config=conf))
else:
	for handler in log_root.handlers:
		handler.setFormatter(formatters.JSONFormatter(datefmt=datefmt))
```

这样整个log初始化就完成了

## 使用log
使用log的时候最终都会进入logging库的__init__.py中的`Logger`类中的`_log`方法
```python
def _log(self, level, msg, args, exc_info=None, extra=None, stack_info=False):
	"""
	Low-level logging routine which creates a LogRecord and then calls
	all the handlers of this logger to handle the record.
	"""
	sinfo = None
	if _srcfile:
		#IronPython doesn't track Python frames, so findCaller raises an
		#exception on some versions of IronPython. We trap it here so that
		#IronPython can use logging.
		try:
			fn, lno, func, sinfo = self.findCaller(stack_info)
		except ValueError: # pragma: no cover
			fn, lno, func = "(unknown file)", 0, "(unknown function)"
	else: # pragma: no cover
		fn, lno, func = "(unknown file)", 0, "(unknown function)"
	if exc_info:
		if isinstance(exc_info, BaseException):
			exc_info = (type(exc_info), exc_info, exc_info.__traceback__)
		elif not isinstance(exc_info, tuple):
			exc_info = sys.exc_info()
	record = self.makeRecord(self.name, level, fn, lno, msg, args,
							 exc_info, func, extra, sinfo)
	self.handle(record)
```
进入`handle`方法
```python
def handle(self, record):
	"""
	Call the handlers for the specified record.

	This method is used for unpickled records received from a socket, as
	well as those created locally. Logger-level filtering is applied.
	"""
	if (not self.disabled) and self.filter(record):
		self.callHandlers(record)

def callHandlers(self, record):
	"""
	Pass a record to all relevant handlers.

	Loop through all handlers for this logger and its parents in the
	logger hierarchy. If no handler was found, output a one-off error
	message to sys.stderr. Stop searching up the hierarchy whenever a
	logger with the "propagate" attribute set to zero is found - that
	will be the last logger whose handlers are called.
	"""
	c = self
	found = 0
	while c:
		for hdlr in c.handlers:
			found = found + 1
			if record.levelno >= hdlr.level:
				hdlr.handle(record)
		if not c.propagate:
			c = None    #break out
		else:
			c = c.parent
	if (found == 0):
		if lastResort:
			if record.levelno >= lastResort.level:
				lastResort.handle(record)
		elif raiseExceptions and not self.manager.emittedNoHandlerWarning:
			sys.stderr.write("No handlers could be found for logger"
							 " \"%s\"\n" % self.name)
			self.manager.emittedNoHandlerWarning = True		
```
到了这里我们发现，最终会调用在setup中介绍到的各个`handler`的`handle`方法

这里以`WatchedFileHandler`处理器为例
所有的handler均继承自logging库的__init__.py中的`Handler`类，调用各子类的handle方法实际上是调用的该基类的handle
```python
def handle(self, record):
	"""
	Conditionally emit the specified logging record.

	Emission depends on filters which may have been added to the handler.
	Wrap the actual emission of the record with acquisition/release of
	the I/O thread lock. Returns whether the filter passed the record for
	emission.
	"""
	rv = self.filter(record)
	if rv:
		self.acquire()
		try:
			self.emit(record)
		finally:
			self.release()
	return rv
```
各子类重写了`emit`方法
```python
def emit(self, record):
	"""
	Emit a record.

	If underlying file has changed, reopen the file before emitting the
	record to it.
	"""
	self.reopenIfNeeded()
	logging.FileHandler.emit(self, record)
```
经过层层调用，最终进入到logging库的__init__.py中的`Handler`类
```python
def format(self, record):
	"""
	Format the specified record.

	If a formatter is set, use it. Otherwise, use the default formatter
	for the module.
	"""
	if self.formatter:
		fmt = self.formatter
	else:
		fmt = _defaultFormatter
	return fmt.format(record)
```
通过上面`增加handlers`和`设置formatter`，我们知道，这里的`fmt`实际上就是oslo_log库的formatters.py下的`ContextFormater`类实例
```python
def format(self, record):
	"""Uses contextstring if request_id is set, otherwise default."""
	if six.PY2:
		should_use_unicode = True
		args = (record.args.values() if isinstance(record.args, dict)
				else record.args)
		for arg in args or []:
			try:
				six.text_type(arg)
			except UnicodeDecodeError:
				should_use_unicode = False
				break
		if (not isinstance(record.msg, six.text_type)
				and should_use_unicode):
			record.msg = _ensure_unicode(record.msg)

	# store project info
	record.project = self.project
	record.version = self.version

	# FIXME(dims): We need a better way to pick up the instance
	# or instance_uuid parameters from the kwargs from say
	# LOG.info or LOG.warn
	instance_extra = ''
	instance = getattr(record, 'instance', None)
	instance_uuid = getattr(record, 'instance_uuid', None)
	context = _update_record_with_context(record)
	if instance:
		try:
			instance_extra = (self.conf.instance_format
							  % instance)
		except TypeError:
			instance_extra = instance
	elif instance_uuid:
		instance_extra = (self.conf.instance_uuid_format
						  % {'uuid': instance_uuid})
	elif context:
		# FIXME(dhellmann): We should replace these nova-isms with
		# more generic handling in the Context class.  See the
		# app-agnostic-logging-parameters blueprint.
		instance = getattr(context, 'instance', None)
		instance_uuid = getattr(context, 'instance_uuid', None)

		# resource_uuid was introduced in oslo_context's
		# RequestContext
		resource_uuid = getattr(context, 'resource_uuid', None)

		if instance:
			instance_extra = (self.conf.instance_format
							  % {'uuid': instance})
		elif instance_uuid:
			instance_extra = (self.conf.instance_uuid_format
							  % {'uuid': instance_uuid})
		elif resource_uuid:
			instance_extra = (self.conf.instance_uuid_format
							  % {'uuid': resource_uuid})

	record.instance = instance_extra

	# NOTE(sdague): default the fancier formatting params
	# to an empty string so we don't throw an exception if
	# they get used
	for key in ('instance', 'color', 'user_identity', 'resource',
				'user_name', 'project_name'):
		if key not in record.__dict__:
			record.__dict__[key] = ''

	# Set the "user_identity" value of "logging_context_format_string"
	# by using "logging_user_identity_format" and
	# get_logging_values of oslo.context.
	if context:
		record.user_identity = (
			self.conf.logging_user_identity_format %
			_ReplaceFalseValue(_dictify_context(context))
		)

	if record.__dict__.get('request_id'):
		fmt = self.conf.logging_context_format_string
	else:
		fmt = self.conf.logging_default_format_string

	# Cache the formatted traceback on the record, Logger will
	# respect our formatted copy
	if record.exc_info:
		record.exc_text = self.formatException(record.exc_info, record)

	record.error_summary = _get_error_summary(record)
	if '%(error_summary)s' in fmt:
		# If we have been told explicitly how to format the error
		# summary, make sure there is always a default value for
		# it.
		record.error_summary = record.error_summary or '-'
	elif record.error_summary:
		# If we have not been told how to format the error and
		# there is an error to summarize, make sure the format
		# string includes the bits we need to include it.
		fmt += ': %(error_summary)s'

	if (record.levelno == logging.DEBUG and
			self.conf.logging_debug_format_suffix):
		fmt += " " + self.conf.logging_debug_format_suffix

	self._compute_iso_time(record)

	if sys.version_info < (3, 2):
		self._fmt = fmt
	else:
		self._style = logging.PercentStyle(fmt)
		self._fmt = self._style._fmt

	try:
		return logging.Formatter.format(self, record)
	except TypeError as err:
		# Something went wrong, report that instead so we at least
		# get the error message.
		record.msg = 'Error formatting log line msg={!r} err={!r}'.format(
			record.msg, err).replace('%', '*')
		return logging.Formatter.format(self, record)
```

在该方法中调用了这样一个方法获取context
```python
def _update_record_with_context(record):
    """Given a log record, update it with context information.

    The request context, if there is one, will either be passed with the
    incoming record or in the global thread-local store.
    """
    context = record.__dict__.get(
        'context',
        context_utils.get_current()
    )
    if context:
        d = _dictify_context(context)
        # Copy the context values directly onto the record so they can be
        # used by the formatting strings.
        for k, v in d.items():
            setattr(record, k, v)

    return context
```
这里会将context中的属性一一加入到record中

然后回到format方法中
```python
if record.__dict__.get('request_id'):
	fmt = self.conf.logging_context_format_string
else:
	fmt = self.conf.logging_default_format_string
```
根据contex是否带有request_id来决定输出什么格式的日志