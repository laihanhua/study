### 基本用法
#### 结构体匹配
一般来说，我们都会在需要校验的结构体参数加上标签，下面是一些常用的标签：

```
import (
    "fmt"

    "github.com/asaskevich/govalidator"
)

type foo struct {
    URL     string `json:"url" valid:"url,optional"`
    AID     int    `json:"aid" valid:"range(1|100)"`
    SN      string `json:"sn" valid:"maxstringlength(7)"`
    Number  string `json:"number" valid:"numeric"`
    IP      string `json:"ip" valid:"ip"`
    Oversea bool   `json:"Oversea" valid:"required"`
}

func main() {

    f := foo{
        URL:     "http:netease.com",
        AID:     101,
        SN:      "sn345678",
        Number:  "aa",
        IP:      "127.0.0.4.1",
        Oversea: false,
    }

    var err error
    result, err := govalidator.ValidateStruct(f)
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println()
    fmt.Println(result)
}
```

输出

```
Oversea: non zero value required;aid: 101 does not validate as range(1|100);ip: 127.0.0.4.1 does not validate as ip;number: aa does not validate as numeric;sn: sn345678 does not validate as maxstringlength(7);url: http:netease.com does not validate as url

false
```

foo参数均不符合的情况，err会把所有错误的参数都输出。我们根据自己项目的需求，整理一下报错信息：

```
  if err != nil {
    errs := err.(govalidator.Errors).Errors()
    for _, e := range errs {
      fmt.Println(e.Error())
    }
  }
```
输出

```
url: http:netease.com does not validate as url
aid: 101 does not validate as range(1|100)
sn: sn345678 does not validate as maxstringlength(7)
number: aa does not validate as numeric
ip: 127.0.0.4.1 does not validate as ip
Oversea: non zero value required
```

注意：

1. 参数必须是导出型，也就是必须大写字母开头，govalidator才会去理会。
2. 对于一些tag，例如`valid:"range(1|100)"`，字符串类型也是可以支持的。
3. 对于optional标签，例如`valid:"url,optional"`，表示字段非空才校验
4. 对于required标签，是传了非默认值才能通过校验；例如

	```
	type foo struct {
    Oversea bool `json:"Oversea" valid:"required"`
	}
	
	func main() {
	    f := foo{
	        Oversea: false,
	    }
	
	    var err error
	    result, err := govalidator.ValidateStruct(f)
	    if err != nil {
	        fmt.Println(err)
	    }
	    fmt.Println(result)
	}
	
	required必须要非默认值，才认为校验通过。
	Oversea: non zero value required
	false
	```

#### 函数使用
除了用结构体标签外，govalidator也可以直接使用函数来进行匹配

```
func main() {
    input := `{"key":"value"}`
    fmt.Println(govalidator.IsJSON(input))
    input = `127.0.0.1`
    fmt.Println(govalidator.IsJSON(input))
}
```
输出

```
true
false
```
大家也都猜到，使用标签也是调用对应的函数

#### 嵌套匹配
govalidator支持结构体嵌套校验的

```
type foo struct {
    Oversea bool `json:"Oversea" valid:"required"`
    F
}

type F struct {
    AID int `json:"aid" valid:"required"`
}

func main() {
    f := foo{
        Oversea: true,
    }

    var err error
    result, err := govalidator.ValidateStruct(f)
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println(result)
}
```
输出

```
F.aid: non zero value required
false
```

嵌套的结构体也必须是导出型的才可以校验。如果我们不想对嵌套结构体校验，那么可以增加`valid:"-"`标签

#### 全局强制校验
我们可以要求，对需要校验的结构体字段，都必须加上标签，从而增加代码稳定性。那么可以在main()中或者init()中，设置全局变量

```
func init() {
    govalidator.SetFieldsRequiredByDefault(true)
}

type foo struct {
    Oversea bool `json:"Oversea"`
}

func main() {
    f := foo{
        Oversea: true,
    }

    var err error
    result, err := govalidator.ValidateStruct(f)
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println(result)
}
```
输出

```
Oversea: All fields are required to at least have one validation defined
false
```

#### 自定义标签
当govalidator提供的标签不能满足我们需求时，我们还可以很方便的自定义一个标签

```
func init() {
    govalidator.CustomTypeTagMap.Set("testTag", func(i interface{}, context interface{}) bool {
        switch v := i.(type) {
        case string:
            if v == "testTag" {
                return true
            }
        }
        return false
    })
}

type foo struct {
    Input string `valid:"testTag"`
}

func main() {
    f := foo{"aaa"}
    _, err := govalidator.ValidateStruct(f)
    fmt.Println(err)

    f.Input = "testTag"
    _, err = govalidator.ValidateStruct(f)
    fmt.Println(err)
}
```
输出

```
Input: aaa does not validate as testTag
<nil>
```
 我们自定义一个testTag标签，只有满足字符串为testTag的才校验通过，通过以下设置
  
```
// after
govalidator.CustomTypeTagMap.Set("testTag", func(i interface{}, context interface{}) bool {
  // ...
})
```
其中i为输入的参数，context为整个结构体（即当校验时，我们甚至可以拿到结构体的其他参数进行一些hook处理）。

除了自定义固定标签外，我们还可以自定义函数标签

```
func init() {
	// 设置标签函数，其中str为参数值；params为标签自定义参数，例如min[7]中的7
    govalidator.ParamTagMap["min"] = govalidator.ParamValidator(func(str string, params ...string) bool {
        if len(params) == 1 {
            input, _ := govalidator.ToFloat(str)
            min, _ := govalidator.ToFloat(params[0])
            return input >= min
        }
        return false
    })
    // 设置标签的正则匹配，通过该正则govalidator才能找到正确的min标签
    govalidator.ParamTagRegexMap["min"] = regexp.MustCompile("^min\\((\\d+)\\)$")
}

type foo struct {
    Number int `valid:"min(7)"`
}

func main() {

    f := foo{6}

    var err error
    result, err := govalidator.ValidateStruct(f)
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println(result)
}
```
输出

```
Number: 6 does not validate as min(7)
false
```


### 注意事项
#### 零值问题
如果参数值为默认值，但是不带`required`标签，govalidator默认会放过（即requried标签的优先级最高），例如：

```
type foo struct {
    Number int `valid:"min(7)"`
    Range  int `valid:"range(1|10)"`
}

func main() {

    f := foo{0, 0}
    var err error
    result, err := govalidator.ValidateStruct(f)
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println(result)
}
```
当Number,Range=0,0这样是能够通过校验的，这显然是和我们的期望不一样。这时候就需要加上`required`标签才行。

```
type foo struct {
    Number int `valid:"min(7),required"`
    Range  int `valid:"range(1|10),required"`
}

结构体改成上述后，则报错；符合预期
Number: non zero value required;Range: non zero value required
```

#### 错误问题
govalidator是返回(bool,error)(是否通过校验，错误)，一般情况下，当返回false时，err是不为空的。我们贪图方便可能会这样写代码：

```
    ok, err := govalidator.ValidateStruct(f)
    if !ok || err != nil {
        fmt.Printf("param: %s fail\n", strings.Split(err.Error(), ":")[0])
    }
```

不过官网在函数描述时这样写道：

```
// ValidateStruct use tags for fields.
// result will be equal to `false` if there are any errors.
// todo currently there is no guarantee that errors will be returned in predictable order (tests may to fail)
func ValidateStruct(s interface{}) (bool, error){}
```
即也有可能返回false,nil的组合，这样我们在调用err.Error()时就会panic。不巧我就遇到过出现false,nil的情况。

```
type foo struct {
    Range int `valid:"range(1|10),required"`

    Log mylog.MyLogger
}

func main() {

    f := foo{Range: 5}
    var err error
    ok, err := govalidator.ValidateStruct(f)
    if !ok || err != nil {
        fmt.Printf("param: %s fail\n", strings.Split(err.Error(), ":")[0])
    }
}
```
报错

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x18 pc=0x116445e]

goroutine 1 [running]:
main.main()
	/Users/lhh/gopath/src/test/test_govalidator.go:23 +0x7e
exit status 2
```
程序的本意只是想校验下Range参数的，不过却出现了panic。明显就是err.Error()的问题了。设置一下Log看看

```
    f := foo{Range: 5, Log: mylog.New("DEBUG", nil, 1)}
    var err error
    ok, err := govalidator.ValidateStruct(f)
    if !ok || err != nil {
        fmt.Println(ok)
        fmt.Println(err)
    }
    
```
输出

```
false
validator: unsupported type: logrus.LevelHooks
```
和普通的参数报错不一样，提示validator不支持logrus类型。

如果是导出类型，不管有没有设置valid标签，该字段都会进入validator的校验函数

```
func typeCheck(v reflect.Value, t reflect.StructField, o reflect.Value, options tagOptionsMap) (isValid bool, resultErr error) {
    if !v.IsValid() {
        return false, nil
    }
    ...
}
```
在typeCheck中，v.IsValid()中就返回false,nil了。**所以要避免这个问题，在校验的结构体中，非参数的类型最好使用非导出变量**

### 总结
暂时govalidator初体验如上，后续使用该包有其他的心得或者发现有其他问题，也会同步到此km文档。

### 参考文档
1. https://github.com/asaskevich/govalidator
