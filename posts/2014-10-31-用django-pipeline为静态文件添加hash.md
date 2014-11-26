用django-pipeline为静态文件添加hash
===

为什么需要hash静态文件？
---

请看[大公司里怎样开发和部署前端代码？](http://www.zhihu.com/question/20790576) 张云龙的答案。

这样，当静态文件有修改时，会很方便的拿到最新的修改版本，而未修改的静态文件则依然使用缓存。这样避免了修改后用户静态文件不更新的尴尬，并且可以充分利用缓存。

demo
---

[django_pipeline_demo](https://github.com/duoduo369/django_pipeline_demo)

安装
---
    sudo mkdir /opt/projects
    git clone https://github.com/duoduo369/django_pipeline_demo.git
    cd django_pipeline_demo
    ln -s $(pwd) /opt/projects
    ln -s /opt/projects/django_pipeline_demo/deploy/nginx/django_pipeline.conf /etc/nginx/sites-enabled
    pip install -r requirements.txt
    python manage.py runserver 0.0.0.0:9888
    nginx -s reload
    vim /etc/hosts 添加 127.0.0.1:9888 django_pipline_demo.com


django的库pipeline
---

[mako](http://www.makotemplates.org/),  [django-mako](https://github.com/jurgns/django-mako),  [django-pipeline-demo](https://github.com/duoduo369/django_pipeline_demo)

效果是这样的,以 [django_pipeline_demo](https://github.com/duoduo369/django_pipeline_demo) 为例。

先说最终用法
---

1. debug必须为False(上线本来就是False),如果为True则使用django默认查找静态文件的方式,不会使用pipeline。
2. `python manage.py collectstatic`
3. 重启django项目

重点代码解释
---
settings.py的几个配置,
如何安装配置django-pipeline,请移步[文档](http://django-pipeline.readthedocs.org/).

解释几个collect有关的配置

    # python manage.py collectstatic 后文件会扔到STATIC_ROOT下面
    STATIC_ROOT = './statics'

    # django的模板会从这些目录下查找
    TEMPLATE_DIRS = (
        os.path.join(BASE_DIR, 'templates'),
    )

    # 开发时css的路径，collectstatic会从这里查找然后丢到STATIC_ROOT下
    # 使用pipeline后会在静态文件中添加hash码，例如css/index.css
    # collectstatic后会变成 css/index.as1df14jah8dfh.css
    STATICFILES_DIRS = (
        os.path.join(BASE_DIR, "static_dev"),
    )


templates/common/static_pipeline.html

    这是用mako定义了一个url，以后静态文件使用这个url导入，就可以找到hash的版本了。

    <%!
    from django.contrib.staticfiles.storage import staticfiles_storage
    %>

    <%def name='url(file)'><%
    try:
        url = staticfiles_storage.url(file)
    except:
        url = file
    %>${url}</%def>

index.html

    首先导入/common/static_pipeline.html,需要引用静态文件的地方使用${static.url('未hash的文件路径')}

    <%namespace name='static' file='/common/static_pipeline.html'/>
    ....
        <link rel="stylesheet" href="${static.url('css/index.css')}" type="text/css" media="all" />
    ....

