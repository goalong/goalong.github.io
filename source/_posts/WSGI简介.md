---
title: WSGI简介
date: 2018-06-09
tags: 
---
WSGI是Web Server Gateway Interface的简称，它是一个规范，在 PEP 333中描述，规定了web server和python web 框架／应用 之间的标准接口, 其目的是提供一个相对简单而通用的接口来支持Web server和Web 框架的几乎所有交互，此外也方便了对中间件的支持（pre-request post-request)。通俗的讲，WSGI 规范了一种简单的接口，解耦了 server 和 application，使得双边的开发者更加专注自身特性的开发。

几乎所有的python web框架都实现了对wsgi的支持，例如flask, Django, Bottle等。

### WSGI Application

wsgi application是一个可调用的对象：一个函数，方法，类或者包含object.__call__的实例，这个可调用对象必须接受两个位置参数：

1. 一个字典包含各种环境变量
2. 一个可调用的函数，用来让application 发送http状态码和http头部给web server

此外它必须返回一个包装字符串而成的可迭代的对象作为响应的正文.

一个application的例子：

	def application(environ, start_response):  # environ是一个字典， start_response是一个函数
		response_body = "Request method:  %s" % environ['REQUEST_METHOD']
		status = '200 OK'
		response_headers = [('Content-Type', 'text/plain'),
								('Content-Length', str(len(response_body))]
		start_response(status, response_headers) # 调用start_response, 发送http状态码和头部
		return [respponse_body]  # 返回可迭代对象，内容是响应的正文
		
注意返回的是[response_body], 一个列表里面包含着字符串，当然也可以返回response_body, 不过这样会比较慢，因为每次迭代的时候只能读取一个字节，web server通过一次遍历来读取application返回的可迭代对象，然后将读取到的内容写入响应正文发送给客户端。

### Environment dict

	# Python's bundled WSGI server
	from wsgiref.simple_server import make_server
	
	def application (environ, start_response):
	
	    # Sorting and stringifying the environment key, value pairs
	    response_body = [
	        '%s: %s' % (key, value) for key, value in sorted(environ.items())
	    ]
	    response_body = '\n'.join(response_body)
	
	    status = '200 OK'
	    response_headers = [
	        ('Content-Type', 'text/plain'),
	        ('Content-Length', str(len(response_body)))
	    ]
	    start_response(status, response_headers)
	
	    return [response_body]
	
	# Instantiate the server
	httpd = make_server (
	    'localhost', # The host name
	    8000, # A port number where to wait for the request
	    application # The application object name, in this case a function
	)
	
	# Wait for a single request, serve it and quit
	httpd.handle_request()
	
  将上面的代码保存为environ.py, 然后在终端执行python environ.py, 便启动了一个会返回所有的环境变量的server，然后在浏览器输入localhost:8000来查看响应，可以看到环境变量包含了很多内容，例如请求的路径，方法等。
  
  
### WSGI Server

server端主要专注HTTP层面的业务，重点是接收HTTP请求和提供并发。每当收到 HTTP 请求，server/gateway 必须调用可调用对象，处理请求的核心流程是：

	iterable = app(environ, start_response)
	for data in iterable:
	   # send data to client
app 即是wsgi application, 在wsgi server 中调用app(environ, start_response)然后返回可迭代对象，遍历这个可迭代对象，然后将数据发送给客户端，这就是wsgi server的工作内容。


### WSGI 优缺点
优点：

1. 组件之间的高度解耦和多样的部署选择，理论上任意一个符合wsgi规范的Application都可以和任意一个符合wsgi的server来搭配部署，这带来了极大的灵活性。
2. 轻松支持多进程，多线程

缺点：
不支持跨服务器通信

### uWSGI
uWSGI是一个WSGI服务器，即uWSGI是一个实现了WSGI规范的服务器，它可以与支持WSGI接口的app进行通信, 类似的wsgi服务器还有Gunicorn等。

参考：

[An Introduction to the Python Web Server Gateway Interface (WSGI)](http://ivory.idyll.org/articles/wsgi-intro/what-is-wsgi.html)

[WSGI Tutorial](http://wsgi.tutorial.codepoint.net/)

