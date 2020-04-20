## 如何实现一个web下载功能
### 1、web端
这里采用了angularjs框架
demo代码如下：
```js
function get_file() {
	$http.get("url", {responseType: "blob"}).success(function (data) {
		var BOM = "\uFEFF"
		var blob = new Blob([BOM + data], {type: "text/csv;charset=utf-8;"});
		if (blob.size > 0) {
			var a = document.createElement("a");
			document.body.appendChild(a);
			a.download = "ts.csv";
			a.href = URL.createObjectURL(blob);
			a.click();
		} else {
			console.log("download file failed.")
		}
	}).catch(function (reason) {
		console.log("download file exc.")
	});
};
```
#### 注意：
`BOM = "\uFEFF"`一定要加上这句，不然下载的文件中中文是乱码的

然后在html中的下载按钮中调用下该方法，即可实现立即下载到Windows默认下载路径下

### 2、后端处理
这里使用的是python语言
使用到了webob三方库
demo代码如下：
```py
def get(self, req, body=None):
	"""The post interface body is nothing.

	:param req:
	:param body:
	:return:
	"""
	tmp = dirname(abspath(__file__))
	test_file = tmp + "/ts.xlsx"
	file_size = os.path.getsize(test_file)

	response = webob.Response(app_iter=self._file_iterator(test_file))
	response.headers['Content-Type'] = 'application/octet-stream'
	response.headers["Content-Disposition"] = \
		'attachment;filename="{}"'.format("ts.xlsx")
	response.headers['Content-length'] = str(file_size)

	return response

@staticmethod
def _file_iterator(file_path, chunk_size=512):
	with open(file_path, 'rb') as f:
		while True:
			chunk = f.read(chunk_size)
			if chunk:
				yield chunk
			else:
				break
```

### 总结
1、主要用到了web的blob技术
2、采用了content_type为二进制流传输的方式