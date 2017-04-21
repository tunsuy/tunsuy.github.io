---
title: 为Protobuf编译器protoc编写插件
date: 2017-02-20 09:31:47
tags: [python,json,protobuf]
categories: python
keywords: [python,json,protobuf,插件,protoc,Protobuf编译器]
description:
---

Google的 `Protocol Buffer` 是一种以二进制格式对消息进行编码和解码的库，可针对不同平台之间的紧凑性和可移植性进行优化。目前，核心库可以为 `C/C++`，`Java` 和 `Python` 生成代码，但通过编写Protobuf编译器的插件可以实现自动生成其他语言代码。

已经有一个支持第三方语言的插件列表，但是你可以更加自己的需要来编写插件。在这篇文章中，将实践举例如何编写一个Protobuf编译器的插件。

# 一、 配置
在开始编写插件之前，我们需要安装 `Protocol Buffer` 编译器：
```sh
yum install protobuf
```

为了能够编译 `.proto` 文档，我们还需要安装相关语言的protobuf包, 这里我们用Python编写插件：
```sh
pip install protobuf
```

<!-- more -->

# 二、 定义proto
先定义一个proto示例文件，以便后面演示
```proto
enum Greeting {
    NONE = 0;
    MR = 1;
    MRS = 2;
    MISS = 3;
}

message Hello {
    required Greeting greeting = 1;
    required string name = 2;
}
```

# 三、 编写插件
编译器 `protoc ` 的接口非常简单：编译器将在 `stdin` 上传递一个 `CodeGeneratorRequest` 消息，你的插件将在 `stdout` 的 `CodeGeneratorResponse` 中输出生成的代码。所以第一步是编写读取请求的代码并写一个空的响应：
```python
#!/usr/bin/env python

import sys

from google.protobuf.compiler import plugin_pb2 as plugin


def generate_code(request, response):
    pass


if __name__ == '__main__':
    # Read request message from stdin
    data = sys.stdin.read()

    # Parse request
    request = plugin.CodeGeneratorRequest()
    request.ParseFromString(data)

    # Create response
    response = plugin.CodeGeneratorResponse()

    # Generate code
    generate_code(request, response)

    # Serialise response message
    output = response.SerializeToString()

    # Write to stdout
    sys.stdout.write(output)
```

编译器protoc遵循关于插件名称的命名约定，你可以在 `PATH` 中将代码保存在名为 `protoc-gen-custom` 的文件中，或使用你喜欢的任何名称保存（如 `my-plugin.py`），并将插件的名称和路径传递给 `--plugin` 命令行选项。

我们选择第二个选项，所以我们将把插件保存为 `my-plugin.py`，然后编译器的调用将如下所示（假设构建目录已经存在）：
```js
protoc --plugin=protoc-gen-custom=my-plugin.py --custom_out=./build hello.proto
```
上面的命令不会产生任何输出，因为我们的插件什么都不做，现在要写一些有意义的输出。

让我们修改 `generate_code()` 函数来生成 `.proto` 文件的JSON表示。在这篇文章中，根据之前我们定义的proto文件内容，首先我们需要一个遍历AST并返回所有枚举，消息和嵌套类型的函数：
```python
def traverse(proto_file):

    def _traverse(package, items):
        for item in items:
            yield item, package

            if isinstance(item, DescriptorProto):
                for enum in item.enum_type:
                    yield enum, package

                for nested in item.nested_type:
                    nested_package = package + item.name

                    for nested_item in _traverse(nested, nested_package):
                        yield nested_item, nested_package

    return itertools.chain(
        _traverse(proto_file.package, proto_file.enum_type),
        _traverse(proto_file.package, proto_file.message_type),
    )
```

现在 `generate_code()` 代码如下所示：
```python
import itertools
import json

from google.protobuf.descriptor_pb2 import DescriptorProto, EnumDescriptorProto


def generate_code(request, response):
    for proto_file in request.proto_file:
        output = []

        # Parse request
        for item, package in traverse(proto_file):
            data = {
                'package': proto_file.package or '&lt;root&gt;',
                'filename': proto_file.name,
                'name': item.name,
            }

            if isinstance(item, DescriptorProto):
                data.update({
                    'type': 'Message',
                    'properties': [{'name': f.name, 'type': int(f.type)}
                                   for f in item.field]
                })

            elif isinstance(item, EnumDescriptorProto):
                data.update({
                    'type': 'Enum',
                    'values': [{'name': v.name, 'value': v.number}
                               for v in item.value]
                })

            output.append(data)

        # Fill response
        f = response.file.add()
        f.name = proto_file.name + '.json'
        f.content = json.dumps(output, indent=2)
```
对于 `protoc` 请求中的每个 `.proto` 文件，我们遍历所有项目（枚举，消息和嵌套类型），并在字典中写入一些信息。然后，我们向 `protoc` 的响应中添加一个新文件，我们设置文件名，在该例中等于原始文件名加上 `.json` 扩展名，以及字典的JSON表示形式的内容。

如果再次运行protobuf编译器，它将在build目录中输出一个名为 `hello.proto.json` 的文件，内容如下：
```json
[
  {
    "type": "Enum",
    "filename": "hello.proto",
    "values": [
      {
        "name": "NONE",
        "value": 0
      },
      {
        "name": "MR",
        "value": 1
      },
      {
        "name": "MRS",
        "value": 2
      },
      {
        "name": "MISS",
        "value": 3
      }
    ],
    "name": "Greeting",
    "package": "&lt;root&gt;"
  },
  {
    "properties": [
      {
        "type": 14,
        "name": "greeting"
      },
      {
        "type": 9,
        "name": "name"
      }
    ],
    "filename": "hello.proto",
    "type": "Message",
    "name": "Hello",
    "package": "&lt;root&gt;"
  }
]
```

# 四、 总结
在这篇文章中，我们通过创建一个 `Protocol Buffer` 插件来将 `.proto` 文件编译成JSON格式的简化表示。核心部分是从 `stdin` 读取protoc请求的接口代码，遍历AST并在stdout上写入响应。该例中我们只处理了 `.proto` 中所有的枚举，消息和嵌套类型，如果 `.proto` 文档有其他类型，还需要加相关的代码解析处理。

根据这样的思路，及插件编写过程，你可以定制任何你想要的输出形式。

