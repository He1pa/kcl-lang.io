---
sidebar_position: 1
---

# 简介

KCL 语言提供 C/Rust/Go/Python/Java 等通用编程语言接口，相关语言正在开发完整中。

## C/Rust 语言

KCL 核心采用 Rust 语言开发，对外导出 C 语言 API 供 Go/Python/Java 等高级语言包装和集成。

## Go 语言

Go 语言是通过 CGO 包装 KCL 提供的 C-API，同时提供更深度的定制特性以满足上层工具的需求。

### API 抽象模型

Go 语言 API 的抽象模型如下图：

```
┌─────────────────┐         ┌─────────────────┐           ┌─────────────────┐
│     kcl files   │         │    KCL-Go-API   │           │  KCLResultList  │
│  ┌───────────┐  │         │                 │           │                 │
│  │    1.k    │  │         │                 │           │                 │
│  └───────────┘  │         │                 │           │  ┌───────────┐  │         ┌───────────────┐
│  ┌───────────┐  │         │  ┌───────────┐  │           │  │ KCLResult │──┼────────▶│x.Get("a.b.c") │
│  │    2.k    │  │         │  │ Run(path) │  │           │  └───────────┘  │         └───────────────┘
│  └───────────┘  │────┐    │  └───────────┘  │           │                 │
│  ┌───────────┐  │    │    │                 │           │  ┌───────────┐  │         ┌───────────────┐
│  │    3.k    │  │    │    │                 │           │  │ KCLResult │──┼────────▶│x.Get("k", &v) │
│  └───────────┘  │    │    │                 │           │  └───────────┘  │         └───────────────┘
│  ┌───────────┐  │    ├───▶│  ┌───────────┐  │──────────▶│                 │
│  │setting.yml│  │    │    │  │RunFiles() │  │           │  ┌───────────┐  │         ┌───────────────┐
│  └───────────┘  │    │    │  └───────────┘  │           │  │ KCLResult │──┼────────▶│x.JSONString() │
└─────────────────┘    │    │                 │           │  └───────────┘  │         └───────────────┘
                       │    │                 │           │                 │
┌─────────────────┐    │    │                 │           │  ┌───────────┐  │         ┌───────────────┐
│     Options     │    │    │  ┌───────────┐  │           │  │ KCLResult │──┼────────▶│x.YAMLString() │
│WithOptions      │    │    │  │MustRun()  │  │           │  └───────────┘  │         └───────────────┘
│WithOverrides    │────┘    │  └───────────┘  │           │                 │
│WithWorkDir      │         │                 │           │                 │
│WithDisableNone  │         │                 │           │                 │
└─────────────────┘         └─────────────────┘           └─────────────────┘
```

其中输入的文件包含 KCL 文件和 `setting.yml` 配置文件，`Options` 可以用于指定额外的参数和工作目录等信息。“KCL-Go-API”部分是提供的 KCL 执行函数，执行函数根据输入文件和额外的参数执行 KCL 程序，最终输出 `KCLResultList` 结果。`KCLResultList` 是一个 `KCLResult` 构成的列表，每个 `KCLResult` 对应一个生成的配置文件或 `map[string]interface{}`。

### 例子

```go
package main

import (
	"fmt"

	"kusionstack.io/kclvm-go/api/kcl"
)


func main() {
	const k_code = `
import kcl_plugin.hello

name = "kcl"
age = 1

two = hello.add(1, 1)

schema Person:
    name: str = "kcl"
    age: int = 1

x0 = Person{}
x1 = Person{age:101}
`

	result := kcl.MustRun("hello.k", kcl.WithCode(k_code)).First()
	fmt.Println(result.YAMLString())

	fmt.Println("----")
	fmt.Println("x0.name:", result.Get("x0.name"))
	fmt.Println("x1.age:", result.Get("x1.age"))

	fmt.Println("----")

	var person struct {
		Name string
		Age  int
	}
	fmt.Printf("person: %+v\n", result.Get("x1", &person))
}
```

输出结果:

```yaml
age: 1
name: kcl
two: 2
x0:
    age: 1
    name: kcl
x1:
    age: 101
    name: kcl

----
x0.name: kcl
x1.age: 101
----
person: &{Name:kcl Age:101}
```

## REST-API

KCL 提供的 C-API 并没有 REST-API，REST-API 是通过 Protobuf 定义，最终由上层的 Go-SDK 提供实现。

### 启动 REST 服务

通过以下方式可以启动 RestAPI 服务：

```shell
python3 -m pip install kclvm -U
python3 -m gunicorn "kclvm.program.rpc-server.__main__:create_app()" -t 120 -w 4 -k uvicorn.workers.UvicornWorker -b :2021
```

然后可以通过 POST 协议请求服务：

```shell
$ curl -X POST http://127.0.0.1:2021/api:protorpc/BuiltinService.Ping --data '{}'
{
	"error": "",
	"result": {}
}
```

其中 POST 请求和返回的 JSON 数据和 Protobuf 定义的结构保持一致。

### `BuiltinService` 服务

其中 `/api:protorpc/BuiltinService.Ping` 路径表示 `BuiltinService` 服务的 `Ping` 方法。

完整的 `BuiltinService` 由 Protobuf 定义：

```protobuf
service BuiltinService {
	rpc Ping(Ping_Args) returns(Ping_Result);
	rpc ListMethod(ListMethod_Args) returns(ListMethod_Result);
}

message Ping_Args {
	string value = 1;
}
message Ping_Result {
	string value = 1;
}

message ListMethod_Args {
	// empty
}
message ListMethod_Result {
	repeated string method_name_list = 1;
}
```

其中 `Ping` 方法可以验证服务是否正常，`ListMethod` 方法可以查询提供的全部服务和函数列表。

### `KclvmService` 服务

`KclvmService` 服务是和 KCL 功能相关的服务。用法和 `BuiltinService` 服务一样。

比如有以下的 `Person` 结构定义：

```python
schema Person:
    key: str

    check:
        "value" in key  # 'key' is required and 'key' must contain "value"
```

然后希望通过 `Person` 来校验以下的 JSON 数据：

```json
{ "key": "value" }
```

可以通过 `KclvmService` 服务的 `ValidateCode` 方法完成。参考 `ValidateCode` 方法的 `ValidateCode_Args` 参数结构：

```protobuf
message ValidateCode_Args {
	string data = 1;
	string code = 2;
	string schema = 3;
	string attribute_name = 4;
	string format = 5;
}
```

根据 `ValidateCode_Args` 参数结构构造 POST 请求需要的 JSON 数据，其中包含 `Person` 定义和要校验的 JSON 数据：

```json
{
  "code": "\nschema Person:\n    key: str\n\n    check:\n        \"value\" in key  # 'key' is required and 'key' must contain \"value\"\n",
  "data": "{\"key\": \"value\"}"
}
```

将该 JSON 数据保存到 `vet-hello.json` 文件，然后通过以下命令进行校验：

```shell
$ curl -X POST \
    http://127.0.0.1:2021/api:protorpc/KclvmService.ValidateCode \
    -H  "accept: application/json" \
    --data @./vet-hello.json
{
    "error": "",
    "result": {
        "success": true
    }
}
```

说明校验成功。

## Python 语言

使用 Python SDK 要求您本地具备高于 3.7.3 的 Python 版本和 pip 包管理工具，可以通过如下命令进行安装并获得帮助信息

```bash
python3 -m pip install kclvm --user && python3 -m kclvm --help
```

### 命令行工具

编写名为 `main.k` 的 KCL 文件:

```python
name = "kcl"
age = 1

schema Person:
    name: str = "kcl"
    age: int = 1

x0 = Person {}
x1 = Person {
    age = 101
}
```

执行如下命令并获得输出:

```shell
$ python3 -m kclvm hello.k
name: kcl
age: 1
x0:
  name: kcl
  age: 1
x1:
  name: kcl
  age: 101
```

### API

此外，我们还可以通过 Python 代码实现对 KCL 文件的执行

编写名为 `main.py` 的 python 文件:

```python
import kclvm.program.exec as kclvm_exec
import kclvm.vm.planner as planner

print(planner.plan(kclvm_exec.Run(["hello.k"]).filter_by_path_selector()))
```

执行如下命令并获得输出:

```shell
$ python3 main.py
name: kcl
age: 1
x0:
  name: kcl
  age: 1
x1:
  name: kcl
  age: 101
```

可以看出通过命令行工具和 API 可以获得同样的输出。

目前 KCL Python SDK 还处于早期预览版本，后续 KCL 团队会持续更新并提供更丰富的功能，更多信息请参阅：[https://github.com/kcl-lang/kcl-py](https://github.com/kcl-lang/kcl-py)

## 其它语言

用户也可以基于 C-API 和 RestAPI 包装其它语言的 SDK。
