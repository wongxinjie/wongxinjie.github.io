title: 我的Python开发总结
date: 2017-03-01 21:43:11
categories: Python
tags:
    - 工作总结
---

在全职用Python开发接近两年后，发现自己虽然长时间用Python，也看多很多的Best Pratices，读了很多关于别人如何用Python进行开发，但是自己从来没有总结，现在就找时间将关于这两年的经验总结一下。

#### Linux, Vim

开发环境首推的MacOS，或者退而求其次是Linux。用*Unix系统可以避免很多奇奇怪怪的问题。当然如果是直接SSH到开发机开发那另说，否则直接上Ubuntu。

开发Python的IDE是PyCharm，社区版就估计够用了。但是个人偏好使用Vim。在Linux下使用Terminal+Vim基本可以完成开发工作。

Terminal使用[oh-my-zsh](http://ohmyz.sh/) ，有同事推荐[fish](https://fishshell.com/) ，这两个都是极佳的，目前用的前者。关于终端还有一个很好用的终端分屏工具, [tmux](https://tmux.github.io/)，功能很强大，但自从用了双屏，就很少使用了。

裸使用Vim那简直就是给自己过不去，假如自己有时间和兴趣和精力可以自己写配置，我用的是GitHub上的humiaozuzu的[Vim配置](https://github.com/humiaozuzu/dot-vimrc)。 自己clone了一份[配置](https://github.com/wongxinjie/dot-vimrc)，比如使用Flask8检查器强制是代码遵循PEP8规则，增加Golang支持以及其他的一些私人配置。Vim配置有很多，其中[spf13](https://github.com/spf13/spf13-vim)可以试一试。

<!--more-->
#### Python2 Vs Python3

关于Python2还是Python3的争论历来已久，但是根据[PEP373](https://www.python.org/dev/peps/pep-0373/) Python2 从2020年开始就不在维护了。所以到现在不应该有什么争论了，假如是历史项目，用Python2就继续用Python2，新项目开始全部使用Python3，这里的Python3是指Python3.4+。项目中使用的Python3.5，Python3.6出来了也可以试试。就版本而言，现在很多第三方库都已经支持Python3，至于有一些库只支持Python2，那么这个库基本就可以不用了，因为在Python3必然会找到替代的库。

Python3是一门经过重新设计的Python语言，单纯就源文件默认编码是utf-8而不是ASCII就是一大进步，文件开头不用特地表明 # -*- coding: utf-8 -*- ，也不用为中文编码纠缠半天了。更不提Python3内置的线程池，异步IO支持等等。如果说Python2还有有点，那么大概 就是Python2的解释器到目前为止比Python3还快。

#### pip & virtualenv

基本上一个合格的Python程序员在日常开发中离不开这两个程序。鉴于国内的网络，可以将pip的源设为阿里源或者豆瓣源，用vim 编辑~/.pip/pip.conf :

```
 [global]
 trusted-host =  mirrors.aliyun.com
 index-url = http://mirrors.aliyun.com/pypi/simple
```

至于[vritualenv](https://virtualenv.pypa.io/en/stable/), 是项目开发中的一个依赖隔离的神器，配合[virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/index.html)，简单配置之后对虚拟环境管理易如反掌：

```
mkvirtualenv deom # 创建虚拟环境
workon demo # 激活虚拟环境
deactive # 在虚拟环境中退出
pip freeze > requirements.txt # 在虚拟环境中导出项目的依赖包
rmvirtualenv # 删除虚拟环境
```

 其他的解决方案还有autoenv等等，可以查阅这个[页面](http://docs.python-guide.org/en/latest/dev/virtualenvs/)。

虚拟环境还可以设置环境变量，在项目中类似数据库配置的一般都不推荐直接Hardcode到代码，而从环境变量获取。在虚拟环境中设置环境变量，编辑 $PROJECT_VENV/bin/postactive:

```
#!/usr/bin/zsh
# This hook is sourced after this virtualenv is activated.
export ENV_VAR_A=A
export ENV_VAR_B=B
```

 比如在Django/Flask项目中，经常用一个DEBUG标志是否开启debug模式，设置成:

```Python
DEBUG = int(os.environ.get("DEBUG", 0)) == 1
```

线下开发时设置环境变量```DEBUG=1``` 便不会有项目上线忘记关闭DEBUG模式这种乌龙的出现。

#### code style & project structure

代码风格上遵循[PEP8](https://www.python.org/dev/peps/pep-0008/), 也许有人会对PEP一行79个字符有异意，认为其实可以120个字符没问题，见仁见智。关于代码风格，有几个资料可以读一读:

* [Code Style](http://docs.python-guide.org/en/latest/writing/style/)
* [The Best of the Best Practices (BOBP) Guide for Python](https://gist.github.com/sloria/7001839)
* [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)

项目结构方面，大多数Python开发这估计都是用Django、Flask、Tornado或者aiohttp开发项目，关于项目结构各个公司估计有各个公司的风格，结构清晰维护方便其实都可以，这里有一篇关于非Web项目结构的资料：

* [Structuring Your Project](http://docs.python-guide.org/en/latest/writing/structure/)

#### document && HOWTO

无论是Django、Flask或者包括Python本身，最好的学习资料便是英文文档本身，没事的时候翻翻，经常会有收获。

搜索引擎用Google，程序员用VPN其实不贵。StackOverflow是好朋友，大多数问题都能解决，用第三方库时，先读文档或者README.md，遇到问题可以看一看issue，说不定会有收获。

用Git对项目源码等进行版本控制，记得要编辑.gitignore，将一些不必或者不能纳入版本控制的文件（夹）写进去。

#### books && websites

关于Python的书有很多，挑选大家评价比较高的就行，但是豆瓣排行不可全信，好比我认为现在《Python基础教材》已经不再适合初学者入门了，因为这本书是基于Python2的。

有时候多看看优秀开源库作者的博客，或者其他一些网站都是有收获的，我就收集了一些：

* [The Hitchhiker’s Guide to Python!](http://docs.python-guide.org/en/latest/) ，python最佳实践集合
* [Full Stack Python](http://www.fullstackpython.com/table-of-contents.html), Python技术栈
* [OSOABOOK](http://aosabook.org/en/index.html), 虽然不是纯Python，但是绝对经典
* ...(想到再补)

#### finally

现在越来越觉得，其实将自己定位为Python程序员有时候会徒曾自己的无力感。虽然有同行表示Python不搞个三五年不能算精通，但不可否认这其实不大现实。在招聘市场上Python一直算是小众语言，招聘网站上招Python的大多数是创业公司，稍微大点的公司招Python都是偏运维方向或者是对学历要求是硕士（机器学习、数据挖掘方面）。所以每年都有Python程序员转向前端、转Java等等。

话说回来，只会Python的程序员不是好Python程序员。作为计算机专业的学生，大学的时候没有认真学习，等毕业了才发现自己大学白上了，计算机网络多好啊，操作系统知识其实挺有趣，数据结构与算法对一个人写程序其实是有影响的，编译原理了解了对自己有好处。懂一点点操作系统和Linux编程只是，才能理解Python的GIL是怎么一回事，C10K是什么，异步又有什么好处，哪种情景下适用……

当Python程序员重新拾起了C++/Java，才会发现自己的路并不窄，最近在看Go，配好Vim后写起来也挺舒服的，但是为了好找工作，还是先把Java/C++学起来，数据结构和算法自己消化一下。

