在测试之前，我们先说下http的相关状态

### HTTP状态
* CLOSE_WAIT：haproxy向backend发起的连接，backend发起了FIN，haproxy没有回应FIN
* FIN_WAIT2： client向haproxy发起的连接，haproxy发起了FIN，没有收到client发送的FIN

CLOSE_WAIT和FIN_WAIT2的连接基本是一一对应的关系，可以推测：
* backend要终止连接，向haproxy发起了FIN
* haproxy与backend的连接进入CLOSE_WAIT状态，需要等待与client的连接断开后，才发送FIN关闭连接
* haproxy向client发起FIN，并得到了client的回应，进入FIN_WAIT2状态
* client迟迟不发送FIN给haproxy，导致haproxy与client的连接一直出于FIN_WAIT2状态，haproxy与backend的连接也始终为CLOSE_WAIT状态。

可以看下http的四次挥手的过程，就能够更加的明白这两个状态。网上很多这样的教程，自行google。

### 性能优化
下面是一个初步试验

#### 测试代码
```python
import json
import sys

from eventlet import GreenPool
import requests
import threading


def test_api_green_thread(concurrent_num):
    gp_pool = GreenPool(concurrent_num)
    threads = []

    for i in range(concurrent_num):
        threads.append(gp_pool.spawn(api_request))
    gp_pool.waitall()
    results = [result.wait() for result in threads if result is not None]

    print("test finished")

    success_times = 0
    failed_times = 0
    for result in results:
        if result is True:
            success_times += 1
        elif result is False:
            failed_times += 1

    print("success times: %s, failed times: %s" % (success_times,
                                                   failed_times))


def test_api_thread(concurrent_num):
    threads = []
    results = []

    for i in range(concurrent_num):
        th = threading.Thread(target=api_request, args=(results,))
        th.start()
        threads.append(th)

    for i in threads:
        i.join()

    print("test finished")

    success_times = 0
    failed_times = 0
    for result in results:
        if result is True:
            success_times += 1
        elif result is False:
            failed_times += 1

    print("success times: %s, failed times: %s" % (success_times,
                                                   failed_times))


def api_request(results=list()):
    url = "https://1.1.1.1:8799/v1.5/xxx/api"
    headers = {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        "X-Auth-Token": "xxx"
    }
    kwargs = {
        "headers": headers,
        "verify": False,
        "timeout": 60
    }

    try:
        response = requests.request('GET', url, **kwargs)
        if response.status_code == 200:
            rsp_body = json.loads(response.content)
            print("rsp content: %s" % rsp_body)
            results.append(True)
            return True
        print("api request failed, code: %s, msg: %s" %
              (response.status_code, response.content))
        results.append(False)
        return False
    except Exception as err:
        print("api request occur exc, msg: %s" % err.message)
        results.append(False)
        return False


if __name__ == "__main__":
    args = sys.argv
    if not args or len(args) != 2:
        print("Usage: python xx.py [concurrent_num]")
        exit(1)

    # test_api_green_thread(int(args[1]))
    test_api_thread(int(args[1]))

```

#### 测试结果
```sh
2019-04-11 22:22:39.657 localhost haproxy [pid:28229]info  25.10.0.104:38370 [11/Apr/2019:22:21:59.602] test_8799~ b_def_test_8799/controller1 44/30005/-1/-1/40053 503 212 - - sC-- 231/166/166/80/+3 0/0 "GET /v1.5/xxx/api HTTP/1.1"
2019-04-11 22:22:39.679 localhost haproxy [pid:28229]info  25.10.0.104:38372 [11/Apr/2019:22:21:59.602] test_8799~ b_def_test_8799/controller1 67/30007/-1/-1/40075 503 212 - - sC-- 230/165/165/78/+3 0/0 "GET /v1.5/xxx/api HTTP/1.1"
2019-04-11 22:22:39.679 localhost haproxy [pid:28229]info  25.10.0.104:38374 [11/Apr/2019:22:21:59.626] test_8799~ b_def_test_8799/controller1 43/30007/-1/-1/40051 503 212 - - sC-- 229/164/164/78/+3 0/0 "GET /v1.5/xxx/api HTTP/1.1"
2019-04-11 22:22:39.701 localhost haproxy [pid:28229]info  25.10.0.104:38378 [11/Apr/2019:22:21:59.626] test_8799~ b_def_test_8799/controller1 57/30005/-1/-1/40073 503 212 - - sC-- 228/163/163/77/+3 0/0 "GET /v1.5/xxx/api HTTP/1.1"
2019-04-11 22:22:39.704 localhost haproxy [pid:28229]info  25.10.0.104:38382 [11/Apr/2019:22:21:59.646] test_8799~ b_def_test_8799/controller1 48/30007/-1/-1/40057 503 212 - - sC-- 227/162/162/76/+3 0/0 "GET /v1.5/xxx/api HTTP/1.1"
```
其中sC表示haproxy的session状态flag 
```sh
The "timeout connect" stroke before a connection to the server could
          complete. When this happens in HTTP mode, the status code is likely a
          503 or 504 here.
```

完整的flag官方定义如下：
https://cbonte.github.io/haproxy-dconv/2.0/configuration.html#9.1

#### 参数调整
根据系统配置调整相关的参数。 
* timeout参数调整参考：https://delta.blue/blog/haproxy-timeouts
* 系统内核参数调整参考：https://fangpeishi.com/haproxy_best_practice_notes.html
