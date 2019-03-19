---
title: protobuf
date: 2016-04-07 19:48:28
tags: protobuf
categories: java tool
---
这篇protobuf指南描述了怎样使用**proto3**构造protocol buffer数据，包括`.proto`文件语法以及怎样从.proto文件生成数据访问类。
<!--more-->
其实就是[proto3指南][protobuf]的翻译啦。有误之处，麻烦指出（英文水平...）。
<!--more-->
### 定义消息类型
#### 使用`.proto`文件定义消息类型
假设想定义一个“搜索请求”的消息格式，每一个请求含有一个查询字符串、查询结果所在的页数，每一页显示查询结果的数量。可以采用如下的方式来定义消息类型的.proto文件了：
```
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```
* `syntax = "proto3";`标明使用`proto3`语法，如果不标明则protocol buffer编译器会假定你使用的是`proto2`。必须是文件中空行、注释之外的第一行。
* 消息`SearchRequest` 包含了三个字段，每个字段都包含`type`、`name`、`unique numbered tag`。
* `tag`用来标识字段，范围为`1 ~ 229 - 1(536,870,911)`，其中`19000 ~ 19999`为protocol Buffers内部预留，不可使用。
其中`1 ~ 15`使用1个byte编码，`16 ~ 2047`使用两个bytes编码。

#### 字段规则
消息的字段可以是以下中的一个
    1. singular：0个或1个
    2. `repeated`：0个或多个，即数组，顺序会被保留。使用`[packed=true]`可以提高编码效率。例如:
    `repeated int32 samples = 4 [packed=true];`

#### 多个消息类型
 一个`.proto`文件可以定义多个消息结构
```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```
#### 添加注释
* 可以使用`//`添加注释
```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

#### 保留字段
当消息类型移除一些字段后，为了防止`name`、`tag`被之后的开发者使用，可以使用`reserved`
```
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```
需要注意，不能在一个`reserved`声明中混合字段的名称和`tag`。

#### `.proto`文件生成了什么
当用protocolbuffer编译器来运行.proto文件时，编译器将生成所选择语言的代码，这些代码可以操作在.proto文件中定义的消息类型，包括获取、设置字段值，将消息序列化到一个输出流中，以及从一个输入流中解析消息。
* 对于java来说，会生成一个`.java`文件，包含每个消息类型，以及创建消息类型的`Builder`类

### 类型
消息中的字段可以使用以下类型，并列出了与之对应的java类型
| .proto Type        | Java Type          |Notes |
| :------------- |:-------------:|:-------------|
|double    | double | 
| float   | float|
| int32 | int      | 使用可变长(variable-length)编码方式。编码负数时效率低。如果字段包含负数，使用sint32。 |
| int64 | long      | 使用可变长(variable-length)编码方式。编码负数时效率低。如果字段包含负数，使用sint64。 |
| uint32| int | 使用可变长(variable-length)编码方式。|
| uint64| long |  使用可变长(variable-length)编码方式。|
| sint32 | int      | 使用可变长(variable-length)编码方式。有符号整形，对负数进行编码比int32高效  |
| sint64 | long      | 使用可变长(variable-length)编码方式。有符号整形，对负数进行编码比int64高效  |
| fixed32 | int      | 总是4个字节，如果数值经常大于228 ，比uint32高效| 
| fixed64| long | 总是8个字节，如果数值经常大于 256 ，比uint64高效| 
| sfixed32| int      | 总是4个字节|
| sfixed64| long| 总是8个字节|
| bool| boolean| 
|string| String| 字符串，必须是utf-8或者7-bit ASCII编码|
| bytes| ByteString| 
In Java, unsigned 32-bit and 64-bit integers are represented using their signed counterparts, with the top bit simply being stored in the sign bit.

### 默认类型
* strings ：空字符串
* bytes： 空bytes
* bools：false
* 数值类型：0
* enums：enum的第一个元素（0）
* 结构体：null

### 枚举
* 通过`enum`可以定义枚举，枚举的第一个元素必须为0。
```
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```
* `option allow_alias = true;`可以为枚举定义别名，即多个字段可以拥有相同的值
```
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;
}
enum EnumNotAllowingAlias {
  UNKNOWN = 0;
  STARTED = 1;
  // RUNNING = 1;  // Uncommenting this line will cause a compile error inside Google and a warning message outside.
}
```
* 枚举的值必须在`32-bit integer`范围内
* 如果一个枚举定义在消息体内部，其他消息通过如下语法使用：
`MessageType.EnumType`

### 使用其他消息类型
可以将消息当做其他消息的一个字段类型来使用，如下：
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
`Result` 与`SearchResponse`必须位于同一个`.proto`文件中。

### 导入定义
* 当`Result` 与`SearchResponse`位于不同的文件时，可以在定义`SearchResponse`消息的`.proto`文件中使用如下语法导入 `Result`的定义：
`import "myproject/other_protos.proto";`
* 有时候需要将一个`.proto`文件转移到其他位置，最好的方法不是直接转移`.proto`文件并更新其他文件中此文件的路径，而是在目标路径下建立新的`.proto`文件，拷贝旧文件内容，并在旧文件中设置`import public`，如下：
```
// new.proto
// All definitions are moved here
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```

### 嵌套消息
* 可以在一个消息的内部定义、使用消息，如下：
```
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```
* 如果想在父消息外的其他消息中使用内部消息，参考`Parant.Type`形式：
```
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```
* 可以嵌套任意层消息
```
message Outer {                  // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      int64 ival = 1;
      bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      int32 ival = 1;
      bool  booly = 2;
    }
  }
}
```

### 更新消息
如果一个已有的消息类型已经不能满足全部需求，比如需要增加一个新的字段，但仍然旧版本代码仍然可用。可以更新消息类型而不破坏已存在的代码。只要注意以下规则：
1. 不能改变已有字段的`tag`
2. 如果增加了新的字段，新的代码依然可以解析由旧代码序列化的消息，此时，新增的字段为默认值。旧的代码也可以解析由新代码序列化的消息，但会忽略新增的字段。
3. 字段可以被移除，但`tag`不能在更新后的消息类型中使用。可以使用`reserved`防止被重复使用。
4. `int32`,`uint32`,`int64`,`uint64`,`bool`是相互兼容的，将这些类型中的一个转换为另外一个，不会破坏向前、向后兼容(forwards- or backwards-compatibility)
5. `sint32`,`sint64`相互兼容，但与其他的整数类型不兼容
6. `string`和`bytes`是兼容的，只要`bytes`是有效的UTF-8编码。
7. 内嵌消息与`bytes`是兼容的，只要bytes包含该消息的一个编码过的版本。
8. `fixed32`与`sfixed32`是兼容的，`fixed64`与`sfixed64`是兼容的。

### Any  Oneof ???

### Oneof
如果一个消息有多个字段，但同一时间只有一个字段会被设置，可以使用`oneof`特性来强制这种行为，同时节省内存。
在一个oneof中的所有字段会共享内存，同一时间至多有一个字段可被设置。设置其中的一个字段，会清除之前设置的字段。可以使用`case()`或`WhichOneof()`方法检测哪一个字段被设置了。
#### 使用`oneof`关键字定义oneof，如下：
```
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```
可以在oneof的定义中增加任意类型的oneof字段，但不能使用`repeated`字段。
在生成的代码中，oneof字段像普通字段一样有`getter`,`setter`方法。还有一个特殊方法，用来检测哪个字段被设置。

#### oneof特性
* 设置一个oneof字段的值将自动清理oneof的其他值。所以，如果设置了几个值，只有最后被设置的字段有值。
```
SampleMessage message;
message.set_name("name");
CHECK(message.has_name());
message.mutable_sub_message();   // Will clear name field.
CHECK(!message.has_name());
```
* If the parser encounters multiple members of the same oneof on the wire, only the last member seen is used in the parsed message.--怎么翻译。。。
* oneof 不能是`repeated`
* 反射 Apis对oneof字段是有效的
* 当增加或者删除oneof字段时一定要小心. 如果检查oneof的值返回None/NOT_SET, 它可能意味着oneof字段没有被赋值或者在一个不同的版本中赋值了。没有任何办法分辨两者。

### Maps
* 使用如下语法可以在消息中使用map
`map<key_type, value_type> map_field = N;`
* `key_type`可以是任意的整数或字符串（除了浮点型、`bytes`）,`value_type`可以是任意类型
* map前不能使用`repeated`
* map不能保证顺序
* map 与以下语法是等效的(equivalent)
```
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```
### Packages
* 为防止消息类型的命名冲突，可以在`.proto`文件中定义`package·：
```
package foo.bar;
message Open { ... }
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```
* 在`Java`中，如果在`.proto`文件中没有提供`option java_package`，`package`将被用做java的package

### Defining Services？？？
### JSON Mapping？？？

### Options
所有可用的options的定义在`google/protobuf/descriptor.proto`。
Options分为`file-level`,`message-level`,`field-level`等几种。
以下为常用的options：
* `java_package` (file option)
生成java代码时的包名，在其他语言中不起作用。如果没有指定，将由`package`生成包名。
`option java_package = "com.example.foo";`
* `java_outer_classname`(file option)
生成java代码时的类名称，在其他语言中不起作用。如果没有指定，类名称将会根据.proto文件的名称采用驼峰式的命名方式进行生成，比如`foo_bar.proto`->`FooBar.java`
`option java_outer_classname = "Ponycopter";`
* `optimize_for`(file option):可以被设置为：`SPEED`,`CODE_SIZE`,`LITE_RUNTIME`
    1. `SPEED`(default)：protocol buffer编译器将在消息类型上生成序列化、解析、其他常用操作的代码。这些代码是极其优化的。
    2. `CODE_SIZE`：生成最少的代码，通过共享或基于反射的代码来实现序列化、语法分析及各种其它操作。代码量比`SPEED`小的多，但操作慢。
    3. `LITE_RUNTIME`：依赖`lite`runtime library(`libprotobuf-lite` instead of `libprotobuf`)来生成代码。这种简化的运行时由于忽略了一些特性，如描述符及反射，因此要比全类库小得多。这种模式经常在移动手机平台应用多一些。编译器生成的方法是像`SPEED`模式一样的快实现（fast implementations）。生成的代码会只实现提供了`Message`接口方法子集的`MessageLite`接口。
`option optimize_for = CODE_SIZE;`
* `packed`(field option)：在一个`repeated`的数字类型上设置为true，在编码时会使用更紧密的编码方式。
2.3.之前的版本，一个没有设置的代码去解析设置过的数据，会忽略此数据。2.3.0及以后，转变是安全的。
`repeated int32 samples = 4 [packed=true];`

### Custom Options 
protocol buffer 允许自定义选项。因为options 是被定义在`google/protobuf/descriptor.proto`文件中消息所定义的，所以自定义选项就是简单（extend）扩展这些消息。
**`extend`是proto2的功能，在proto3中只允许被使用在自定义选项中**
例如：
```
import "google/protobuf/descriptor.proto";

extend google.protobuf.MessageOptions {
  optional string my_option = 51234;
}

message MyMessage {
  option (my_option) = "Hello world!";
}
```
如上，通过继承`MessageOptions`定义了一个`message-level`的选项。使用这个选项的时候，选项名称必须被放置在`()`里，以表明这是一个扩展。在`java`中，可以用如下方式读取`my_option`的值：
```
String value = MyProtoFile.MyMessage.getDescriptor().getOptions()
  .getExtension(MyProtoFile.myOption);
```
自定义选项可以被定义为proto中的任意类型，以下是每种选项的列子：
```
import "google/protobuf/descriptor.proto";

extend google.protobuf.FileOptions {
  optional string my_file_option = 50000;
}
extend google.protobuf.MessageOptions {
  optional int32 my_message_option = 50001;
}
extend google.protobuf.FieldOptions {
  optional float my_field_option = 50002;
}
extend google.protobuf.EnumOptions {
  optional bool my_enum_option = 50003;
}
extend google.protobuf.EnumValueOptions {
  optional uint32 my_enum_value_option = 50004;
}
extend google.protobuf.ServiceOptions {
  optional MyEnum my_service_option = 50005;
}
extend google.protobuf.MethodOptions {
  optional MyMessage my_method_option = 50006;
}

option (my_file_option) = "Hello world!";

message MyMessage {
  option (my_message_option) = 1234;

  optional int32 foo = 1 [(my_field_option) = 4.5];
  optional string bar = 2;
}

enum MyEnum {
  option (my_enum_option) = true;

  FOO = 1 [(my_enum_value_option) = 321];
  BAR = 2;
}

message RequestType {}
message ResponseType {}

service MyService {
  option (my_service_option) = FOO;

  rpc MyMethod(RequestType) returns(ResponseType) {
    // Note:  my_method_option has type MyMessage.  We can set each field
    //   within it using a separate "option" line.
    option (my_method_option).foo = 567;
    option (my_method_option).bar = "Some string";
  }
}
```
如果要在非定义自定义选项的包中使用自定义选项，必须加包名做为前缀，就像消息类型一样，例如：
```
// foo.proto
import "google/protobuf/descriptor.proto";
package foo;
extend google.protobuf.MessageOptions {
  optional string my_option = 51234;
}
// bar.proto
import "foo.proto";
package bar;
message MyMessage {
  option (foo.my_option) = "Hello world!";
}
```
自定义选项必须像其他的字段、扩展一样分配`tag`。如果想要在公共应用中使用自定义选项，必须保证`tag`是全局唯一的。要想获得全局唯一`tag`，可以发邮件给`protobuf-global-extension-registry@google.com`，只需提供工程名称、工程站点地址。通常只需要一个扩展`tag`。可以将多个选项定义到一个子消息中来使用一个扩展`tag`，如下：
```
message FooOptions {
  optional int32 opt1 = 1;
  optional string opt2 = 2;
}

extend google.protobuf.FieldOptions {
  optional FooOptions foo_options = 1234;
}

// usage:
message Bar {
  optional int32 a = 1 [(foo_options).opt1 = 123, (foo_options).opt2 = "baz"];
  // alternative aggregate syntax (uses TextFormat):
  optional int32 b = 2 [(foo_options) = { opt1: 123 opt2: "baz" }];
}
```
每一种选项类型（file-level, message-level, field-level, etc.）都有自己的`tag`空间，所以可以使用相同的数字`tag`定义`FieldOptions`、`MessageOptions`扩展。


### 生成代码
`protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --javanano_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto`
* `IMPORT_PATH`：指定在解析`import`时寻找`.proto`的路径，如果没有指定，则使用当前目录。设置多个`--proto_path`可以指定多个目录，这些目录会被按顺序寻找。
可以简写为`-I=IMPORT_PATH`。
* 可以提供一个或多个输出指令
    1. `--java_out`：java代码输出路径
    2. `--cpp_out`, `-python_out`, `--go_out`, `--ruby_out`, `--javanano_out`,`-objc_out`,`--csharp_out`
如果`DST_DIR`以`zip`,`.jar`结尾，编译器会生成单独的`zip`,`jar`文件。`jar`文件会添加`manifest`文件。如果目标文件已经存在，**会被覆盖，而不是添加新的文件**。
* 必须提供一个或多个`.proto`文件作为输入文件。多个`.proto`文件可以一次指定。

### 与proto2的区别
* proto3中不能使用`Required`,`optional`,`[default = xxx]`

[protobuf]: https://developers.google.com/protocol-buffers/docs/proto3?hl=zh-cn#top_of_page
