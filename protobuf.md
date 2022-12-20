# ProtoBuf 语法

ProtoBuf 语法主要分为三部分，分别是 proto定义、message、service 三种



## proto定义

### syntax

```
syntax = "proto3";
```

定义proto文件的版本 



### import

```
import "google/protobuf/any.proto";
import "google/protobuf/type.proto";
```

用于引入外部的proto文件



### optional

```protobuf
//将每个message生成一个java文件
option java_multiple_files = true;

//定义java包的名称
option java_package = "org.apache.skywalking.apm.network.language.profile.v3";

//定义java输出文件的名称
option java_outer_class = "xxx"
option csharp_namespace = "SkyWalking.NetworkProtocol.V3";
option go_package = "skywalking.apache.org/repo/goapi/collect/language/profile/v3";
```





## Message

Message 主要包含了各种类型定义以及基础的语法

### 基础类型

| .proto Type | Notes                                                        | C++ Type | Java Type  | Python Type[2] | Go Type | Ruby Type                      | C# Type    | PHP Type       |
| ----------- | ------------------------------------------------------------ | -------- | ---------- | -------------- | ------- | ------------------------------ | ---------- | -------------- |
| double      |                                                              | double   | double     | float          | float64 | Float                          | double     | float          |
| float       |                                                              | float    | float      | float          | float32 | Float                          | float      | float          |
| int32       | 使用变长编码，对于负值的效率很低，如果你的域有可能有负值，请使用sint64替代 | int32    | int        | int            | int32   | Fixnum 或者 Bignum（根据需要） | int        | integer        |
| uint32      | 使用变长编码                                                 | uint32   | int        | int/long       | uint32  | Fixnum 或者 Bignum（根据需要） | uint       | integer        |
| uint64      | 使用变长编码                                                 | uint64   | long       | int/long       | uint64  | Bignum                         | ulong      | integer/string |
| sint32      | 使用变长编码，这些编码在负值时比int32高效的多                | int32    | int        | int            | int32   | Fixnum 或者 Bignum（根据需要） | int        | integer        |
| sint64      | 使用变长编码，有符号的整型值。编码时比通常的int64高效。      | int64    | long       | int/long       | int64   | Bignum                         | long       | integer/string |
| fixed32     | 总是4个字节，如果数值总是比总是比228大的话，这个类型会比uint32高效。 | uint32   | int        | int            | uint32  | Fixnum 或者 Bignum（根据需要） | uint       | integer        |
| fixed64     | 总是8个字节，如果数值总是比总是比256大的话，这个类型会比uint64高效。 | uint64   | long       | int/long       | uint64  | Bignum                         | ulong      | integer/string |
| sfixed32    | 总是4个字节                                                  | int32    | int        | int            | int32   | Fixnum 或者 Bignum（根据需要） | int        | integer        |
| sfixed64    | 总是8个字节                                                  | int64    | long       | int/long       | int64   | Bignum                         | long       | integer/string |
| bool        |                                                              | bool     | boolean    | bool           | bool    | TrueClass/FalseClass           | bool       | boolean        |
| string      | 一个字符串必须是UTF-8编码或者7-bit ASCII编码的文本。         | string   | String     | str/unicode    | string  | String (UTF-8)                 | string     | string         |
| bytes       | 可能包含任意顺序的字节数据。                                 | string   | ByteString | str            | []byte  | String (ASCII-8BIT)            | ByteString | string         |





### 枚举类型

```
  enum Type {
    RPC = 0;
    JSON=2;
    YAML=3;
  }
```





### 数组类型

```protobuf
 repeated string  tags = 5;
```



### map 类型 

```
map<string,int32> = 1;
```





### 可选类型

可选类型的意思是指 在oneof内部定义的字段只有一个能被设置，就是只能设置其中一个的值

```
oneof file_name {
	string text  = 1;
	string yaml = 2;
	string properties = 3;
}
```



### reserved

reserved限定修饰符：指定保留字段名称（Field Name）和分配标识号（Assigning  Tags），用于将来的扩展。下面是一个简单的reserved限定修饰符使用的例子： 

```protobuf
message MsgFoo{ 
     //… 
     reserved 12, 15, 9 to 11; // 预留将来使用的分配标识号（Assigning Tags）, 
     reserved "foo", "bar"; // 预留将来使用的字段名（field name） 
}
```





## Service

#### 定义RPC访问

```
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```



#### 定义stream请求

``` protobuf
service SearchService {
  rpc Search (stream SearchRequest) returns (SearchResponse);
}
```



定义stream 返回流

```
service SearchService {
  rpc Search (stream SearchRequest) returns (stream SearchResponse);
}
```

