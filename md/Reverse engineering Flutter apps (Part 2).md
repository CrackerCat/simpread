> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.tst.sh](https://blog.tst.sh/reverse-engineering-flutter-apps-part-2/)

This is a continuation of [Part 1](https://blog.tst.sh/reverse-engineering-flutter-apps-part-1/) which covered how Flutter compiles apps and what snapshots look like internally.

As you have probably guessed so far, reverse engineering its not an easy task.

* * *

### [Calling conventions](#calling-conventions)

Let's first cover some basics about Dart's type system:

```
void main() {
  void foo() {}
  int bar([int aaa]) {}
  Null biz({int aaa}) {}
  int baz(int aa, {int aaa}) {}
  
  print(foo is void Function());
  print(bar is void Function());
  print(biz is void Function());
  print(baz is void Function());
}

```

Which functions do you think print true?

It turns out the Dart type system is much more flexible than you might expect, as long as a function takes the same positional arguments and has compatible return type it is a valid function subtype.

Because of this, all but the `baz` print true.

Here's another experiment:

```
void main() {
  int foo({int a}) {}
  int bar({int a, int b}) {}
  
  print(foo is int Function());
  print(foo is int Function({int a}));
  print(bar is int Function({int a}));
  print(bar is int Function({int b}));
  print(bar is int Function({int b, int c}));
}


```

This code checks if functions have a valid subtype when they have a subset of named arguments, all but the last prints true.

For a formal description of function types, see _"9.3 Type of a Function"_ in the [Dart language specification](https://dart.dev/guides/language/specifications/DartLangSpec-v2.2.pdf), you might also want to check out my [recipe on the type system](https://recipes.tst.sh/docs/faq/type-system.html).

Mixing and matching parameter signatures are a nice feature but pose some problems when implementing them at a low level, for example:

```
void main() {
  void Function({int a, int c}) foo;
  
  foo = ({int a, int b, int c}) {
    print("Hi $a $b $c");
  };
  
  foo(a: 1, c: 2);
}


```

In order for this to work, `foo` needs some way of knowing the caller provided `a` and `c` but not `b`, this piece of information is called an argument descriptor.

Internally argument descriptors are defined by `vm/dart_entry.h`. The implementation is just an interface over a regular Array object which the callee provides via the argument descriptor register.

For example:

```
void bar({int a}) {
  print("Hi $a");
}

void foo() {
  bar(a: 42);
}

```

Rather than using Dart's built-in disassembler I'll be using a custom one that provides proper annotations for calls, object pool entries, and other constants.

Disassembly of `foo`, the caller:

```
034 | ...

038 | mov ip, #0x54        

03c | str ip, [sp, #-4]!   

040 | add r4, pp, #0x2000  

044 | ldr r4, [r4, #0x4a3] 

048 | bl 0x1b8             

04C | ...

```

foo

The argument descriptor for the call to bar is the following RawArray:

```
[
  0, 
  1, 
  0, 
  
  
  "x", 0,
  
  null, 
]

```

r4

The descriptor is used in the prologue of the callee to map stack indices to their respective argument slots and verify the proper arguments were received. Here is the disassembly of the callee:

```
000 | stmdb sp!, {fp, lr}
004 | add fp, sp, #0
008 | sub sp, sp, #4
00c | ldr r0, [r4, #0x13] 
010 | ldr r1, [r4, #0xf]  
014 | cmp r0, #0          
018 | bgt 0x74            
01c | ldr r0, [r4, #0x17]  
020 | add ip, pp, #0x2000  
024 | ldr ip, [ip, #0x4a7] 
028 | cmp r0, ip           
02c | bne 0x20             
030 | ldr r0, [r4, #0x1b]    
034 | sub r2, r1, r0         
038 | add r0, fp, r2, lsl #1 
    |                        
03c | ldr r0, [r0, #4]       
040 | mov r2, r0             
044 | mov r0, #2             
048 | b 12                   
04c | ldr r2, [thr, #0x68] 
050 | mov r0, #0           
054 | str r2, [fp, #-4] 
058 | cmp r1, r0 
05c | bne 0x30   
060 | ldr ip, [thr, #0x24] 
064 | cmp sp, ip           
068 | blls -0x5af00        
06c | ...
08c | ldr r6, [pp, #0x33] 
090 | sub sp, fp, #0      
094 | ldmia sp!, {fp, lr} 
098 | ldr pc, [r6, #3]    

```

bar

To summarize, it loops the array assigning slots to any matching arguments, throwing a NoSuchMethodError if any are not part of the function type. Also keep in mind argument checking is only required for polymorphic calls,  most (including the hello world example) are monomorphic.

This code is generated at a high level in `vm/compiler/frontend/prologue_builder.cc` `PrologueBuilder::BuildOptionalParameterHandling` meaning registers and subroutines may be layed out differently depending on the types of arguments and what optimizations it feels like doing.

* * *

### [Integer arithmetic](#integer-arithmetic)

The `num`, `int`, and `double` classes are special in the Dart type system, for performance reasons they cannot be extended or implemented.

This quirk guarantees a non-null `num` can always be used in arithmetic directly, if this restriction was missing the compiler would have to generate relatively expensive method calls instead.

All objects in dart are stored as `RawObject*` however only pointers tagged with `kHeapObjectTag` point to actual memory, otherwise they are smis, short for small ints.

Because of pointer tagging you will see a lot of `tst r0, #1` and similar instructions in generated code, most of which are simply discriminating between smis and heap objects. You will also see a lot of odd-numbered offset loads and stores, these are just subtracting the heap flag.

There is a nice document with details on how and why this optimization was introduced, you can find it here: [https://github.com/dart-lang/sdk/blob/master/docs/language/informal/int64.md](https://github.com/dart-lang/sdk/blob/master/docs/language/informal/int64.md)

Fun fact: The core `int` type used to be a bigint before Dart 2.0.

Any integer that can fit within the word size minus one bit (31 bits on A32) can be stored as an smi, otherwise larger integers are stored as 64 bit mint (medium int) instances on the heap.

You know an object is an smi if the LSB is clear:

![](https://blog.tst.sh/content/images/2020/02/smi-1.png)

Smis can contain negative numbers too of course, it uses an arithmetic right shift to sign extend the number back into place.

For example, here is a simple function that adds two ints:

```
int hello(int x, int y) => x + y;

```

This might seem trivial but it can be a little messy under the hood.

To start, `x` and `y` are each unboxed into pairs of registers, Dart ints are 64 bit so two registers are needed for each arg on A32:

```
024 | ...
028 | ldr r1, [fp, #12]    
02c | ldr ip, [thr, #0x68] 
030 | cmp r1, ip           
034 | bleq -0x50954        
038 | ...
048 | mov r3, r1, asr #0x1f 
04c | movs r4, r1, asr #1   
050 | bcc 12                
054 | ldr r4, [r1, #7]      
058 | ldr r3, [r1, #11]     
05c | ...

```

x + y

After `x` and `y` are in pairs of registers it can perform the actual 64 bit add:

```
070 | adds r7, r4, r6 

074 | adcs r2, r3, r1 

```

x + y

Before returning the result gets re-boxed:

```
074 | ...
078 | mov r0, r7, lsl #1      
07c | cmp r7, r0, asr #1      
080 | cmpeq r2, r0, asr #0x1f 
084 | beq 0x34                
088 | ldr r0, [thr, #0x3c] 
08c | adds r0, r0, #0x10   
090 | ldr ip, [thr, #0x40] 
094 | cmp ip, r0           
098 | bls 0x28             
09c | str r0, [thr, #0x3c] 
0a0 | sub r0, r0, #15      
0a4 | mov ip, #0x2204      
0a8 | movt ip, #0x31       
0ac | str ip, [r0, #-1]    
0b0 | str r7, [r0, #7]  
0b4 | str r2, [r0, #11] 
0b8 | sub sp, fp, #0
0bc | ldmia sp!, {fp, pc}
0c0 | stmdb sp!, {r2, r7} 
0c4 | bl 0x651f4          
0c8 | ldmia sp!, {r2, r7} 
0cc | b -0x1c             

```

x + y

Boxing looks more expensive than it actually is, most of the time the value will be returned immediately as an smi and only hits the secondary code paths when the number is larger than 31 bits.

* * *

### [Instances](#instances)

The code below creates an instance by calling an allocation stub followed by a call to the constructor:

```
makeFoo() => Foo<int>();

```

Disassembled:

```
014 | ...

018 | ldr ip, [pp, #0x93] 

01c | str ip, [sp, #-4]!  

020 | bl -0x628           

024 | add sp, sp, #4      

028 | str r0, [fp, #-4]   

02c | str r0, [sp, #-4]!  

030 | bl -0x9f0           

034 | add sp, sp, #4      

038 | ldr r0, [fp, #-4]   

03c | ...

```

makeFoo

Each class has a corresponding allocation stub that allocates and initializes an instance (very similar to how boxing creates an object), these stubs are generated for any classes that can be constructed.

Unfortunately for us, field information is removed from the snapshot so we can't directly get their names. You can however see the names of implicit getter and setter methods (assuming they haven't been inlined).

Offsets for fields are calculated at `Class::CalculateFieldOffsets`, the rules go as follows:

1.  Start at end of super class, otherwise start at `sizeof(RawInstance)`
2.  Use the type arguments field of parent, else put it at the start
3.  Lay out remaining (non static) fields sequentially

Because type arguments are shared with the super, instantiating the following class gives us a type arguments field containing `<String, int>`:

```
class Foo<T> extends Bar<String> {}
var x = Foo<int>(); 

```

Whereas if the type arguments are the same for parent and child, the list will only contain `<int>`:

```
class Foo<T> extends Bar<T> {}
var x = Foo<int>(); 

```

Another fun feature of Dart is that all field access is done via setters and getters, this may sound very slow but in practice dart eliminates a ton of overhead with the following optimizations:

1.  Whole-program static analysis
2.  Inlining calls on known types
3.  Code de-duplication
4.  Inline cache (via ICData)

These optimizations apply to all methods including getters and setters, in the following example the setter is inlined:

```
class Foo {
  int x;
}

Foo bar() => Foo()..x = 42;

```

Disassembled:

```
028 | ...

02c | ldr r0, [fp, #-4] 

030 | mov ip, #0x54     

034 | str ip, [r0, #3]  

038 | ...

```

bar

But when we call this setter through an interface:

```
abstract class Foo {
  set x(int x);
}

class FooImpl extends Foo {
  int x;
}

void bar(Foo foo) {
  foo.x = 42;
}

```

Disassembled:

```
010 | ...

014 | ldr ip, [fp, #8]     

018 | str ip, [sp, #-4]!   

01c | mov ip, #0x54        

020 | str ip, [sp, #-4]!   

024 | ldr r0, [sp, #4]     

028 | add lr, pp, #0x2000  

02c | ldr lr, [lr, #0x4a3] 

030 | add r9, pp, #0x2000  

034 | ldr r9, [r9, #0x4a7] 

038 | blx lr               

03c | ...

```

bar

Here it invokes an unlinkedCall stub which is a magic bit of code that handles polymorphic method invocation, it will patch its own object pool entry so that further calls are quicker.

I'd love to get into more detail about how this works at runtime but all we need to know is that it invokes the method specified in the RawUnlinkedCall. If you are interested, there is a great article on the internals of DartVM that explains more: [https://mrale.ph/dartvm/](https://mrale.ph/dartvm/)

* * *

### [Type Checking](#type-checking)

Type checking is a fundamental component of polymorphism, dart provides this through the `is` and `as` operators.

Both operators do a subtype check with the exception of `as` allowing null values, here is the `is` operator in action:

```
class FooBase {}
class Foo extends FooBase {}
class Bar extends FooBase {}

bool isFoo(FooBase x) => x is Foo;

```

Disassembled:

```
024 | ...

028 | ldr r1, [fp, #8]       

02c | ldrh r2, r3, [r1, #1]  

030 | mov r2, r2, lsl #1     

034 | cmp r2, #0x12c         

038 | ldreq r0, [thr, #0x6c] 

03c | ldrne r0, [thr, #0x70] 

040 | ...

```

x is Foo

Since whole-program analysis determined `Foo` only has one implementer, it can simply check equality of the class ID, but what if it has a child class?

```
class Baz extends Foo {}

```

We now get:

```
028 | ...
02c | ldr r1, [fp, #8]      
030 | ldrh r2, r3, [r1, #1] 
034 | mov r2, r2, lsl #1    
038 | mov r1, #0x12c        
03c | mov r4, r1, asr #1    
040 | mov r3, r4, asr #0x1f 
044 | mov r6, r2, asr #1    
048 | mov r1, r6, asr #0x1f 
04c | cmp r1, r3            
050 | bgt 0x10              
054 | blt 0x40              
058 | cmp r6, r4            
05c | blo 0x38              
060 | mov r2, #0x12e        
064 | mov r4, r2, asr #1    
068 | mov r3, r4, asr #0x1f 
06c | cmp r1, r3            
070 | blt 0x18              
074 | bgt 12                
078 | cmp r6, r4            
07c | bls 12                
080 | ldr r2, [thr, #0x70]  
084 | b 8                   
088 | ldr r2, [thr, #0x6c]  
08c | mov r0, r2            
090 | b 8                   
094 | ldr r0, [thr, #0x70]  
098 | ...

```

x is Foo

Gah! This code is awful so here is a basic translation:

```
bool isFoo(FooBase* x) {
  if (x.classId < FooClassId) return false;
  return x.classId <= BazClassId;
}

```

All it is doing here is checking if the class id falls within a set of ranges, in this case there is only one range to check.

This is definitely a place where DartVM could improve on ARM, it's doing 64 bit smi range checks for 16 bit class ids instead of just comparing it directly.

The range checks also do not take into consideration the super type its comparing from which can cause a range to be split by a type that does not implement the super, perhaps as a result of unsoundness.

* * *

### [Control flow](#control-flow)

Dart uses a relatively advanced flow graph, represented as an SSA (Single Static Assignment) intermediate similar to modern compilers like gcc and clang. It can perform many optimizations that change the control flow structure of the program, making reasoning about its generated code a bit harder.

Here is a simple if statement:

```
void hello(bool condition) {
  if (condition) {
    print("foo");
  } else {
    print("bar");
  }
}

```

Disassembled:

```
010 | ...
014 | ldr r0, [fp, #8]      
018 | ldr ip, [thr, #0x68]  
01c | cmp r0, ip            
020 | bne 0x18              
024 | str r0, [sp, #-4]!    
028 | ldr r9, [thr, #0x178] 
02c | mov r4, #1            
030 | ldr ip, [thr, #0xd0]  
034 | blx ip                
038 | ldr r0, [fp, #8]      
03c | ldr ip, [thr, #0x6c]  
040 | cmp r0, ip            
044 | bne 0x1c              
048 | add ip, pp, #0x2000   
04c | ldr ip, [ip, #0x4a3]  
050 | str ip, [sp, #-4]!    
054 | bl -0x33b1c           
058 | add sp, sp, #4        
05c | b 0x18                
060 | add ip, pp, #0x2000   
064 | ldr ip, [ip, #0x4a7]  
068 | str ip, [sp, #-4]!    
06c | bl -0x33b34           
070 | add sp, sp, #4        
074 | ...

```

hello

That null check is an example of a "runtime entry" dynamic call, this is the bridge from dart code to subroutines defined in `vm/runtime_entry.cc`.

In this case it is a specialized entry that throws a `Failed assertion: boolean expression must not be null`, as you would expect if the condition of an if statement is null.

Whole program optimization (and sound non-nullability in the future) allows this null check to be elided, for example if `hello` never gets called with a possible null value then it won't do the check at all:

```
void main() {
  hello(true);
  hello(false);
}

void hello(bool condition) {
  if (condition) {
    print("foo");
  } else {
    print("bar");
  }
}

```

Disassembled:

```
010 | ...
014 | ldr r0, [fp, #8]     
018 | ldr ip, [thr, #0x6c] 
01c | cmp r0, ip           
020 | bne 0x1c             
024 | add ip, pp, #0x2000  
028 | ldr ip, [ip, #0x4a3] 
02c | str ip, [sp, #-4]!   
030 | bl -0x33a90          
034 | add sp, sp, #4       
038 | b 0x18               
03c | add ip, pp, #0x2000  
040 | ldr ip, [ip, #0x4a7] 
044 | str ip, [sp, #-4]!   
048 | bl -0x33aa8          
04c | add sp, sp, #4       
050 | ldr r0, [thr, #0x68] 
054 | ...

```

hello

* * *

### [Closures](#closures)

Closures are the implementation of first-class functions under the `Function` type, you can acquire one by creating an anonymous function or extracting a method.

A simple function `hi` that returns an anonymous function:

```
void Function() hi() {
  return (x) { print("Hi $x"); };
}

```

Disassembled:

```
024 | ...

028 | bl 0x3a6e0           

02c | add ip, pp, #0x2000  

030 | ldr ip, [ip, #0x2cf] 

034 | str ip, [r0, #15]    

038 | ...

```

hi

And to call the closure:

```
010 | ...

014 | bl -0x6c           

018 | str r0, [sp, #-4]! 

01c | ldr ip, [thr, #0x68] 

020 | cmp r0, ip

024 | bleq -0x3fdc4 

028 | ldr r1, [r0, #15]   

02c | mov r0, r1          

030 | ldr r4, [pp, #0xfb] 

034 | ldr r6, [r0, #0x2b] 

038 | ldr r2, [r0, #7]    

03c | mov r9, #0          

040 | blx r2              

044 | add sp, sp, #4      

048 | ...

```

hi()()

Pretty simple, but what if the lambda depends on a local variable from the parent function?

```
int Function() hi() {
  int i = 123;
  return () => ++i;
}

```

Disassembled:

```
014 | ...

018 | mov r1, #1          

01c | ldr r6, [pp, #0x1f] 

020 | ldr lr, [r6, #3]    

024 | blx lr              

028 | str r0, [fp, #-4]   

02c | ...

040 | ldr r0, [fp, #-4]    

044 | mov ip, #0xf6        

048 | str ip, [r0, #11]    

04c | bl 0x3a84c           

050 | add ip, pp, #0x2000  

054 | ldr ip, [ip, #0x2cf] 

058 | str ip, [r0, #15]    

05c | ldr r1, [fp, #-4]    

060 | str r1, [r0, #0x13]  

064 | ...

```

hi

Instead of storing the variable `i` in the stack frame like a regular local variable, the function will store it in a `RawContext` and pass that context to the closure.

When called, the closure can access that variable from the closure argument:

```
028 | ...

02c | ldr r1, [fp, #8]    

030 | ldr r2, [r1, #0x13] 

034 | ldr r0, [r2, #11]   

038 | ...

```

() => ++i

Another way to get a closure is method extraction:

```
class Potato {
  int _foo = 0;
  int foo() => _foo++;
}

int Function() extractFoo() => Potato().foo;

```

When you call `get:foo` on Potato, Dart will generate that getter method as follows:

```
010 | ...

014 | mov r4, #0           

018 | add ip, pp, #0x2000  

01c | ldr r1, [ip, #0x303] 

020 | add ip, pp, #0x2000  

024 | ldr r6, [ip, #0x2ff] 

028 | ldr pc, [r6, #11]    

```

get:foo

`get:foo` invokes buildMethodExtractor, which eventually returns a RawClosure and stores the receiver (`this`) in its context and loads that back into `r0` when called, just like a regular instance call.

* * *

### [Where the fun starts](#where-the-fun-starts)

With a good starting point to reverse engineer real world applications, the first big Flutter app that comes to mind is [Stadia](https://twitter.com/jtmcdole/status/1192853350179491840).

So let's take a crack at it, first step is to grab an APK off of apkmirror, in this case version 2.2.289534823:

![](https://blog.tst.sh/content/images/2020/03/image.png)

(I don't recommend downloading apps from third party websites, it's just the easiest way to grab an apk file without a compatible android device)

The important part here is that the version information contains `arm64-v8a + armeabi-v7a` which are A64 and A32 respectively.

![](https://blog.tst.sh/content/images/2020/03/image-1.png)

The interesting bits are in the lib folder like `libflutter.so` which is the flutter engine, and `libproduction_android_library.so` which is just a renamed `libapp.so`.

Before being able to do anything with the snapshot we must know the _exact_ version of Dart that was used to build the app, a quick search of `libflutter.so` in a hex editor gives us a version string:

![](https://blog.tst.sh/content/images/2020/03/image-6.png)

That `c547f5d933` is a commit hash in the Dart SDK which you can view on GitHub: [https://github.com/dart-lang/sdk/tree/c547f5d933](https://github.com/dart-lang/sdk/tree/c547f5d933), after some digging this corresponds to Flutter version `v1.13.6` or commit `659dc8129d`.

Knowing the exact version of dart is important because it gives you a reference to know how objects are layed out and provides a testbed.

Once decoded, the next step is to search for the root library, in this version of dart it's located at index 66 of the root objects list:

![](https://blog.tst.sh/content/images/2020/10/image.png)

Neat, we can see the package name of this app is `chrome.cloudcast.client.mobile.app`, which you might notice is not actually valid for a pub package, what's going on here?

The reason for the weird package name is that Google doesn't actually use pub for internal projects and instead uses it's internal Google3 repository. You can occasionally see issues on the Flutter GitHub labelled `customer: ... (g3)`, this is what it refers to.

By extracting uris from every library defined in the app, we can view the complete file structure for every packages it contains.

As you might expect from a large project, it depends on quite a few packages: [https://gist.github.com/PixelToast/7503be142607df38582b454f3b1e8153](https://gist.github.com/PixelToast/7503be142607df38582b454f3b1e8153)

We can gather it uses some of the following technologies:

*   protobuf
*   markdown boilerplate (from AdWords?)
*   firebase
*   rx
*   bloc
*   provider

Most of which appear to be internal implementations.

Going deeper, here is the root of the lib folder: [https://gist.github.com/PixelToast/f2029cd88d5343c0991f706403012f62](https://gist.github.com/PixelToast/f2029cd88d5343c0991f706403012f62)

I picked a random widget to look at, `SocialNotificationCard` from `profile/view/social_notification_card.dart`.

The library containing this widget is structured as follows:

```
enum SocialNotificationIconType {
  avatarUrl,
  apiImage,
  defaultIcon,
  partyIcon,
}

class SocialNotificationCard extends StatelessWidget {
  SocialNotificationCard({
    dynamic socialNotificationIconType,
    dynamic title,
    dynamic body,
    dynamic timestamp,
    dynamic avatarUrl,
    dynamic apiImage,
  }) { }
  
  NessieString get title { }

  Widget _buildNotificationMessage(dynamic arg1) { }
  Widget _buildNotificationTimestamp(dynamic arg1) { }
  Widget _buildGeneralNotificationIcon() { }
  Widget _buildPartyIcon(dynamic arg1) { }
  Widget _buildAvatarUrlIcon(dynamic arg1) { }
  Widget _buildApiImage(dynamic arg1) { }
  Widget _buildNotificationIconImage(dynamic arg1) { }
  Widget _buildNotificationIcon(dynamic arg1) { }
  Widget build(dynamic arg1) { }
}

```

profile/view/social_notification_card.dart

The type information on these parameters is missing, but since they are build methods we can assume they all take a `BuildContext`.

The full disassembly of the `_buildPartyIcon` method goes as follows:

```
000 | tst r0, #1

004 | ldrhne ip, [r0, #1]

008 | moveq ip, #0x30

00c | cmp r9, ip, lsl #1

010 | ldrne pc, [thr, #0x108]

014 | stmdb sp!, {fp, lr}

018 | add fp, sp, #0

01c | bl 0x27ef84          

020 | add ip, pp, #0x54000 

024 | ldr ip, [ip, #0xdc3] 

028 | str ip, [r0, #7]     

02c | ldr ip, [thr, #0x70] 

030 | str ip, [r0, #0x43]  

034 | add ip, pp, #0x44000 

038 | ldr ip, [ip, #0x6b]  

03c | str ip, [r0, #0x13]  

040 | add ip, pp, #0x44000 

044 | ldr ip, [ip, #0x6b]  

048 | str ip, [r0, #0x17]  

04c | add ip, pp, #0x46000 

050 | ldr ip, [ip, #0xe2b] 

054 | str ip, [r0, #0x27]  

058 | add ip, pp, #0xd000  

05c | ldr ip, [ip, #0xacf] 

060 | str ip, [r0, #0x2b]  

064 | add ip, pp, #0x38000 

068 | ldr ip, [ip, #0x653] 

06c | str ip, [r0, #0x2f]  

070 | ldr ip, [thr, #0x70] 

074 | str ip, [r0, #0x37]  

078 | ldr ip, [thr, #0x70] 

07c | str ip, [r0, #0x3b]  

080 | add ip, pp, #0x38000 

084 | ldr ip, [ip, #0x657] 

088 | str ip, [r0, #0x1f]  

08c | sub sp, fp, #0

090 | ldmia sp!, {fp, pc}

```

SocialNotificationCard._buildPartyIcon

This one is quite easy to turn back into code by hand since it constructs a single `Image` widget:

```
Widget _buildPartyIcon(BuildContext context) {
  return Image.asset(
    
    
    "assets/social/party_invite.png",
    fit: BoxFit.cover,
    width: 48,
    height: 48,
    
  );
}

```

Note that object construction generally happens in 3 parts:

1.  Invoke allocation stub, passing type arguments if needed
2.  Evaluate parameter expressions and assigning them to fields in-order
3.  Calling the constructor body, if any

The initializer list and default parameters seem to be unconditionally inlined to the caller, leading to a bit more noise.

Finally let's disassemble the actual `build` method of `SocialNotificationCard`:

```
000 | tst r0, #1              
004 | ldrhne ip, [r0, #1]     
008 | moveq ip, #0x30         
00c | cmp r9, ip, lsl #1      
010 | ldrne pc, [thr, #0x108] 
014 | stmdb sp!, {fp, lr}     
018 | add fp, sp, #0          
01c | sub sp, sp, #0x14       
020 | ldr ip, [thr, #0x24]    
024 | cmp sp, ip              
028 | blls -0x52ee40          
02c | bl 0x3e3ce4          
030 | str r0, [fp, #-4]    
034 | bl 0x3e11f4          
038 | str r0, [fp, #-8]    
03c | add ip, pp, #0x5000  
040 | ldr ip, [ip, #0x9db] 
044 | str ip, [r0, #3]     
048 | add ip, pp, #0xf000  
04c | ldr ip, [ip, #0xd7]  
050 | str ip, [r0, #7]     
054 | add ip, pp, #0x5000  
058 | ldr ip, [ip, #0x9db] 
05c | str ip, [r0, #11]    
060 | add ip, pp, #0xf000  
064 | ldr ip, [ip, #0xd7]  
068 | str ip, [r0, #15]    
06c | bl 0x217984          
070 | str r0, [fp, #-12]   
074 | add ip, pp, #0x31000 
078 | ldr ip, [ip, #0x677] 
07c | str ip, [sp, #-4]!   
080 | add r1, pp, #0x31000 
084 | ldr r1, [r1, #0x677] 
088 | mov r2, #6           
08c | ldr r6, [pp, #7]     
090 | ldr lr, [r6, #3]     
094 | blx lr               
098 | str r0, [fp, #-0x10] 
09c | ldr ip, [fp, #12]    
0a0 | str ip, [sp, #-4]!   
0a4 | ldr ip, [fp, #8]     
0a8 | str ip, [sp, #-4]!   
0ac | bl -0x180            
0b0 | add sp, sp, #8       
0b4 | ldr r1, [fp, #-0x10]   
0b8 | add r9, r1, #11        
0bc | str r0, [r9]           
0c0 | tst r0, #1             
0c4 | beq 0x1c               
0c8 | ldrb ip, [r1, #-1]     
0cc | ldrb lr, [r0, #-1]     
0d0 | and ip, lr, ip, lsr #2 
0d4 | ldr lr, [thr, #0x30]   
0d8 | tst ip, lr             
0dc | blne -0x52f1ac         
0e0 | add ip, pp, #0x31000 
0e4 | ldr ip, [ip, #0xaab] 
0e8 | str ip, [sp, #-4]!   
0ec | bl 0x39ae5c          
0f0 | add sp, sp, #4       
0f4 | str r0, [fp, #-0x14] 
0f8 | ldr ip, [fp, #12]  
0fc | str ip, [sp, #-4]! 
100 | ldr ip, [fp, #8]   
104 | str ip, [sp, #-4]! 
108 | bl -0x1ae634       
10c | add sp, sp, #8     
110 | ldr r1, [fp, #-0x14] 
114 | mov ip, #2           
118 | str ip, [r1, #15]    
11c | add ip, pp, #0x31000 
120 | ldr ip, [ip, #0xab7] 
124 | str ip, [r1, #0x13]  
128 | str r0, [r1, #7]     
12c | ldrb ip, [r1, #-1]     
130 | ldrb lr, [r0, #-1]     
134 | and ip, lr, ip, lsr #2 
138 | ldr lr, [thr, #0x30]   
13c | tst ip, lr             
140 | blne -0x52f020         
144 | str r1, [sp, #-4]!   
148 | bl -0x529018         
14c | add sp, sp, #4       
150 | ldr r1, [fp, #-0x10] 
154 | ldr r0, [fp, #-0x14] 
158 | add r9, r1, #15      
15c | str r0, [r9]         
160 | tst r0, #1             
164 | beq 0x1c               
168 | ldrb ip, [r1, #-1]     
16c | ldrb lr, [r0, #-1]     
170 | and ip, lr, ip, lsr #2 
174 | ldr lr, [thr, #0x30]   
178 | tst ip, lr             
17c | blne -0x52f24c         
180 | ldr ip, [fp, #12]  
184 | str ip, [sp, #-4]! 
188 | ldr ip, [fp, #8]   
18c | str ip, [sp, #-4]! 
190 | bl -0x894          
194 | add sp, sp, #8     
198 | ldr r1, [fp, #-0x10] 
19c | add r9, r1, #0x13    
1a0 | str r0, [r9]         
1a4 | tst r0, #1             
1a8 | beq 0x1c               
1ac | ldrb ip, [r1, #-1]     
1b0 | ldrb lr, [r0, #-1]     
1b4 | and ip, lr, ip, lsr #2 
1b8 | ldr lr, [thr, #0x30]   
1bc | tst ip, lr             
1c0 | blne -0x52f290         
1c4 | ldr ip, [fp, #-0x10] 
1c8 | str ip, [sp, #-4]!   
1cc | bl -0x52948c         
1d0 | add sp, sp, #8       
1d4 | ldr r1, [fp, #-12]   
1d8 | add ip, pp, #0x3a000 
1dc | ldr ip, [ip, #0x1a3] 
1e0 | str ip, [r1, #11]    
1e4 | add ip, pp, #0x31000 
1e8 | ldr ip, [ip, #0xa8b] 
1ec | str ip, [r1, #15]    
1f0 | add ip, pp, #0x31000 
1f4 | ldr ip, [ip, #0xa93] 
1f8 | str ip, [r1, #0x13]  
1fc | add ip, pp, #0x31000 
200 | ldr ip, [ip, #0xa77] 
204 | str ip, [r1, #0x17]  
208 | add ip, pp, #0x31000 
20c | ldr ip, [ip, #0xa9b] 
210 | str ip, [r1, #0x1f]  
214 | str r0, [r1, #7]     
218 | tst r0, #1             
21c | beq 0x1c               
220 | ldrb ip, [r1, #-1]     
224 | ldrb lr, [r0, #-1]     
228 | and ip, lr, ip, lsr #2 
22c | ldr lr, [thr, #0x30]   
230 | tst ip, lr             
234 | blne -0x52f114         
238 | str r1, [sp, #-4]! 
23c | bl -0x52910c       
240 | add sp, sp, #4     
244 | ldr r0, [fp, #-8]  
248 | ldr r1, [fp, #-4]  
24c | str r0, [r1, #11]  
250 | ldrb ip, [r1, #-1]     
254 | ldrb lr, [r0, #-1]     
258 | and ip, lr, ip, lsr #2 
25c | ldr lr, [thr, #0x30]   
260 | tst ip, lr             
264 | blne -0x52f144         
268 | ldr r0, [fp, #-12] 
26c | str r0, [r1, #7]   
270 | ldrb ip, [r1, #-1]     
274 | ldrb lr, [r0, #-1]     
278 | and ip, lr, ip, lsr #2 
27c | ldr lr, [thr, #0x30]   
280 | tst ip, lr             
284 | blne -0x52f164         
288 | mov r0, r1          
28c | sub sp, fp, #0      
290 | ldmia sp!, {fp, pc} 

```

SocialNotificationCard.build

There was a bit more GC related code this time, if you are interested these write barriers are required due to the [tri-color invariant](https://v8.dev/blog/concurrent-marking). Hitting a write barrier is actually pretty rare, so it has minimal impact on performance with the benefit of allowing parallel garbage collection.

The equivalent dart code:

```
Widget build(BuildContext context) {
  return Padding(
    padding: EdgeInsets.symmetric(vertical: 16.0),
    child: Row(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: <Widget>[
        _buildNotificationIcon(context),
        Expanded(
          child: _buildNotificationMessage(context),
        ),
        _buildNotificationTimestamp(context),
      ],
    ),
  );
}

```

A little more tedious to reverse due to the amount of code, but still relatively easy given tools to identify object pool entries and call targets.

* * *

This was a super fun project and I thoroughly enjoyed picking apart assembly code. I hope this series inspires others to also learn more about compilers and the internals of Dart.

### [Can someone steal my app?](#can-someone-steal-my-app)

Technically was always possible, given enough time and resources.

In practice this is not something you should worry about (yet), we are far off from having a full decompilation suite that allows someone to steal an entire app.

### [Are my tokens and API keys safe?](#are-my-tokens-and-api-keys-safe)

Nope!

There will never be a way to fully hide secrets in any client-side application. Note that things like the [google_maps_flutter](https://pub.dev/packages/google_maps_flutter) API key is not actually private.

If you are currently using hard coded credentials or tokens for third party apis in your app you should switch to a real backend or [cloud function](https://firebase.google.com/docs/functions) ASAP.

### [Will obfuscation help?](#will-obfuscation-help)

Yes and no.

Obfuscation will randomize identifier names for things like classes and methods, but it won't prevent us from viewing class structure, library structure, strings, assembly code, etc.

A competent reverse engineer can still look for common patterns like http API layers, state management, and widgets. It is also possible to partially symbolize code that uses publicly available packages, e.g. you can build signatures for functions in `package:flutter` and correlate them to ones in an obfuscated snapshot.

I generally don't recommend obfuscating Flutter apps because it makes reading error messages harder without doing much for security, you can read more about it [here](https://flutter.dev/docs/deployment/obfuscate).