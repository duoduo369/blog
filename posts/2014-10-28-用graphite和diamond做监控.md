用graphite diamond做监控
===

文章
---
开局先贴两个文章，值得一读

[很赞的blog](http://www.dongwm.com/archives/shi-yong-grafanahe-diamondgou-jian-graphitejian-kong-xi-tong/)

[另一篇介绍graphite的文章](https://kevinmccarthy.org/blog/2013/07/18/10-things-i-learned-deploying-graphite/)

恩怨
---
无论是什么系统，只要上线，就需要运维，这时候很想看一些监控的图表，graphite就很方便的实现了这个需求。

而graphite采用metrics的方式，又有很多其他的tool为他做支持，所监控的不仅仅是机器的一些东西，你可以监控你爬虫的指标，
log的INFO,ERROR频次，nginx网站的访问数量等等，基本是你需要监控什么，很容易的就可以做到。

我从2014年初就在自己的TODOList添加了要玩graphite, 陆续玩了3、4次都失败了，原因都是安装里面某些步骤失败，
这两天终于搞成功了，写个博客记录一下。

[graphite-web](https://github.com/graphite-project/graphite-web) 大部分的安装方式比较简单，都是用pip就可以安装，但是装完后有个坑,
[文档](http://graphite.readthedocs.org/en/latest/install-pip.html)中说使用`pip install graphite-web`,但是pip中的graphite-web太老了，
导致有个cairo,库在ubuntu下打死也装不上，在新的源码中此bug已经修复。我已经提了[issue 1004](https://github.com/graphite-project/graphite-web/issues/1004)

因为用的graphite-index,直接拿了他的几张图来看最终效果

![1](https://raw.github.com/duoduo369/skill_issues/master/imgs/blog/grapite/1.png)
![2](https://raw.github.com/duoduo369/skill_issues/master/imgs/blog/grapite/2.png)
![3](https://raw.github.com/duoduo369/skill_issues/master/imgs/blog/grapite/3.png)


安装
---

我用的是ubuntu, 写在最上面, 并且我假设你了解基本的python语法，用过pip, virtualenv, 没用过也没问题。

文档需要翻墙，因此贴出主要的安装步骤.
最好安装到python的virtualenv中，具体virtualenv的使用可以参考[这里](https://github.com/duoduo369/skill_issues/blob/master/python/python_tools.issue.md)
首先，查看graphite-web的[requirements.txt](https://github.com/graphite-project/graphite-web/blob/master/requirements.txt)，发现需要装一些系统的库, `sudo apt-get install libcairo2-dev`。


    pip install https://github.com/graphite-project/ceres/tarball/master
    pip install whisper
    pip install carbon
    pip install graphite-web


这里我先贴下最终整个系统搭起来后的各个python库版本, 其中logster是一个做日志监控的东西，先`git clone`的本机，然后`pip install -e logster`项目地址即可


    Django==1.4.8
    Twisted==11.1.0
    argparse==1.2.1
    astroid==1.2.1
    cairocffi==0.6
    ceres==0.10.0
    cffi==0.8.6
    configobj==5.0.6
    diamond==3.5.0
    django-tagging==0.3.3
    ipython==2.3.0
    logilab-common==0.62.1
    -e git+https://github.com/etsy/logster.git@4606bfc6b000ec0fd57de639d08cea9629525304#egg=logster-master
    mock==1.0.1
    psutil==2.1.3
    pycparser==2.10
    pylint==1.3.1
    pylint-django==0.5.5
    pylint-plugin-utils==0.2.2
    pyparsing==1.5.7
    python-memcached==1.47
    simplejson==2.1.6
    six==1.8.0
    txAMQP==0.4
    whisper==0.9.12
    wsgiref==0.1.2
    zope.interface==4.1.1

graphite配置与启动
---
根据文档的步骤安装完成后，你会发现`/opt/graphite`下多了一堆东西，将`/opt/graphite/conf`下的*.example,拷贝到去掉example即可

graphite有个服务在2003,2004接口上，你的metrics需要扔到2003上，具体请看文档，现在不用在意这些细节。

metrics就是类似这样的字符串 前缀.前缀.前缀....... blabala, graphite就是根据这种东西画图的,具体请看文档，不用在意这些细节,
因为其他的工具都有封装。

*. 启动carbon, metrics会扔到carbon这个小屋里面


    /opt/graphite/bin/carbon-cache.py start


*. 制造一些metrics, 更改host，或者server, 这里只是做测试，之后会用diamond来采集metrics


    vim /etc/hosts
    添加 127.0.0.1   graphite, 或者其他的东西

    python /opt/graphite/examples/example-client.py
    这些数据存在 /opt/graphite/storage/whisper, 尝试修改example-client.py发点不一样的东西


*. 配置并修改graphite-web的几行代码，启动这个django项目

    cp /opt/graphite/webapp/graphite/local_settings.py{.example,}
    python /opt/graphite/webapp/graphite/manage.py syncdb
    vim /opt/graphite/webapp/graphite/render/glypy.py
    找到import cairo, ....(这就是坑)
    改为import ...
    try:
        import cairo
    except ImportError:
        import cairocffi as cairo

    启动django
    python /opt/graphite/webapp/graphite/manage.py runserver 0.0.0.0:12222(或者其他端口)

4. 浏览器打开http://127.0.0.1:12222, http://127.0.0.1:12222/dashboard这两个页面玩一下,你会看到左侧tree那边有一些数据
这些数据存在`/opt/graphite/storage/whisper`

使用diamond收集metrics
---
给graphite填数据的方式太多了，这里使用diamond,因为豆瓣有一层graphite+diamond的皮, 下面会说

安装
---


    git clone https://github.com/BrightcoveOS/Diamond.git
    cd Diamond
    pip install -e ./


配置并启动
---


    cp /etc/diamond/diamond.conf{.example,}

    vim /etc/diamond/diamond.conf
    找到host, host = graphite(还记得之前配的host么)
    查看下这个文件，你可以cd到collectors_path, handlers_path去看看里面的文件, 因为定制自己的
    diamand collector时需要根据这些东西来写(继承Collector,重写collect方法)，此篇不谈

    service diamond restart


给graphite换层皮, graphite-index
---
graphite的界面实在是不敢恭维，因此很多人为它写UI，这里选择豆瓣的[graphite-index](https://github.com/douban/graph-index)
选择它是因为配置简单

下载
---


    git clone https://github.com/douban/graph-index.git
    cd graph-index


配置
---


    vim config.py
    graphite_url天上你graphite的ip已经端口
    graphite_url = 'http://127.0.0.1:12222'


更新metrics
---


    ./update-metrics.py
    crontab -e
    */5 * * * * python 绝对路径到/update-metrics.py
    ./graph-index.py


使用logster做日志监控
---
日志监控还是需要的，出了nginx的访问日志之外，对于application的异常等等可能也需要监控，这时候使用[logster](https://github.com/duoduo369/logster),就非常方便了，因为他内置了像graphite发metrics的方法，so easy, 这里给了一个我fork的地址，因为我是一个pythoner，logster默认
的parser有apache等等，但是没有python的，我写了一个，提了一个patch.

安装:

    git clone git@github.com:duoduo369/logster.git
    cd logster
    pip install -e ./

用法:
    logster  --output=graphite --graphite-host=graphite的ip已经端口 你的parser 日志绝对路径
    logster  --output=graphite --graphite-host=127.0.0.1:2003 PythonLogster /var/log/adx/adxsterr.log

如果你需要自己定制parser,参照`logster/logster/parsers`下的东西写一个就好。

因为logster自带向graphite发metrics,无须向diamond集成(写Collector),只要起一个定时任务即可。




Finally
---

当然，如果你熟悉django，可以把graphite, graphite-index人给gunicorn和supervisor,这不是重点，需要的可以参考我github上的[demo](https://github.com/duoduo369/django_supervisor_gunicorn_demo).

至于定制你的diamond Collector,监控你想监控的东西，请自己翻阅文档 (继承Collector,重写collect方法),将写好的Collector放在collectors_path下.

