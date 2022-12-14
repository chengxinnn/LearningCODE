[C++11：noexcept关键字 - 简书 (jianshu.com)](https://www.jianshu.com/p/08a53d8c9670?u_atoken=86c7ba98-f703-4f07-9184-1bc9eb5a09e9&u_asession=01WBO_9-vVbNRUU9lHh1Jmmx0Jvp5GLAWTbS1wAXGHkArMUlsp0N9T7uJzxMeu4ROGX0KNBwm7Lovlpxjd_P_q4JsKWYrT3W_NKPr8w6oU7K9voD-9z8GulD7NFKjqCGb-nHmbkqVcEgdObpAroqY1_GBkFo3NEHBv0PZUm6pbxQU&u_asig=05UHfmQX1ntO6dvjZFc9Yg8PZGwWOKTgZRZe8i2DKRzkFgi6pFrhxbBHeyL_AmAH78IurMJ27Oy1TWDnWi8gp05uRJg1JGRquFNUAmasI-XofmFvUOMbU9agcEYDc0zRFwX0fIgl7mnMJ0kH6PST9M6IHvGlV-v47IQKwErdCHP_f9JS7q8ZD7Xtz2Ly-b0kmuyAKRFSVJkkdwVUnyHAIJzQB_V900CtfucUs5CLHPl-nWaxgIi_scUrXEphQ6WOm86xbSxAaWh9ph0bRUFW-6vO3h9VXwMyh6PgyDIVSG1W9El5XLLKDkLdW_k1er2YeB2t06Kw4WXjL8pvgPxmcQyrDytKXaIDwRr9d30CpMYxuLpR4V9ttyZZQE9k9xAP9lmWspDxyAEEo4kbsryBKb9Q&u_aref=1uBfs5klQZfKzrtdoJK5C9ns5Xo%3D)

### 1、介绍

C++11新标准引入的noexcept运算符，可以用于指定某个函数不抛出异常。预先知道函数不会抛出异常有助于简化调用该函数的代码，而且编译器确认函数不会抛出异常，它就能执行某些特殊的优化操作。

### 2、noexcept异常说明

（1）对于一个函数来说，noexcept说明要么出现在该函数的所有生命语句和定义语句中，要么一次也不出现。

（2）可以再函数指针的声明和定义中指定noexcept。

（3）在typedef和类型别名中不可以出现noexcept。

（4）在成员函数中，noexcept需要跟在const以及引用限定符之后，在final、override或虚函数=0之前。

对于程序违反了异常说明，编译器在编译阶段不会检查报错。

上面的程序违反了异常说明，但是编译通过，在程序执行过程中，程序会调用terminate以确保遵守不在运行时抛出异常的承诺。

旧版本的不抛出异常声明

noexcept可以接受一个可选的实参，该参数必须能转换为bool类型

### 3、noexcept运算符

noexcept运算符是一个一元运算符，它的返回值是一个bool类型的右值常量表达式，用于表示给定的表达式是否会抛出异常。

使f()和g()的异常抛出情况相同：

### 4、异常说明与指针、虚函数和拷贝控制

（1）函数指针及该指针指向的函数必须具有一致的异常说明。如果一个指针做出了不抛出异常的声明，则该指针将只能指向不抛出异常的函数。

（2）如果显示或隐式说明了指针可能抛出异常，那么该指针可以指向任何函数。

（3）如果一个虚函数承诺不会抛出异常，则后续派生出来的衍生类的虚函数也必须做出同样的承诺。反之则不需要。

（4）当编译器合成拷贝控制成员的时候，也会生成一个异常说明。如果对所有成员和基类的所有操作都承诺了不抛出异常，则合成的成员是noexcept的；如果有任意一个函数可能抛出异常，则合成的成员是noexcept(false)。

### 总结

noexcept关键字主要用于对一个函数做出不抛出异常的说明，不抛出异常可以优化函数代码，但是具体改善多少，就以后在研究了。

作者：一天不工作浑身难受
链接：https://www.jianshu.com/p/08a53d8c9670
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
