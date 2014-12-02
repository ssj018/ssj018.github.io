---
layout: post
title: linux常用命令
description: linux常用命令
category: blog
---

声明：  
本博客欢迎转发，但请保留原作者信息!  
新浪微博：[@孔令贤HW](http://weibo.com/lingxiankong)；   
博客地址：<http://lingxiankong.github.io/>  
内容系本人学习、研究和总结，如有雷同，实属荣幸！

---

有的OS不支持像`ll`这样的快捷命令，添加`alias ll='ls -l'`到~/.bashrc

清理当前目录中pyc：`find . -name "*.pyc" -exec rm -f '{}' \;`  
还有一个更好的方法：`find . -name "*.pyc" -delete`  
参见[这里](http://www.slashroot.in/which-is-the-fastest-method-to-delete-files-in-linux)

循环观察一个命令的执行：`watch -d -n 5 "nova show 9e3619c7-ffa9-489b-9425-0b1a10968623"`，-d表示--difference, -n表示--interval