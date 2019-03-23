## python中LLT的mock妙用

### 1、mock的位置
* 将mock放在class上：作用于所有的test用例
* 将mock放在method上：作用于该test用例
* 也可以将mock用在测试用例类，使用with，将只作用于该代码块
```python
with mock.patch.multiple(
		mock_obj1,
		mock_obj2,
		mock_obj3
) as mocks:
	mocks["mock_obj1"].field(method) = xxx
```
* 单个域的mock
```python
class.method = mock.MagicMock()
class.method.return_value = xxx
```

### 2、mock的patch参数
如果是这种@mock.patch("mock_class")形式作用于用例上，则该用例将需要增加接收mock的参数，比如
```python
@mock.patch("mock_class")
def test_method(xx, mock_obj):
	mock_obj.field = xxx
	mock_obj.method = xxx
```

如果是这种@mock.patch("method", mock_method)形式作用于用例上，则用例不需要做任何处理，比如
```python
@mock.patch("method", mock_method)
def test_method(xx):
	todo
```

### 3、检查method调用数
```python
assertEqual(class.method.call_count, intxx)
```

### 4、以固定的参数调用
* 调用过任意次
```python
class.method.assert_any_call(param1, param2, xxx)
```
* 调用过一次
```python
class.method.assert_called_once_with(param1, param2, xxx)
```

### 5、mock同一个方法
根据代码调用顺序需要，针对不同的调用需要mock返回不同的结果，比如
```python
requests.request = mock.MagicMock()
requests.request.side_effect = [
	MockResponse(400, jsonutils.dumps(result_failed)),
	MockResponse(400, jsonutils.dumps(result_failed)),
	MockResponse(400, jsonutils.dumps(result_failed)),
	MockResponse(200, jsonutils.dumps(result_success)),
]
```