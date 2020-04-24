curl操作

请求参数
1、-H：请求header参数，每个参数对应一个`-H`
比如 -H "Content-Type:application/json" -H "X-Auth-Token:'xxx'"

2、-X：请求method
PUT、GET、DELETE、POST等等

3、-d：请求body，json字符串
比如：-d '{"address": "1.1.1.1", "port": 1111}'

4、-k/--insecure：不校验证书

回显参数
1、-i：显示response的header信息
2、-v：显示一次http通信的整个过程，包括端口连接和http request头信息
3、-trace [output_file]：比-v更加详细

curl -i -k -H "Content-Type:application/json" -H "X-Auth-Token:'xxx'" -X PUT -d '{"address": "10.200.8.28", "port": 8799}' https://10.200.8.57:31943/silvan/rest/v1.0/endpoints/csbs/regions/hn-smx-1



推荐一个gui工具：postman