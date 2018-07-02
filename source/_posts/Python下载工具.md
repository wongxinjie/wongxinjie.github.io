title: Python下载工具
date: 2016-04-10 21:01:47
categories: Python
tags:
- download tools
- 笔记
---

由于业务需求，最近整理了一些关于python下载文件的一些套路。利用python下载文件，大致有以下几种方式：
* [urllib](https://docs.python.org/2/library/urllib.html)，标准库自带，缺点是不支持cookie，只支持HTTP/FTP/本地文件，也不支持SSL。
* [urllib2](https://docs.python.org/2/library/urllib2.html)，一个完整的HTTP/FTP客户端实现，基本实现了必须的功能，支持cookie，但HTTP只支持GET和POST.
* [urllib3](https://pypi.python.org/pypi/urllib3),支持复用socket连接，文件上传，内置重定向和重试，而且线程安全。requests就是基于urllib3.
* [httplib](https://docs.python.org/2/library/httplib.html)，标准库自带，只支持HTTP/HTTPS协议，一般不会直接使用，urllib部分功能基于此模块.
* [httplib2](https://pypi.python.org/pypi/httplib2),只支持HTTP/HTTPS协议，一个较完善的HTTP客户端，支持Keep-Alive，验证、缓存及HTTP GET/POST/HEAD等请求方法等等。
* [mechanize](http://wwwsearch.sourceforge.net/mechanize/),一个可以模拟用户操作的web browsing拓展包。
* [requests](http://docs.python-requests.org/en/master/),这个基本都知道, urllib for human。
* [PyCurl](http://pycurl.io/docs/latest/),libcurl的一个python封装，特点是快（比requests快几倍），支持多种协议，可定制性高。

<!--more-->
除了这些python实现，也查找了一些Linux命令行下载工具，比如curl一个wget，利用subprocess.check_call等调用下载文件。curl的作者自己总结了curl和wget的一些异同。
相同之处:
* 都是支持HTTP/FTP/HTTPS下载文件的命令行工具。
* 都可以发送HTTP POST请求，以及都支持cookies。
* 都不是为了图形界面而设计。
* 都是90s开始的开源项目

不同的是，
curl方面(不翻译了)

* library. curl is powered by libcurl - a cross-platform library with a stable API that can be used by each and everyone. This difference is major since it creates a completely different attitude on how to do things internally. It is also slightly harder to make a library than a "mere" command line tool.
* pipes. curl works more llke the traditional unix cat command, it sends more stuff to stdout, and reads more from stdin in a "everything is a pipe" manner. Wget is more like cp, using the same analogue.
* Single shot. curl is basically made to do single-shot transfers of data. It transfers just the URLs that the user specifies, and does not contain any recursive downloading logic nor any sort of HTML parser.
* More protocols. curl supports FTP, FTPS, Gopher, HTTP, HTTPS, SCP, SFTP, TFTP, TELNET, DICT, LDAP, LDAPS, FILE, POP3, IMAP, SMB/CIFS, SMTP, RTMP and RTSP. Wget only supports HTTP, HTTPS and FTP.
* More portable. curl builds and runs on lots of more platforms than wget. For example: OS/400, TPF and other more "exotic" platforms that aren't straight-forward unix clones.
* More SSL libraries and SSL support. curl can be built with one out of eleven (11!) different SSL/TLS libraries, and it offers more control and wider support for protocol details. curl supports public key pinning.
* HTTP auth. curl supports more HTTP authentication methods, especially over HTTP proxies: Basic, Digest, NTLM and Negotiate
* SOCKS. curl supports several SOCKS protocol versions for proxy access
* Bidirectional. curl offers upload and sending capabilities. Wget only offers plain HTTP POST support.
* HTTP multipart/form-data sending, which allows users to do HTTP "upload" and in general emulate browsers and do HTTP automation to a wider extent
* curl supports gzip and inflate Content-Encoding and does automatic decompression
* curl offers and performs decompression of Transfer-Encoded HTTP, wget doesn't
* curl supports HTTP/2 and it does dual-stack connects using Happy Eyeballs
* Much more developer activity. While this can be debated, I consider three metrics here: mailing list activity, source code commit frequency and release frequency. Anyone following these two projects can see that the curl project has a lot higher pace in all these areas, and it has been so for 10+ years. Compare on openhub

Wget方面
* Wget is command line only. There's no library.
* Recursive! Wget's major strong side compared to curl is its ability to download recursively, or even just download everything that is referred to from a remote resource, be it a HTML page or a FTP directory listing.
* Older. Wget has traces back to 1995, while curl can be tracked back no earlier than the end of 1996.
* GPL. Wget is 100% GPL v3. curl is MIT licensed.
* GNU. Wget is part of the GNU project and all copyrights are assigned to FSF. The curl project is entirely stand-alone and independent with no organization parenting at all with almost all copyrights owned by Daniel.
* Wget requires no extra options to simply download a remote URL to a local file, while curl requires -o or -O.
* Wget supports only GnuTLS or OpenSSL for SSL/TLS support
* Wget supports only Basic auth as the only auth type over HTTP proxy
* Wget has no SOCKS support
* Its ability to recover from a prematurely broken transfer and continue downloading has no counterpart in curl.
* Wget enables more features by default: cookies, redirect-following, time stamping from the remote resource etc. With curl most of those features need to be explicitly enabled.
* Wget can be typed in using only the left hand on a qwerty keyboard!
