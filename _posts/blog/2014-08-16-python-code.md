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