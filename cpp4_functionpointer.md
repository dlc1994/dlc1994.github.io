---
title: 如何编写高效、优雅、可信代码系列（4）——今天教你学会用函数指针
date: 2021-07-19 23:59:53
toc: true
categories: 
- C++
tags: 
- C++
- 函数指针
top: 16
---
故事的起源来自于，没错，又是来自于业务。

事实是这样的，我在重构一段代码的时候，发现有两个函数的代码近乎80%都是相同的，区别在于根据不同的条件局部调用不同的两个函数，很难搞，因为一是抽个abstract class太麻烦，二是函数里是循环，在循环里判断又影响性能，不管吧，也影响代码重复率。这个时候我想到了函数指针！
<!--more-->
# 1. 函数指针的定义和作用

我们先来看看wiki的定义

> 【from wiki】 A function pointer, also called a `subroutine pointer` or `procedure pointer`, is a pointer that points to a function. As opposed to referencing a data value, a function pointer points to executable code within memory. Dereferencing the function pointer yields the referenced function, which can be invoked and passed arguments just as in a normal function call. Such an invocation is also known as an "indirect" call, because the **function is being invoked indirectly through a variable instead of directly through a fixed identifier or address**.
> 
> Function pointers can be used to **simplify code** by providing a simple way to select a function to execute based on **run-time** values.  

重点已经在上面标出来了，函数指针的作用之一是简化代码。

那么c++标准委员会仅仅是为了让我们写clean code吗？

Obviously not! Another use for function pointers is setting up **"listener"** or **"callback"** functions that are invoked when a particular event happens. 

什么是回调函数呢？比如你为图形用户界面 (GUI) 编写代码时。大多数情况下，用户将与允许鼠标指针移动并重绘界面的循环进行交互。但是，有时用户会单击按钮或在字段中输入文本。这些操作是“事件”，可能需要您的程序需要处理的响应。你的代码怎么知道发生了什么？使用回调函数！用户的点击应该会导致界面调用您编写的用于处理事件的函数。

# 2. 函数指针的用法

说了这么多，你是不是已经跃跃欲试，恨不得马上重构自己的代码以降低重复率了，hold on! hold on!先看完用法再去操作，磨刀不误砍柴工。

## 2.1 函数指针的声明
先来看看函数指针的声明：

```
void *(*foo)(int *);
```

表达式的最里面的元素是 `*foo`，它应该指向一个返回 `void *` 并采用 `int *`作为参数的函数。因此，foo是指向这样一个函数的指针。

还可以用`typedef`简化函数指针的定义

```
#include <iostream>
int test(int a)
{
    return a;
}
int main(int argc, const char * argv[])
{
    typedef int (*foo)(int a);
    foo f = test;
    cout << foo(2) << endl;
    return 0;
}
```

## 2.2 函数指针的初始化

为了初始化一个函数指针，需要给它一个程序内的函数地址，比如下面的函数：

```
#include <stdio.h>
void MyFun(int x)
{
    printf( "我是%d颗大西瓜。\n", x );
}

int main()
{
    // way 1
    void (*foo)(int);
    foo = &MyFun;
    // way 2
    void (*foo)(int) = &MyFun;
    // way 3
    auto foo = &MyFun;

    return 0;
}
```
上面三种方式都是可以的。如果是指向类函数呢？

```
#include <iostream>
#include <cstdio>
using namespace std;
 
class A
{
    public:
        A(int aa = 0):a(aa){}
        ~A(){}
        void SetA(int aa = 1)
        {
            a = aa;
        }
        virtual void Print()
        {
            cout << "A: " << a << endl;
        }
    private:
        int a;
};
int main(void)
{
    A a;
    void (A::*ptr)(int) = &A::setA;
    A* pa = &a;
    
    //对于非虚函数，返回其在内存的真实地址
    printf("A::Set(): %p\n", &A::SetA);
    //对于虚函数， 返回其在虚函数表的偏移位置
    printf("A::Print(): %p\n", &A::print);
 
    a.Print();
    a.SetA(10);
    //对于指向类成员函数的函数指针，引用时必须传入一个类对象的this指针，所以必须由类实体调用(如果是在类内调用就用*this)
    (pa->*ptr)(1000);
}
```

## 2.3 让我们来看看成果

下面是用不同的order进行的选择排序代码，请看成片

```
#include <utility> // for std::swap
#include <iostream>
 
// Note our user-defined comparison is the third parameter
void selectionSort(int *array, int size, bool (*comparisonFcn)(int, int))
{
    // Step through each element of the array
    for (int startIndex {0}; startIndex < (size - 1); ++startIndex) {
        // bestIndex is the index of the smallest/largest element we've encountered so far.
        int bestIndex {startIndex};
        // Look for smallest/largest element remaining in the array (starting at startIndex+1)
        for (int currentIndex{ startIndex + 1 }; currentIndex < size; ++currentIndex) {
            // If the current element is smaller/larger than our previously found smallest
            if (comparisonFcn(array[bestIndex], array[currentIndex])) { // COMPARISON DONE HERE
                // This is the new smallest/largest number for this iteration
                bestIndex = currentIndex;
            }
        }
        // Swap our start element with our smallest/largest element
        std::swap(array[startIndex], array[bestIndex]);
    }
}
 
// Here is a comparison function that sorts in ascending order
bool ascending(int x, int y)
{
    return x > y; // swap if the first element is greater than the second
}
 
// Here is a comparison function that sorts in descending order
bool descending(int x, int y)
{
    return x < y; // swap if the second element is greater than the first
}
 
// This function prints out the values in the array
void printArray(int *array, int size)
{
    for (int index{ 0 }; index < size; ++index) {
        std::cout << array[index] << ' ';
    }
    std::cout << '\n';
}
 
int main()
{
    int array[9] { 3, 7, 9, 5, 6, 1, 8, 2, 4 };
 
    // Sort the array in descending order using the descending() function
    selectionSort(array, 9, descending);
    printArray(array, 9);
 
    // Sort the array in ascending order using the ascending() function
    selectionSort(array, 9, ascending);
    printArray(array, 9);
 
    return 0;
}
```

# 3. 后记

是不是看起来还挺优雅的，函数指针的好处可以总结如下
- **GOOD1: 函数指针提供了一种传递有关如何做某事的指令的方法**
- **GOOD2: 可以编写灵活的函数和库，允许程序员通过将函数指针作为参数传递来选择行为（当然这种灵活性也可以通过使用具有虚函数的类来实现）**
- **GOOD3: 可以简化代码**

但是every coin has two sides，函数指针也不是百利而无一害，我也总结了一些，大家可以继续补充
- **BAD1: 性能开销。** 让我们回到它的定义，正如一个指针变量保存的是变量的地址一样，只不过函数指针保存的是函数的地址，并且是运行时才能确定的变量。理论来说，直接函数调用开销更小，因为函数指针调用需要先访问数据区，再访问函数，增加指令开销，同时，数据取值与函数指令加载必须串行执行，影响CPU流水性能；普通的函数调用可以做内联优化。*不过幸好编译器可以将函数指针的调用开销优化到跟普通的函数调用相同（优化选项O1及以上，具体可参考https://zhuanlan.zhihu.com/p/84887035）*
- **BAD2: 代码可读性。** 区别于传统的函数调用，我们在进入到使用函数指针作为参数的函数中时，有时会很抓狂，比如上面的`selectionSort()`，我在阅读到`comparisonFcn()`的时候无法确认这个函数的行为，需要往外跳一层才能知道什么情况下这个函数是升序还是降序；更令人抓狂的是，更为复杂的函数中（假如作者不写任何注释），你会破口大骂谁写的函数指针，而且点跳转还无法跳转到相应的函数实现中（没错，我有时候是会骂自己的T.T）。



