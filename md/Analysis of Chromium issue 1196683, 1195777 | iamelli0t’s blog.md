> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [iamelli0t.github.io](https://iamelli0t.github.io/2021/04/20/Chromium-Issue-1196683-1195777.html)

On April 12, a code commit[1] in Chromium get people’s attention. This is a bugfix for some vulnerability in Chromium Javascript engine v8. At the same time, the regression test case regress-1196683.js for this bugfix was also submitted. Based on this regression test case, some security researcher published an exploit sample[2]. Due to Chrome release pipeline, the vulnerability wasn’t been fixed in Chrome stable update until April 13[3].  

Coincidentally, on April 15, another code commit[4] of some bugfix in v8 has also included one regression test case regress-1195777.js. Based on this test case, the exploit sample was exposed again[5]. Since the latest Chrome stable version does not pull this bugfix commit, the sample can still exploit in render process of latest Chrome. When the vulnerable Chormium browser accesses a malicious link without enabling the sandbox (–no-sandbox), the vulnerability will be triggered and caused remote code execution.  

RCA of Issue 1196683
--------------------

The bugfix for this issue is shown as follows:  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/1.png)  

This commit fixes the issue of incorrect instruction selection for the ChangeInt32ToInt64 node in the instruction selection phase of v8 TurboFan. Before the commit, instruction selection is according to the input node type of the ChangeInt32ToInt64 node. If the input node type is a signed integer, it selects the instruction X64Movsxlq for sign extension, otherwise it selects X64Movl for zero extension. After the bugfix, X64Movsxlq will be selected for sign extension regardless of the input node type.  

First, let’s analyze the root cause of this vulnerability via regress-1196683.js:  

```
(function() {
  const arr = new Uint32Array([2**31]);
  function foo() {
    return (arr[0] ^ 0) + 1;
  }
  %PrepareFunctionForOptimization(foo);
  assertEquals(-(2**31) + 1, foo());
  %OptimizeFunctionOnNextCall(foo);
  assertEquals(-(2**31) + 1, foo());
});


```

The foo function that triggers JIT has only one code line. Let’s focus on the optimization process of (arr[0] ^ 0) + 1 in the key phases of TurboFan:

**1) TyperPhase**  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/2.png)

The XOR operator corresponds to node 32, and its two inputs are the constant 0 (node 24) and arr[0] (node 80).  

**2) SimplifiedLoweringPhase**  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/3.png)  

The original node 32: SpeculativeNumberBitwiseXor is optimized to Word32Xor, and the successor node ChangeInt32ToInt64 is added after Word32Xor. At this time, the input node (Word32Xor) type of ChangeInt32ToInt64 is Signed32.  

**3) EarlyOptimizationPhase**  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/4.png)  

We can see that the original node 32 (Word32Xor) has been deleted and replaced with node 80 as the input of the node 110 ChangeInt32ToInt64. Now the input node (LoadTypedElement) type of ChangeInt32ToInt64 is **Unsigned32**.  

The v8 code corresponding to the logic is as follows:  

```
template <typename WordNAdapter>
Reduction MachineOperatorReducer::ReduceWordNXor(Node* node) {
  using A = WordNAdapter;
  A a(this);

  typename A::IntNBinopMatcher m(node);
  if (m.right().Is(0)) return Replace(m.left().node());  // x ^ 0 => x
  if (m.IsFoldable()) {  // K ^ K => K  (K stands for arbitrary constants)
    return a.ReplaceIntN(m.left().ResolvedValue() ^ m.right().ResolvedValue());
  }
  if (m.LeftEqualsRight()) return ReplaceInt32(0);  // x ^ x => 0
  if (A::IsWordNXor(m.left()) && m.right().Is(-1)) {
    typename A::IntNBinopMatcher mleft(m.left().node());
    if (mleft.right().Is(-1)) {  // (x ^ -1) ^ -1 => x
      return Replace(mleft.left().node());
    }
  }

  return a.TryMatchWordNRor(node);
}


```

As the code shown above, for the case of x ^ 0 => x, the left node is used to replace the current node, which introduces the wrong data type.  

**4) InstructionSelectionPhase**  
According to the previous analysis, in instruction selection phase, because the input node (LoadTypedElement) type of ChangeInt32ToInt64 is Unsigned32, the X64Movl instruction is selected to replace the ChangeInt32ToInt64 node finally:  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/5.png)  

Because the zero extended instruction X64Movl is selected incorrectly, (arr[0] ^ 0) returns the wrong value: 0x0000000080000000.  

Finally, using this vulnerability, a variable x with an unexpected value 1 in JIT can be obtained via the following code (the expected value should be 0):  

```
const _arr = new Uint32Array([0x80000000]);

function foo() {
	var x = (_arr[0] ^ 0) + 1;
	x = Math.abs(x);
	x -= 0x7fffffff;
	x = Math.max(x, 0);
	x -= 1;
	if(x==-1) x = 0;
	return x;
}


```

RCA of Issue 1195777
--------------------

The bugfix for this issue is shown as follows:  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/6.png)  

This commit fixes a integer conversion node generation error which used to convert a 64-bit integer to a 32-bit integer (truncation) in SimplifiedLowering phase. Before the commit, if the output type of current node is Signed32 or Unsigned32, the TruncateInt64ToInt32 node is generated. After the commit, if the output type of current node is Unsigned32, the type of use_info is needed to be checked next. Only when use_info.type_check() == TypeCheckKind::kNone, the TruncateInt64ToInt32 node wiill be generated.  

First, let’s analyze the root cause of this vulnerability via regress-1195777.js:  

```
(function() {
  function foo(b) {
    let x = -1;
    if (b) x = 0xFFFFFFFF;
    return -1 < Math.max(0, x, -1);
  }
  assertTrue(foo(true));
  %PrepareFunctionForOptimization(foo);
  assertTrue(foo(false));
  %OptimizeFunctionOnNextCall(foo);
  assertTrue(foo(true));
})();


```

The key code in foo function which triggers JIT is ‘return -1 <Math.max(0, x, -1)’. Let’s focus on the optimization process of Math.max(0, x, -1) in the key phases of TurboFan:

**1) TyperPhase**  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/7.png)  
Math.max(0, x, -1) corresponds to node 56 and node 58. The output of node 58 is used as the input of node 41: SpeculativeNumberLessThan (<) .  

**2) TypedLoweringPhase**  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/8.png)  

The two constant parameters 0, -1 (node 54 and node 55) in Math.max(0, x, -1) are replaced with constant node 32 and node 14.  

**3) SimplifiedLoweringPhase**  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/9.png)  

The original NumberMax node 56 and node 58 are replaced by Int64LessThan + Select nodes. The original node 41: SpeculativeNumberLessThan is replaced with Int32LessThan. When processing the input node of SpeculativeNumberLessThan, because the output type of the input node (Select) is Unsigned32, the vulnerability is triggered and the node 76: TruncateInt64ToInt32 is generated incorrectly.  

The result of Math.max(0, x, -1) is truncated to Signed32. Therefore, when the x in Math.max(0, x, -1) is Unsigned32, it will be truncated to Signed32 by TruncateInt64ToInt32.  

Finally, using this vulnerability, a variable x with an unexpected value 1 in JIT can be obtained via the following code (the expected value should be 0):  

```
function foo(flag){
	let x = -1;
	if (flag){ 
		x = 0xFFFFFFFF;
	}
	x = Math.sign(0 - Math.max(0, x, -1));
	return x;
}


```

Exploit analysis
----------------

According to the above root cause analysis, we can see that the two vulnerabilities are triggered when TurboFan performs integer data type conversion (expansion, truncation). Using the two vulnerabilities, a variable x with an unexpected value 1 in JIT can be obtained.  
According to the samples exploited in the wild, the exploit is following the steps below:

**1) Create an Array which length is 1 with the help of variable x which has the error value 1;**  
**2) Obtain an out-of-bounds array with length 0xFFFFFFFF through Array.prototype.shift();**

The key code is as shown follows:

```
var arr = new Array(x);	// wrong: x = 1
arr.shift();			// oob
var cor = [1.8010758439469018e-226, 4.6672617056762661e-62, 1.1945305861211498e+103];
return [arr, cor];


```

The JIT code of var arr = new Array(x) is:  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/10.png)  

Rdi is the length of arr, which value is 1. It shift left one bit (rdi+rdi) by pointer compression and stored in JSArray.length property (+0xC).  

The JIT code of arr.shift() is:  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/11.png)  

After arr.shift(), the length of arr is assigned by constant 0xFFFFFFFE directly, the optimization process is shown as follows:

（1）TyperPhase  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/12.png)

The array length assignment operation is mainly composed of node 152 and node 153. The node 152 caculates Array.length-1. The node 153 saves the calculation result in Array.length (+0xC).  

（2）LoadEliminationPhase  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/13.png)  
Since the value of x which collected by Ignition is 0, constant folding (0-1=-1) happens here to get the constant 0xFFFFFFFF. After shift left one bit, it is 0xFFFFFFFE, which is stored in Array.length (+0xC). Thus, an out-of-bounds array with a length of 0xFFFFFFFF is obtained.  

After the out-of-bounds array is obtained, the next steps are common:  
**3) Realize addrof/fakeobj with the help of this out-of-bounds array;**  
**4) Fake a JSArray to achieve arbitrary memory read/write primitive wth the help of addrof/fakeobj;**  

The memory layout of arr and cor in exploit sample is:  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/14.png)  

(1) Use the vulnerability to obtain an arr with the length of 0xFFFFFFFF (red box)  
(2) Use the out-of-bounds arr and cor to achieve addrof/fakeobj (green box)  
(3) Use the out-of-bounds arr to modify the length of cor (yellow box)  
(4) Use the out-of-bounds cor, leak the map and properties of the cor (blue box), fake a JSArray, and use this fake JSArray to achieve arbitrary memory read/write primitive  

**5) Execute shellcode with the help of WebAssembly;**  
Finally, a memory page with RWX attributes is created with the help of WebAssembly. The shellcode is copied to the memory page, and executed in the end.  

The exploitation screenshot:  
![](https://iamelli0t.github.io/images/Chromium-Issue-1196683-1195777/15.png)  

References
----------

[1] https://chromium-review.googlesource.com/c/v8/v8/+/2820971  
[2] https://github.com/r4j0x00/exploits/blob/master/chrome-0day/exploit.js  
[3] https://chromereleases.googleblog.com/2021/04/stable-channel-update-for-desktop.html  
[4] https://chromium-review.googlesource.com/c/v8/v8/+/2826114  
[5] https://github.com/r4j0x00/exploits/blob/master/chrome-0day/exploit.js