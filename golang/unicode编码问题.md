## 由emoji引发的编码问题

章鱼哥系统接口需要返回玩家中文名字的unicode码，由于有些玩家过于调皮，名字中带有💀emoji字符，这会导致我们自定义后的json.Marshal编码错误，下面就来探讨一下Unicode的一些编码问题以及golang内的处理。

## 探究unicode、utf-8、utf-16

### 简要介绍
大家可能已经对这三种编码方式比较熟悉了，我就简单总结一下：

Unicode只是一个统一的符号集，规定了符号的二进制代码，并没有规定如何去存储；Unicode的编码空间从U+0000到U+10FFFF。从U+xx0000到U+xxFFFF其中可以包含17个平面码位（xx可以表示00-FF），正常来说我们使用基本平面（基础字段）就够了，即U+0000到U+FFFF。其余的为辅助扩展字段。

UTF-8是Unicode的一种实现，golang内的编码也是UTF-8编码。它是一种变字节的编码方式，可以使用1～4个字节代码一个符号，一般来说字母一个字节就可以表示了，普通的中文也不会超过3个字节。不过
💀字符就使用了4个字节。（咦～上面不是说Unicode基本字段U+0000到U+FFFF，应该是2个字节呀？莫急，后面有解释）

UTF-16也是Unicode的编码实现，它规定使用2个字节或4个字节来；其实在2个字节的情况，UTF-16编码的16进制数就是对应unicode码点；而在4字节的情况，需要使用代理对(surrogate pair)来表示一个Unicode码点；

### 编码实现
其实UTF是“Unicode Transformation Format”的缩写，刚刚说了，Unicode码点是全球统一的，那么具体UTF-8/UTF-16是如何编码的呢？拿💀字符为例说明，其中该字符的Unicode码点为U+1F480，参考戳[这里](https://apps.timwhitlock.info/unicode/inspect/hex/1F480#block-U1F300)

#### UTF-8
我们就通过一个表格来看下将Unicode字符转换为UTF-8编码方式的具体步骤:

|Unicode码点范围|UTF-8编码方式
|---|---|
|U+0000~U+007F|0xxxxxx
|U+0080~U+07FF|110xxxxx 10xxxxxx
|U+0800~U+FFFF|1110xxxx 10xxxxxx 10xxxxxx
|U+10000~U+10FFFF|11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
从表中可以看到，由于UTF-8的编码实现方式，所以0800~FFFF看似两个字节，实际存储时使用的是三个字节。那么字符💀的码点为U+1F480，则使用四个字节存储。

1F480的二进制为11111 010010 000000，根据表格中从左往右的填充规则，那么得到的UTF-8编码为

```
1111 0000 =F0
1001 1111 =9F
1001 0010 =92
1000 0000 =80
```

#### UTF-16
同样我们也通过表格来看下编码方式

|Unicode码点范围|UTF-16编码方式
|---|---|
|U+0000~U+FFFF|2 Byte存储，编码后等于Unicode值
|U+10000~U+10FFFF|4 Byte存储，现将Unicode值减去（0x10000），得到20bit长的值。再将Unicode分为高10位和低10位。UTF-16编码的高位是2 Byte，高10位Unicode范围为0-0x3FF，将Unicode值加上0XD800，得到高位代理（或称为前导代理，存储高位）；低位也是2 Byte，低十位Unicode范围一样为0~0x3FF，将Unicode值加上0xDC00,得到低位代理（或称为后尾代理，存储低位）

对于两个字节的情况就比较简单粗暴，存储值就是对应的码点；还有大头（FE FF）/小头(FF FE)方式存储，存储的顺序也不一样，不过编码规则是一样的。

因为1F480>FFFF，所以还是用4字节存储。

```
1. 将Unicode值减去0x1F480-0x10000得到0x0F480=0000 1111 0100 1000 0000
2. 高10位和低10位
	0000111101=0x003D
	0010000000=0x0080
3. 高10位Unicode值加上0xD800得到高位代理：0xD83D
4. 低10位Unicode值加上0xDC00得到低位代理：0xDC80
```
得到UTF-16的代理对位0xD830/0xDC80，根据小头方式排列的UTF-16 LE编码为3D D8 80 DC

#### golang验证

```
package main

import (
    "fmt"
    "strconv"
    "unicode/utf16"
)

func main() {
    str := "💀"
    fmt.Println("Unicode码点:")
    fmt.Printf("%s\n", strconv.QuoteToASCII(str))
    fmt.Println("utf-8编码:")
    fmt.Printf("%x\n", str) //默认以utf-8存储，%x输出16进制
    fmt.Println("utf-16编码:")
    hPair, lPair := utf16.EncodeRune('💀') //把对应字符(rune类型)转成utf-16代理对
    fmt.Printf("%x %x\n", hPair, lPair)

}

lhh@laihanhuadeMacBook-Air  ~/gopath/src/test  go run test_unicodekm.go
Unicode码点:
"\U0001f480"
utf-8编码:
f09f9280
utf-16编码:
d83d dc80
```

## 实际问题

### 输出编码
我们为了能够输出编码成Unicode，我们写了如下程序：

```
type QuoteString string

func (q QuoteString) MarshalJSON() ([]byte, error) {
    return []byte(strconv.QuoteToASCII(string(q))), nil
}

type Test struct {
    Name QuoteString `json:"name"`
}

func main() {
    out := Test{Name: "编码"}
    b, err := json.Marshal(out)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(string(b))
}

 lhh@laihanhuadeMacBook-Air  ~/gopath/src/test  go run test_unicodekm.go
{"name":"\u7f16\u7801"}
```
可以看出，我们能够把“编码”字符串输出成Unicode编码，满足我们的要求。一般来说对于基本字符来说是没问题的，但是对于一些拓展特殊字符呢？还是拿emjoy表情💀为例试一下

```
func main() {
    out := Test{Name: "💀"}
    b, err := json.Marshal(out)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Println(string(b))
}

lhh@laihanhuadeMacBook-Air  ~/gopath/src/test  go run test_unicodekm.go
json: error calling MarshalJSON for type main.QuoteString: invalid character 'U' in string escape code
```
那么就报错了，所以上面的代码还不能适用于所有的字符。实际上，对于Unicode扩展字符(unicode escape)，即码点>U+FFFF的字符，上述json.Marshal时候就会报错。在[规范中](https://tools.ietf.org/html/rfc7159#section-7)说明，对于unicode escape字符，可以使用上面说的surrogate pair来表示。修改一下MarshalJSON()代码：

```
func (q QuoteString) MarshalJSON() ([]byte, error) {
    var result = []byte("\"")
    for _, r := range []rune(q) {
        if r < 0x10000 {
            v := "\\u" + strconv.FormatInt(int64(r), 16)
            result = append(result, []byte(v)...)
            continue
        }
        r1, r2 := utf16.EncodeRune(r)
        v1 := "\\u" + strconv.FormatInt(int64(r1), 16)
        v2 := "\\u" + strconv.FormatInt(int64(r2), 16)
        result = append(append(result, []byte(v1)...), []byte(v2)...)
    }

    result = append(result, []byte("\"")...)
    return result, nil

}

lhh@laihanhuadeMacBook-Air  ~/gopath/src/test  go run test_unicodekm.go
{"name":"\ud83d\udc80"}
```
可以看到，能够正常输出💀的代理对surrogate pair。这个代理对也能够被正常的识别到为💀字符。小伙伴可以使用\ud83d\udc80在浏览器上unicode转码测试下。

### string循环处理
从刚刚代码中我们可以延伸出一个问题，平时我们对于字符串循环处理时，如果遇到有中文是怎么样的情况呢？拿“i 是 emoji💀”来测试一下。

一般我们循环有len(str)和range两种，我们来对比下这两种区别

```
func main() {
    s := "i 是 emoji💀"
    for i := 0; i < len(s); i++ {
        fmt.Printf("% x", s[i])
    }
    fmt.Println()
    for _, v := range s {
        fmt.Printf("% x", v)
    }
    fmt.Println()
}

 lhh@laihanhuadeMacBook-Air  ~/gopath/src/test  go run test_unicodekm.go
 69 20 e6 98 af 20 65 6d 6f 6a 69 f0 9f 92 80
 69 20 662f 20 65 6d 6f 6a 69 1f480
```
在for len(s)中，遍历的次数明显比range s多。这是因为对于len(s)的情况是遍历字符串中的字节（使用下标访问），然后range s则是遍历字符串中的字符（fmt.("%x")如果接受的对象是字符，即rune类型，此时输出的是十六进制Unicode码点）。大家可以看下具体的输出，“是”在UTF-8中存储三个字节，即e6 98 af；而💀存储4个字节，即“f0 9f 92 80”，这样就对得上了。

## 总结

由于章鱼哥系统处理的字符情况比较多，也遇到了不少的编码问题。特别是对于中文、特殊字符的情况，我们要牢记golang中的编码都是UTF-8，从而来解决自己的问题。如果上面有说得不对，或者有更好的解决方案，欢迎讨论。

## 参考
https://tools.ietf.org/html/rfc7159#section-7
https://apps.timwhitlock.info/unicode/inspect/hex/1F480#block-U1F300
https://cloud.tencent.com/developer/article/1341908
https://github.com/golang/go/issues/39137
https://zh.wikipedia.org/wiki/UTF-16
https://studygolang.com/articles/439
