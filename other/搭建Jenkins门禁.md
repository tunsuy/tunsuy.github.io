# 搭建Jenkins门禁

## 一、Flake8
### 1、生成执行结果
使用python工具flake8，通过pip install flake8安装即可
```sh
flake8 --ignore=E121,E123,E126,E133,W503,W504,W505,H306,H405,H237,N802,N803,H301,H403,N805,E261,H404,H702 --output-file flake8.txt [path]
```
其中path表示需要扫描的目录
--ignore表示忽略哪些规则

### 2、转换jenkins报告
```sh
if [ $? -eq 0 ];then
  echo "<?xml version='1.0' encoding='utf-8'?>" > flake8_junit.xml
  echo '<testsuite errors="0" failures="0" name="flake8" tests="999" time="1"><testcase></testcase></testsuite>' >> flake8_junit.xml
  exit 0
else
  python -m junit_conversor flake8.txt flake8_junit.xml
  exit 1
fi
```
这里用到了另一个工具：junit_conversor
也是通过pip安装即可

### 3、查看jenkins报告
通过在jenkins的配置中，添加Publish Junit test result report，
然后将flake8_junit.xml添加到其路径中即可

### 4、门禁
通过解析扫描结果问题，获取问题数，然后跟期望门禁对比即可
```sh
count=`cat xx/flake8.txt | wc -l`
echo 本次MR的flake8问题数共有 $count 个

expect_count=0
if [[ "${Target_Branch}" = "xxx" ]]; then
	echo "Target branch is private_py3_branch"
	expect_count=xxx
fi


if [ "$count" -gt ${expect_count} ]; then
    echo "门禁值${expect_count}，实际值${count}，门禁失败"
	exit -1
else
    echo "门禁值${expect_count}，实际值${count}，门禁成功"
	exit 0
fi
```

## 二、pylint
### 1、生成执行结果
这里使用到了python的pylint工具，通过pip install pylint安装即可
```sh
find $SCAN_PATHS -iname "*.py" | xargs pylint --output-format=parseable --rcfile=/opt/pylint_rule.conf > $WORKSPACE/pylint_report/pylint.log | exit 0
```

### 2、查看jenkins报告
通过在jenkins的配置中，添加Publish pylint report，
pylint.log添加到其路径中即可

### 3、门禁
通过解析扫描结果问题，获取问题数，然后跟期望门禁对比即可
```sh
count=`cat $WORKSPACE/pylint_report/pylint.log | egrep -v "\*\*\*|---|Your code" | wc -l`
count=$((count-2))
echo 本次MR的pylint问题数共有 ${count} 个

expect_count=0
if [[ "${Target_Branch}" = "xxxxx" ]]; then
	echo "Target branch is private_py3_branch"
	expect_count=xxx
fi

if [ "$count" -gt ${expect_count} ]; then
    echo "门禁值${expect_count}，实际值${count}，门禁失败"
	exit -1
else
    echo "门禁值${expect_count}，实际值${count}，门禁成功"
	exit 0
fi

```

## 三、Radon
### 1、生成执行结果
这里使用到了python的radon工具，通过pip install radon安装即可
```sh
/usr1/tools/python2.7.9/bin/radon cc $WORKSPACE/ -a > radon.txt
/usr1/tools/python2.7.9/bin/radon cc --xml $WORKSPACE/ > radon.xml
```

### 2、查看jenkins报告
这里用到了jenkins的CCM插件，通过在jenkins的配置中，添加Publish CCM analysis results
radon.xml添加到其路径中即可

### 3、门禁
#### 3.1 平均圈复杂度
通过解析扫描结果问题，获取问题数，然后跟期望门禁对比即可
```python
# -*- coding: utf-8 -*-
import os
import re
import sys
import shutil
result = ''
with open('radon.txt')as file_object:
    for line in file_object:
        if 'Average' in line:
            result = re.findall(r"\d+\.?\d*", line)
average = float(result[0])
print '======================================================================='
print '                 Average complexity is: ',average
print '======================================================================='
if average >= 5:
    print '=================radon gate failed===================='
    exit (1)
else:
    print 'Everything is OK!'

```

#### 3.2 超大圈复杂度
通过解析扫描结果问题，获取问题数，然后跟期望门禁对比即可
```sh
count=`awk -F " " '{print $NF}' radon.txt | egrep "^C|^D|^E|^F" |wc -w`
echo 本次MR圈复杂度超过10的共有 $count 个方法

expect_count=0
if [[ "${Target_Branch}" = "xxx" ]]; then
	echo "Target branch is xxx"
	expect_count=xx
fi


if [ "$count" -gt ${expect_count} ]; then
    echo "门禁值${expect_count}，实际值${count}，门禁失败"
	exit -1
else
    echo "门禁值${expect_count}，实际值${count}，门禁成功"
	exit 0
fi
```

## 四、LLT
### 1、生成执行结果
这里使用到了python的nosetests和coverage工具，通过pip install安装即可
```sh
cd xxx/
python setup.py egg_info >> /dev/null 2>&1
nosetests -q --processes=-1 --process-timeout=600 --process-restartworker --with-cov --cov-report xml --with-xunitmp --exe path
#coverage combine --timid -a --rcfile=./.coveragerc
coverage report --ignore-error --omit=xx,xx
```
其中path表示需要扫描的目录
--omit表不需要计算覆盖率的目录
上面将会自动产生nosetests.xml, converage.xml报告


### 2、查看jenkins报告
#### 2.1 查看LLT覆盖率
这里用到了jenkins的Cobertura插件，通过在jenkins的配置中，添加Publish Cobertura converage report
converage.xml添加到其路径中即可
#### 2.2 查看LLT执行结果
通过在jenkins的配置中，添加Publish Junit test result report
nosetests.xml添加到其路径中即可

### 3、门禁
通过第2点中可以设置阈值