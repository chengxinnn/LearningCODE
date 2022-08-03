参考[https://blog.csdn.net/sinat_22991367/article/details/73695039](https://blog.csdn.net/sinat_22991367/article/details/73695039)

# 一、基本概念

_declspec(dllexport)与_declspec(dllimport)都是DLL内的关键字，即导出与导入。他们是将DLL内部的类与函数以及数据导出与导入时使用的。

主要区别在于：dllexport是在这些类、函数以 及数据的申明的时候使用。用过表明这些东西可以被外部函数使用，即（dllexport）是把DLL中的相关代码（类，函数，数据）暴露出来为其他应用程 序使用。而 dllimport关键字是在外部程序需要使用DLL内相关内容时使用的关键字。

# 二、初步理解

考虑下面的需求，使用一个方法，一个是提供者，一个是使用者，二者之间的接口是头文件。头文件中声明了方法，在提供者那里方法应该被声明为__declspec(dllexport)，在使用者那里，方法应该被声明为__declspec(dllimport)。二者使用同一个头文件，作为接口，怎么办呢？

使用条件编译：定义一个变量，针对提供者和使用者，设置不同的值。

```cpp
#ifndef DLL_H_
#define DLL_H_

#ifdef DLLProvider
#define DLL_EXPORT_IMPORT __declspec(dllexport)
#else
#define DLL_EXPORT_IMPORT __declspec(dllimport)
#endif

DLL_EXPORT_IMPORT int add(int ,int);

#endif
```

以上解释似乎可以说明_declspec(dllimport)存在的意义，但是在实际工程过程中慢慢发现，不用_declspec(dllimport)，我们也可以将动态链接库进行导入使用，那么_declspec(dllimport)的作用究竟是什么？？


# 三、深入剖析

先来看看MSDN里面的解释：
不使用 __declspec(dllimport) 也能正确编译代码，但使用 __declspec(dllimport) 使编译器可以生成更好的代码。编译器之所以能够生成更好的代码，是因为它可以确定函数是否存在于 DLL 中，这使得编译器可以生成跳过间接寻址级别的代码，而这些代码通常会出现在跨 DLL 边界的函数调用中。但是，必须使用 __declspec(dllimport) 才能导入 DLL 中使用的变量。
初看起来，这段话前面的意思是，不用它也可以正常使用DLL的导出库，但最后一句话又说，必须使用 __declspec(dllimport) 才能导入 DLL 中使用的变量这个是什么意思？？
试验一下，假定，你在DLL里只导出一个简单的类，注意，我假定你已经在项目属性中定义了 SIMPLEDLL_EXPORT
//SimpleDLLClass.h

#ifdef SIMPLEDLL_EXPORT
#define DLL_EXPORT __declspec(dllexport)
#else
#define DLL_EXPORT
#endif

class DLL_EXPORT SimpleDLLClass
{
  public:
    SimpleDLLClass();
    virtual ~SimpleDLLClass();

    virtual getValue() { return m_nValue;};
  private:
    int m_nValue;
};

//SimpleDLLClass.cpp

#include "SimpleDLLClass.h"

SimpleDLLClass::SimpleDLLClass()
{
   m_nValue=0;
}

SimpleDLLClass::~SimpleDLLClass()
{
}

然后你再使用这个DLL类，在你的APP中include SimpleDLLClass.h时，你的APP的项目不用定义 SIMPLEDLL_EXPORT 所以，DLL_EXPORT 就不会存在了，这个时候，你在APP中，不会遇到问题。这正好对应MSDN上说的__declspec(dllimport)定义与否都可以正常使用。但我们也没有遇到变量不能正常使用呀。
那好，1）我们改一下SimpleDLLClass,把它的m_nValue改成static。

    2）然后在cpp文件中加一行：

int SimpleDLLClass::m_nValue=0;
 改完之后，再去LINK一下，你的APP，看结果如何，结果是LINK告诉你找不到这个m_nValue。明明已经定义了，为什么又没有了？？肯定是因为我把m_nValue定义为static的原因。但如果我一定要使用Singleton的Design Pattern的话，那这个类肯定是要有一个静态成员，每次LINK都没有，那不是完了？ 如果你有Platform SDK，用里面的Depend程序看一下，DLL中又的确是有这个m_nValue导出的呀。
再回去看看我引用MSDN的那段话的最后一句。 那我们再改一下SimpleDLLClass.h，把那段改成下面的样子:

#ifdef SIMPLEDLL_EXPORT
#define DLL_EXPORT __declspec(dllexport)
#else
#define DLL_EXPORT __declspec(dllimport)
#endif
再LINK，一切正常。原来dllimport是为了更好的处理类中的静态成员变量的，如果没有静态成员变量，那么这个__declspec(dllimport)无所谓。

# 四、总结

 _declspec(dllexport)与_declspec(dllimport)是相互呼应，只有在DLL内部用dllexport作了声明，才能 在外部函数中用dllimport导入相关代码。实际上，在应用程序访问DLL时，实际上就是应用程序中的导入函数与DLL文件中的导出函数进行链接。而且链接的方式有两种：隐式链接和显式链接。
　　隐式链接是指通过编译器提供给应用程序关于DLL的名称和DLL函数的链接地址，面在应用程序中不需要显式地将DLL加载到内存，即在应用程序中使用dllimport即表明使用隐式链接。不过不是所有的隐式链接都使用dllimport。
       显式链接即应用程序用语句显式地加载DLL，编译器不需要知道任何关DLL的信息。
