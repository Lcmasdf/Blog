---
layout: post
title: "string到底是在哪里定义的？"
date: 2016-01-04 22:23:12
categories: C++
excerpt: 我们在使用string时，总会下意识的包含iostream和string两个头文件。但是你知道么。很多时候这些其实string并不是必要的！！
published: true
---


最近闲着无聊写了下面这段代码

	#include <iostream>
	using namespace std;
	int main () {
		string tmp = “123”；
        //此处省略若干代码
        return 0;
	}

　　本来是打算复习一下string的初始化方法以及C++中字符串的操作的，但是突然发现在头文件中我没有包含string、string.h或者cstring这三个头文件中任意一个（我当时确实不太清楚这三个头文件的关系）。但这个程序竟然能够正确的编译通过，没有warning而且可以正确的运行！！！通过Go to Definition我知道string是定义在xstring中的，但是xstring在哪里呢？    
　　有问题问度娘啊。我搜索了“string定义”关键字，得到的结果是string就定义在string头文件中。（这里确实是我的锅，如果我搜索string定义在哪，百度百科第一条搜索记录就会告诉我）    
　　起初，我认为这个是VC独有的黑科技，可以让你在不包含string头文件时使用string，但是我在linux下用g++编译了相同的代码，同样可以通过编译并且正确运行。所以，这个跟IDE以及C++的版本没有关系。    
　　然后我把矛头指向了编译器，以为编译器在处理的时候做了额外帮我添加了一些东西。通过查看个g++的build的指令就知道，编译器只做了编译和链接的部分，肯定不是这里的问题。    
　　不是C++的问题，也不是编译器的锅，那就是我的问题喽。*string到底藏在哪里呢？*当时手里还有其他的问题，而且周围小伙伴也表示：咦，这个东西好神奇哦，所以就不了了之了。    
　　在一个月黑风高的夜晚，我突然意识到string应该就在iostream里面啊，我只包含了这一个头文件，福尔摩斯那句话是怎么说来着：`When you have eliminated the impossible, whatever remains, however improbable, must be the truth.`。定义string的头文件一定就藏在ios特然吗中。果然，没几个Go to Definition就找到xstring，原来iostream里面内容如此丰富。╮(╯▽╰)╭，既然这样，那就来仔细看看iostream中到底包含了什么东西吧。

## VC++中的iostream
![iostream_vc++](http://7xprz9.com1.z0.glb.clouddn.com/Iostream.jpeg)

* iostream  : 定义了cin、cout、cerr、clog以及wcin、wcout、wcerr、wclog    
* istream   : 定义了basic_istream    
* ostream   : 定义了basic_ostream    
* ios       : 定义了basic_ios    
* xlocnum   : a set of feature that are culture-specific,应该就是定义了一些跟具体实现相关的特性，详情参见[locale](www.cplusplus.com/reference/locale)    
* climits   : Microsoft C/C++定义的一些与具体实现相关的值，参见yvals.h和limits.h    
* yvals.h   : values for microsoft C/C++    
* limits.h  : C语言中用到的与具体实现相关的值    
* cstdio    : 在C++中可以使用的C的标准输入输出    
* cstdlib   : 定义了一些通用的函数，主要包括动态内存管理、随机数生成、整数算法、排序和转换。
* streambuf : 定义了所有用来处理narrow characters的stream buffer类的基类，xstring就藏在streambuf中

## G++中的iostream
![iostream_g++](http://7xprz9.com1.z0.glb.clouddn.com/Iostream_g%2B%2B.jpeg)

* iostream  : 定义了cin、cout、cerr、clog以及wcin、wcout、外侧让人、wclog，与VC++相同   
* c++config.h : 预定义了一些序号和宏，例如， GLIBCSS_ABI_TAG_CXX11、_GLIBCXX_USE_CONSTEXPR   
* ostream   : 定义了basic_ostream   
* ios       : 包含了定义iostreams base class的头文件（只包含了若干头文件）   
* iosfwd    : iostreams base class中所有相关内容的前置定义   
* stringfwd.h : string的前置定义，包含了：typedef basic_string<char> string   
* memeoryfwd.h ：与内存相关的类的前置定义    
* postypes.h : 代表stream positions的类型    
* atomic_lockfree_defines.h : 关于原子性定义的一些东西(我也不懂- -，后面来填这个坑)。    
* char_traits.h : string和iostream中用到的字符的特性，包括assign、copy、move等。    
* localefwd.h : locales声明了本地化函数，这些函数用于把程序调整到特定的区域设置，localefwd中包含了前置定义。    
* ios_base.h : 定义了ios_base。   
* ostream_insert.h : 定义了__ostream_write、__ostream_fill、__ostream_insert.h。头文件中注释显示这是一个Helpers for ostream inserters   
* cxxabi_forced.h : cancellation的子集。

## 总结

弄清这个问题最大的感触并不是了解了iostream内部的构造，了解了string中其实只包含了某些运算符和函数的重载。最大的感触是如果我们能够对身边的事情有更敏锐的观察，不要忽视某些细小的问题，总是会有一定的收获的。