---
layout: post
title: linux常用
description: linux常用
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

cut是一个选取命令，就是将一段数据经过分析，取出我们想要的.  
提取每一行的第3个字节: `who|cut -b 3`  
自定义分隔符，提取域：`cat /etc/passwd|head -n 5|cut -d : -f 1`  
不足：如果文件里面的某些域是由若干个空格来间隔的，那么用cut就有点麻烦了

head，默认显示开头前10行  
前5行：`head -n 5 file`或`head -5 file`  
不显示最后5行：`head -n -5 /etc/passwd`  
tail，默认显示最后10行  
后5行：`tail -n 5 file`或`tail -5 file`  

## 使用文件锁
避免脚本重入。

    readonly FLOCK=/usr/bin/flock
    readonly LOCKFILE_DIR=/var/lock
    readonly LOCK_FD=200
    
    # The locking function using flock to ensure the script is not running twice:
    lock() {
        local prefix=$1
        local fd=${2:-$LOCK_FD}
        local lock_file=$LOCKFILE_DIR/$prefix.lock

        # Create the lock file:
        eval "exec $fd>$lock_file"

        # Acquire the lock:
        $FLOCK -n $fd \
            && return 0 \
            || return 1
    }
    
    myexit() {
        local error_str="$@"

        echo $error_str
        exit 1
    }
    
    main() {
        lock $SCRIPTNAME \
          || myexit "Only one instance of $SCRIPTNAME can run at one time."
    }

    main
