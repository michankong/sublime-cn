+[Sublime Text官方说明文档](https://www.sublimetext.com/docs/3/syntax.html)+

Sublime Text可以使用 ++.sublime-syntax++ 或者 ++.tmLanguage++ 文件来定义语法高亮。本文档主要介绍 ++.sublime-syntax++ 的用法。

# 概述
Sublime语法文件是一个YAML文件，有一个小的头信息，接下来是一个语法环境列表。每个语法环境都包含一个pattern列表，描述如何高亮代码，如何改变当前代码内容。

下面的例子说明了如何高亮一个C语言

```
%YAML 1.2
---
name: C
file_extensions: [c, h]
scope: source.c

contexts:
  main:
    - match: \b(if|else|for|while)\b
      scope: keyword.control.c
```
在其核心，语法定义指定了作用域(e.g., `keyword.control.c`)对文本区域进行控制。颜色选择是根据这些作用域来高亮代码。

这个语法文件包含了一个语法环境，`main`，将会匹配 `[if, else, for, while]` 等关键字，并指定作用域为 `keyword.control.c` 。有一个特殊的代码标识名 `main`：每个语法必须声名 `main`，它将会在文件的开头使用。
`match`关键字的值是一个正则表达式，使用 `ruby`语法。在上面的例子中，`\b`用于确保只匹配整个单词，而不会匹配到`elsewhere`中的`else`。

注意：由于YMAL语法，制表符不能出现在 ++.sublime-syntax++ 文件中。
# 头部 -- header
头部区域可以出现的关键字如下：

- name 定义在菜单栏中显示的语法名字。可选，如果不定义，将从文件名中派生出来。
- file_extensions 字符串列表，定义什么文件扩展名可以使用该语法
- first_line_match 当文件被打开时，没有得到确认的扩展名，文件内容的第一行将对该regex进行测试，看看是否应该应用语法。
- scope 指定文件中所有文本的默认作用域
- hidden 隐藏的语法定义不会显示在菜单中,但仍然可以由插件指定，或者由其他语法定义包含。

# 上下文环境 -- contexts

对于大多数语言，您需要不止一个语法环境，举个例子，在C语言中，我们不需要高亮一个字符串中的`for`做为关键字。下面举个例子说明如何处理这个问题：
```
%YAML 1.2
---
name: C
file_extensions: [c, h]
scope: source.c

contexts:
  main:
    - match: \b(if|else|for|while)\b
      scope: keyword.control.c
    - match: '"'
      push: string

  string:
    - meta_scope: string.quoted.double.c
    - match: \\.
      scope: constant.character.escape.c
    - match: '"'
      pop: true
```
每二个`pattern`已经添加到`main`环境中，用来匹配双引号符号（注意，这里用'"'来表示，因为单独的引号符号会触发YAML语法错误），并在语法环境栈里增加一个新的环境`string`。这就意味着文件中剩下内容将使用`string`环境进行处理，而不使用`main`环境，直到`string`环境从语法环境栈中跳出。
`string`环境引入了一个新的`pattern`：`meta_scope`。在`string`环境还在语法环境栈时，指定所有代码的作用域为`string.quoted.double.c`。
当在Sublime Text进行编辑时，你可以通过下面的方法查询光标所在文本的作用域
- `ontrol + shift + p` (OSX) 
- `ctrl + alt + shift + p` (Windows and Linux)
`string`代码块包含两个`pattern`：第一个匹配反斜杠和一个任意字符，第二个匹配引号。注意，最后一个`pattern`指定一个动作：当遇到一个没有转义的引号符时，`string`环境将会跳出所有语法环境栈，返回到`main`环境指定的作用域。
当上下文同时可以匹配多个`pattern`时，总是先匹配最左边的。当多个`pattern`同时匹配一个上下文位置时，第一个定义的`pattern`将被选择。

## Meta Patterns

- meta_scope 给上下文所有的代码指定定义好的作用域，包括上下文入栈和出栈的`pattern`。

- meta_content_scope 同上，但是不会应用到触发该上下文的文本（eg.,在上面的字符串示例中，内容的作用域不会应用于引号字符）。
- meta_include_prototype 用于停止当前上下文自动包括原型上下文。
- clear_scopes 这个设置允许从当前的栈中移除作用域名。可以用整数或都`true`来移除所有的作用域名。它可以在 `meta_scope` 和 `meta_content_scope` 之前使用。这个通常只用于一种语法嵌入另一种语法的情况。

`meta pattern` 必须在上下文的最前面写列出来，优先于任何`match` 或 `include` 的 `pattern`。

## Match Patterns

一个 `match pattern` 可以包含以下的`keys`：
- match  正则表达式匹配指定的文本。YAML语法允许使用不带引号的多个字符串，这样正则表达式看起来更清晰，但是要清楚什么时候需要使用引号包含正则表达式。当正则表达式中包含 `#` `:` `-` `{` `[` `>`等字符时，就需要使用引号了。
- scope 给匹配的文本指定作用域
- captures 数字到作用域的映射表，指定被正则捕获的分组部分的作用域。请看下面的例子。 
- push 将上下文推入栈内。这可能是一个单一的上下文名字，上下文名字列表，内联的上下文名字或匿名上下文。
- pop 将上下文推出栈。这个键只有一个唯一值：`true`
- set 接收和 `push` 相同的参数，但是是先推出栈，然后将给定的上下文推入栈。
- syntax 请见下面的 `Including Other Files` 内容。

注意：`push` `pop` `set` `syntax` 这些动作是独立的，在一个单独的`pattern`中，只能使用其中一个。
在下面的例子中，正则表达式包括了两个捕获分组，并且捕获列表的键值指定被捕获分组不同的作用域：
```
- match: "^\\s*(#)\\s*\\b(include)\\b"
  captures:
    1: meta.preprocessor.c++
    2: keyword.control.include.c++
```

## Include Patterns

通常，在另一个上下文中包含一个上下文的内容是很方便的。例如，你可能定义了多个不同的上下文环境来解释C语言，而且几乎所有上下文都包含了对注释的解释。与其将相关的匹配 `pattern` 复制到每一个上下文环境中，不如使用下面的方法：
```
expr:
  - include: comments
  - match: \b[0-9]+\b
    scope: constant.numeric.c
  ...
```

这里，就把所有的 `match patterns` 和 `include patterns` 所定义的注释相关的上下文环境都包含进来了。它们将被插入到 `include pattern` 位置，这样你仍然可以控制 `pattern` 的顺序。任何 `meta patterns` 中定义的有关注释的上下文环境都将被忽略。

有了诸如注释之类的元素，在每个上下文环境中包含他们会变得很简单，且通用，需要例外情况的时候，只需要替换之前的即可。你可能使用一个关键字 `prototype` 来创建这样的上下文环境，将会在其他的每一个上下文环境的顶部自动包含这个属性，除非在上下文环境使用了 `meta_include_prototype` `meta pattern`。例如：
```
prototype:
  - include: comments

string:
  - meta_include_prototype: false
  ...
```

在C中，在字符串中的 `/*` 不是注释的开始，所以在 `string` 上下文环境中应该指明 `protoype` 属性不生效。

# Including Other Files

`Sublime Text` 语法文件支持嵌入另一个语法定义的概念。例如，HTML代码中会包含嵌入的javascript代码。这里有一个关于HTML基本语法定义的例子。

```
scope: text.html

contexts:
  main:
    - match: <script>
      push: Packages/JavaScript/JavaScript.sublime-syntax
      with_prototype:
        - match: (?=</script>)
          pop: true
    - match: "<"
      scope: punctuation.definition.tag.begin
    - match: ">"
      scope: punctuation.definition.tag.end
```

注意上面第一条规则。它指明当我们遇到 `<script>` 标签时，++JavaScript.sublime-syntax++ 中的 `main` 环境将会被推入当前的上下文栈中。它还定义了另一个关键字 `with_prototype`。它包含一个模式列表，这些模式将被插入到 ++JavaScript.sublime-syntax++ 中定义的所有上下文中。注意的是， `with_prototype` 从概念上和 `prototype` 相类似，无论如何，它总是会被插入到每一个引用到的上下文中，而不考虑它们的 `meta_include_prototype` 设置。

在这个case中，当在接下来的文字中遇到 `</script>` 标签，插入的模式将被弹出当前上下文环境。注意，它并不是匹配实际上的 `</script>` 标签，而是使用了先行断言，这里有两个关键的作用：它不仅允许HTML规则匹配了对应的结束标签，正确地高亮代码，还确保 `Javascript` 的上下文环境跳出当前的上下文环境。例如，上下文环境栈还在 `Javascript` 字符串的中，当遇到 `</script>` 标签， `Javascript` 字符串和 `main` 上下文环境都将被弹出。

注意，虽然 Sublime Text 同时支持 ++.sublime-syntax++ ++.tmLanguage++ 两种语法文件，但是对于在 ++.sublime-syntax++ 文件中包含 ++.tmLanguage++ 文件这种方式，是不支持的。

另一个常见的场景是，在HTML中包含一个模板语言。下面是一个例子，这次是 `Jinja` 的一小部分代码。

```
scope: text.jinja
contexts:
  main:
    - match: ""
      push: "Packages/HTML/HTML.sublime-syntax"
      with_prototype:
        - match: "{{"
          push: expr

  expr:
    - match: "}}"
      pop: true
    - match: \b(if|else)\b
      scope: keyword.control
```

这个就和 `HTML` 嵌入 `javascript` 代码的例子不太一样了，因为模板文件倾向于从内部进行操作：默认情况下，需要以HTML语言环境编辑环境，只有在特定的表达式中才能逃到底层的模板语言。

在上面的例子中，默认是在HTML语法环境中的： `main` 上下文环境包含了一个始终匹配的单一模式，不使用任何文本，只包括HTML语法。

在包含HTML语法的位置，`Jinja` 语法指令（ {{ ... }} ）通过 `with_prototype` 键进行包含，因此，在HTML语法（以及javascript，通过传递性）中注入到每个上下文。

# 变量 --- Variables

一些正则表达式存在相同的常用部分是很常有的。为了避免重复地输入，你可以使用变量：

```
variables:
  ident: '[A-Za-z_][A-Za-z_0-9]*'
contexts:
  main:
    - match: '\b{{ident}}\b'
      scope: keyword.control
```

变量必须在 ++.sublime-syntax++ 文件的顶部区域进行定义，在正则中使用  `{{varname}}` 来引用。变量本身可以包含其他变量。 注意，任何不匹配 `{{[A-Za-z0-9_]+}}` 的文本，都不会被认为是变量，所以正则还是可以包含 `{{` 字符的。

# Selected Examples

## Bracket Balancing 括号平衡

这个例子说明缺少左括号的的右括号的高亮方式：

```
name: C
scope: source.c

contexts:
  main:
    - match: \(
      push: brackets
    - match: \)
      scope: invalid.illegal.stray-bracket-end

  brackets:
    - match: \)
      pop: true
    - include: main
```

## Sequential Contexts 连续出现的情况

该示例高亮了包含很多分号的C语言的语句：

```
for_stmt:
  - match: \(
    set: for_stmt_expr1
for_stmt_expr1:
  - match: ";"
    set: for_stmt_expr2
  - match: \)
    pop: true
  - include: expr
for_stmt_expr2:
  - match: ";"
    set: for_stmt_expr3
  - match: \)
    pop: true
  - include: expr
for_stmt_expr3:
  - match: \)
    pop: true
  - match: ";"
    scope: invalid.illegal.stray-semi-colon
  - include: expr
```

## Advanced Stack Usage

在C中，标识符通常用typedef关键字定义。所以 `Goto` 定义可以使用这些标识符，这些标识符应该有一个隶属于它们的作用域 `entity.name.type`。

```
typedef int coordinate_t;

typedef struct
{
    int x;
    int y;
} point_t;
```

要识别这些，在匹配typedef关键字之后，两个上下文环境将被推入栈：第一个将会识别类型名称，然后弹出，而第二个将识别类型的引用名称。

```
main:
  - match: \btypedef\b
    scope: keyword.control.c
    set: [typedef_after_typename, typename]

typename:
  - match: \bstruct\b
    set:
      - match: "{"
        set:
          - match: "}"
            pop: true
  - match: \b[A-Za-z_][A-Za-z_0-9]*\b
    pop: true

typedef_after_typename:
  - match: \b[A-Za-z_][A-Za-z_0-9]*\b
    scope: entity.name.type
    pop: true
```

上面的例子， `typename` 是一个可以复用的上下文，将会读入一个 typename，并且在结束时，弹出堆栈。它可以在类型需要被消费到任何上下文中，比如在typedef中，或者作为函数函数参数。

`main` 环境使用了推入两个上下文环境到堆栈的匹配模式，列表中最右边的将会进入栈中的优先级更高的上下文环境。一旦 `typename` 上下文环境弹出 ，`typedf_after_typename` 将会上堆栈的最优先位置。

还要注意上面，在 `typename` 上下文中使用了匿名上下文。

## PHP Heredocs

该例子展示了怎么匹配PHP语法中的 `heredoc`。在 `main` 环境中的匹配模式被捕获到 heredoc 标识符，而 `heredoc` 上下文对应的弹出模式的匹配的文字将引用 `\1` 标识符捕获到的文本。

```
name: PHP
scope: source.php

contexts:
  main:
    - match: <<<([A-Za-z][A-Za-z0-9_]*)
      push: heredoc

  heredoc:
    - meta_scope: string.unquoted.heredoc
    - match: ^\1;
        pop: true
```

# 测试

构建语法定义时，不必使用 `show_scope_name` 命令手动的测试作用域，你可以定义一个语法测试文件，它将为你进行检查：

```
// SYNTAX TEST "Packages/C/C.sublime-syntax"
#pragma once
// <- source.c meta.preprocessor.c++
 // <- keyword.control.import

// foo
// ^ source.c comment.line
// <- punctuation.definition.comment

/* foo */
// ^ source.c comment.block
// <- punctuation.definition.comment.begin
//     ^ punctuation.definition.comment.end

#include "stdio.h"
// <- meta.preprocessor.include.c++
//       ^ meta string punctuation.definition.string.begin
//               ^ meta string punctuation.definition.string.end
int square(int x)
// <- storage.type
//  ^ meta.function entity.name.function
//         ^ storage.type
{
    return x * x;
//  ^^^^^^ keyword.control
}

"Hello, World! // not a comment";
// ^ string.quoted.double
//                  ^ string.quoted.double - comment
```

要编写一个这样的测试文件，遵循下面的规则：

1. 确保测试文件命名以 ++syntax_test_++。
2. 确保文件保存在 Packages 目录下，最好和对应的 .sublime_syntax 文件同一级目录。
3. 确保文件的第一行以 `<comment_token> SYNTAX TEST "<syntax_file>"` 开始。注意这里的语法文件包括 ` .sublime-syntax` `.tmLanguage`。

一旦上述条件得到满足，使用语法测试或语法定义文件运行 `build` 命令将测试进行所有的语法测试，在输出面板输出所有结果。
按下一个结果（F4）可以导航到第一个失败的测试点。

在语法测试文件中的每一个测试都要以注释符号开始（在第一行的基础上，它实际上不需要根据语法来做注释），后面接 `^` `<-` 符号。

测试语法中的两个类型：

- 插入：`^` 会在接下来的非测试行中测试对应作用域的选择器。在 `^` 的相同列中测试。连续的 `^` 将测试对应选择器中的每一列。
- 箭头：`<-` 会在接下来的非测试行中测试对应作用域的选择器。在 `<-` 的相同列中测试。它将会测试和注释符号相同的列。
