# Thrift

> [Thrift: Scalable Cross-Language Services Implementation](https://thrift.apache.org/static/files/thrift-20070401.pdf)

## 简介

Thrift 是由 Fackbook 团队开发的跨语言的 RPC 框架，于 2007 年开源，后贡献给 Apache 基金会。

Thrift 采用了 C/S 架构，并通过 IDL(Interface Description Language) 定义接口，之后会协助生成目标语言的代码。生成的代码包括将数据结构和服务接口转换为目标语言的类和接口。

## IDL 文件构成

> [Thrift Types](https://thrift.apache.org/docs/types)
> [Thrift interface description languagel](https://thrift.apache.org/docs/idl)

For Thrift version 0.20.0.

Thrift 可以按照 IDL 文件中定义的数据结构与服务，生成特定语言的代码，以达到跨语言通信的功能。

IDL 文件使用了 Thrift 定义的一些基础类型，使用者无需再考虑其他语言。

### Basic Definitions

- `Literal`：字面量，匹配所有单引号或双引号包裹起来的内容。

    ```text
    Literal         ::=  ('"' [^"]* '"') | ("'" [^']* "'")
    ```

- `Letter` & `Digit`：字母和数字的集合。

    ```text
    Letter          ::=  ['A'-'Z'] | ['a'-'z']
    Digit           ::=  ['0'-'9']
    ```

- `Identifier`：标识符，用来定义变量名，结构名，服务名，等等。只能以字母或 '\_' 开头，只能包含字母、数字、'\.' 和 '\_'。

    ```text
    Identifier      ::=  ( Letter | '_' ) ( Letter | Digit | '.' | '_' )*
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

`namespace` 可以声明该 thrift 最终生成代码时，其内部定义的变量、结构、服务等将针对这些语言生成对应的代码。

例如在文件中包含了如下定义，则最终生成代码时，会在 `silk/example/go` 的路径下，生成对应的 go 文件，在 `silk/example/java` 的路径下，生成对应的 java 文件。

```thrift
namespace go silk.example.go
namespace python silk.example.python
```
