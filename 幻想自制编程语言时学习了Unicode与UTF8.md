# 幻想自制编程语言时学习了Unicode与UTF8

> 幻想自制我的 Coral-Lang 的过程中，由于是用 Go 写的，我一直想把这个语言做成像是 Go 那样的原生支持 UTF8 的。然而由于有些基础知识没有打牢实，写起来有些码不实在。
>
> 而现在我已在单元测试中实现了在字符串读入时，可含有转义字符、Unicode 转义两个功能

![image.png](https://blog.shenqingchuan.top/upload/2020/05/image-931ffeb9d29b46a9bf0d884a36cc8b76.png)

## Unicode 和 UTF8 的区别

简单来说:

- Unicode 是「字符集」
- UTF-8 是「编码规则」

其中：

- 字符集：为每一个「字符」分配一个唯一的ID，学名叫做码位/码点/Code Point
- 编码规则：将「码位」转换位字节序列的规则（即对应一种编码/解码过程）

广义的 Unicode 是一个标准，定义了一个字符集以及一系列的编码规则，即 Unicode 字符集和 UTF-8、UTF-16、UTF-32 等...

Unicode 字符集为每一个字符分配一个码位，例如 “ **知** ” 这个字的码位是 30693（记作：U+77E5，即 30693 的十六进制）

对于 Unicode 来说，每一个字符对应一个十六进制数字。

而计算机只懂二进制，因此，严格按照 Unicode 的方式(UCS-2)，应该这样存储：
```txt
      7    7    e    5
知： 0111 0111 1110 0101
```

这个字符串总共占用了 18 个字节，但是对比中英文的二进制码，可以发现，英文前 9 位都是 `0`！浪费啊！浪费硬盘，浪费流量。

而 UTF-8 是这样做的：

1. 单字节的字符，字节的第一位设为`0`，对于英语文本，UTF-8 码只占用一个字节，和 ASCII 码完全相同；
2. `n` 个字节的字符(`n>1`)，第一个字节的前 `n` 位设为 `1`，第 `n+1` 位设为 `0`，后面字节的前两位都设为 `10`。

    这 `n` 个字节的其余空位填充该字符 Unicode 码，高位用 `0` 补足。这样就形成了如下的 UTF-8 标记位：

```txt
U+0000  - U+  007F: 0xxxxxxx
U+0080  - U+  007F: 110xxxxx 10xxxxxx
U+0800  - U+  007F: 1110xxxx 10xxxxxx 10xxxxxx
U+10000 - U+10FFFF: 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

根据上表中的编码规则，之前的 “ **知** ” 字的码位 U+77E5 属于第三行的范围：

```txt
规则 1110xxxx 10xxxxxx 10xxxxxx

知： 11100111 10011111  10100101
    ^^^^     ^^        ^^
```

## 编译原理 - 词法分析中的实现：

假如我们的语言中有如下的一段：

```
var str = "我就是\t想装个逼：\u77e5道unicode是这样的"
```

那么很明显其内容中除了我们人眼可见可识别出的汉字、英文字母外，还应该保证 **制表符也有效果、`\u77e5` 变为 `知` 字**

```go
// 读出一个字符串，含转义字符的处理
func (lexer *Lexer) ReadString() (*Token, *CoralError) {
	var str string
	lexer.GoNextChar() // 移过当前的 '"' 双引号

	for !lexer.PeekChar().MatchRune('"') {
		if lexer.PeekChar().MatchRune('\\') { // 可能遇到转义字符
			switch lexer.PeekNextChar(lexer.PeekChar().ByteLength).Rune {
			case 'a':
				str += "\a"
				lexer.GoNextCharByStep(2)
			case 'b':
				str += "\b"
				lexer.GoNextCharByStep(2)
			case 't':
				str += "\t"
				lexer.GoNextCharByStep(2)
			case 'v':
				str += "\v"
				lexer.GoNextCharByStep(2)
			case 'n':
				str += "\n"
				lexer.GoNextCharByStep(2)
			case 'r':
				str += "\r"
				lexer.GoNextCharByStep(2)
			case 'f':
				str += "\f"
				lexer.GoNextCharByStep(2)
			case '"':
				str += "\""
				lexer.GoNextCharByStep(2)
			case 'u':
				// Unicode 需要是：\uXXXX 格式：
				lexer.GoNextCharByStep(2) // 移过当前的 '\u'
				unicodeBitCount := 0
				sUnicode := ""
				for lexer.PeekChar().IsLegalHexadecimal() {
					unicodeBitCount++
					sUnicode += string(lexer.PeekChar().Rune)
					lexer.GoNextChar()
				}
				if unicodeBitCount != 4 {
					// 说明不满 4 位，解码出错
					return nil, NewCoralError("Syntax", "(unicode error) 'unicodeEscape' codec can't decode bytes in position 0-3: truncated \\uXXXX escape", LexUnicodeEscapeFormatError)
				}
				gotUTF8Decoded := utils.UnicodeToUTF8(sUnicode)
				str += gotUTF8Decoded
			}
		} else {
			// 正常添加字符
			str += string(lexer.PeekChar().Rune)
			lexer.GoNextChar()
		}
	}

	return lexer.makeToken(TokenTypeString, str), nil
}
```

也就是说，我们需要把源代码字符串字面量中的 `\t` 交给 Go 当作一个制表字符来处理，而针对四位的十六进制数转为一个 UTF-8 字符我们需要额外的处理：

```go
func UnicodeToUTF8(source string) string {
	res := []string{""}
	// 无论你是 \uXXXX 还是没有前缀的 XXXX 都可
	sUnicode := strings.Split(source, "\\u")  // 切分出四位 unicode 的编码
	context := ""
	for _, v := range sUnicode {
		additional := ""
		if len(v) < 1 {
			continue
		}
		if len(v) > 4 {  // 长度大于 4
			rs := []rune(v)
			v = string(rs[:4])  // 切分出前 4 位
			additional = string(rs[4:]) // 后面的当作正常字符处理
		}
		temp, err := strconv.ParseInt(v, 16, 32) // <- 32bit 即允许四个字节整数
		if err != nil {
			context += v
		}
		context += fmt.Sprintf("%c", temp) // 使用 Go 原生支持的 UTF-8 的 %c 来输出该字符
		context += additional // 添加上余下多出的一些正常字符
	}
	res = append(res, context)
	return strings.Join(res, "")
}
```