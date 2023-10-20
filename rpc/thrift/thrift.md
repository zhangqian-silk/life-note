# Thrift

> [Thrift: Scalable Cross-Language Services Implementation](https://thrift.apache.org/static/files/thrift-20070401.pdf)

## 简介

Thrift 是由 Fackbook 团队开发的跨语言的 RPC 框架，于 2007 年开源，后贡献给 Apache 基金会。

Thrift 采用了 C/S 架构，并通过 IDL(Interface Description Language) 定义接口，之后会协助生成目标语言的代码。生成的代码包括将数据结构和服务接口转换为目标语言的类和接口。

## IDL

> [Thrift Types](https://thrift.apache.org/docs/types)
> [Thrift interface description languagel](https://thrift.apache.org/docs/idl)

For Thrift version 0.20.0.

Thrift 可以按照 IDL 中定义的数据类型与服务，生成特定语言的代码，以达到跨语言通信的功能。使用者无需再考虑其他语言。

### Basic Definitions

```text
Literal         ::=  ('"' [^"]* '"') | ("'" [^']* "'")

Letter          ::=  ['A'-'Z'] | ['a'-'z']

Digit           ::=  ['0'-'9']

Identifier      ::=  ( Letter | '_' ) ( Letter | Digit | '.' | '_' )*

ListSeparator   ::=  ',' | ';'
```

- `Literal`：字面量，匹配所有单引号或双引号包裹起来的内容。
- `Letter` & `Digit`：字母和数字的集合。
- `Identifier`：标识符，用来定义变量名，结构名，服务名，等等。只能以字母或 '\_' 开头，只能包含字母、数字、'\.' 和 '\_'。
- `ListSeparator`：分隔符，用来标识语句的结束，通常是可选项。

### Types

```text
FieldType       ::=  Identifier | BaseType | ContainerType

BaseType        ::=  'bool' | 'byte' | 'i8' | 'i16' | 'i32' | 'i64' | 'double' | 'string' | 'binary' | 'uuid'

ContainerType   ::=  MapType | SetType | ListType

MapType         ::=  'map' CppType? '<' FieldType ',' FieldType '>'

SetType         ::=  'set' CppType? '<' FieldType '>'

ListType        ::=  'list' CppType? '<' FieldType '>' 

CppType         ::=  'cpp_type' Literal
```

Thrift 中的字段类型(`FieldType`)支持自定义类型(`Identifier`)、基础类型(`BaseType`)以及容器类型(`ContainerType`)。

基础类型如下所示：


### Constant Values

```text
ConstValue      ::=  IntConstant | DoubleConstant | Literal | Identifier | ConstList | ConstMap

IntConstant     ::=  ('+' | '-')? Digit+

DoubleConstant  ::=  ('+' | '-')? Digit* ('.' Digit+)? ( ('E' | 'e') IntConstant )?

ConstList       ::=  '[' (ConstValue ListSeparator?)* ']'

ConstMap        ::=  '{' (ConstValue ':' ConstValue ListSeparator?)* '}'
```

### Field

```text
Field           ::=  FieldID? FieldReq? FieldType Identifier ('=' ConstValue)? XsdFieldOptions ListSeparator?
FieldID         ::=  IntConstant ':'
FieldReq        ::=  'required' | 'optional' 
```

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

通过 `include` 关键字，加上一串用于表示文件路径的 `Literal` ，可以使另一个文件中的所有符号可见（带有前缀），并将相应的 `include` 语句添加到为此 Thrift 文件生成的代码中。

例如在 base.thrift 中，我们有如下定义：

```thrift
struct Base {
    ...
}
```

在 silk.thrift 中，就可以通过 `include` 引入该文件，并使用其内部的符号 `Base`（带有前缀 `base`）。

```thrift
include 'base.thrift'

struct Example {
    1: base.Base ExampleBase
}
```

#### C++ Include

```text
CppInclude      ::=  'cpp_include' Literal
```

`cpp_include` 可以将一个自定义的 C++ 引入声明添加到此 thrift 文档最终生成的 C++ 代码中。

#### Namespace

```text
Namespace       ::=  ( 'namespace' ( NamespaceScope Identifier ) )

NamespaceScope  ::=  '*' | 'c_glib' | 'cpp' | 'delphi' | 'haxe' | 'go' | 'java' | 'js' | 'lua' | 'netstd' | 'perl' | 'php' | 'py' | 'py.twisted' | 'rb' | 'st' | 'xsd'
```

`Namespace` 可以声明该 thrift 最终生成代码时，其内部定义的变量、结构、服务等将针对这些语言生成对应的代码。

例如在文件中包含了如下定义，则最终生成代码时，会在 `silk/example/go` 的路径下，生成对应的 go 文件，在 `silk/example/java` 的路径下，生成对应的 java 文件。

```thrift
namespace go silk.example.go
namespace python silk.example.python
```

### Definition

```text
Definition      ::=  Const | Typedef | Enum | Struct | Union | Exception | Service
```

`Definition` 可以包含常量(`Const`)、类型声明(`Typedef`)、枚举(`Enum`)、结构体(`Struct`)、联合体(`Union`)、异常(`Exception`)、服务(`Service`)。这部分是 IDL 的核心内容。

#### Const

```text
Const           ::=  'const' FieldType Identifier '=' ConstValue ListSeparator?
```

常量(`Const`)声明的构成包括 `const` 关键字，常量的类型(`FieldType`)，常量的标识符(`Identifier`)，赋值符号(`=`)，常量值(`ConstValue`)以及可选的分隔符(`ListSeparator`)。

例如:

```thrift
const i8 constInt = 100
const string constString = 'hello, world';
```

#### Typedef

```text
Typedef         ::=  'typedef' DefinitionType Identifier
DefinitionType  ::=  BaseType | ContainerType
```

类型定义(`Typedef`)以 `typedef` 开头，用于为 Thrift 中声明的类型(`DefinitionType`)创建别名(`Identifier`)。

要注意，目前 `Typedef` 还不支持为自定义类型创建别名。

例如：

```thrift
typedef i8 int8
const int8 constInt = 100
```

#### Enum

```text
Enum            ::=  'enum' Identifier '{' (Identifier ('=' IntConstant)? ListSeparator?)* '}'
```

枚举(`Enum`)用来创建一种可以被枚举的类型，并对每一种值给定特定的命名。

以关键字 `enum` 开头，紧跟着类型标识符(`Identifier`)，用花括号(`{}`)包裹起来该枚举对应的所有的值，每个枚举值都有特定的标识符(`Identifier`)，并且可以为其赋值一个非负的整形(`IntConstant`)，最后可以以分隔符结尾(`ListSeparator`)。

若没有显示的给枚举值赋值，则首位枚举值默认为 0，其他枚举值默认是前一个枚举值的结果加 1。例如：

```thrift
enum silk {
    began   // 默认为 0
    pause   // 默认为 began + 1 = 1
    ended   // 默认为 pause + 1 = 2
}
```

我们也可以通过手动赋值，来实现位图的效果，例如：

```thrift
enum silk {
    status_0 = 1>>0
    status_1 = 1>>1
    status_2 = 1>>2
    status_3 = 1>>3
}
```

#### Struct

```text
Struct          ::=  'struct' Identifier '{' Field* '}'
```

结构体(`Struct`)是 Thrift 中的基本组合类型，其中每个字段(`Field`)的名称在结构体内部都要求是唯一的。

以关键字 `struct` 开头，紧跟着类型标识符(`Identifier`)，用花括号(`{}`)包裹起来该结构体所包含的所有字段(`Field`)。例如：

```thrift
enum silk {
    status_0 = 1>>0
    status_1 = 1>>1
    status_2 = 1>>2
    status_3 = 1>>3
}
```

#### Union

#### Exception

#### Service
