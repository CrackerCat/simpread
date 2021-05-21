> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267687.htm)

最近网上查了查还原平坦化相关的资料。看了无名侠大佬的文章。由于本人太菜了。好多地方看的有点懵懵懂懂的。最终还是自己摸索着尝试还原平坦化。写的哪里不对。还请大佬指正。

 

先贴上几位大佬的文章

 

[ARM64 OLLVM 反混淆][https://bbs.pediy.com/thread-252321.htm]

 

[利用符号执行去除控制流平坦化][https://security.tencent.com/index.php/blog/msg/112]

 

翻阅过大佬的文章后。本菜菜得到两个简单的结论。

 

**一、fla 主要功能是将 if else 这类的逻辑给转换成 while+switch 组合来处理。达到将简单的逻辑给复杂化，增加静态分析难度的目的。**

 

**二、模拟执行找出真实块之间的关联，patch 修改相应的代码。直接将真实块关联起来，从而实现反混淆**

 

接下来。我将采用先射箭后画靶的方式。来逐步的还原一个标准的 ollvm fla 的案例。

 

首先是准备我们的案例，如下是未混淆前的源码

```
std::string calcKey(std::string data){
    if (data.length()>10){
        data="ceshi";
    }else if(data.length()>30){
        data="ollvm";
    }else{
        data="fla";
    }
    return data;
}
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_ollvmdemo2_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
 
    std::string hello = "Hello from C++";
    hello=calcKey(hello);
 
    return env->NewStringUTF(hello.c_str());
}

```

然后是混淆的参数配置是`add_definitions("-mllvm -fla -mllvm -split -mllvm -split_num=3")`

 

贴上混淆后的样本：链接: https://pan.baidu.com/s/1Dd0T49xhsjgWrJw7PSTQFA 密码: rrem

 

然后贴上 ida 解析出来的代码

```
unsigned __int64 __usercall calcKey@(unsigned __int64 result@, __int64 a2@)
{
  signed int v2; // w8
  signed int v3; // w8
  unsigned __int64 v4; // [xsp+10h] [xbp-40h]
  __int64 v5; // [xsp+18h] [xbp-38h]
  signed int v6; // [xsp+24h] [xbp-2Ch]
  bool v7; // [xsp+3Fh] [xbp-11h]
  unsigned __int64 v8; // [xsp+40h] [xbp-10h]
  bool v9; // [xsp+4Fh] [xbp-1h]
 
  v6 = 910266824;
  v5 = a2;
  v4 = result;
  while ( v6 != -1929161090 )
  {
    switch ( v6 )
    {
      case -1578842400:
        v6 = 56578246;
        break;
      case -1521928633:
        v9 = v8 > 0x1E;
        v6 = -124633339;
        break;
      case -124633339:
        if ( v9 )
          v3 = 1872828810;
        else
          v3 = 1744272424;
        v6 = v3;
        break;
      case 56578246:
        v6 = 1089662144;
        break;
      case 256858465:
        if ( v7 )
          v2 = 1938015978;
        else
          v2 = 1363893773;
        v6 = v2;
        break;
      case 910266824:
        v6 = 2056492010;
        break;
      case 1089662144:
        result = sub_F8E4(v5, v4);
        v6 = -1929161090;
        break;
      case 1363893773:
        result = sub_F77C(v4);
        v8 = result;
        v6 = -1521928633;
        break;
      case 1493239651:
        v6 = 1089662144;
        break;
      case 1697837166:
        v6 = 56578246;
        break;
      case 1744272424:
        result = sub_F830(v4, "fla");
        v6 = 1697837166;
        break;
      case 1872828810:
        result = sub_F830(v4, "ollvm");
        v6 = -1578842400;
        break;
      case 1938015978:
        result = sub_F830(v4, "ceshi");
        v6 = 1493239651;
        break;
      case 2056492010:
        result = sub_F77C(v4);
        v7 = result > 0xA;
        v6 = 256858465;
        break;
    }
  }
  return result;
} 
```

我们对比看下混淆前和混淆后的代码。原本我们只需要看 if 条件就知道意义的。但是现在就需要不停的顺着条件。进入 case 一步步的分析每一步。如果是这么简单的例子。我们还是可以静态分析看的出来真正的条件。但是代码量过于庞大时就很难跟踪分析了。

[](#思路：先射箭后画靶)思路：先射箭后画靶
-----------------------

先简单说下如何先射箭后画靶，上面这个例子的逻辑比较简单。初学的情况，我们可以直接静态分析下，找出所有的真实块。然后用 ida 的 keypatch 插件来手动将真实块关联起来。再用 ida 解析看看结果是否和我们的混淆前的一致。结果一致后。我们就可以开始写 unidbg 代码来根据特征匹配真实块。特征匹配的结果可以和我们静态分析的结果对比看是否一致。最后只要 unidbg 实现出我们手动关联的效果，就完成了自动化反混淆了。下面我就先手动的还原。

[](#一、手动还原)一、手动还原
-----------------

1、找出所有真实块以及对应的汇编地址，标准的 ollvm 虚假块中一般只有简单的修改 v6 的值，其他的基本都是真实块

```
case -1521928633:                            //0xF694
        v9 = v8 > 0x1E;
        v6 = -124633339;
        break;
case -124633339:                            //0xF6BC
        if ( v9 )
          v3 = 1872828810;
        else
          v3 = 1744272424;
        v6 = v3;
        break;
case 256858465:                                //0xF624
        if ( v7 )
          v2 = 1938015978;
        else
          v2 = 1363893773;
        v6 = v2;
        break;
case 1089662144:                            //0xF750
        result = sub_F8E4(v5, v4);
        v6 = -1929161090;
        break;
case 1363893773:                            //0xF678
        result = sub_F77C(v4);
        v8 = result;
        v6 = -1521928633;
        break;
case 1744272424:                            //0xF710
        result = sub_F830(v4, "fla");
        v6 = 1697837166;
        break;
case 1872828810:                            //0xF6E0
        result = sub_F830(v4, "ollvm");
        v6 = -1578842400;
        break;
case 1938015978:                            //0xF648
        result = sub_F830(v4, "ceshi");
        v6 = 1493239651;
        break;
case 2056492010:                            //0xF5F8
                result = sub_F77C(v4);
        v7 = result > 0xA;
        v6 = 256858465;
        break;

```

找出所有真实块的地址后。接着就是顺着逻辑将他们全部串联起来。这里我举两个比较典型的例子。先从函数开始的地方开始。下面简单整理下第一个真实块的关联

 

根据 v6=910266824 第一次赋值。我们锁定到下面的 case 分支

```
unsigned __int64 __usercall calcKey@(unsigned __int64 result@, __int64 a2@)
{
  v6 = 910266824;
  v5 = a2;
  v4 = result;
  while ( v6 != -1929161090 )
  {
    switch ( v6 )
    {
      ....
      case 910266824:
        v6 = 2056492010;
        break;
      ....
      case 2056492010:                            //0xF5F8        第一个真实块
        result = sub_F77C(v4);
        v7 = result > 0xA;
        v6 = 256858465;
        break;
     ....
     case 256858465:                                //0xF624        第二个真实块
        if ( v7 )
          v2 = 1938015978;
        else
          v2 = 1363893773;
        v6 = v2;
    }
  }
  return result;
} 
```

根据上面的代码。我们第一个真实块。应该是直接跳转到 0xF5F8 这个位置。而第一个跳转的位置。应该是在 while 循环前。先贴上第一个 block 的汇编代码

```
.text:000000000000F470 loc_F470                            
.text:000000000000F470                 LDR             W8, [SP,#0x50+var_2C]
.text:000000000000F474                 MOV             W9, #0x567E
.text:000000000000F478                 MOVK            W9, #0x8D03,LSL#16
.text:000000000000F47C                 CMP             W8, W9
.text:000000000000F480                 STR             W8, [SP,#0x50+var_44]
.text:000000000000F484                 B.EQ            loc_F76C
.text:000000000000F488                 B               loc_F48C

```

我们可以直接第一句就跳转到真实指令。因为这里使用的 while 是 fla 混淆的所以这些数据我们并不需要去执行。修改代码如下

```
.text:000000000000F470 loc_F470                          
.text:000000000000F470                 LDR             W8, [SP,#0x50+var_2C]
.text:000000000000F474                 B               loc_F5F8 ; Keypatch modified this from:
.text:000000000000F474                                         ;   MOV W9, #0x567E
.text:000000000000F478 ; ---------------------------------------------------------------------------
.text:000000000000F478                 MOVK            W9, #0x8D03,LSL#16
.text:000000000000F47C                 CMP             W8, W9
.text:000000000000F480                 STR             W8, [SP,#0x50+var_44]
.text:000000000000F484                 B.EQ            loc_F76C
.text:000000000000F488                 B               loc_F48C

```

这样第一个块就关联好了。然后继续关联第二个真实块 0xF624。下面先列出第一个真实块的汇编

```
.text:000000000000F5F8 loc_F5F8                               
.text:000000000000F5F8                                    
.text:000000000000F5F8                 LDR             X0, [SP,#0x50+var_40]
.text:000000000000F5FC                 BL              sub_F77C
.text:000000000000F600                 CMP             X0, #0xA
.text:000000000000F604                 CSET            W8, HI
.text:000000000000F608                 MOV             W9, #1
.text:000000000000F60C                 AND             W8, W8, W9
.text:000000000000F610                 STURB           W8, [X29,#var_11]
.text:000000000000F614                 MOV             W8, #0x5961
.text:000000000000F618                 MOVK            W8, #0xF4F,LSL#16
.text:000000000000F61C                 STR             W8, [SP,#0x50+var_2C]
.text:000000000000F620                 B               loc_F778

```

汇编的最后是跳转到了 0xf778。这里就是又进入控制分发器了。我们直接让他跳转到第二个真实块。修改最后一行汇编如下

```
.text:000000000000F618                 MOVK            W8, #0xF4F,LSL#16
.text:000000000000F61C                 STR             W8, [SP,#0x50+var_2C]
.text:000000000000F620                 B               loc_F624 ; Keypatch modified this from:
.text:000000000000F620                                         ;   B loc_F778

```

第二个真实块这里就有点特殊了。这是一个分支让我们的第三个真实块可能是 0xF648 位置也可能是 0xF678 位置。我们先看看 c++ 的部分

```
case 256858465:
        if ( v7 )                //根据条件让第三个真实块不固定了
          v2 = 1938015978;
        else
          v2 = 1363893773;
        v6 = v2;

```

那么这里我们就不能简单的 b 跳转哪个真实块了。这里我的预想是把这块的代码修改成如下

```
case 256858465:
        if ( v7 )                //根据条件让第三个真实块不固定了
          //直接跳转到0xF648
        else
          //直接跳转到0xF678
        v6 = v2;

```

但是我们肯定不能修改 c++ 的代码。所以看看这个真实块的汇编部分

```
.text:000000000000F624 loc_F624                               
.text:000000000000F624                                        
.text:000000000000F624                 LDURB           W8, [X29,#var_11]
.text:000000000000F628                 MOV             W9, #0xC6EA
.text:000000000000F62C                 MOVK            W9, #0x7383,LSL#16
.text:000000000000F630                 MOV             W10, #0x5E0D
.text:000000000000F634                 MOVK            W10, #0x514B,LSL#16
.text:000000000000F638                 TST             W8, #1
.text:000000000000F63C                 CSEL            W8, W9, W10, NE
.text:000000000000F640                 STR             W8, [SP,#0x50+var_2C]
.text:000000000000F644                 B               loc_F778

```

这里可以看到那个 v7 的值就是 w8。所以我们修改成判断 w8=1。就跳转到 0xf648。否则跳转到 0xf678。下面贴上修改后的结果

```
.text:000000000000F624 loc_F624                               
.text:000000000000F624                                        
.text:000000000000F624                 LDURB           W8, [X29,#var_11]
.text:000000000000F628                 MOV             W9, #0xC6EA
.text:000000000000F62C                 MOVK            W9, #0x7383,LSL#16
.text:000000000000F630                 MOV             W10, #0x5E0D
.text:000000000000F634                 MOVK            W10, #0x514B,LSL#16
.text:000000000000F638                 CMP             W8, #1  ; Keypatch modified this from:
.text:000000000000F638                                         ;   TST W8, #1
.text:000000000000F63C                 B.EQ            loc_F648 ; Keypatch modified this from:
.text:000000000000F63C                                         ;   CSEL W8, W9, W10, NE
.text:000000000000F640                 B               loc_F678 ; Keypatch modified this from:
.text:000000000000F640                                         ;   STR W8, [SP,#0x50+var_2C]

```

再往后基本就都是这两种方式关联了。下面我就直接列一下手动修改后的跳转关联

```
0xF474        --->        0xF5F8
0xF620        --->        0xF624
0xF63C        --->        0xF648
0xF640        --->        0xF678
0xF664        --->        0xF750
0xF690        --->        0xF694
0xF6B8        --->        0xF6BC
0xF6D4        --->        0xF6E0
0xF6D8        --->        0xF710
0xF6FC        --->        0xF750

```

上面的跳转修改完后。最后把让我们 while 循环的跳转给改成 nop。

```
.text:000000000000F778 loc_F778                               
.text:000000000000F778                                       
.text:000000000000F778                 B               loc_F470            //修改成nop

```

最后我们 f5 解析一下。基本结果就差不多了。

```
void __usercall calcKey(unsigned __int64 result@, __int64 a2@)
{
  __int64 v2; // x0
  unsigned __int64 v3; // [xsp+10h] [xbp-40h]
  __int64 v4; // [xsp+18h] [xbp-38h]
  bool v5; // [xsp+4Fh] [xbp-1h]
 
  v4 = a2;
  v3 = result;
  if ( (unsigned __int64)sub_F77C(result) > 0xA == 1 )
  {
    sub_F830(v3, "ceshi");
  }
  else
  {
    v5 = (unsigned __int64)sub_F77C(v3) > 0x1E;
    if ( v5 == 1 )
      sub_F830(v3, "ollvm");
    else
      sub_F830(v3, "fla");
  }
  v2 = sub_F8E4(v4, v3);
  sub_F77C(v2);
} 
```

[](#二、自动化反混淆)二、自动化反混淆
---------------------

第一步先用 unidbg 跑通流程

```
public class FlaTest {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final DvmClass mainActivityDvm;
    public static void main(String[] args) {
        FlaTest bcfTest = new FlaTest();
        bcfTest.call_calckey();
    }
    private FlaTest(){
        emulator = AndroidEmulatorBuilder
                .for64Bit()
                .build();
        Memory memory = emulator.getMemory();
        LibraryResolver resolver = new AndroidResolver(23);
        memory.setLibraryResolver(resolver);
        vm = emulator.createDalvikVM(null);
        vm.setVerbose(false);
        mainActivityDvm = vm.resolveClass("com/example/ollvmdemo2/MainActivity");
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/resources/example_binaries/ollvm_fla/libnative-lib.so"), false);
        dm.callJNI_OnLoad(emulator);
 
    }
    //主动调用目标函数
    private void call_calckey(){
        StringObject res = mainActivityDvm.callStaticJniMethodObject(emulator, "stringFromJNI()Ljava/lang/String;");
        System.out.println(res.toString());
    }
}

```

先找找真实块和混淆块的特征，然后区分它们。可以看出来这个案例中的控制流程只要靠 v6 这个变量来驱动。真实块中有条件判断。有函数调用。而混淆块中只有对 v6 的修改。接下来看看混淆块的汇编代码是什么样的。当然不同的混淆参数也会导致特征不一样。要根据实际情况来进行调整。下面是一个混淆块

```
.text:000000000000F700 loc_F700 
.text:000000000000F700                 MOV             W8, #0x50C6
.text:000000000000F704                 MOVK            W8, #0x35F,LSL#16
.text:000000000000F708                 STR             W8, [SP,#0x50+var_2C]
.text:000000000000F70C                 B               loc_F778

```

这个例子的混淆特征就是一个 4 行指令的 block。mov、movk、str、b。由于我们看到例子中用于驱动控制流程的变量是同一个。所以 str 的写入地址也可以当做特征。如果无法通过混淆 block 的特征区分精准。就通过真实块的特征区分。比如 block 中有 bl 指令跳转到函数的是真实块。有运算符操作的是真实块。有条件判断的是真实块

 

能够区分出混淆块和真实块之后。最后的工作就是完成 block 之间的关系连接。找出所有的真实块。然后直接跳转过去

 

还原流程分析完后。接着列一下实现的步骤

 

1、将所有执行的 block 保存下来

 

2、遍历所有 block。将真实的 block 筛选出来

 

3、遍历所有真实 block。用 b 指令将他们串联起来。

 

下面是保存所有执行块的代码部分

```
    //用来保存所有执行过的block
    ArrayList> blocks=new ArrayList>();
//主动调用目标函数
private void call_calckey(){
      //这里BlockHook就是按照一个block的触发
    emulator.getBackend().hook_add_new(new BlockHook() {
        @Override
        public void hook(Backend backend, long address, int size, Object user) {
              //这里的insns是整个block。
            Capstone.CsInsn[] insns = emulator.disassemble(address, size,0);
            blocks.add(new Pair(address,insns));
        }
    },0x4000F44C,0x4000F44C+0x330,null);
    //调用一个返回值为object的静态的jni函数
    StringObject res = mainActivityDvm.callStaticJniMethodObject(emulator, "stringFromJNI()Ljava/lang/String;");
    System.out.println(res.toString());
    for(Pair block : blocks){
        System.out.println(String.format("block address:0x%x",block.getKey()) );
    }
} 
```

这样就将所有执行的 block 给保存了出来。其中保存的 address 就是块的第一个地址。后面我们连线到真实块。就是要直接跳到第一个地址。接下来筛选出所有的真实 block。这里我的筛选比较简单。能满足我这个例子。其他比较复杂需要调整下。

```
        //如果指令在队列中。则返回true。这是用来筛选block。如果block中有非以下指令的。说明很有可能是一个真实块
        private boolean OpstrContains(String opstr){
        ArrayList flags= new ArrayList(Arrays.asList("ldur","ldr","str","b","movz","movk","cmp","b.eq"));
        if(flags.contains(opstr)){
            return true;
        }
        return false;
    }
        //存储所有真实的block块
        ArrayList> readlyBlock=new ArrayList>();
        //过滤出真实块
        private void LoadReadlyAddress(){
          //第一个块肯定是要有的
        readlyBlock.add(blocks.get(0));
        for(int i=1;i pdata=blocks.get(i);
            Capstone.CsInsn[] insns=pdata.getValue();
            Long address=pdata.getKey();
            boolean isReadly=false;
              //遍历所有指令。检查出真实块。保存起来
            for(Capstone.CsInsn ins :insns){
                if(readlyBlock.contains(address)){
                    continue;
                }
                if(!OpstrContains(ins.mnemonic)){
                    isReadly=true;
                    String opstr= ARM.assembleDetail(emulator,ins,address,false,false);
                    System.out.println(String.format("block readly address:0x%x opstr:%s",ins.address,opstr) );
                    break;
                }
                else{
//                    String opstr= ARM.assembleDetail(emulator,ins,address,false,false);
//                    System.out.println(String.format("block fla address:0x%x opstr:%s",ins.address,opstr) );
                }
            }
            if(isReadly){
                readlyBlock.add(new Pair(address,insns));
            }
        }
    } 
```

获取到真实块之后。我们就需要进行跳转。首先处理我们的第一种关联方式。分支的关联方式比较复杂我们放的下面再说

```
                LoadReadlyAddress();
        String modulePath="unidbg-android/src/test/resources/example_binaries/ollvm_fla/libnative-lib.so";
        byte[] sodata=readFile(modulePath);
        //遍历真实块。然后直接修改成跳转真实块
        for(int i=0;iblock=readlyBlock.get(i);
                System.out.println(String.format("curBlock address:0x%x",block.getKey()));
                //获取下一个真实块
                PairnextBlock=readlyBlock.get(i+1);
 
                //取出当前真实块最后一个指令的地址
                int end_address= GetEndAddress(block.getValue()).intValue();
                if(end_address<=0){
                    continue;
                }
                //获取下一个真实块第一个指令的地址
                int start_address=GetStartAddress(nextBlock.getValue()).intValue();
                //这里是最后跳转出结束的地方。由于已经被我们nop掉了循环。所以如果下一个块是跳转出while的。可以直接跳过了
                if(nextBlock.getKey().intValue()==0x4000f76c){
                    continue;
                }
 
                //准备转换汇编代码进行替换
                try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm64, KeystoneMode.LittleEndian)) {
                    int subAddress=start_address-end_address;
                    //用来patch修改的asm指令。这里是要计算出当前地址的相对地址跳转。所以上面要减一下。
                    String asmStr=String.format("b #0x%x",subAddress);
                    //这个是我们显示日志看结果的。看看和我们之前手动分析的是不是差不多
                    String showStr=String.format("b #0x%x",start_address);
//                    System.out.println(String.format("address:0x%x chang asm:%s",end_address,showStr));
                    //转换汇编
                    KeystoneEncoded encoded = keystone.assemble(asmStr);
                    byte[] patch = encoded.getMachineCode();
                    if (patch.length <=0) {
                        System.out.println("转换汇编失败");
                        return;
                    }
//                    System.out.println(bytesToHexString(patch));
                    //替换原来的字节数据
                    for(int y =0;y
```

然后看看日志。因为我们之前已经静态分析解混淆过一次了。所以结果我们是清楚的。那么下面的结果对比我们手动的答案要不同

```
address:0xf484 chang asm:b #0xf5f8
address:0xf620 chang asm:b #0xf624
address:0xf644 chang asm:b #0xf648
address:0xf654 chang asm:b #0xf750
address:0xf758 chang asm:b #0xf76c

```

问题其实就是因为我们没有处理分支。只是单纯根据执行结果在跳转。那么这里我们需要判断一下 csel 来特殊处理一下。

 

需要处理的地方有两处。

 

1、分支的当前关联。也就是仿照我们手动关联时的 cmp + b.eq + b 来实现跳转两个地方

 

2、未执行分支的后续关联。手动关联是我们虽然不知道执行哪个分支。但是看 v6 的变化可以找到下一个真实块。但是自动化时。我们前面获取到的真实块。都是基于执行流程的。未执行部分的 block 块我们并未保存下来。所以我们要不就是一步步解析未执行分支的后续关联。要不就是直接修改分支判断的寄存器。把另一条分支的执行块也给记录下来。

 

我们先调整下 hook 部分的代码。将分支不同的执行块结果保存出来。

```
                emulator.getBackend().hook_add_new(new BlockHook() {
            @Override
            public void hook(Backend backend, long address, int size, Object user) {
                //这里的insns是整个block。
                Capstone.CsInsn[] insns = emulator.disassemble(address, size,0);
//                System.out.println(String.format("address:0x%x size:0x%x",address,insns.length));
                if(brance_data==0){
                    blocks.add(new Pair(address,insns));
                }else{
                    branchBlocks.add(new Pair(address,insns));
                }
 
            }
        },0x4000F44C,0x4000F778,null);
        //碰到csel的时候把w8的值修改下。控制走其他分支
        emulator.getBackend().hook_add_new(new CodeHook() {
            @Override
            public void hook(Backend backend, long address, int size, Object user) {
                //这里的insns是整个block。
                Capstone.CsInsn[] insns = emulator.disassemble(address, size,0);
                for(Capstone.CsInsn ins :insns){
                    //这里就是控制如果brance_data为0就固定走第一个分支。为1就固定走第二个分支
                    if(ins.mnemonic.equals("csel")){
                        int w9=emulator.getBackend().reg_read(Arm64Const.UC_ARM64_REG_W9).intValue();
                        int w10=emulator.getBackend().reg_read(Arm64Const.UC_ARM64_REG_W10).intValue();
                        if(brance_data==0){
                            emulator.getBackend().reg_write(Arm64Const.UC_ARM64_REG_W10,w9);
                        }else{
                            emulator.getBackend().reg_write(Arm64Const.UC_ARM64_REG_W9,w10);
                        }
                    }
                }
            }
        },0x4000F44C,0x4000F778,null);
 
        //调用一个返回值为object的静态的jni函数
        StringObject res = mainActivityDvm.callStaticJniMethodObject(emulator, "stringFromJNI()Ljava/lang/String;");
        System.out.println(res.toString());
        brance_data=1;
        StringObject res2 = mainActivityDvm.callStaticJniMethodObject(emulator, "stringFromJNI()Ljava/lang/String;");
        System.out.println(res2.toString());
        LoadReadlyAddress(); 
```

然后就可以在跳转的地方获取到两个跳转的真实块地址了。然后打上 patch 替换掉原来的三行汇编

```
                                //获取当前块中的csel地址。如果没有csel则为0
                Long cselAddress=GetCselddress(block.getValue());
                //存在csel指令说明是有分支。然后要特殊处理
                if(cselAddress>0){
                    //获取另一个分支的下一个真实块
                    int start_address2=GetBranchReadlyAddress(block.getKey());
                    try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm64, KeystoneMode.LittleEndian)){
                        /*
                        *要改以下三行的汇编。前面获取出来的cselAddress是CSEL的地址。
                        * TST             W8, #1
                        * CSEL            W8, W9, W10, NE
                        * STR             W8, [SP,#0x50+var_2C]
                        * */
                        int subAddress1=start_address-(cselAddress.intValue())+4;
                        int subAddress2=start_address2-(cselAddress.intValue())+4;
                        String asm1="cmp w8,#0x1";
                        String showAsm1=String.format("b.eq 0x%x",start_address);
                        String showAsm2=String.format("b #0x%x",start_address2);
//                        System.out.println(String.format("address:0x%x chang asm1:%s",cselAddress,showAsm1));
//                        System.out.println(String.format("address:0x%x chang asm2:%s",cselAddress,showAsm2));
                        String asm2=String.format("b.eq 0x%x",subAddress1);
                        String asm3=String.format("b 0x%x",subAddress2);
                        //转换汇编
                        KeystoneEncoded encoded = keystone.assemble(Arrays.asList(asm1,asm2,asm3));
                        byte[] patch = encoded.getMachineCode();
                        if (patch.length <=0) {
                            System.out.println("转换汇编失败");
                            return;
                        }
                        System.out.println(bytesToHexString(patch));
                        //这里去掉基址。再往上一个指令的位置开始写入
                        int replace_address=cselAddress.intValue()-4;
                        //替换原来的字节数据
                        for(int y =0;y
```

贴上执行完成后解析的结果

```
__int64 __usercall calcKey@(__int64 a1@, __int64 a2@)
{
  __int64 v2; // x0
  __int64 v4; // [xsp+10h] [xbp-40h]
  __int64 v5; // [xsp+18h] [xbp-38h]
 
  v5 = a2;
  v4 = a1;
  if ( (unsigned __int64)sub_F77C(a1) > 0xA == 1 )
  {
    sub_F830(v4, "ceshi");
    v2 = sub_F8E4(v5, v4);
  }
  else
  {
    v2 = sub_F77C(v4);
  }
  return sub_F77C(v2);
} 
```

对比之前我们手动还原的结果看了下。还是差了点。但是当前的执行流程的反混淆结果可以看了。未执行部分的分支如何去关联我想了好久越想越复杂了。就没有继续倒腾了。有知道的大佬麻烦提点下。

 

中间还有一些细节处理的地方没有贴代码，我就直接丢上地址了。代码我都尽量写上注释了。

 

https://github.com/dqzg12300/unidbg_tools.git

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)