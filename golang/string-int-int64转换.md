```
#string到int
int,err:=strconv.Atoi(string)
#int到string
string:=strconv.Itoa(int)

#string到int64
int64, err := strconv.ParseInt(string, 10, 64)
#int64到string
string:=strconv.FormatInt(int64,10)
```