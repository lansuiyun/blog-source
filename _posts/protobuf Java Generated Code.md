---
title: protobuf生成的java代码
date: 2016-04-20 15:54:50
tags: protobuf
categories: java tool
---
这篇文章描述了protobuf编译生成的java文件内容，原文地址[Java Generated Code][java-generated]
<!--more-->
### Message
消息定义
```
message Foo {}
```
protobuf编译器将生成一个名字为`Foo`的java class，它实现了[Message][message]接口。`Foo`被声明为`final`，不允许有子类。
`Foo`继承了[GeneratedMessage][GeneratedMessage]。
[Message][message]接口定义了检查、操作、读写消息的方法。除此之外，`Foo`还定义了一下静态方法：
* `static Foo getDefaultInstance()`：返回一个`Foo`的实例，与`Foo.newBuilder().build()`效果相同。即所有的字段都使用默认值。
* `static Descriptor getDescriptor()`：返回类型的描述，包括有哪些字段以及字段的类型。可以与`Message`的反射方法配合使用。
* `static Foo parseFrom(...)`：将指定的数据源解析为`Foo`。
* `static Parser parser()`：返回一个实现了多种`parseFrom()`方法的`Parser`的实例。
* `Foo.Builder newBuilder()`：创建一个构建对象。
* `Foo.Builder newBuilder(Foo prototype)`：创建一个新的构建对象，所有字段的值与`prototype`一样。

### Builders
消息对象，例如`Foo`的实例是像Java中的`String`一样是不可变对象。想要构建消息对象，就想要使用`builder`。每个消息类都有自己的构建类。
`builder`修改内容的方法，包括字段的`setter`方法，总是返回`builder`本身的引用。这样可以进行方法链调用，例如：
```
builder.mergeFrom(obj).setFoo(1).setBar("abc").clearBaz();
```

### Sub Builders
消息包含子消息，编译器也会生成`sub builders`。这样可以重复修改嵌套类而无需重新构建(reBuilding)。例如
```
message Foo {
  optional int32 val = 1;
  // some other fields.
}

message Bar {
  optional Foo foo = 1;
  // some other fields.
}

message Baz {
  optional Bar bar = 1;
  // some other fields.
}
```
如果已存在`Baz`对象，想要改变`Foo`的值，无需这样:
```
baz = baz.toBuilder().setBar(
      baz.getBar().toBuilder().setFoo(
          baz.getBar().getFoo().toBuilder().setVal(10).build()
      ).build()).build();
```
可以：
```
 Baz.Builder builder = baz.toBuilder();
  builder.getBarBuilder().getFooBuilder().setVal(10);
  baz = builder.build();
```

### Fields
protobuf编译器为定义在`proto`文件消息中的每一个字段都生成了相应的读、写的方法。其中message只定义了读取值得方法，而与之关联的builder中则定义了读、写方法。
编译器还为每个字段对应的`tag`生成了int型的常量。例如，字段为`optional int32 foo_bar = 5;`，编译器将生成`public static final int FOO_BAR_FIELD_NUMBER = 5;`。
#### 单个字段
定义了如下字段：
```
int32 foo = 1;
```
编译器在messge和builder中都生成了读取方法
- `boolean hasFoo()` ：如果`foo`字段已经被赋值，则返回`true`
- `int getFoo()` ：如果foo还没有被赋值，将返回默认值
只在builder中生成的方法
- `Builder setFoo(int value)`
- `Builder clearFoo()` ：清理`foo`的值。调用此方法后，`hasFoo()`返回`false`，`getFoo()`返回默认值
#### 枚举字段
message、builder都有对应的读取方法
- `int getFooValue()` ：返回枚举的int值
只有builder生成的方法
- `Builder setFooValue(int value)`：为枚举设置int值

#### Repeated Fields
定义了如下字段
```
repeated int32 foo = 1;
```
message、builder都生成的方法
- `int getFooCount()` 
- `int getFoo(int index)`
- `List<Integer> getFooList()`：message返回的list是不可变的。builder返回的list是不可修改的。
只在builder中生成的方法
- `Builder setFoo(int index, int value)`
- `Builder addFoo(int value)`
- `addAllFoo(Iterable<? extends Integer> value)`
- `Builder clearFoo()`


[message]: https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/Message?hl=zh-cn
[GeneratedMessage]: https://developers.google.com/protocol-buffers/docs/reference/java/com/google/protobuf/GeneratedMessage?hl=zh-cn
[java-generated]: https://developers.google.com/protocol-buffers/docs/reference/java-generated