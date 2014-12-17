---
layout: post
title: python代码片段
description: python代码片段
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@孔令贤HW](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

---

## 获取一个类的所有子类
代码来源：rally

    def itersubclasses(cls, _seen=None):
        """Generator over all subclasses of a given class in depth first order."""

        if not isinstance(cls, type):
            raise TypeError(_('itersubclasses must be called with '
                              'new-style classes, not %.100r') % cls)
        _seen = _seen or set()
        try:
            subs = cls.__subclasses__()
        except TypeError:   # fails only when cls is type
            subs = cls.__subclasses__(cls)
        for sub in subs:
            if sub not in _seen:
                _seen.add(sub)
                yield sub
                for sub in itersubclasses(sub, _seen):
                    yield sub
                    
## 简单的线程配合
代码来源：rally

    import threading
    
    is_done = threading.Event()
    consumer = threading.Thread(
        target=self.consume_results,
        args=(key, self.task, runner.result_queue, is_done))
    consumer.start()
    self.duration = runner.run(
            name, kw.get("context", {}), kw.get("args", {}))
    is_done.set()
    consumer.join() #主线程堵塞，直到consumer运行结束

多说一点，threading.Event()也可以被替换为threading.Condition()，condition有notify(), wait(), notifyAll()。解释如下：

> The wait() method releases the lock, and then blocks until it is awakened by a notify() or notifyAll() call for the same condition variable in another thread. Once awakened, it re-acquires the lock and returns. It is also possible to specify a timeout.  
The notify() method wakes up one of the threads waiting for the condition variable, if any are waiting. The notifyAll() method wakes up all threads waiting for the condition variable.  
Note: the notify() and notifyAll() methods don’t release the lock; this means that the thread or threads awakened will not return from their wait() call immediately, but only when the thread that called notify() or notifyAll() finally relinquishes ownership of the lock.

    # Consume one item
    cv.acquire()
    while not an_item_is_available():
        cv.wait()
    get_an_available_item()
    cv.release()

    # Produce one item
    cv.acquire()
    make_an_item_available()
    cv.notify()
    cv.release()
    
## 计算运行时间
代码来源：rally

    class Timer(object):
        def __enter__(self):
            self.error = None
            self.start = time.time()
            return self

        def __exit__(self, type, value, tb):
            self.finish = time.time()
            if type:
                self.error = (type, value, tb)

        def duration(self):
            return self.finish - self.start

    with Timer() as timer:
        func()
    return timer.duration()
    
## 元类
参考来源：<http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386820064557c69858840b4c48d2b8411bc2ea9099ba000>

`__new__()`方法接收到的参数依次是：  
当前准备创建的类的对象；  
类的名字；  
类继承的父类集合；  
类的方法集合；

    class ModelMetaclass(type):
        def __new__(cls, name, bases, attrs):
            if name=='Model':
                return type.__new__(cls, name, bases, attrs)
            mappings = dict()
            for k, v in attrs.iteritems():
                if isinstance(v, Field):
                    print('Found mapping: %s==>%s' % (k, v))
                    mappings[k] = v
            for k in mappings.iterkeys():
                attrs.pop(k)
            attrs['__table__'] = name # 假设表名和类名一致
            attrs['__mappings__'] = mappings # 保存属性和列的映射关系
            return type.__new__(cls, name, bases, attrs)
            
    class Model(dict):
        __metaclass__ = ModelMetaclass

        def __init__(self, **kw):
            super(Model, self).__init__(**kw)

        def __getattr__(self, key):
            try:
                return self[key]
            except KeyError:
                raise AttributeError(r"'Model' object has no attribute '%s'" % key)

        def __setattr__(self, key, value):
            self[key] = value

        def save(self):
            fields = []
            params = []
            args = []
            for k, v in self.__mappings__.iteritems():
                fields.append(v.name)
                params.append('?')
                args.append(getattr(self, k, None))
            sql = 'insert into %s (%s) values (%s)' % (self.__table__, ','.join(fields), ','.join(params))
            print('SQL: %s' % sql)
            print('ARGS: %s' % str(args))
            
    class Field(object):
        def __init__(self, name, column_type):
            self.name = name
            self.column_type = column_type
        def __str__(self):
            return '<%s:%s>' % (self.__class__.__name__, self.name)
            
    class StringField(Field):
        def __init__(self, name):
            super(StringField, self).__init__(name, 'varchar(100)')

    class IntegerField(Field):
        def __init__(self, name):
            super(IntegerField, self).__init__(name, 'bigint')
            
    class User(Model):
        # 定义类的属性到列的映射：
        id = IntegerField('id')
        name = StringField('username')
        email = StringField('email')
        password = StringField('password')

    # 创建一个实例：
    u = User(id=12345, name='Michael', email='test@orm.org', password='my-pwd')
    # 保存到数据库：
    u.save()
    
输出如下：
    
    Found model: User
    Found mapping: email ==> <StringField:email>
    Found mapping: password ==> <StringField:password>
    Found mapping: id ==> <IntegerField:uid>
    Found mapping: name ==> <StringField:username>
    SQL: insert into User (password,email,username,uid) values (?,?,?,?)
    ARGS: ['my-pwd', 'test@orm.org', 'Michael', 12345]
    
## SQLAlchemy简单使用

    # 导入:
    from sqlalchemy import Column, String, create_engine
    from sqlalchemy.orm import sessionmaker
    from sqlalchemy.ext.declarative import declarative_base

    # 创建对象的基类:
    Base = declarative_base()

    # 定义User对象:
    class User(Base):
        # 表的名字:
        __tablename__ = 'user'

        # 表的结构:
        id = Column(String(20), primary_key=True)
        name = Column(String(20))

    # 初始化数据库连接:
    engine = create_engine('mysql+mysqlconnector://root:password@localhost:3306/test') # '数据库类型+数据库驱动名称://用户名:口令@机器地址:端口号/数据库名'
    # 创建DBSession类型:
    DBSession = sessionmaker(bind=engine)
    # 创建新User对象:
    new_user = User(id='5', name='Bob')
    # 添加到session:
    session.add(new_user)
    # 提交即保存到数据库:
    session.commit()
    # 创建Query查询，filter是where条件，最后调用one()返回唯一行，如果调用all()则返回所有行:
    user = session.query(User).filter(User.id=='5').one()
    # 关闭session:
    session.close()
    
## WSGI简单使用和Web框架Flask的简单使用

    from wsgiref.simple_server import make_server

    def application(environ, start_response):
        start_response('200 OK', [('Content-Type', 'text/html')])
        return '<h1>Hello, web!</h1>'

    # 创建一个服务器，IP地址为空，端口是8000，处理函数是application:
    httpd = make_server('', 8000, application)
    print "Serving HTTP on port 8000..."
    # 开始监听HTTP请求:
    httpd.serve_forever()

了解了WSGI框架，我们发现：其实一个Web App，就是写一个WSGI的处理函数，针对每个HTTP请求进行响应。  
但是如何处理HTTP请求不是问题，问题是如何处理100个不同的URL。  
一个最简单和最土的想法是从environ变量里取出HTTP请求的信息，然后逐个判断。
    
    from flask import Flask
    from flask import request

    app = Flask(__name__)

    @app.route('/', methods=['GET', 'POST'])
    def home():
        return '<h1>Home</h1>'

    @app.route('/signin', methods=['GET'])
    def signin_form():
        return '''<form action="/signin" method="post">
                  <p><input name="username"></p>
                  <p><input name="password" type="password"></p>
                  <p><button type="submit">Sign In</button></p>
                  </form>'''

    @app.route('/signin', methods=['POST'])
    def signin():
        # 需要从request对象读取表单内容：
        if request.form['username']=='admin' and request.form['password']=='password':
            return '<h3>Hello, admin!</h3>'
        return '<h3>Bad username or password.</h3>'

    if __name__ == '__main__':
        app.run()
        
## 格式化显示json

    print(json.dumps(data, indent=4))
    # 或者
    import pprint
    pprint.pprint(data)

## 实现类似Java或C中的枚举
代码来源，Rally

    #!/usr/bin/env python
    # -*- coding: utf-8 -*-

    import itertools
    import sys

    class ImmutableMixin(object):
        _inited = False

        def __init__(self):
            self._inited = True

        def __setattr__(self, key, value):
            if self._inited:
                raise Exception("unsupported action")
            super(ImmutableMixin, self).__setattr__(key, value)

    class EnumMixin(object):
        def __iter__(self):
            for k, v in itertools.imap(lambda x: (x, getattr(self, x)), dir(self)):
                if not k.startswith('_'):
                    yield v

    class _RunnerType(ImmutableMixin, EnumMixin):
        SERIAL = "serial"
        CONSTANT = "constant"
        CONSTANT_FOR_DURATION = "constant_for_duration"
        RPS = "rps"

    if __name__=="__main__":
        print _RunnerType.CONSTANT
        
## 创建文件时指定权限
代码来源，Nova

    import os

    def write_to_file(path, contents, umask=None):
        """Write the given contents to a file

        :param path: Destination file
        :param contents: Desired contents of the file
        :param umask: Umask to set when creating this file (will be reset)
        """
        if umask:
            saved_umask = os.umask(umask)

        try:
            with open(path, 'w') as f:
                f.write(contents)
        finally:
            if umask:
                os.umask(saved_umask)
                
    if __name__ == '__main__':
        write_to_file('/home/kong/tmp', 'test', 31)
        # Then you will see a file is created with permission 640.
        # Warning: If the file already exists, its permission will not be changed.
        # Note：For file, default all permission is 666, and 777 for directory.
        
## 多进程并发执行
代码来源：Rally

    import multiprocessing
    import time
    import os

    def run(flag):
        print "flag: %s, sleep 2s in run" % flag
        time.sleep(2)
        print "%s exist" % flag
        return flag
        
    if __name__ == '__main__':
        pool = multiprocessing.Pool(3)
        iter_result = pool.imap(run, xrange(6))
        print "sleep 5s\n\n"
        time.sleep(5)
        for i in range(6):
            try:
                result = iter_result.next(600)
            except multiprocessing.TimeoutError as e:
                raise
            
            print result
                
        pool.close()
        pool.join()
		
## 运行时自动填充函数参数
代码来源：Rally

	import decorator
	
	def default_from_global(arg_name, env_name):
	    def default_from_global(f, *args, **kwargs):
	        id_arg_index = f.func_code.co_varnames.index(arg_name)
	        args = list(args)
	        if args[id_arg_index] is None:
	            args[id_arg_index] = get_global(env_name)
	            if not args[id_arg_index]:
	                print("Missing argument: --%(arg_name)s" % {"arg_name": arg_name})
	                return(1)
	        return f(*args, **kwargs)
	    return decorator.decorator(default_from_global)

	# 如下是一个装饰器，可以用在需要自动填充参数的函数上。功能是：
	# 如果没有传递函数的deploy_id参数，那么就从环境变量中获取（调用自定义的get_global函数）
	with_default_deploy_id = default_from_global('deploy_id', ENV_DEPLOYMENT)    
    
## 嵌套装饰器
代码来源：Rally  

    validator函数装饰func1，func1使用时接收参数(*arg, **kwargs)，而func1又装饰func2（其实就是Rally中的scenario函数），给func2增加validators属性，是一个函数的列表，函数的接收参数config, clients, task。这些函数最终调用func1，传入参数（config, clients, task, *args, **kwargs），所以func1定义时参数是（config, clients, task, *arg, **kwargs）  
    最终实现的效果是，func2有很多装饰器，每个都会接收自己的参数，做一些校验工作。

    def validator(fn):
        """Decorator that constructs a scenario validator from given function.

        Decorated function should return ValidationResult on error.

        :param fn: function that performs validation
        :returns: rally scenario validator
        """
        def wrap_given(*args, **kwargs):
            """Dynamic validation decorator for scenario.

            :param args: the arguments of the decorator of the benchmark scenario
            ex. @my_decorator("arg1"), then args = ('arg1',)
            :param kwargs: the keyword arguments of the decorator of the scenario
            ex. @my_decorator(kwarg1="kwarg1"), then kwargs = {"kwarg1": "kwarg1"}
            """
            def wrap_validator(config, clients, task):
                return (fn(config, clients, task, *args, **kwargs) or
                        ValidationResult())

            def wrap_scenario(scenario):
                wrap_validator.permission = getattr(fn, "permission",
                                                    consts.EndpointPermission.USER)
                if not hasattr(scenario, "validators"):
                    scenario.validators = []
                scenario.validators.append(wrap_validator)
                return scenario

            return wrap_scenario

        return wrap_given