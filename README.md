# Django-处理流程
#### 记录一下django框架的处理流程从runserver开始：
一般开发过程中用python manage.py runserver ip:port启动django wsgi server。创建一个新的项目时django会自动生成manage.py文件，里面要做的事情就是解析命令行要执行的脚本。源码位置django.core.management.execute_from_command_line。这里执行的是runserver命令。可以在命令行输入python manage.py help --commands，返回的是django内部的命令。
##### 以runserver为例，描述命令调用过程：
execute_from_command_line创建了ManagementUtility的实例调用execute,execute获取给定命令行参数，预先处理django setting and python path,这些配置会影响命令执行，它会计算出哪个子命令正在运行，创建适合该命令的解析器并运行它，这里涉及到BaseCommand所有命令的基类。ManagementUtility的实例调用execute，execute返回runserver.run_from_argv,run_from_argv即为runserver从BaseCommand继承的方法，调用自身的handler()。接下来就是调用命令自身的方法run--inner_run--<b>get_handler（返回WSGIHandler）/django.core.servers.basehttp.run</b>

<pre><code>
django.core.servers.basehttp.run源码
def run(addr, port, wsgi_handler, ipv6=False, threading=False, server_cls=WSGIServer):
    server_address = (addr, port)
    if threading:
        httpd_cls = type('WSGIServer', (socketserver.ThreadingMixIn, server_cls), {})
    else:
        httpd_cls = server_cls
    httpd = httpd_cls(server_address, WSGIRequestHandler, ipv6=ipv6)
    if threading:
        httpd.daemon_threads = True
    httpd.set_app(wsgi_handler)
    httpd.serve_forever()
 </code></pre>
 
runserver命令具体做了哪些操作：
 - 解析参数，获取WSGIHandler
 <pre>
 通过django.core.servers.basehttp.get_internal_wsgi_application获取wsgi_handler,
 这里的get_internal_wsgi_application获取了settings.py中的WSGI_APPLICATION的值app.wsgi.application。
 这里settings.py、wsgi.py也是django创建项目时生成，app为创建的项目名称。
 在app.wsgi模块下获取application的值get_wsgi_handler(),返回WSGIHandler
 </pre>
 - 根据ip_address和port生成一个WSGIServer对象，接受用户请求
