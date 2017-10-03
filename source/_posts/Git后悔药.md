title: Git后悔药
date: 2015-07-07 23:05:36
categories: 经验
tags:
- Git
- Tools
---
公司使用的代码版本控制是GitHub。随着对GitHub的深入使用，越发觉得git大法就是好啊，最基本的git给你后悔的机会。
最基本的，要是觉得某天不小心修改了某段代码(真相是写得太烂)，需要将其恢复到修改之前的状态，使用checkout，具体讲
{% code %}
git checkout filename
{% endcode %}
checkout命令会将文件的内容恢复到上一次commit的状态，妥妥的从头开始。
对于已经提交的，又开始后悔不该提交，git也提供了后悔的方法，reset(版本撤回)命令。
<!--more-->
{% code %}
git reset --mixed [commit] #回退commit，保留源码，默认方式。
git reset --soft [commit] #回退至某个版本，只回退commit信息。
git reset --hard [commit] #彻底回退到某个版本
{% endcode %}
常用的有：
{% code %}
git reset HEAD^ 回退所有的内容到上一个版本
git reset —soft HEAD~3 撤销最近的三次提交
git reset HEAD^ test.py 将test.py回退到上一个提交版本
git reset —hard origin/master 将本地状态回退到和远程一样
git reset 067de 回退到commit hash为067de版本
{% endcode %}
在一次提交代码中，由于将多余的东西commit了，所以想撤销commit。毕竟第一次用git reset，很naive地使用了git reset –hard head^．然后发现自己写了一整天的代码奇迹的小时了。去google一下，发现—hard 选项是将代码干干净净地回退到上一个版本，完全擦除。
好在git提供了后悔的余地。使用
{% code %}
git reflog
{% endcode %}
查看所有的提交记录，函授的一串形如098e8d的便是每次commit的编号，找到使用git reset —hard head^前一次的提交编号 xxxxx.
{% code %}
git reset --hard xxxxx
{% endcode %}
代码妥妥地回来了。使用—hard选项前需三思，最好将代码备份一下，免得虚惊一场。
假如已经将代码commit并push到了远程服务器，还想修改commit，我试过用revert（撤销操作）命令。
撤销某次操作，此次操作之前的commit会被保留，而之后的会被全部撤销。
{% code %}
git revert commit
git revert --[continue|quit]
{% endcode %}
常见用法：
{% code %}
git revert HEAD~3: 丢弃最近三次的commit，将代码回复到第四个commit的状态，并提交一个commit记录。
git revert 089e4d 丢弃098e4d之后的commit
{% endcode %}

**reset与revert的区别**
git revert是使用一次新的commit来回滚之前的commit，而reset则直接删除指定的commit.
在回滚这一操作上看，效果差不多。但是在日后继续merge以前的老版本时有区别。因为git revert是用一次逆向的commit“中和”之前的提交，因此日后合并老的branch时，导致这部分改变不会再次出现，但是git reset是之间把某些commit在某个branch上删除，因而和老的branch再次merge时，这些被回滚的commit应该还会被引入。
git reset 是把HEAD向后移动了一下，而git revert是HEAD继续前进，只是新的commit的内容和要revert的内容正好相反，能够抵消要被revert的内容。
