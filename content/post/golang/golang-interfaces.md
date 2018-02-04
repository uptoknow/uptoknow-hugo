+++
title = "Golang Interfaces"
date = 2018-02-04T15:31:04+08:00
tags = ["blog", "uptoknow", "golang"]
categories = ["golang"]
+++

Interface make more sense when you make the code more flexible, scalable, and maintainable, interface in go is a different than other languages like java.


* In java we need use `implement` to say that type implements an interface, in golang, Every type that
implement an interface automatically satisfies that interface, it's automatically deteced by Go compiler.
```golang
type I interface{
  func M()
}
type Foo struct{}
func (f *foo) M(){}
```
We declare an interface I that have an func M(), and the Foo struct have an `M()` func, so we can treat Foo is and implementation of interface I

* nil interface value. Interface type have two components, dynamic type and dynamic value. **Interface type value is nil if both it's type and value is nil, this is intresting, let's see an example**
```golang
type T struct {}
func (T) M() {}
func main() {
    var t *T
    if t == nil {
        fmt.Println("t is nil")
    } else {
        fmt.Println("t is not nil")
    }
    var i I = t
    if i == nil {
        fmt.Println("i is nil")
    } else {
        fmt.Println("i is not nil")
    }
}
```
Output:
```
t is nil
i is not nil
```
As we can see t is nil, because it has nil type and nil value, when we assign t to i, the dynamic type of i becomes T, and it's value is nil, so now i is not nil, **So let's say, a interface variable points to an nil interface variable, it's not nil.**

* Empty interface. An interface that dont' have any func defined, we can call it's a empty interface. Empty interface is automatically satisfied by any type, so any type can be assigned to such interface type variable.

* Type assertion. Golang has assignability rules that allow assin to a variable value with different type. See:
```golang
type T struct {
    name string
}
func main() {
    v1 := struct{ name string }{"foo"}
    fmt.Printf("%T\n", v1) // struct { name string }
    var v2 T
    v2 = v1
    fmt.Printf("%T\n", v2) // main.T
}
```
The syntax of type assertion expression is as follows:
```
v.(T)
```
where v is of interface type and T is either abstract or concrete type.
 This is an example of **Concrete Type**
```golang
type I interface {
    M()
}
type T struct{}
func (T) M() {}
func main() {
    var v1 I = T{}
    v2 := v1.(T)
    fmt.Printf("%T\n", v2) // main.T
}
```
This is an example of **Interface Type**:
```golang
type I1 interface {
    M()
}
type I2 interface {
    I1
    N()
}
type T struct{
    name string
}
func (T) M() {}
func (T) N() {}
func main() {
    var v1 I1 = T{"foo"}
    var v2 I2
    v2, ok := v1.(I2)
    fmt.Printf("%T %v %v\n", v2, v2, ok) // main.T {foo} true
}
```
Special case: When v is nil, then type assertion always fails with panic, no matter `T` is an interface type of concret type.

* Type switch:
Type assertion is a method to do a sigle check, if the code needs do do multiple such check against a single variable, we can use type switch.
```golang
type I1 interface {
    M1()
}
type T1 struct{}
func (T1) M1() {}
type T2 struct{}
func (T2) M1() {}
func main() {
    var v I1 = T2{}
    switch v.(type) {
    case nil:
            fmt.Println("nil")
    case T1, T2:
            fmt.Println("T1 or T2")
    }
}
```



Ref:
[Interfaces in Go (part I)](https://medium.com/golangspec/interfaces-in-go-part-i-4ae53a97479c).
[Interfaces in Go (part II)](https://medium.com/golangspec/interfaces-in-go-part-ii-d5057ffdb0a6).
[Interfaces in Go (part III)](https://medium.com/golangspec/interfaces-in-go-part-iii-61f5e7c52fb5).

