title: Python应用部署
date: 2016-03-06 14:32:03
categories: Python
tags:
    - 工作笔记
---

公司部署python web application的方式是nginx+supervisor+gunicorn。之前开发的时候都是直接用web framework自带的WSGI server来运行，绑定公网地址也可以访问，那么为什么还要用这种方式来部署，带着这些疑问整合了一些资料。

#### CGI与WSGI
CGI是原始的动态网站开发方式。动态网站的必然要经过处理客户端请求，处理请求并生成相应的结果，然后返回结果的过程。通常，客户端和服务端是通过HTTP协议来通信的。因为在处理用户请求时，程序会做很多重复的工作，比如解析HTTP请求等一系列工作，这些工作可以抽出来作为一个程序，于是Web服务器就出现了。Web服务器HTTP请求，然后把这个请求的各种参数写进进程的环境变量，比如REQUEST_METHOD，PATH_INFO之类的。之后呢，服务器会调用相应的程序来处理这个请求，这个程序也就是我们所要写的CGI程序了。它会负责生成动态内容，然后返回给服务器，再由服务器转交给客户端。服务器和CGI程序之间通信，一般是通过进程的环境变量和管道。
但是每一次处理用户请求都要调用一次CGI程序，也就是要创建一个CGI进程，然后处理请求，让后结束子进程。这便是fork-an-excute模式，所以用CGI方式的服务青有多少连接就要开多上CGI子进程。而每一个进程都要占用内容、CPU时间，在用户访问量比较多时，瓶颈就出现了。

WSGI是基于现存的CGI标准设计的接口协议。是为Python语言定义的Web服务器和Web应用程序或框架之间的一种简单而通用的接口。自从WSGI被开发出来以后，许多其它语言中也出现了类似接口。WSGI是作为Web服务器与Web应用程序或应用框架之间的一种低级别的接口，以提升可移植Web应用开发的共同点。WSGI是为了让python web framework与web服务器无关，使不同的web framework能在不同服务器可移植的一套标准（PEP0333)。

WSGI区分为两个部份：一为“服务器”或“网关”，另一为“应用程序”或“应用框架”。在处理一个WSGI请求时，服务器会为应用程序提供环境资讯及一个回调函数（Callback Function）。当应用程序完成处理请求后，透过前述的回调函数，将结果回传给服务器。所谓的 WSGI 中间件同时实现了API的两方，因此可以在WSGI服务和WSGI应用之间起调解作用：从WSGI服务器的角度来说，中间件扮演应用程序，而从应用程序的角度来说，中间件扮演服务器。
<!--more-->

#### FastCGI、uWSGI和Gunicorn
当代的python web framework基本都实现了WSGI协议，所以程序员无须关注底层，直接跑程序就能调试了。但为什么在部署的时候为什么还需要FastCGI等这些东西呢？原因就是因为无论是CGI还是WSGI，处理并较大用户量和并发请求的能力都很弱。

##### FastCGI
而Fast CGI是基于CGI的一种改良。FastCGI是一个可伸缩地、高速地在HTTP server和动态脚本语言间通信的接口。多数流行的HTTP server都支持FastCGI，包括Apache、Nginx和lighttpd等，同时，FastCGI也被许多脚本语言所支持，比如Python/PHP等。
FastCGI是从CGI发展改进而来的。传统CGI接口方式的主要缺点是性能很差，因为每次HTTP服务器遇到动态程序时都需要重新启动脚本解析器来执行解析，然后结果被返回给HTTP服务器。这在处理高并发访问时，几乎是不可用的。FastCGI像是一个常驻(long-live)型的CGI，它可以一直执行着，只要激活后，不会每次都要花费时间去fork一次(这是CGI最为人诟病的fork-and-execute 模式)。CGI 就是所谓的短生存期应用程序，FastCGI 就是所谓的长生存期应用程序。由于 FastCGI 程序并不需要不断的产生新进程，可以大大降低服务器的压力并且产生较高的应用效率。它的速度效率最少要比CGI技术提高5倍以上。它还支持分布式的运算, 即 FastCGI 程序可以在网站服务器以外的主机上执行并且接受来自其它网站服务器来的请求。

FastCGI是语言无关的、可伸缩架构的CGI开放扩展，其主要行为是将CGI解释器进程保持在内存中并因此获得较高的性能。众所周知，CGI解释器的反复加载是CGI性能低下的主要原因，如果CGI解释器保持在内存中并接受FastCGI进程管理器调度，则可以提供良好的性能、伸缩性、Fail-Over特性等等。FastCGI接口方式采用C/S结构，可以将HTTP服务器和脚本解析服务器分开，同时在脚本解析服务器上启动一个或者多个脚本解析守护进程。当HTTP服务器每次遇到动态程序时，可以将其直接交付给FastCGI进程来执行，然后将得到的结果返回给浏览器。这种方式可以让HTTP服务器专一地处理静态请求或者将动态脚本服务器的结果返回给客户端，这在很大程度上提高了整个应用系统的性能。

FastCGI的工作流程:
web server启动时载入FastCGI进程管理器 ==> FastCGI进程处理器初始化，启动多个CGI解释器进程等待web server连接 ==> 当有用户请求时，FastCGI管理器便连接到一个CGI解释器，将CGI环境变量和标准输入发送到子进程，然后将子进程输出返回给web server。然后等待下一个连接。

##### uWSGI和Gunicorn
uWSGI和Gunicorn都是WSGI应用服务器，都实现了WSGI协议。
虽然现在的python web framework都自带wsgi服务器，但是都只能简单的开单进程。而像gunicorn是prefork模式，从nginx转发的每一个请求，它就fork一个进程去处理，并缓存相关的数据。相比裸跑框架自带的wsgi服务器，专业的wsgi服务器是针对线上环境开发的，能更好的配置和应对更复杂的情况，性能也更好。
运行和配置详见[uWSGI](https://uwsgi-docs.readthedocs.org/en/latest/)和[gunicorn](http://gunicorn.org/#docs)


#### 为什么supervisor
supervisor是unix-like系统的一个进程管理工具。使用 Supervisor可以更好的控制、管理、重启我们希望运行的进程。
详见[supervisor](http://supervisord.org/introduction.html)

#### 为什么nginx
nginx不只是一个HTTP服务器，同时也是一个反向代理服务器。在python application部署中，它就充当了反向代理服务器的功能，接受到请求，对静态文件的请求自己处理，而动态请求则直接重定向到uWSGI服务器，再不下游服务器的结果返回给用户。
使用nginx作为前端服务器：
 * 高效率处理静态文件,web server都是用c开发，调用是native的函数，对IO，文件传输都做针对性的优化
 * 充当一个简易的网络防火墙，可以denny一些ip，简单的控制并发连接数量等等
 * 处理高并发短连接请求，把成千上万用户的request 通过内网的几十个长连接进行转发，原因一个是web server处理高并发很专业，另外一个原因是大部分的application所用的框架都不具备处理高并发的能力
 * 负载均衡
 * 缓存
