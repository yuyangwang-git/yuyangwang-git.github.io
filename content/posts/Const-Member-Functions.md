---
title: "C++ 常量成员函数"
date: 2022-07-08T08:39:32+08:00
draft: true
---

在成员函数的声明后添加`const`关键字即成为常量成员函数：

```Cpp
// <return_type> <function_name>() const;
int get_data() const;
```

## 普通指针无法绑定一个常量对象

```Cpp
const double pi = 3.14;
double *ptr = &pi;      // C2440 “初始化”: 无法从“const double *”转换为“double *”
```

这段代码会抛出一个错误，原因很简单，`pi`为常量，不允许改变`pi`的值，因此不可以用一个普通指针指向常量`pi`，正确的做法是在声明指针时使用`const`关键字：

```Cpp
const double pi = 3.14
const double *ptr = &pi;
```

这样`ptr`就成为了一个“指向常量的指针”（pointer to const）。

注，C++ Primer（中文版）一书中，将`const pointer`翻译为“常量指针”，`pointer to const`翻译为“指向常量的指针”。前面提到的普通指针则指常量指针和非常量指针：

```Cpp
const double *ptr;      // 指向常量的指针
double const *ptr;      // 指向常量的指针, 两种写法等价

// 普通指针
double * ptr;           // 非常量指针
double * const ptr;     // 常量指针
```

## this 常量指针

> 默认情况下，`this`的类型是指向类类型非常量版本的**常量指针**。
>
> By default, the type of `this` is a const pointer to the nonconst version of the class type.
>
> 例如，在`Sales_data`成员函数中，`this`的类型是`Sales_data *const`。
>
> For example, by default, the type of `this` in a `Sales_data` member function is `Sales_data *const`.
>
> 尽管 `this` 是隐式的，但它仍然需要遵循初始化规则，意味着（在默认情况下）我们不能把 `this` 绑定到一个常量对象上。
>
> Although `this` is implicit, it follows the normal initialization rules, which means that (by default) we cannot bind this to a const object.

```Cpp
class Dog
{
    public:
        std::string name = "null";

    void print_name()
    {
        std::cout << this->name << std::endl;
    }

    void print_name_const() const      // 常量成员函数
    {
        std::cout << this->name << std::endl;
    }
}

int main()
{
    const Dog dog;
    dog.print_name();               // 报错，C2662 “void Dog::print_name(void)”: 不能将“this”指针从“const Dog”转换为“Dog &”
    dog.print_name_const();         // 正确运行

    return 0;
}
```

1. `main`函数第一行中创建了一个常量对象`dog`，意味着`dog`对象的内容不允许被修改；

2. 前面提到了，默认情况下，传给成员函数的`this`是一个常量指针，即`Dog *const`，也就无法将其直接绑定到常量对象上；

3. 我们已经把`dog`声明为了常量对象，所以我们需要把`this`修改为指向常量的指针，即`const Dog *const`，才能正常调用`dog`的成员函数；

4. 但是`this`是隐式传递给成员函数的，例如`print_name()`函数的参数列表中并没有显式包含`this`，也就意味着无法直接通过函数参数修改`this`类型；

5. 那么如何把`this`修改为`const Dog *const`类型呢？C++提供了这样一个方法：在声明成员函数时，末尾添加`const`，使之成为静态成员函数，如本例的`print_name_const()`。

## 总结

传给成员函数的`this`指针，其类型为常量指针，意味着可以通过`this`修改其指向的对象的内容，但不能修改`this`指向的对象；

传给静态成员函数的`this`指针，其类型为指向常量的常量指针，意味着既不可以通过`this`修改其指向的对象的内容，也不能修改`this`指向的对象。
