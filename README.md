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
 - 根据ip_address和port生成一个WSGIServer对象，接受用户请求
 <pre>
 通过django.core.servers.basehttp.get_internal_wsgi_application获取wsgi_handler,
 这里的get_internal_wsgi_application获取了settings.py中的WSGI_APPLICATION的值app.wsgi.application。
 这里settings.py、wsgi.py也是django创建项目时生成，app为创建的项目名称。
 在app.wsgi模块下获取application的值get_wsgi_handler(),返回WSGIHandler
 </pre>
 - 解析参数，获取WSGIHandler
 <pre>
 class WSGIHandler(base.BaseHandler):
    def __call__(self, environ, start_response):
	    pass

 </pre>
 WSGIHandler继承BaseHandler,主要做的事情导入项目中的 settings模块、导入 自定义例外类，最后调用自己的 load_middleware 方法（只有第一次请求时会调用 ），加载所有列在 MIDDLEWARE_CLASSES 中的 middleware 类并且内省它们
##### WSGI规范
<pre>
它规定应用是可调用对象(函数/方法)，然后它接受2个固定参数：一个是含有服务器端的环境变量，另一个是可调用对象，这个对象用来初始化响应，
给响应加上status code状态码和httpt头部，并且返回一个可调用对象。
def simplr_wsgi_app(environ, start_response):
	# 固定两个参数，django中也使用同样的变量名
	status = '200 OK'
	headers = [{'Content-type': 'text/plain'}]
	# 初始化响应, 必须在返回前调用
	start_response(status, headers)
	# 返回可迭代对象
	return ['hello world!']
</pre>

##### 中间件
一 个 middleware 类可以包括请求响应过程的四个阶段：request，view，response 和 exception。对应的方法：process_request，process_view， process_response 和 process_exception。我们在middleware中间件中定义其中的方法。Middleware是在Django BaseHandler的load_middleware方法执行时加载的，加载之后会建立四个列表作为处理器的实例变量：

  -  _request_middleware：process_request方法的列表
  -  _view_middleware：process_view方法的列表
  -  _response_middleware：process_response方法的列表
  -  b_exception_middleware：process_exception方法的列表
