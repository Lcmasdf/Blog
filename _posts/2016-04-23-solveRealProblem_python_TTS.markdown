---
layout: post
title: "想把我读给你听"
data: 2016-04-23 21:33:27
categories: python
excerpt: 闭上眼，听听我的博客吧
---

某一天，我在逛github时发现一个[精妙的python程序](https://github.com/Aaron1011/python-email-reader)，是一个用来发声读新收到邮件的程序。嗯哼～，我也搞[一个读我博客的程序]()，这样岂不是想看就看，想听就听（什么？你说既不想看又不想听怎么办？友谊的小船说翻就翻了～）

# python-email-reader分析

<img src="http://7xprz9.com1.z0.glb.clouddn.com/python-email-reader.png" alt="python-email-reader" width="100%">

如上图所示，python-email-reader主要实现了三部分功能：从服务器拉取最新未读邮件、解析拉取信息的内容和将感兴趣的内容让机器念出来。主要涉及了方面的知识[Internet Message Access Protocol](https://en.wikipedia.org/wili/Internet_Message_Access_Protocol)和[Text-To-Speech](https://en.wikipedia.org/wiki/Speech_synthesis)。掌握了这两方面的内容，读懂python-email-reader的代码分分钟的事儿。

ps: pyttsx仅仅支持python2.x，通过简单的修改并不能够使之适应python3.x。所以使用python3.x的童鞋看下思想就好～

# lcm-blog-reader设计

lcm-blog-reader所要实现的功能也可以简单的分解为三部分：获取网页内容、解析网页内容得到需要阅读的部分和将这部分内容发声读出来，如下图所示。

<img src="http://7xprz9.com1.z0.glb.clouddn.com/lcm-blog-reader.png" alt="lcm-blog-reader" width="100%">

针对三部分的解决方案如下：

1. 使用[urllib](https://docs.python.org/2/library/urllib.html)解决获取网页内容的问题

2. 使用[htmllib](https://docs.python.org/2/library/htmllib.html)解决解析网页内容问题

3. 使用[pyttsx](http://pyttsx.readthedocs.org/en/latest)来实现发声的问题

# 难点

看似简单的三步走就解决了上述的问题，但是在实现过程中，还是有一些坑要踏平。

## 中文发音问题

python-email-reader的作者是个歪果仁(一出生就点满了英语天赋那种)，所以他的邮件内容应该全是英语。作为计算机的本命语言，不论你使用哪种操作系统，选择哪种默认语言以及使用何种TTS引擎，英语都是默认支持的。

但是作为龙的传人，我更喜欢使用一种适合书写诗歌的语言-汉语来写博客。对于汉语的支持，各个TTS引擎就做的不像对英语的支持那么好了。想要正确的发声往往需要设置好正确的引擎以及语言包。

如何自动选择合适的引擎以及语言包成为了我们首要解决的问题——没声音，再好的戏也出不来。

在不同的系统下语音引擎的设置是不一样的：windows自带SAPI5, MacOSX自带NSSpeechSynthesizer以及可以运行在linux下的eSpeaker，还有一起其它的可以运行在不同平台上的语音引擎，目前还有基于云的引擎。如果用户能够自己正确设置引擎以及语言包，那么就能获得更好的效果；如果按照默认的操作来，效果会差一些，但是简便很多。最终我觉得决定后一种方式，易用性效果不错要比难用效果好来的靠谱！

关于语音包的问题，目前只有一个粗糙的办法，就是判断语音包的名称中是否有"ZH" "CH"等字样。如果有，那么这个语音包大概率是中文语音包。WIN10默认提供的语言包如下所示：

![WIN10提供的语音包](http://7xprz9.com1.z0.glb.clouddn.com/voice_item.png)

当然，这并不能解决所有问题，目前我还没有想到其它简单粗暴能够解决此问题的办法。如果你有好的idea，快快跟我分享一下啊！！

## 关于公式、特殊字符的读法

博客中有一部分的内容机器很难判断应该怎么读，例如：

1. 公式： [A,B] +' [C,D] +' [E,F]  +' [G,H]

2. 特殊符号： 在MarkDown里，在需要列表显示的文字前加上'-'或者'就可变为无需列表

3. 稀奇古怪的符号： @-@, ~_~


很显然，对于上面第一条中所述的公式，如果没有认真看过InternetCheckSum文章的人都无法正确的读出来。如何能期待TTS引擎能够正确识别它并且用正确的发音将其读出来呢，所以，通过升级语音引擎来解决这类问题不太现实，至少难度太大。

我突发奇想，我们希望这些内容可以被正确的显示，也希望这些内容能够正确的读出来。但是并没有规定这是同一部分！将内容以一种形式显示，以另一种形式进行发音（例如，对于 +' 我们规定其发音为‘一补码加法’）！采用了如下的方法来解决这个问题：对于上述较难发音的内容，使用span标签分别将显示与发音的部分标记起来，对于需要显示的部分，添加属性TTSShow=True，对于需要发声的部分，添加属性hidden=‘hidden’通过html解析将两部分内容分别提取出来，这样就实现了复杂语句的阅读功能了。

# 总结

最近读完了《黑客与画家》,感觉自己太看重编程语言而忽视了更重要的东西。能够解决实际问题的知识才是最重要的知识，而通过实现lcm-blog-reader也印证了这一点：python的语法我并不熟悉，但是我依旧可以解决实际的问题。

能解决实际问题的码弄才是好程序员！