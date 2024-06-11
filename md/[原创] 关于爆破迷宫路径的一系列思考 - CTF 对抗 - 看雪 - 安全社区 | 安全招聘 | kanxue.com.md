> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282091.htm)

> [原创] 关于爆破迷宫路径的一系列思考

前几天学长给了一道 maze 的题，动态调试过程可谓非常艰难 (对我现在的水平而言)，于是学长提供了一种用 subprocess 的方法

程序分析
----

首先通过字符串定位到主函数那里，我们发现 cmp 那里其实是一个长度校验，判断路径的长度是不是为 140  
![](https://bbs.kanxue.com/upload/attach/202406/994194_SHX9RY983J8E2HU.webp)  
这里为判断路径方向  
![](https://bbs.kanxue.com/upload/attach/202406/994194_K5RKRSSJ3A6RWSG.webp)  
中间过程太复杂，我们直接跳到后面，我们发现他判断所处位置是不是为 0，还有判断最后到达位置对不对的  
![](https://bbs.kanxue.com/upload/attach/202406/994194_QDJRXNA55YHRMC3.webp)  
经过分析我们并不知道迷宫在哪，因此我们才要尝试爆破的方法。

爆破处理
----

首先第一步，我们要把长度校验给 patch 掉  
![](https://bbs.kanxue.com/upload/attach/202406/994194_T77YHCJDPA8X568.webp)  
重点的 patch 在后面，我们为了一步一步进行爆破，我们要对每一步进行爆破  
但是有一个问题，后面不管走不走对都会输出  
我们可以先写一个简单的试验一下

```
import subprocess
printable=['w','s','a','d']
for i in range(len(printable)):
    sh=subprocess.Popen("./mapmap", shell=True, stdout=subprocess.PIPE, stdin=subprocess.PIPE)
    sh.stdin.write((printable[i]+'\n').encode())
    sh.stdin.flush()
    output ,_ =sh.communicate()
    print(output)

```

![](https://bbs.kanxue.com/upload/attach/202406/994194_GCG6U8MBJYXYSHH.webp)  
我们发现根本无法判断是不是走对了，这个时候我们再 patch 一下  
![](https://bbs.kanxue.com/upload/attach/202406/994194_33ZWDPFZYZ6JZHH.webp)  
![](https://bbs.kanxue.com/upload/attach/202406/994194_4N7EWSBWA4RYYUE.webp)  
把下面那个指向 wrong 的地址改为指向'what()'的地址  
我们这个时候可以再看一下  
![](https://bbs.kanxue.com/upload/attach/202406/994194_UZP5TCTAX6FSK9P.webp)  
发现有不一样的回显了，我们根据这个可以写一个脚本，但是这里有一个问题，它在程序里面每走一步不会把走过的给标记，所以我们得自己标记一下，由于没有写这个标记，导致脚本跑了好久也没跑出来。

Exp
---

```
import subprocess
 
def dfs(path, n, x_location, y_location, count, try_direction, visited_paths, coordinates):
    if n < count:
        for direction in try_direction:
            process = subprocess.Popen("./mapmap", shell=True, stdout=subprocess.PIPE, stdin=subprocess.PIPE)
            next_path = path + direction
            process.stdin.write((next_path + '\n').encode())
            process.stdin.flush()
            output, _ = process.communicate()
             
            if b'wrong\n' not in output:
                new_x_location, new_y_location = x_location, y_location
                if direction == 'd':
                    new_x_location += 1
                elif direction == 'a':
                    new_x_location -= 1
                elif direction == 'w':
                    new_y_location -= 1
                elif direction == 's':
                    new_y_location += 1
                if (new_x_location, new_y_location) not in coordinates:
                    print(path)
                    coordinates[(new_x_location, new_y_location)] = next_path
                    dfs(next_path, len(next_path), new_x_location, new_y_location, count, try_direction, visited_paths, coordinates)
    elif n == count:
            visited_paths.append(path)
            coordinates[(x_location, y_location)] = path
 
# DFS start
if __name__ == '__main__':
    start_path = ''
    try_direction = ['w', 'd', 's', 'a']
    count = 140
    visited_paths = []
    coordinates = {}
 
    x_location = 0
    y_location = 0
    coordinates[(x_location, y_location)] = start_path
    print("\nVisit paths:")
    dfs(start_path, len(start_path), x_location, y_location, count, try_direction, visited_paths, coordinates)
     
    print("\nFinal paths:")
    for path in visited_paths:
        process = subprocess.Popen("./mapmap", shell=True, stdout=subprocess.PIPE, stdin=subprocess.PIPE)
        process.stdin.write((path + '\n').encode())
        process.stdin.flush()
        output, _ = process.communicate()
        if b'flag{md5(your input)}\n' in output:
            print(path)

```

![](https://bbs.kanxue.com/upload/attach/202406/994194_7624WQBH97YT3R5.webp)  
差不多二十秒的时间跑出来

我自己的虚拟环境是 kali 2023.3 内存 4G 处理器 2x2

如果师傅们有更好的思路欢迎来交流

参考资料
----

[https://docs.python.org/zh-cn/3/library/subprocess.html#using-the-subprocess-module](https://docs.python.org/zh-cn/3/library/subprocess.html#using-the-subprocess-module)

[https://www.cnblogs.com/lordtianqiyi/articles/16588649.html](https://www.cnblogs.com/lordtianqiyi/articles/16588649.html)

[https://bbs.kanxue.com/thread-281796.html](https://bbs.kanxue.com/thread-281796.html)

[http://z221x.cc/index.php/2024/01/06/nctf-ezvm%e6%8f%92%e6%a1%a9%e7%88%86%e7%a0%b4%e7%9a%84%e5%a4%8d%e7%8e%b0/](http://z221x.cc/index.php/2024/01/06/nctf-ezvm%e6%8f%92%e6%a1%a9%e7%88%86%e7%a0%b4%e7%9a%84%e5%a4%8d%e7%8e%b0/)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工 作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 2 天前 被 5m10v3 编辑 ，原因：

[#Reverse](forum-37-1-111.htm)

上传的附件：

*   [mapmap.zip](javascript:void(0)) （801.55kb，1 次下载）