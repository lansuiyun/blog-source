---
title: 解析proto文件
date: 2016-04-22 10:33:50
tags: protobuf
categories: java tool
---
这篇文章主要描述如何使用程序获取`proto`文件的信息。

#### 生成`desc`文件命令
```
protoc -I=IMPORT_PATH --descriptor_set_out=DST_FILE path/to/file.proto
```
<!--more-->
proto提供了api可以解析`desc`文件来获取proto文件的信息。

####  定义`proto`文件
- 定义一个包含自定义选项的`proto`文件`options.proto`，内容如下：
```
syntax = "proto3";
import "protoc/google/protobuf/descriptor.proto";
 
option java_package = "sky.test";
option java_outer_classname = "Options";
 
extend google.protobuf.MessageOptions {
  int32 msgid = 54321;
}
```
- 定义一个使用了自定义选项的`proto`文件`test.proto`，内容如下：
```
syntax = "proto3";
 
import "options.proto";
 
option java_package = "sky.test";
option java_outer_classname = "Test";
 
message persion {
    string name = 1;
    int32 age = 2;
    repeated string like = 3;
}
 
message people {
    option (msgid) = 1111;
    repeated persion persions = 1;
    string country = 2;
}
```
#### 生成`desc`文件
```
protoc --descriptor_set_out=test.desc options.proto test.proto
```
使用以上指令将`options.proto`，`test.proto`两个文件的描述信息生成到同一个`desc`文件中。
#### 获取扩展信息
```java
public static void getExtendInfo(String extendDescFile) throws Exception {
    DescriptorProtos.FileDescriptorSet fdSet = DescriptorProtos.FileDescriptorSet.parseFrom(new FileInputStream(extendDescFile));
    for (DescriptorProtos.FileDescriptorProto fileDescriptorProto : fdSet.getFileList()) {
        //fileDescriptorProto.getExtensionList() 获取扩展（proto中的extend）列表
        for (DescriptorProtos.FieldDescriptorProto fieldDescriptorProto : fileDescriptorProto.getExtensionList()) {
            System.out.println(fieldDescriptorProto);
            System.out.println(fieldDescriptorProto.getName());
            System.out.println(fieldDescriptorProto.getNumber());
        }
    }
}
```
运行代码，输出如下：
```
name: "msgid"
extendee: ".google.protobuf.MessageOptions"
number: 54321
label: LABEL_OPTIONAL
type: TYPE_INT32
json_name: "msgid"

msgid
54321
```
#### 获取`proto`文件的name，options，message等信息
```java
public static void getMsgInfo(String descFile) throws Exception {
    DescriptorProtos.FileDescriptorSet fdSet = DescriptorProtos.FileDescriptorSet.parseFrom(new FileInputStream(descFile));
    //可以将多个proto文件的描述信息生成到同一个desc文件，每个proto文件对应一个FileDescriptorProto对象
    //FileDescriptorProto包含proto文件的信息，比如name、option（比如定义的java_package，java_outer_classname等）、消息等
    for (DescriptorProtos.FileDescriptorProto fileDescriptorProto : fdSet.getFileList()) {
        System.out.println("proto file name ：" + fileDescriptorProto.getName());
        System.out.println("proto file Options ：" + fileDescriptorProto.getOptions());
        //DescriptorProto代表proto文件中的一个消息
        for (DescriptorProtos.DescriptorProto descriptorProto : fileDescriptorProto.getMessageTypeList()) {
            System.out.println("msg  " + descriptorProto.getName() + " 的字段信息");
            System.out.println();
            //FieldDescriptorProto 代表消息中的一个字段，包含名称、tag、类型等信息
            for (DescriptorProtos.FieldDescriptorProto fieldDescriptorProto : descriptorProto.getFieldList()) {
                System.out.println(fieldDescriptorProto);
            }
            //如果消息中使用了自定义选项，通过UnknownFieldSet可以获取到相关信息
            UnknownFieldSet uf = descriptorProto.getOptions().getUnknownFields();
            for (Map.Entry<Integer, UnknownFieldSet.Field> entry : uf.asMap().entrySet()) {
                System.out.println("UnknownFieldSet.Field  key:" + entry.getKey());
                UnknownFieldSet.Field value = entry.getValue();
                System.out.println(value.getVarintList());
            }
        }
    }
}
```
运行代码，输出如下：
```
proto file name ：options.proto
proto file Options ：java_package: "sky.test"
java_outer_classname: "Options"

proto file name ：test.proto
proto file Options ：java_package: "sky.test"
java_outer_classname: "Test"

msg  persion 的字段信息

name: "name"
number: 1
label: LABEL_OPTIONAL
type: TYPE_STRING
json_name: "name"

name: "age"
number: 2
label: LABEL_OPTIONAL
type: TYPE_INT32
json_name: "age"

name: "like"
number: 3
label: LABEL_REPEATED
type: TYPE_STRING
json_name: "like"

msg  people 的字段信息

name: "persions"
number: 1
label: LABEL_REPEATED
type: TYPE_MESSAGE
type_name: ".persion"
json_name: "persions"

name: "country"
number: 2
label: LABEL_OPTIONAL
type: TYPE_STRING
json_name: "country"

UnknownFieldSet.Field  key:54321
[1111]
```