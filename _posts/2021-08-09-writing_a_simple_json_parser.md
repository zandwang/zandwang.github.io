---
layout: post
title:  "用python编写一个简单的json解析器"
date:   2021-08-09 03:35:49 +0800
categories: python
---

> 本文为译文，原文地址：https://notes.eatonphil.com/writing-a-simple-json-parser.html

编写一个JSON解析器是熟悉解析技术最简单的方法。它的格式非常简单。它是递归定义，所以与解析Brainfuck(一门深奥的编程语言，译者注)相比，解析JSON只是一点轻微挑战；并且你可能也在用JSON。除了最后一点，解析Scheme(一门函数语言)的S表达式可能是一项更简单的任务。

如果你只是想看下pj库的代码，请从github上检出

# 解析是什么，不是什么
解析通常分为两个阶段：词法分析和语义分析。词法分析将原始输入分为简单的语言可分解元素，即所谓的“符号”。语义分析（通常也称之为解析）接受一串符号并尝试在它们中找到符合所解析语言的模式。

解析不能确定输入源的语义可行性。输入源的语义可行性可能包含变量是否在使用前已经定义，或者调用函数的参数是否正确，或者一个变量是否能在某些作用域二次声明

> 当然，人们选择解析和应用语义规则不尽相同，我假设用“传统”的方法来解析核心概念。

# JSON库的接口
最终，应该有一个`from_string`方法接受JSON编码的字符串并且返回对应的python字典。
比如：
```python
assert_equal(from_string('{"foo":1}'), {"foo":1})
```
# 词法分析
词法分析将输入的字符串分解为符号。注释和空白字符一般会在词法分析阶段丢弃，所以只会留下一些简单的输入用来在语法分析阶段搜索语法匹配。

假设一个简单的词法分析器，你可能需要把所有的输入的字符遍历一遍，把它们分解为基础的/非递归的语言结构比如整数/字符串和布尔类型。特别的，字符串必须是词法分析的一部分，因为你不能在不知道它不是字符串的一部分的情况下丢弃空格。

> 在一个有用的词法分析器中你可以追踪你跳过的空格和注释/当前行数和当前文件，这样你可以在源文件分析产生错误的任何步骤回头看一下。最近V8 JS引擎可以重现函数的确切源代码，这至少需要词法分析器的帮助才能实现

# 实现一个JSON词法分析器
JSON 词法分析的要点是对输入源遍历并试着找到字符串/数字/布尔/空值或者JSON符号像左括号/左花括号，最终返回这些元素的列表

这里给出一个样例输入后词法分析器应该返回的值

```python
assert_equal(lex('{"foo": [1,2, {"bar":2}]}'), ['{', 'foo', ':', '[', 1, ',', 2, ',', '}', 'bar', ':', 2, '}', ']', '}'])
```

这里给出这段逻辑开始的样子：

```python
def lex(string):
    tokens = []
    while len(string):
        json_string, string = lex_string(string)
        if json_string is not None:
            tokens.append(json_string)
            continue
        # TODO: lex booleans, nulls, numbers

        if string[0] in JSON_WHITESPACE:
            string = string[1:]
        elif string[0] in JSON_SYNTAX:
            tokens.append(string[0])
            string = string[1:]
        else:
            raise Exception('Unexpected character: {}'.format(string[0]))
    return tokens
```

这里的目标是尝试匹配字符串/数字/布尔和空值，并将它们加到符号列表中。如果都不匹配，检查这些字符是不是空白字符，如果是则丢弃。另外如果是JSON的语法关键字（比如左括号）的一部分则按照符号保存起来。最终如果字符不匹配任何一种模式则抛错。

让我们稍微扩展下核心逻辑来支持所有的种类并添加函数存根

```python
def lex_string(string):
    return None, string

def lex_number(string):
    return None, string

def lex_bool(string):
    return None, string

def lex_null(string):
    return None, string

def lex(string):
    tokens = []

    while len(string):
        json_string, string = lex_string(string)
        if json_string is not None:
            tokens.append(json_string)
            continue
        
        json_number, string = lex_number(string)
        if json_number is not None:
            tokens.append(json_number)
            continue
        
        json_bool, string = lex_bool(string)
        if json_bool is not None:
            tokens.append(json_bool)
            continue
        
        json_null, string = lex_null(string)
        if json_null is not None:
            tokens.append(None)
            continue
        
        if string[0] in JSON_WHITESPACE:
            string = string[1:]
        elif string[0] in JSON_SYNTAX:
            tokens.append(string[0])
            string = string[1:]
        else:
            raise Exception('Unexpected character: {}'.format(string[0]))
    return tokens
```

# 分析字符串
对于`lex_string`函数其要点是检查第一个字符是否是引号。如果是，遍历输入的字符串直到发现结束引号。如果你没有发现出师引号，返回None和原始list。如果你发现了初始引号和一个结束引号，返回引号间的字符串和剩下的未检查字符串。

```python
def lex_string(string):
    json_string = ''

    if string[0] == JSON_QUOTE:
        string = string[1:]
    else:
        return None, string
    
    for c in string:
        if c == JSON_QUOTE:
            return json_string, string[len(json_string)+1:]
        else:
            json_string += c
    raise Exception('Expected end of string quote')
```

# 分析数字
对于`lex_number`函数，其要点是遍历输入直到你发现一个字符不是数字的一部分。（当然这大大简化了，实现更高的精度就留给读者来练习）在找到一个字符不是数字后，如果累计的字符数大于0，则返回一个浮点数或整数。否则返回None和原始字符串输入。

```python
def lex_number(string):
    json_number = ''

    number_characters = [str(d) for d in range(0, 10)] + ['-', 'e', '.']

    for c in string:
        if c in number_characters:
            json_number += c
        else:
            break
    
    rest = string[len(json_number):]
    if not len(json_number):
        return None, string

    if '.' in json_number:
        return float(json_number), rest
    
    return int(json_number), rest
```

# 解析布尔值和空值
找布尔值和空值仅仅是简单的字符串匹配

```python
def lex_bool(string):
    string_len = len(string)

    if string_len >= TRUE_LEN and string[:TRUE_LEN] == 'true':
        return True, string[TRUE_LEN]
    elif string_len >= FALSE_LEN and string[:FALSE_LEN] == 'false':
        return False, string[FALSE_LEN:]
    
    return None, string

def lex_null(string):
    string_len = len(string)

    if string_len >= NULL_LEN and [:NULL_LEN] == 'null':
        return True, string[NULL_LEN]
    
    return None, string
```
现在分析器的代码写完了，查看lexer.py 来获得完整代码

# 语法分析
语法分析的基础工作是遍历一维的符号列表，并根据语言的定义将标记组匹配到语言的各个部分。如果，在语法分析的任何点，分析器不能将当前符号组匹配到合法的语言语法，分析器将会失败并可能给你一些有用的信息比如你提供了什么输入，什么地方发生了错误，以及你期望是什么。

# 实现一个JSON解析器
JSON解析器的要点是遍历来自`lex`的结果符号列表，并尝试匹配到对象、列表或者普通值。

这里给出一个解析器对一个样例输入应该的返回
```python
tokens = lex('{"foo": [1, 2, {"bar": 2}]}')
assert_equal(tokens, ['{', 'foo', ':', '[', 1, ',', 2, '{', 'bar', ':', 2, '}', ']', '}'])
assert_equal(parse(tokens), {'foo': [1, 2, {'bar': 2}]})
```

下面是解析器逻辑部分的样例

```python
def parse_array(tokens):
    return [], tokens

def parse_object(tokens):
    return {}, tokens

def parse(tokens):
    t = tokens[0]

    if t == JSON_LEFTBRACKET:
        return parse_array(tokens[1:])
    elif t == JSON_LEFTBRACE:
        return parse_object(tokens[1:])
    else:
        return t, tokens[1:]
```

词法分析和语法分析一个关键的结构性不同是词法分析返回一个一维的符号列表。语法分析经常是递归定义的并且返回一个递归的，树状的结构体。因为JSON是数据序列化而不是一种语言，所以语法解析器需要生成python的对象，而不是对语法树执行更多的分析（或者是生成代码在编译器的情况下）。
此外，独立于语法分析器的词法分析器的好处是这两段代码都很简单并且只关注特殊元素。

# 解析数组
解析数组就是解析数组成员，期望有一个分号在成员之间或者右括号在数组的最后。

```python
def parse_array(tokens):
    json_array = []
    
    t = tokens[0]
    if t == JSON_RIGHTBRACKET:
        return json_array, tokens[1:]

    while True:
        json, tokens = parse(tokens)
        json_array.append(json)

        t = tokens[0]
        if t == JSON_RIGHTBRACKET:
            return json_array, tokens[1:]
        elif t != JSON_COMMA:
            raise Exception('Expected comma after object in array')
        else:
            tokens = tokens[1:]
    
    raise Exception('Expected end-of-array bracket')
```

# 解析结构体
解析结构体就是解析内部通过冒号分割的键值对，外部通过逗号分割，直到到达结构体结束

```python
def parse_object(tokens):
    json_object = {}

    t = tokens[0]
    if t == JSON_RIGHTBRACE:
        return json_object, tokens[1:]

    while True:
        json_key = tokens[0]
        if type(json_key) is str:
            tokens = tokens[1:]
        else:
            raise Exception('Expected string key, got: {}'.format(json_key))
        
        if tokens[0] != JSON_COLON:
            raise Exception('Expected colon after key in object, got: {}')
        
        json_value, tokens = parse(tokens[1:])
        json_object[json_key] = json_value

        t = tokens[0]
        if t == JSON_RIGHTBRACE:
            return json_object, tokens[1:]
        elif t != JSON_COMMA:
            raise Exception('Expected comma after pair in object, got: {}')
        
        tokens = tokend[1:]
    
    raise Exception('Expected end-of-object brace')
```

这样解析器的代码就完成了！完整的代码可以看parse.py

# 统一库
为了提供理想的接口，创造一个`from_string`函数包装`lex`和`parse`函数。

```python
def from_string(string):
    tokens = lex(string)
    return parse(tokens)[0]
```

这样库就完成了。在github上可以检出完成的实现包含基础测试代码

