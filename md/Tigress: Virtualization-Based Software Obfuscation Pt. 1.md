> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.mitchellzakocs.com](https://www.mitchellzakocs.com/blog/tigress)

> Virtualization-Based Obfuscation - A software obfuscation technique that transforms functions into cu......

*   [Terminology](#terminology)
*   [Introduction](#introduction)
*   [Setup](#setup)
*   [Analysis](#analysis)
    *   [VM Entry](#vm-entry)
    *   [Virtual Stack](#virtual-stack)
    *   [Local Variable Space](#local-variable-space)
    *   [Dispatch](#dispatch)
    *   [Opcodes](#opcodes)
    *   [Instruction Trace](#instruction-trace)
*   [Comparison Data](#comparison-data)
    *   [General](#general)
    *   [Strengths and Weaknesses](#strengths-and-weaknesses)
*   [Conclusion](#conclusion)

_Virtualization-Based Obfuscation_ - A software obfuscation technique that transforms functions into custom virtual machines that emulate the functionality of the original code

_Virtual Machine Entrypoint_ - A routine where the virtual machine is initialized, usually near the start of the virtualized function

_Virtual Instruction_ - An opcode and optional operand that is tied to a certain operation within the virtual machine

_Virtual Stack_ - A separate stack that is used solely by the virtual machine and manipulated through its instructions

_Virtual Instruction Pointer_ - A counter that stores the location of the virtual instruction that is currently executing

_Opcode Handlers_ - A routine that is tied to the execution of a certain opcode

_Dispatcher_ - A routine that matches opcodes to their respective opcode handler

_Context Variables_ - A location that holds data or a pointer to data that is essential to the execution of a VM (ex. Virtual Stack Pointer, Virtual Instruction Pointer, etc.)

Welcome to the first part of my multi-part series on virtualization-based obfuscators. I've recently been fascinated by the complexity of these software protection solutions, so I spent my summer studying the architecture, internals, and functionality of some of the most popular virtualization-based obfuscators out there. I was able to do this research through a summer internship with the SEFCOM Lab at Arizona State University; I greatly appreciate their support and funding.

To give a quick introduction to the topic, software virtualization (and obfuscation in general) is generally the largest barrier to entry for any software reverse engineer. It is frequently used by both good and bad actors to protect their software from the peering eyes of reverse engineers. Because of this, I hope that these write-ups not only serve to highlight the flaws of these protections but also help to improve the quality of them in the future. I also hope that these write-ups help inspire future research and innovation in the field of software obfuscation as it is as important as it is obscure.

Tigress is the first virtualization-based obfuscator I wanted to start with because it is simple, accessible, and free. This obfuscator was primarily developed by Christian Collberg at the University of Arizona for research purposes. To start analyzing Tigress, I needed to install the obfuscator and create a sample binary to analyze. While the source code for Tigress seems to be fairly hard to get ahold of, the pre-compiled binaries can be downloaded by just about anyone. After downloading the binaries from [here](https://tigress.wtf/download.html) on a Linux machine, some light setup needs to be done. The home directory for Tigress needs to be set in the TIGRESS_HOME environment variable and added to the _PATH_ variable as well. Instructions for this can be found in the _INSTALL_ file included with the binaries. Tigress differs from most software virtualizers due to the fact that it's a source obfuscator. Instead of taking a pre-compiled binary as input, it instead uses the C source code of the program you wish to virtualize. Not only does it do that, but it also gives you the virtualized source code of the program after it's been obfuscated. I chose one of the sample programs included with Tigress---test1.c---to begin experimenting with. The original source code for _test1.c_ can be seen below.

```
void fac(int n) {
  int s=1;
  int i;
  for (i=2; i<=n; i++) {
    s *= i;
  }
  printf("fac(%i)=%i\n",n,s);
}

void fib(int n) {
  int a=0;
  int b=1;
  int s=1;

  int i;
  for (i=1; i<n; i++) {
    s=a+b;
    a=b;
    b=s;
  }
  printf("fib(%i)=%i\n",n,s);
}

int main (int argc, char** argv) {
  fac(1);
  fib(1);
  fac(5);
  fib(5);
  fac(10);
  fib(10);
  return 0;
}


```

As you can see, the _fib_ function prints the _n_th Fibonacci number while the _fac_ function prints the factorial of _n_. Now that I had a sample program, it was time to obfuscate it using Tigress. The final command I used can be found below.

```
tigress \--Environment=x86\_:Linux:Gcc:4.6 
        \--Transform=Virtualize \--Functions=fib,fac \--out=test1_out.c test1.c


```

Let me quickly explain the parameters I chose here. The first two parameters are fairly self-explanatory; I wanted the binary to be compiled for Linux in x86 using GCC. There are other options available such as ARM and Clang, but I just wanted a standard Linux x86 binary to analyze. The next parameter specifies the transformations to apply to the binary. Since I'm mainly focused on the virtualization techniques of these obfuscators, I will only select that option for now, but I may investigate the other forms of obfuscation in a future write-up as [Tigress has many](https://tigress.wtf/transformations.html). The next parameter lets you indicate which functions you'd like to virtualize. I chose the two main functions in the test program: _fib_ and _fac_. The final parameter just lets you specify the file name for the obfuscated source code. After running the command, Tigress spits out two files: _test1_out.c_ and _a.out_. The first file is the virtualized version of the programs source code and the second file is the compiled version of the virtualized program (compiled with GCC, as specified). The obfuscated binaries and source code can be found [here](https://github.com/mzakocs/VirtualizationObfuscatorAnalysis/tree/main/Tigress). Now that I had a binary to analyze, it was time to load it into IDA Pro and get started with the analysis.

![](https://www.mitchellzakocs.com/blog/tigress/image6.png)

The *main* function of *test1* in IDA

Starting with the _main_ function in IDA, it looked fairly normal. There's no obfuscation to be seen on the surface, so I decided to keep digging. A function that caught my eye from the start was _megaInit_. I assumed that it was going to be some sort of initialization function for Tigress, but opening it in IDA didn't reveal too much.

![](https://www.mitchellzakocs.com/blog/tigress/image17.png)

The *megaInit* function of *test1* in IDA

![](https://www.mitchellzakocs.com/blog/tigress/image32.png)

Source code for the *megaInit* function in VSCode

This function seemed to do absolutely nothing. Looking at the obfuscated source code (_test1_out.c_), I started to understand why. It was literally a blank function. While this function may not be used in Tigress for the virtualization transformation, it may be used for any of the other ones. Either way, I decided to ignore it. The next function to investigate is _fac,_ the factorial function that I discussed earlier. This was one of the functions I selected to be virtualized while obfuscating the binary.

![](https://www.mitchellzakocs.com/blog/tigress/image23.png)

The fac function in IDA

Taking a quick look at this function, there is clearly something strange happening here. A small factorial function that was only 6 lines in the original source code has been expanded into a monstrous function with hundreds of conditional branches. We can assume that the virtualization-based obfuscation from Tigress had done its job.

VM Entry
--------

![](https://www.mitchellzakocs.com/blog/tigress/image16.png)

The entry point of the fac function

When analyzing virtual machines (or most software for that matter), it's usually a good idea to begin analysis at the start of the function and work your way down to the complex parts. I started analyzing the virtual machine entry point for _fac_ and found plenty of information about the virtual machine just by looking there. Starting in the first block, the inputted number for the function _n_ is moved into _rbp-0x144_. I named this location on the stack _inputNumber_. This instruction also shows that Tigress is using the stack to store local variables for virtual machine context (and accessing them relative to the stack frame, _rbp_). After this, I looked towards the bottom of the first block. I wanted to find the location of the bytecode for the virtual machine. Since symbols weren't stripped for this binary, it's pretty easy to spot where it is. A pointer to data labeled __1_fac_$array_ is moved into _rax_. This register now stores a pointer to the start of the virtualized bytecode. Directly afterward, a _mov_ instruction transfers this address to the stack variable _rbp-0x138_. This variable is now (fairly obviously) the virtual instruction pointer, so I named it _instructionPointer_. All of this incredibly useful information was gathered just by briefly looking over the entry point.

Virtual Stack
-------------

My next step was figuring out how the virtual stack was implemented by Tigress. Looking at the stack variables created in the entry point of the VM, there are _0x100_ bytes allocated on the stack at _rbp-130h_. This immediately caught my attention as it seemed like an appropriate size for a stack. Also, a pointer to this location gets loaded into _rbp-140h_ when the VM was initialized which is very likely a virtual stack pointer. Since there are no other large variables in the entry point, I started to assume that this was our stack. To confirm it, I needed to look at how _rbp-140h_ was being used by the VM.

![](https://www.mitchellzakocs.com/blog/tigress/image20.png)

An opcode routine that heavily utilizes [rbp-140h]

To find out if this space was our stack, I decided to find an opcode routine that utilizes it. At the very start of the routine (shown above), the _instructionPointer_ variable is being incremented by 1, meaning that my assumption was correct about this being the virtual instruction pointer. Since the instruction pointer was only incremented by 1, this is grabbing the byte after the opcode. This is likely an operand attached to the instruction that is used by the opcode handler. Afterward, _rbp-140h_ is moved into a register, incremented by 8, and the operand is pushed onto the stack. After this completes, the stack pointer is incremented by 8 again, the _instructionPointer_ is incremented by 4, and the virtual machine continues to the next instruction. From this routine, we can also see that the size of a standard instruction in this VM is 5 bytes: 1 byte for the opcode, then 4 bytes for an operand. Some of the instructions only use 1 byte as they don't need an operand, but I'll talk about that later. Also, since I've now confirmed that _rbp-140h_ is the virtual stack pointer, I'll rename the variable to _stackPointer._

One aspect of the virtual stack that is slightly unique is how it increments the stack pointer to push elements onto the stack instead of decrementing the pointer like a normal stack. This also means that the stack grows down instead of growing up like a normal stack. This doesn't change the functionality/usability of the stack in any way, but it's still cool to observe how the Tigress developers decided to implement it.

Local Variable Space
--------------------

There is one more part of the virtual machine that confused me initially. Declared at the entry point, there is some space saved on the stack at _rbp-30h_. This seems to be a 32-byte space that is only used in one or two of the opcode routines. I needed to investigate these routines to see what they could possibly be used for.

![](https://www.mitchellzakocs.com/blog/tigress/image31.png)

A routine that uses the variable at [rbp-30h]

In this routine, it seems like 3 separate values are pulled out of this data space and loaded into registers. After this, the values are fed into _printf_. The only other instruction that uses _rbp-30h_ pushes pointers to locations in the space into the stack. Through some assumptions, I concluded that _rbp-30h_ is where local variables are stored. From now on, I'll refer to _rbp-30h_ as _localVariableSpace._ I found this out by using the IDA debugger and looking at the values that are fed into _printf_. Referring to the original function, _[localVariableSpace+0x10]_ is the _"fac(%i)=%i\n"_ string, _[localVariableSpace+0x18]_ is _n_, and _[localVariableSpace+0x1C]_ is _s_.

Dispatch
--------

Now that I understood the virtual stack and local variable implementation, it was time to start analyzing the flow of the VM. After everything is initialized and the VM is ready to start executing, we get to the dispatch handler (shown below).

![](https://www.mitchellzakocs.com/blog/tigress/dispatch.png)

As you can see, this is an extremely simple dispatch handler. It mainly consists of long _cmp_ chains that match an opcode to its specific handler. There's unfortunately not much more to talk about here. This is an incredibly simple dispatcher that leads directly to the opcode handlers with no further obfuscations. Now that we know this, let's look at the opcode handlers that this dispatcher uses.

Opcodes
-------

![](https://www.mitchellzakocs.com/blog/tigress/image18.png)

All of the opcodes for fac

Now that I was starting to understand the overall flow of the VM, I decided to start analyzing the individual opcode routines. I could count about 15 just by looking at the graph view, so I knew it wasn't going to be very difficult. Since the opcode routines are fairly boring to analyze, I'll save you the hassle and list out what each opcode does in the table below. Note that some of the instructions are only 1 byte in size because they do not require extra operands for their functionality. This is very similar to [x86 machine code in that sense](http://www.c-jump.com/CIS77/CPU/x86/lecture.html); each instruction can be a different size based on the need for extra parameters.

<table><thead><tr><th>Opcode</th><th>Size</th><th>Behavior</th><th>Implementation</th></tr></thead><tbody><tr><td>0x1</td><td>1 Byte</td><td>Stores 4-byte value on the stack into an address on the stack</td><td>[[SP]] = [SP - 1]</td></tr><tr><td>0x3</td><td>5 Bytes</td><td>Pushes a pointer to the first parameter (&amp;n) onto stack</td><td>[SP] = &amp;n</td></tr><tr><td>0x7</td><td>1 Byte</td><td>Adds two values on stack</td><td>[SP - 1] = [SP - 1] + [SP]</td></tr><tr><td>0x3A</td><td>1 Byte</td><td>Moves a value on the stack into its same location (useless)</td><td>[SP] = [SP]</td></tr><tr><td>0x40</td><td>5 Bytes</td><td>Feeds three values on the local variable space into printf</td><td>printf((LV + 0x10), (LV + 0x18), (LV + 0x1C))</td></tr><tr><td>0x5A</td><td>1 Byte</td><td>VM Exit</td><td>retn</td></tr><tr><td>0x5B</td><td>5 Bytes</td><td>Instruction pointer jump</td><td>IP += [OPERAND]</td></tr><tr><td>0x64</td><td>5 Bytes</td><td>Pushes a pointer to a local variable onto stack</td><td>[SP + 1] = LV + [OPERAND]</td></tr><tr><td>0x67</td><td>1 Byte</td><td>Loads from an address on the stack</td><td>[SP] = [[SP]]</td></tr><tr><td>0x7B</td><td>5 Bytes</td><td>Pushes a pointer to a string in a string table onto the stack</td><td>[SP + 1] = StringTablePtr + [OPERAND]</td></tr><tr><td>0x84</td><td>5 Bytes</td><td>Pushes an IV from instruction operand onto stack</td><td>[SP + 1] = [IP]</td></tr><tr><td>0x88</td><td>1 Byte</td><td>Compares (&lt;=) two values on the stack</td><td>[SP -1] = [SP - 1] &lt;= [SP]</td></tr><tr><td>0x9A</td><td>5 Bytes</td><td>Jump to offset address from instruction parameters if value on stack is true (conditional jump)</td><td>if [SPv]:<br>IP += [OPERAND]</td></tr><tr><td>0xA3</td><td>1 Byte</td><td>Stores 8-byte value on the stack into a pointer on stack</td><td>[[SP]] = [SP - 1]<br>SP -= 2</td></tr><tr><td>0xBE</td><td>1 Byte</td><td>Multiplies two values on the stack</td><td>[SP - 1] = [SP - 1] ✕ [SP]</td></tr></tbody></table>

One thing that is interesting about Tigress is its opcode randomization. In the other function that I virtualized inside this binary, _fib_, the instruction set is completely different. If I had to guess, Tigress has a preset amount of instruction implementations that it assigns to random opcodes while virtualizing a function. After it has its randomized instruction set, it likely just translates the x86 code directly. This is a simple yet effective implementation for these types of randomized architectures.

Instruction Trace
-----------------

After I was done classifying all of the instructions, I started to analyze the actual virtualized program. I wanted to see how it utilized the virtual instructions to calculate a factorial the same way the original binary would. Similar to gathering info on the opcodes, this process is incredibly boring, so I'll just put all of the instructions in a table below as usual. I created this instruction trace using the Python 3 script found [here](https://github.com/mzakocs/VirtualizationObfuscatorAnalysis/blob/main/Tigress/trace.py).

<table><thead><tr><th>Instruction #</th><th>Opcode</th><th>Operand Bytes</th><th>Explanation</th></tr></thead><tbody><tr><td>1</td><td>0x84</td><td>0x1 0x0 0x0 0x0</td><td>Pushes 1 onto the stack</td></tr><tr><td>2</td><td>0x64</td><td>0x4 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x4 onto the stack</td></tr><tr><td>3</td><td>0x1</td><td></td><td>Loads 1 into local variable 0x4</td></tr><tr><td>4</td><td>0x84</td><td>0x2 0x0 0x0 0x0</td><td>Pushes the constant 2 onto the stack</td></tr><tr><td>5</td><td>0x64</td><td>0x8 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x8 onto the stack</td></tr><tr><td>6</td><td>0x1</td><td></td><td>Stores 2 into the local variable 0x8</td></tr><tr><td>7</td><td>0x5B</td><td>0x4 0x0 0x0 0x0</td><td>Adds 4 to the current instruction pointer</td></tr><tr><td>8</td><td>0x64</td><td>0x8 0x0 0x0 0x0</td><td>(JT) Pushes a pointer to local variable 0x8 onto the stack</td></tr><tr><td>9</td><td>0x67</td><td></td><td>Loads 2 from address on stack</td></tr><tr><td>10</td><td>0x3</td><td>0x0 0x0 0x0 0x0</td><td>Pushes a pointer to the first parameter (&amp;n) onto stack</td></tr><tr><td>11</td><td>0x67</td><td></td><td>Loads 1 from address on stack</td></tr><tr><td>12</td><td>0x88</td><td></td><td>Sees if [SP - 1] &lt;= [SP], evaluates false here</td></tr><tr><td>13</td><td>0x9A</td><td>0xE 0x0 0x0 0x0</td><td>Conditional jump to instruction 29, doesn't do it here</td></tr><tr><td>14</td><td>0x5B</td><td>0x4 0x0 0x0 0x0</td><td>Adds 0x4 to instruction pointer, just goes to next instruction</td></tr><tr><td>15</td><td>0x5B</td><td>0x33 0x0 0x0 0x0</td><td>Adds 0x33 to instruction pointer (jumps to instruction 31)</td></tr><tr><td>16</td><td>0x64</td><td>0x4 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x4 onto the stack</td></tr><tr><td>17</td><td>0x67</td><td></td><td>Loads the value of local variable 0x4 onto the stack</td></tr><tr><td>18</td><td>0x64</td><td>0x8 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x8 onto the stack</td></tr><tr><td>19</td><td>0x67</td><td></td><td>Loads the value of local variable 0x8 onto the stack</td></tr><tr><td>20</td><td>0xBE</td><td></td><td>Multiplies local variable 0x4 and 0x8 and stores the result on the stack</td></tr><tr><td>21</td><td>0x64</td><td>0x4 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x4 onto the stack</td></tr><tr><td>22</td><td>0x1</td><td></td><td>Puts the multiplication result into local variable 0x4</td></tr><tr><td>23</td><td>0x64</td><td>0x8 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x8 onto the stack</td></tr><tr><td>24</td><td>0x67</td><td></td><td>Loads the value of local variable 0x8 onto the stack</td></tr><tr><td>25</td><td>0x84</td><td>0x1 0x0 0x0 0x0</td><td>Pushes 1 onto the stack</td></tr><tr><td>26</td><td>0x7</td><td></td><td>Adds 1 to local variable 0x8</td></tr><tr><td>27</td><td>0x64</td><td>0x8 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x8 onto the stack</td></tr><tr><td>28</td><td>0x1</td><td></td><td>Puts incremented local variable 0x8 back into local variable 0x8</td></tr><tr><td>29</td><td>0x5B</td><td>0xBE 0xFF 0xFF 0xFF</td><td>Jumps to instruction 8 (LOOP)</td></tr><tr><td>30</td><td>0x5B</td><td>0xB9 0xFF 0xFF 0xFF</td><td>Not Reachable</td></tr><tr><td>31</td><td>0x7B</td><td>0x0 0x0 0x0 0x0</td><td>(JT) Pushes a pointer to the string 0x0 onto the stack</td></tr><tr><td>32</td><td>0x3A</td><td></td><td>Useless instruction</td></tr><tr><td>33</td><td>0x64</td><td>0x10 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x10 onto the stack</td></tr><tr><td>34</td><td>0xA3</td><td></td><td>Puts string pointer into local variable 0x10</td></tr><tr><td>35</td><td>0x3</td><td>0x0 0x0 0x0 0x0</td><td>Pushes a pointer to the first parameter (&amp;n) onto stack</td></tr><tr><td>36</td><td>0x67</td><td></td><td>Loads 1 onto stack from address on stack</td></tr><tr><td>37</td><td>0x64</td><td>0x18 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x18 onto the stack</td></tr><tr><td>38</td><td>0x1</td><td></td><td>Loads the value of n into local variable 0x18</td></tr><tr><td>39</td><td>0x64</td><td>0x4 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x4 onto the stack</td></tr><tr><td>40</td><td>0x67</td><td></td><td>Loads 1 into local variable 0x4</td></tr><tr><td>41</td><td>0x64</td><td>0x1C 0x0 0x0 0x0</td><td>Pushes a pointer to local variable 0x1C onto the stack</td></tr><tr><td>42</td><td>0x1</td><td></td><td>Loads value of local variable 0x4 into local variable 0x1C</td></tr><tr><td>43</td><td>0x40</td><td>0x1 0x0 0x0 0x0</td><td>Calls printf with local variables 0x10, 0x18, and 0x1C as parameters</td></tr><tr><td>44</td><td>0x5B</td><td>0x4 0x0 0x0 0x0</td><td>Adds 0x4 to instruction pointer, just goes to next instruction</td></tr><tr><td>45</td><td>0x5A</td><td></td><td>Exits VM</td></tr></tbody></table>

For this instruction trace, I made the mistake of recording the trace for the very first call of the _fac_ function: _fac(1)_. This means that the factorial calculation code never actually runs, but I could still analyze it statically. For this reason, instructions 16-29 are all theoretical as I never observed the execution of them (the factorial of 1 is just one, obviously). It's still fairly easy to see how the virtualized bytecode relates back to the original program. We can see that local variable _0x4_ is very likely the _s_ variable (in the original code) and that local variable _0x8_ is the _i_ variable for the loop. We can also see that local variables _0x10_, _0x18_, and _0x1c_ are used as parameters for _printf_, which explains why local variable _0x4_ gets moved into local variable _0x1C_ near the end of execution. This also explains why the string table value is loaded into local variable _0x10_ as the first parameter to _printf_ is always going to be the main string. With the loop code, it's fairly easy to see how this correlates to the original source code as well. It simply loads the value of _0x4_ and _0x8_ onto the virtual stack (variables _s_ and _i_) and multiplies them together continuously until _i_ equals _n_. All of these functionalities are extremely similar to their implementations in x86, so it seems like these virtual instructions are a direct conversion from x86 (like I guessed earlier).

Throughout this series, I will put some charts at the end of each write-up to allow you to easily compare the different obfuscators.

General
-------

<table><thead><tr><th>🝰</th><th>Tigress</th></tr></thead><tbody><tr><td>Obfuscation Layer</td><td>Source Code</td></tr><tr><td>Architecture Type</td><td>Stack Machine, Randomized Instruction Set</td></tr><tr><td>Context Variables</td><td>Stored on stack and accessed through stack frame offset.<br>Includes a local variable space, a virtual stack (and pointer to virtual stack), a virtual instruction pointer, and any parameters passed to the function</td></tr><tr><td>Dispatch Type</td><td>CMP Chains, can be changed in settings</td></tr><tr><td>Local Variables</td><td>Has its own space allocated on stack and accessed like an array</td></tr><tr><td>Virtual Stack</td><td>Grows positively, never needs to be resized</td></tr><tr><td>Static Obfuscations</td><td>None, can be changed in settings</td></tr><tr><td>Bytecode Encryption</td><td>None</td></tr><tr><td>External Functions</td><td>Creates an opcode that calls the function with local variables as parameters</td></tr></tbody></table>

Strengths and Weaknesses
------------------------

<table><thead><tr><th>Strengths</th><th>Weaknesses</th></tr></thead><tbody><tr><td>Free, Randomized Instruction Set, Incredibly Customizable</td><td>Relatively Simple VM, No Encryption, Only Works with C Source Code</td></tr></tbody></table>

Tigress is a standard & simple virtualization-based obfuscator that is a great introduction into the inner workings of virtual machines. It uses a stack for most of its operations, a separate space for local variables, and implements instructions with simple, consistent routines. Since Tigress uses C source code as its input and output, it will likely be fairly different from most other virtualization-based obfuscators which use pre-compiled binaries as input and output. I am interested to see how they all differ in functionality and effectiveness. Even though there are almost no defining features that make Tigress difficult to reverse, the VM would likely be much more complex if I enabled more settings while obfuscating the binary. Even then, I'm sure that I'll get my fair share of complexity when I reverse some commercial obfuscators in the near future. Wrapping it up, the clean architecture of Tigress was great practice for future projects and I'm glad that I took the time to analyze it.