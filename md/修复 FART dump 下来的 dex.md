> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/luoyesiqiu/p/dexfix.html)

1. 情景[#](#1-情景)
---------------

在使用脱壳机对 Android App 脱壳的时候，我们常常会得到有 nop 指令填充的函数的 dex，nop 是 No Operation 的缩写，意为无操作。

[![](https://img2020.cnblogs.com/blog/1456902/202010/1456902-20201020132557607-1999909018.jpg)](https://img2020.cnblogs.com/blog/1456902/202010/1456902-20201020132557607-1999909018.jpg)

上面是一个函数的 smali 代码，可以清晰的看到函数被多个 nop 指令填充，如果这个函数解析成 Java 代码，则得到一个空的函数：

[![](https://img2020.cnblogs.com/blog/1456902/202010/1456902-20201020132610165-1241482049.jpg)](https://img2020.cnblogs.com/blog/1456902/202010/1456902-20201020132610165-1241482049.jpg)

某些壳在加固的时候会把 dex 文件中的函数的真正函数体给抽掉，用 nop 指令来填充，nop 指令在这个过程中只用作占位，当函数执行的时候再把真正的代码给还原回去，加固壳用这种方式来保护函数中真正的代码。我们修复的过程就是把函数的真正代码写回 dex 文件的过程。函数修复后：

[![](https://img2020.cnblogs.com/blog/1456902/202010/1456902-20201020132622709-1965978952.jpg)](https://img2020.cnblogs.com/blog/1456902/202010/1456902-20201020132622709-1965978952.jpg)

2. 修复[#](#2-修复)
---------------

FART 在 dump dex 的同时，还把 dex 的 CodeItem 给 dump 了下来，这给我们修复 dex 提供了极大的便利，CodeItem 中存着函数中真正代码，CodeItem dump 下来后存在`.bin`文件中。所以我们修复的时候，读取`.bin`文件，把 CodeItem 填入 Dex 文件的相应的位置即可。

我们打开`.bin`文件，可以看到它由多项形如以下格式的文本组成，每一项代表一个函数

```
{name:void junit.textui.ResultPrinter.printFailures(junit.framework.TestResult),method_idx:1565,offset:52440,code_item_len:46,ins:BQACAAQAAADpYAIADwAAAG4QpwUEAAwAbhCmBQQACgEbAhYFAABuQBsGAyEOAA==};


```

我们来看这些数据都是什么：

*   `name` 指示函数的全名，包括完整的参数类型和返回值类型
*   `method_idx` 是函数在 method_ids 中的索引
*   `offset` 指示函数的 insns 相对于 dex 文件的偏移
*   `code_item_len` CodeItem 的长度
*   `ins` base64 字符串，解码后是 dex 结构中的 insns，即函数的真正的代码

在 dex 修复过程中，对我们有用的是 offset 和 ins，可以编写代码将它们从`.bin`文件中提取出来：

```
public static List<CodeItem> convertToCodeItems(byte[] bytes){
    String input = new String(bytes);

    List<CodeItem> codeItems = new ArrayList<>();
    Pattern pattern = Pattern.compile("\\{name:(.+?),method_idx:(\\d+),offset:(\\d+),code_item_len:(\\d+),ins:(.+?)\\}");
    Matcher matcher = pattern.matcher(input);
    while(matcher.find()){
        int offset = Integer.parseInt(matcher.group(3));
        String insBase64 = matcher.group(5);
        byte[] ins = Base64.getDecoder().decode(insBase64);
        CodeItem codeItem = new CodeItem(offset,ins);
        codeItems.add(codeItem);
    }

    return codeItems;
}


```

CodeItem 类：

```
public  class CodeItem{
    public CodeItem(long offset, byte[] byteCode) {
        this.offset = offset;
        this.byteCode = byteCode;
    }

    public long getOffset() {
        return offset;
    }

    public void setOffset(long offset) {
        this.offset = offset;
    }

    public byte[] getByteCode() {
        return byteCode;
    }

    public void setByteCode(byte[] byteCode) {
        this.byteCode = byteCode;
    }

    @Override
    public String toString() {
        return "CodeItem{" +
                "offset=" + offset +
                ", byteCode=" + Arrays.toString(byteCode) +
                '}';
    }

    private long offset;
    private byte[] byteCode;
}


```

接着将需要的修复 dex 复制一份，把 insns 填充到被复制出来的 dex 的相应位置，即修复过程：

```
public static void repair(String dexFile, List<CodeItem> codeItems){
    RandomAccessFile randomAccessFile = null;
    String outFile = dexFile.endsWith(".dex") ? dexFile.replaceAll("\\.dex","_repair.dex") : dexFile + "_repair.dex";
    //copy dex
    byte[] dexData = IoUtils.readFile(dexFile);
    IoUtils.writeFile(outFile,dexData);
    try{
        randomAccessFile = new RandomAccessFile(outFile,"rw");
        for(int i = 0 ; i < codeItems.size();i++){
            CodeItem codeItem = codeItems.get(i);
            randomAccessFile.seek(codeItem.getOffset());
            randomAccessFile.write(codeItem.getByteCode());
        }
    }
    catch (Exception e){
        e.printStackTrace();
    }
    finally {
        IoUtils.close(randomAccessFile);
    }
}


```

是不是很简单，本文完。

3. 完整代码[#](#3-完整代码)
-------------------

[https://github.com/luoyesiqiu/DexRepair](https://github.com/luoyesiqiu/DexRepair)

[![](http://images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)](//images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)

**关注微信公众号：luoyesiqiu，浏览更多内容**