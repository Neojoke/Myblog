---
title: Python-命令行参数解析库Argparse使用指南
date:  2017-12-20 09:39:00
categories:
- 术-技术综合实践
tags:
- iOS
---

![](http://upload-images.jianshu.io/upload_images/24274-daa175226f7dce21.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Python 作为网红脚本语言，不少开发者都会选择它来完成一些日常工作，提高效率，特别是在 Python2.4 以后推出的 subprogress 模块后，Python 与操作系统的交互更加自然，其作为命令行工具开发语言的便利程度直线上升。开发命令行工具，解析命令行参数是一个基本功能，Python 中有 argparse（旧的库为 optparse，已经停止维护）专门解析命令行参数，使用起来非常遍历。

## 基本使用

对于一个命令行工具来说，使用 argparse 实例化一个解析器来接收和处理命令行参数。

```
import argparse
parser = argparse.ArgumentParser()
parser.parse_args()
```

导入 argparse 包，实例化一个 parser 解析器，开始解析。运行结果如下：

```
$ python3 prog.py
//没有任何结果
$ python3 prog.py --help
//显示出usage
usage: prog.py [-h]

optional arguments:
  -h, --help  show this help message and exit

$ python3 prog.py --verbose
usage: prog.py [-h]
prog.py: error: unrecognized arguments: --verbose
//不能识别--verbose这样的参数
$ python3 prog.py foo
usage: prog.py [-h]
prog.py: error: unrecognized arguments: foo
// 不能识别foo这样的位置参数
```

使用 argparse 来实例化一个解析器以后，会自动增加—help 对应的 usage，未定义的参数都会有友好的提示。
现在增加一个位置参数：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("echo", help="echo the string you use here")
args = parser.parse_args()
print(args.echo)
```

增加一个 echo 的位置参数，同时使用 help 参数来表明该参数的说明。
运行：

```
$ python3 prog.py -h
usage: prog.py [-h] echo

positional arguments:
  echo        echo the string you use here

optional arguments:
  -h, --help  show this help message and exit

$ python3 prog.py echo
echo
```

usage 出现 echo 参数的说明。 当传入 echo 作为位置参数后，print(args.echo)则打印出 echo 对应的值为字符串 echo。
还可以对参数类型进行设置：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("square", help="display a square of a given number",
                    type=int)
args = parser.parse_args()
print(args.square**2)
```

运行：

```
$ python3 prog.py 4
16
$ python3 prog.py four
usage: prog.py [-h] square
prog.py: error: argument square: invalid int value: 'four'
```

设置类型后，如果出现类型出错则会报错。

## 可选参数

可选参数往往参数名称前加—，如下：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--verbosity", help="increase output verbosity")
args = parser.parse_args()
if args.verbosity:
    print("verbosity turned on")
```

运行：

```
$ python3 prog.py --verbosity 1
verbosity turned on
$ python3 prog.py
$ python3 prog.py --help
usage: prog.py [-h] [--verbosity VERBOSITY]

optional arguments:
  -h, --help            show this help message and exit
  --verbosity VERBOSITY
                        increase output verbosity
$ python3 prog.py --verbosity
usage: prog.py [-h] [--verbosity VERBOSITY]
prog.py: error: argument --verbosity: expected one argument
```

如果使用了可选参数，但是没有传入对应的值，则会报错，告知—verbosity 期望得到一个参数。对于上面的例子，if 判断语句其实只需要 verbosity 的值为布尔值，则如下设置 action:

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--verbose", help="increase output verbosity",
                    action="store_true")
args = parser.parse_args()
if args.verbose:
    print("verbosity turned on")
```

运行：

```
$ python3 prog.py --verbose
verbosity turned on
$ python3 prog.py --verbose 1
usage: prog.py [-h] [--verbose]
prog.py: error: unrecognized arguments: 1
$ python3 prog.py --help
usage: prog.py [-h] [--verbose]

optional arguments:
  -h, --help  show this help message and exit
  --verbose   increase output verbosity
```

此时，填写了 verbose 可选参数以后，if 条件通过，当指定 verbose 的 action 为“store_true”的时候，args.verbose 则为 true，没有则为 false。

### 短可选参数

—verbose 一般为完整的可选参数填写方式，命令行工具中有一个-加上一个字母的短可选参数的表示，可以像下面这样设置：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("-v", "--verbose", help="increase output verbosity",
                    action="store_true")
args = parser.parse_args()
if args.verbose:
    print("verbosity turned on")
```

运行：

```
$ python3 prog.py -v
verbosity turned on
$ python3 prog.py --help
usage: prog.py [-h] [-v]

optional arguments:
  -h, --help     show this help message and exit
  -v, --verbose  increase output verbosity
```

可以看到 usage 中显示，-v 与—verbose 为等效的。

## 同时设置位置参数与可选参数

一般的命令行工具则会接收多种参数，多种类型参数的设置如下：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("square", type=int,
                    help="display a square of a given number")
parser.add_argument("-v", "--verbose", action="store_true",
                    help="increase output verbosity")
args = parser.parse_args()
answer = args.square**2
if args.verbose:
    print("the square of {} equals {}".format(args.square, answer))
else:
    print(answer)
```

运行：

```
$ python3 prog.py
usage: prog.py [-h] [-v] square
prog.py: error: the following arguments are required: square
$ python3 prog.py 4
16
$ python3 prog.py 4 --verbose
the square of 4 equals 16
$ python3 prog.py --verbose 4
the square of 4 equals 16
```

可以变化可选参数的位置，也可以正常运行。
可以设置可选参数的为几种具体的值，以避免程序出现 bug，如下：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("square", type=int,
                    help="display a square of a given number")
parser.add_argument("-v", "--verbosity", type=int, choices=[0, 1, 2],
                    help="increase output verbosity")
args = parser.parse_args()
answer = args.square**2
if args.verbosity == 2:
    print("the square of {} equals {}".format(args.square, answer))
elif args.verbosity == 1:
    print("{}^2 == {}".format(args.square, answer))
else:
    print(answer)
```

choices 可以设置参数为哪些具体的值。
运行如下：

```
$ python3 prog.py 4 -v 3
usage: prog.py [-h] [-v {0,1,2}] square
prog.py: error: argument -v/--verbosity: invalid choice: 3 (choose from 0, 1, 2)
$ python3 prog.py 4 -h
usage: prog.py [-h] [-v {0,1,2}] square

positional arguments:
  square                display a square of a given number

optional arguments:
  -h, --help            show this help message and exit
  -v {0,1,2}, --verbosity {0,1,2}
                        increase output verbosity
```

usage 同时会出现参数可以填写的值有哪些。
也可以处理一个可选参数出现多次的情况，以避免参数过于冗长的情况，这种情况很常见：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("square", type=int,
                    help="display the square of a given number")
parser.add_argument("-v", "--verbosity", action="count",
                    help="increase output verbosity")
args = parser.parse_args()
answer = args.square**2
if args.verbosity == 2:
    print("the square of {} equals {}".format(args.square, answer))
elif args.verbosity == 1:
    print("{}^2 == {}".format(args.square, answer))
else:
    print(answer)
```

action 的另一种类型为”count”，可以计算某个可选参数出现的次数。
运行：

```
$ python3 prog.py 4
16
$ python3 prog.py 4 -v
4^2 == 16
$ python3 prog.py 4 -vv
the square of 4 equals 16
$ python3 prog.py 4 --verbosity --verbosity
the square of 4 equals 16
$ python3 prog.py 4 -v 1
usage: prog.py [-h] [-v] square
prog.py: error: unrecognized arguments: 1
$ python3 prog.py 4 -h
usage: prog.py [-h] [-v] square

positional arguments:
  square           display a square of a given number

optional arguments:
  -h, --help       show this help message and exit
  -v, --verbosity  increase output verbosity
$ python3 prog.py 4 -vvv
16
```

参数可以设置默认值，如下：

```
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("square", type=int,
                    help="display a square of a given number")
parser.add_argument("-v", "--verbosity", action="count", default=0,
                    help="increase output verbosity")
args = parser.parse_args()
answer = args.square**2
if args.verbosity >= 2:
    print("the square of {} equals {}".format(args.square, answer))
elif args.verbosity >= 1:
    print("{}^2 == {}".format(args.square, answer))
else:
    print(answer)
```

设置默认值以后，verbosity 总会有值，且默认为 0。
运行：

```
$ python3 prog.py 4
16
```

## add_argument

函数原型为：

```
ArgumentParser.add_argument(name or flags...[, action][, nargs][, const][, default][, type][, choices][, required][, help][, metavar][, dest])
```

该方法定义单个命令行参数，参数说明如下：

- name or flags:任一一个名称或者是可选参数的 list，例如：foo 或 -f, —foo。
- action:获取的参数该采取何种提取方式。
- nargs: 提取的命令行参数对应的值的个数。
- const: 如果命令行没有传入该参数，const 指定该参数的默认值，在某些 action 与 nargs 类型下才起作用。
- default: 命令行中如果缺少该参数，default 来提供值。
- type: 要转换命令行参数的类型。
- choice: 参数允许的值。
- required ： 表明参数是否是必须的，只对可选参数有效。
- help: 对该参数的简要说明。
- metavar ：用于指明在帮助说明中，该参数有意义的值是多少。
- dest: 用于指明该命令行参数在获取参数时候的属性是什么。
