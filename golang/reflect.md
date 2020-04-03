## 关于反射
   - 首先，任何接口都可以分为<Value,Type>两种类型，在反射包中，用reflect.Value和reflect.Type表示，可以用reflect.ValueOf(),reflect.TypeOf()来获取
   - 一个Value对象value_1，可以通过value_1.interface()获取到真实的值，并且通过断言来进行类型转换value_1.interface().(具体类型),实现了逆向转换
   - 可以通过Kind()获取到底层的类型，例如接口、结构体、指针这些。
   - reflect.Value是通过reflect.ValueOf(X)获得的，只有当X是指针的时候，才可以通过reflec.Value修改实际变量X的值，即：要修改反射类型的对象就一定要保证其值是“addressable”的。如果满足要求，可以通过SetXXX()来修改值
   - Values.Elem()返回接口的值或者是指针所指向的值（Value的Kind()必须是interface或者Ptr）
   - 类型分为静态类型和具体类型，例如：`type MyInt int// var i int //var j MyInt`,变量 i 的类型是 int，j 的类型是 MyInt。 所以，尽管变量 i 和 j 具有共同的底层类型 int，但它们的静态类型并不一样。不经过类型转换直接相互赋值时，编译器会报错。而TypeOf返回的，是具体类型。
   - https://juejin.im/post/5a75a4fb5188257a82110544

