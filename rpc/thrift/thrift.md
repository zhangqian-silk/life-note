# Thrift

> [Thrift: Scalable Cross-Language Services Implementation](https://thrift.apache.org/static/files/thrift-20070401.pdf)


## 简介

Thrift 是由 Fackbook 团队开发的跨语言的 RPC 框架，于 2007 年开源，后贡献给 Apache 基金会。

Thrift 采用了 C/S 架构，并通过 IDL(Interface Description Language) 定义接口，之后会协助生成目标语言的代码。生成的代码包括将数据结构和服务接口转换为目标语言的类和接口。

## 数据类型

> [Thrift Types](https://thrift.apache.org/docs/types)

Thrift 定义了一些常见类型，同时也更鼓励开发者去使用这些通用的类型，而不去考虑具体的语言(c, java, python, go...)。

### Base Types

Thrift 提供了如下基本类型：

- `bool`: A boolean value (true or false)
- `byte`: An 8-bit signed integer
- `i16`: A 16-bit signed integer
- `i32`: A 32-bit signed integer
- `i64`: A 64-bit signed integer
- `double`: A 64-bit floating point number
- `string`: A text string encoded using UTF-8 encoding

需要注意的是，部分编程语言不支持 `unsigned integer`，出于通用考虑，Thrift 也没有提供相关类型

### Special Types

- `binary`: a sequence of unencoded bytes

在上述基本类型外，Thrift 额外提供了一种二进制格式，是 `string` 类型的补充，用于提高与 java 的交互性。

## IDL 文件构成

> [Thrift interface description languagel](https://thrift.apache.org/docs/idl)

### Document

```text
Document        ::=  Header* Definition*
```

每个 IDL 文件包含 0 个或多个 `Header`，后面紧跟着 0 个或多个 `Definition`。

### Header

```text
Header          ::=  Include | CppInclude | Namespace
```

每个 `Header` 可以是一个 `Include`、`CppInclude` 或是 `Namespace`。

#### Thrift Include

```text
Include         ::=  'include' Literal
```

通过 `include` 关键字加上另外一个文件的字面量描述，可以使另一个文件中的所有符号可见（带有前缀），并将相应的 `include` 语句添加到为此 Thrift 文档生成的代码中。
