## 基于python的selenium实战

### 1、安装selenium
通过python的pip安装

### 2、配置chrome驱动
下载跟浏览器同版本的chromedriver
配置方式有几种：
* 设置PATH环境变量，包含chromedriver.exe的文件夹
* 将chromedriver.exe放在跟脚本同目录下

### 3、模拟登陆大众点评网站

```python
# coding: utf-8
from selenium import webdriver

browser = webdriver.Chrome()
browser.maximize_window()
browser.get('https://account.dianping.com/login?redir=http://www.dianping.com')

# 切换到页面中的iframe
browser.switch_to.frame(0)

browser.find_element_by_class_name('bottom-password-login').click()
browser.find_element_by_id('tab-account').click()

browser.find_element_by_id('account-textbox').send_keys("12345678911")
browser.find_element_by_id('password-textbox').send_keys("12345678911")
browser.find_element_by_id('login-button-account').click()

# 切换回页面主内容
browser.switch_to.default_content()
```