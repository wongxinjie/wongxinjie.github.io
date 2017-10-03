title: Python性能优化
date: 2015-10-04 17:21:49
categories: Python
tags:
- 基础
---

注：本文总结自[PythonSpeed PerformaceTips](https://wiki.python.org/moin/PythonSpeed/PerformanceTips)。
Python在性能上的确存在缺陷，但是在写Python的过程中还是可以通过避免一些不高效的写法稍微地提高性能的。

#### 优化可以优化的细节
**排序**  
凡是Python已经有实现的功能，优先采用Python内置的实现方式。譬如，要实现n元元组的列表根据第N列进行排序。下面的程序是可以实现的。
{% code %}
def sortby(somelist, n):
    nlist = [(x[n], x) for x in somelist]
    nlist.sort()
    return [val for (key, val) in nlist]
{% endcode %}
或者原位实现现：
{% code %}
def sortby_inplace(somelist, n):
    somelist[:] = [(x[n], x) for x in somelist]
    somelist.sort()
    somelist[:] = [val for (key, val) in somelist]
    return
{% endcode %}
但实际上自Python2.4开始，有更好的事项方式了。
<!--more-->
非原位实现排序
{% code %}
nlist = sorted(somelist, key=lambda x: x[n])
{% endcode %}
原位实现：
{% code %}
somelist.sort(key=lambda x: x[n])
# import operator
# somelist.sort(key=operator.itemgetter(n))
{% endcode %}
**连接字符串**
要进行字符串拼接的场合有很多，在实现字符串拼接使要避免：
{% code %}
s = ""
for substring in list:
    s += substring
{% endcode %}
更好的实现是：
{% code %}
s = "".join(list)
{% endcode %}
避免：
{% code %}
out = "<html>" + head + prologue + query + tail + "</html>"
{% endcode %}
更好的实现：
{% code %}
out = "<html>%s%s%s%s</html>" % (head, prologue, query, tail)
# even better
out = "<html>%(head)s%(prologue)s%(query)s%(tail)s</html>" % locals()
{% endcode %}
注：这一点不是很赞同，在Python2.7三个实现方式的效率其实差不多，但是显然第一个的可读性更高一点．
**循环**
一个常见的循环：
{% code %}
oldlist = ('python' for _ in range(10000))
newlist = []
for word in oldlist:
    newlist.append(word.upper())
{% endcode %}
使用列表推倒的实现方式：
{% code %}
newlist = [s.supper for s in oldlist]
{% endcode %}
或者使用map函数实现：
{% code %}
newlist = map(str.upper, oldlist)
{% endcode %}
在循环中避免多次的使用．调用函数：
{% code %}
upper = str.upper
newlist = []
append = newlist.append
for word in oldlist:
    append(upper(word))
{% endcode %}
以上四种实现方式的效率结果如下：
{% code %}
normal loops:  4.31513786316 ms
list comprehension:  2.3820400238 ms
map:  2.33602523804 ms
avoiding dot 3.98087501526 ms
{% endcode %}
避免使用．运算符虽然可以提高一点点性能，但是这提高的性能是以牺牲代码冗余以及降低代码可读性为代价的，在工程实现中代码可读性（可维护）显然比提高的那么一点点性能更为重要．况且有更高效的列表推导和map实现，显然列表推导可读性高于map.
**字典初始化**
假如有一项任务，统计在日志记录中某个ip出现的次数，很容易写出来：
{% code %}
ip_dict = {}
for ip in ips:
    if ip not in ip_dict:
        ip_dict[ip] = 0
    ip_dict[ip] += 1
{% endcode %}
或者存在这样一种情况，这个日志记录中某个ip出现的次数远远大于其他偶尔出现的ip，也可以：
{% code %}
ip_dict = {}
for ip in ips:
    try:
        ip_dict[ip] += 1
    except KeyError:
        ip_dict[ip] = 1
{% endcode %}
更好地实现：
{% code %}
ip_dict = {}
for ip in ips:
    ip_dict[ip] = ip_dict.get(ip, 0) + 1
{% endcode %}
假如默认的值为列表，则可以使用：
{% code %}
ip_dict.setdefault(key, []).append(ip_element)
{% endcode %}
更更好的实现是使用collections.defaultdict:
{% code %}
from collections import defaultdict

ip_dict = defaultdict(int)

for ip in ips:
    ip_dict[ip] += 1
{% endcode %}
