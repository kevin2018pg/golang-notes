## 1 Protobuf  Introduction

protobuf 即 Protocol Buffers，是一种轻便高效的结构化数据存储格式，与语言、平台无关，可扩展可序列化。protobuf 性能和效率大幅度优于 JSON、XML 等其他的结构化数据格式。protobuf 是以二进制方式存储的，占用空间小，但也带来了可读性差的缺点。protobuf 主要用于RPC系统和持续数据存储系统。

Protobuf 在 `.proto` 定义需要处理的结构化数据，可以通过 `protoc` 工具，将 `.proto` 文件转换为 C、C++、Golang、Java、Python 等多种语言的代码，兼容性好，易于使用。

## 2 go中使用protobuf

### 2.1 protoc

从 [Protobuf Releases](https://github.com/protocolbuffers/protobuf/releases) 下载对应系统对应版本的安装包。

```bash
# 下载安装包
$ wget https://github.com/protocolbuffers/protobuf/releases/download/v3.11.2/protoc-3.11.2-linux-x86_64.zip
# 解压到 /usr/local 目录下
$ sudo 7z x protoc-3.11.2-linux-x86_64.zip -o/usr/local
```

把解压路径下的 bin 目录 加入到环境变量。

安装成功。

```bash
$ protoc --version
libprotoc 3.11.2
```

### 2.2 protoc-gen-go

在 Golang 中使用 protobuf，有对应的实现 Protobuf 协议的库 [Github地址](https://github.com/golang/protobuf)，需要安装 protoc-gen-go，这个工具用来将 `.proto` 文件转换为 Golang 代码。

```bash
go get -u github.com/golang/protobuf/protoc-gen-go
```

protoc-gen-go 将自动安装到 `$GOPATH/bin` 目录下，也需要将这个目录加入到环境变量中。

## 3 定义协议

创建一个示例，`feature.proto`

```protobuf
syntax = "proto3";
package main;

// this is a feature comment
message Feature {
  string name = 1;
  bool male = 2;
  repeated int32 scores = 3;
}
```

在当前目录下执行：

```bash
$ protoc --go_out=. *.proto
$ ls
feature.pb.go  feature.proto
```

即将该目录下的所有的 .proto 文件转换为 Go 代码，该目录下多出了一个 Go 文件 *feature.pb.go*。这个文件内部定义了一个结构体 Feature，以及相关的方法：

```go
type Feature struct {
	Name string `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
	Male bool `protobuf:"varint,2,opt,name=male,proto3" json:"male,omitempty"`
	Scores []int32 `protobuf:"varint,3,rep,packed,name=scores,proto3" json:"scores,omitempty"`
	...
}
```

`feature.proto`

- protobuf 有2个版本，默认版本是 proto2，如果需要 proto3，则需要在非空非注释第一行使用 `syntax = "proto3"` 标明版本。
- 消息类型 使用 `message` 关键字定义，Feature 是类型名，name, male, scores 是该类型的 3 个字段，类型分别为 string, bool 和 []int32。字段可以是标量类型，也可以是合成类型。
- 每个字段的修饰符默认是 singular，一般省略不写，`repeated` 表示字段可重复，即用来表示 Go 语言中的数组类型。
- 每个字符 `=`后面的数字称为标识符，每个字段都需要提供一个唯一的标识符。标识符用来在消息的二进制格式中识别各个字段，一旦使用就不能够再改变，标识符的取值范围为 [1, 2^29 - 1] 。
- .proto 文件可以写注释，单行注释 `//`，多行注释 `/* ... */`
- 一个 .proto 文件中可以写多个消息类型，即对应多个结构体(struct)。

以下是一个非常简单的例子，证明被序列化的和反序列化后的实例，包含相同的数据。

```go
package main

import (
	"log"
	"github.com/golang/protobuf/proto"
)

func main() {
	test := &Feature{
		Name: "zhangsan",
		Male:  true,
		Scores: []int32{66, 77, 87},
	}
	data, err := proto.Marshal(test)
	if err != nil {
		log.Fatal("marshaling error: ", err)
	}
	newTest := &Feature{}
	err = proto.Unmarshal(data, newTest)
	if err != nil {
		log.Fatal("unmarshaling error: ", err)
	}
	// Now test and newTest contain the same data.
	if test.GetName() != newTest.GetName() {
		log.Fatalf("data mismatch %q != %q", test.GetName(), newTest.GetName())
	}
}
```

- 保留字段(Reserved Field)

更新消息类型时，可能会将某些字段/标识符删除。这些被删掉的字段/标识符可能被重新使用，如果加载老版本的数据时，可能会造成数据冲突，在升级时，可以将这些字段/标识符保留(reserved)，这样就不会被重新使用了，protoc 会检查。

```
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

## 4 字段语法

### 4.1 字段规则

 字段规则有三种：

- 1、required：该规则规定，消息体中该字段的值是必须要设置的。
- 2、optional：消息体中该规则的字段的值可以存在，也可以为空，optional的字段可以根据defalut设置默认值。
- repeated：消息体中该规则字段可以存在多个（包括0个），该规则对应go语言的slice。

### 4.2 标量类型

| proto类型 | go类型  | 备注                          | proto类型 | go类型  | 备注                       |
| :-------- | :------ | :---------------------------- | :-------- | :------ | :------------------------- |
| double    | float64 |                               | float     | float32 |                            |
| int32     | int32   |                               | int64     | int64   |                            |
| uint32    | uint32  |                               | uint64    | uint64  |                            |
| sint32    | int32   | 适合负数                      | sint64    | int64   | 适合负数                   |
| fixed32   | uint32  | 固长编码，适合大于2^28的值    | fixed64   | uint64  | 固长编码，适合大于2^56的值 |
| sfixed32  | int32   | 固长编码                      | sfixed64  | int64   | 固长编码                   |
| bool      | bool    |                               | string    | string  | UTF8 编码，长度不超过 2^32 |
| bytes     | []byte  | 任意字节序列，长度不超过 2^32 |           |         |                            |

标量类型如果没有被赋值，则不会被序列化，解析时，会赋予默认值。

- strings：空字符串
- bytes：空序列
- bools：false
- 数值类型：0

### 4.3 使用其他消息类型

`Result`是另一个消息类型，在 SearchReponse 作为一个消息字段类型使用。

```
message SearchResponse {
  repeated Result results = 1; 
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

嵌套写也是支持的：

```
syntax = "proto3";
package example;
message Person {
    required string Name = 1;
    required int32 Age = 2;
    required string From = 3;
    optional Address Addr = 4;
    message Address {
        required sint32 id = 1;
        required string name = 2;
        optional string pinyin = 3;
        required string address = 4;
    }
}
```

如果定义在其他文件中，可以导入其他消息类型来使用：

```
import "myproject/other_protos.proto";
```

### 4.4 任意类型(Any)

Any 可以表示不在 .proto 中定义任意的内置类型。

```
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

### 4.5 字段默认值

```protobuf
message Address {
        required sint32 id = 1 [default = 1];
        required string name = 2 [default = '北京'];
        optional string pinyin = 3 [default = 'beijing'];
        required string address = 4;
        required bool flag = 5 [default = true];
    }
```

### 4.6 map

```
message MapRequest {
  map<string, int32> points = 1;
}
```

## 5 定义服务(Services)

如果消息类型是用来远程通信的(Remote Procedure Call, RPC)，可以在 .proto 文件中定义 RPC 服务接口。例如我们定义了一个名为 SearchService 的 RPC 服务，提供了 `Search` 接口，入参是 `SearchRequest` 类型，返回类型是 `SearchResponse`

```
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```

官方仓库也提供了一个[插件列表](https://github.com/protocolbuffers/protobuf/blob/master/docs/third_party.md)，帮助开发基于 Protocol Buffer 的 RPC 服务。

## 6 protoc 其他参数

命令行使用方法

```
protoc --proto_path=IMPORT_PATH --<lang>_out=DST_DIR path/to/file.proto
```

- `--proto_path=IMPORT_PATH`：可以在 .proto 文件中 import 其他的 .proto 文件，proto_path 即用来指定其他 .proto 文件的查找目录。如果没有引入其他的 .proto 文件，该参数可以省略。
- `--<lang>_out=DST_DIR`：指定生成代码的目标文件夹，例如 –go_out=. 即生成 GO 代码在当前文件夹，另外支持 cpp/java/python/ 等语言

## 7 补充

- 文件(Files)
  - 文件名使用小写下划线的命名风格，例如 lower_snake_case.proto
  - 每行不超过 80 字符
  - 使用 2 个空格缩进
- 包(Packages)
  - 包名应该和目录结构对应，例如文件在`my/package/`目录下，包名应为 `my.package`
- 消息和字段(Messages & Fields)
  - 消息名使用首字母大写驼峰风格(CamelCase)，例如`message FeatureRequest { ... }`
  - 字段名使用小写下划线的风格，例如 `string status_code = 1`
- 服务(Services)
  - RPC 服务名和方法名，均使用首字母大写驼峰风格，例如`service FooService{ rpc GetSomething() }`
- 更新规则(Update)
  - 不能修改已存在域中的标识号。
  - 所有新增添的域必须是 optional 或者 repeated。
  - 非 required 域可以被删除。但是这些被删除域的标识号不可以再次被使用。
  - 非required域可以被转化，转化时可能发生扩展或者截断，此时标识号和名称都是不变的。
  - optional兼容repeated。发送端发送repeated域，用户使用optional域读取，将会读取repeated域的最后一个元素。