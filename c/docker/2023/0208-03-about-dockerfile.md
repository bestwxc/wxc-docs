# 关于Dockerfile

## 格式

Dockerfile语法格式如下
```text
# 注释（Comment）
指令（INSTRUCTION） 参数（arguments）
```
其中指令（INSTRUCTION）大小写不敏感，但是通常大写来与后面的参数区分

## FROM 指令
每个Docker必须包含至少一个构建阶段（stage），每个构建阶段从FROM指令开始。

注意： 大多数说法中，Dockerfile必须由FROM指令开始是不严谨的（可能早期docker版本是严谨的），FROM指令前面可以有注释、解析指令和ARG指令。

FROM 指令只能解析全局参数，不能解析其他参数。（全局参数: 第一个FROM之前定义的ARG都是全局ARG参数）

## 全局参数
Dockerfile中第一条参数之前ARG指定的参，特点：
1. 能被FROM使用
2. 可以指定多个
3. 不能被FROM后的指令解析

## 解析指示（directive）
翻译成指示，是因为他是用作干涉解析器解析Dockerfile的，并不是命令。 

格式
```text
# directive=value1
```

特点：
1. 必须在文件最开始的位置
2. 不可以分行，但是可以有多余的空格
3. 相同指示不能使用多次
4. 对于无效的指示，当作普通的注释
5. 如果一条有效指示前面有一条无效指示，无效指示会当作注释，第二条指示由于前面有注释,也会被当作注释

#### 支持的指示
1. syntax

    在BuildKit中使用，传统的builder中将被忽略
2. escape

    用于设置换行字符，默认"\"

    ```text
    # escape=\ (backslash)
    # or
    # escape=` (backtick)
    ```

## 环境变量替换
