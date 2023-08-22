> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278525.htm#msg_header_h1_3)

> frida 全流程分析

server.vala 中通过各种调用最后调用到了`ControlService.vala#satrt`。值得一提的是在`ControlService`初始化的时候使用了一个很新奇的东西。在`control-service.vala:92`，可以看见有这么一行代码：`service.incoming.connect (on_server_connection);`

但当我们来到`WebService`类时发现`incoming`是一个方法而且没有函数体。实际上这里用了一个叫做信号槽的东西，可以看见`incoming`被`signal`修饰, 代表这是一个信号槽。而`service.incoming.connect (on_server_connection);`这段代码恰好是将`on_server_connection`插入了槽内。也就是说后面如果调用`WebService.incoming`其实是在调用`ControlService.on_server_connection`。（**记住后面要考**）

在调用`start`方法前判断了是不是`enable_preload`(frida 启动时的参数默认为 true)，如果是就将 frida 的 agent 注入到`zygote`和`zygote64`进程中。为了分析的连贯性我们先不分析 preload 部分，具体分析详见[注入 zygote](#%E6%B3%A8%E5%85%A5zygote)

`do_start`首先是`new`了一个`Soup.Server`然后绑定了一个`websocket_handler`（即访问`/ws`时会调用绑定的方法，**后面要考！！**）到`on_websocket_opened`方法。然后就是`server.listen`在对应的地址监听连接了。

还记得之前说的吧，这里调用了`WebService.incoming`其实是在调用`ControlService.on_server_connection`。我们继续跟进看看。

在 server 端（其实应该说是 client 严谨一些，不过代码里面写的是 server）也就是 pc 端连接后，new 了一个`ControlChannel`以`DBusConnection`为键存储到了`peers`Map。而在`ControlChannel`的`construct`中使用 DBus 注册`ObjectPath.HOST_SESSION`接口到`session`对象, 也就是 new 出来的`ControlChannel`对象。

首先调用`obtain_for_cpu_type`生成不同架构的 frida-helper，这里跟一下 32bit（32 和 64 的代码逻辑完全一样，只是生成的可执行文件不同。）

自己开启一个 Domain Socket，然后再调用 frida-helper 在对应的路径启动 Domain Socket 与 frida-server 建立连接。连接建立后返回 HelperSession 类

首先调用`frida_inject_instance_emit_payload_code`生成字节码，然后注入到内存空间中。再给`rx`权限，返回`entrypoint`也就是注入的字节码的起始地址。

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41

42

43

44

45

46

47

48

49

50

51

52

53

54

55

56

57

58

59

60

61

62

63

64

65

66

67

68

69

70

71

72

73

74

75

76

77

78

79

80

81

82

83

84

85

86

87

88

89

90

91

92

93

94

95

96

97

98

99

100

101

102

103

104

105

106

107

108

109

110

111

112

113

114

115

116

117

118

119

120

121

122

123

124

125

126

127

128

129

130

131

132

133

134

135

136

137

138

139

140

141

142

143

144

145

146

147

148

149

150

151

152

153

154

155

156

157

158

159

160

161

162

163

164

165

166

167

168

169

170

171

172

173

174

175

176

177

178

179

180

181

182

183

184

185

186

187

188

189

190

191

192

193

`static` `void`

`frida_inject_instance_emit_payload_code (``const` `FridaInjectParams * params, GumAddress remote_address, FridaCodeChunk * code)`

`{`

 `GumArm64Writer cw;`

 `const` `guint worker_offset = 128;`

 `const` `gchar * skip_dlopen =` `"skip_dlopen"``;`

 `const` `gchar * skip_dlclose =` `"skip_dlclose"``;`

 `const` `gchar * skip_detach =` `"skip_detach"``;`

 `gum_arm64_writer_init (&cw, code->cur);`

 `cw.pc = remote_address + params->code.offset + code->size;`

`#ifdef HAVE_ANDROID`

 `EMIT_LDR_ADDRESS (X5, frida_resolve_libc_function (params->pid,` `"pthread_create"``));`

`#else`

 `EMIT_LDR_ADDRESS (X20, remote_address + params->data.offset);`

 `EMIT_CALL_IMM (params->dlopen_impl,`

 `2,`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (pthread_so_string)),`

 `ARG_IMM (params->dlopen_flags));`

 `EMIT_STORE_FIELD (pthread_so, X0);`

 `EMIT_CALL_IMM (params->dlsym_impl,`

 `2,`

 `ARG_REG (X0),`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (pthread_create_string)));`

 `EMIT_MOVE (X5, X0);`

`#endif`

 `EMIT_CALL_REG (X5,`

 `4,`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (worker_thread)),`

 `ARG_IMM (0),`

 `ARG_IMM (remote_address + worker_offset),`

 `ARG_IMM (remote_address + params->data.offset));`

 `gum_arm64_writer_put_brk_imm (&cw, 0);`

 `gum_arm64_writer_flush (&cw);`

 `g_assert (gum_arm64_writer_offset (&cw) <= worker_offset);`

 `while` `(gum_arm64_writer_offset (&cw) != worker_offset - code->size)`

 `gum_arm64_writer_put_nop (&cw);`

 `frida_inject_instance_commit_arm64_code (&cw, code);`

 `gum_arm64_writer_clear (&cw);`

 `gum_arm64_writer_init (&cw, code->cur);`

 `cw.pc = remote_address + params->code.offset + worker_offset;`

 `EMIT_PUSH (FP, LR);`

 `EMIT_MOVE (FP, SP);`

 `EMIT_PUSH (X21, X22);`

 `EMIT_PUSH (X19, X20);`

 `EMIT_MOVE (X20, X0);`

 `EMIT_CALL_IMM (params->open_impl,`

 `2,`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (fifo_path)),`

 `ARG_IMM (O_WRONLY | O_CLOEXEC));`

 `EMIT_MOVE (W21, W0);`

 `EMIT_CALL_IMM (params->write_impl,`

 `3,`

 `ARG_REG (W21),`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (hello_byte)),`

 `ARG_IMM (1));`

 `EMIT_LOAD_FIELD (X19, module_handle);`

 `EMIT_CBNZ (X19, skip_dlopen);`

 `{`

`#ifdef HAVE_ANDROID`

 `EMIT_CALL_IMM (params->dlopen_impl,`

 `3,`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (so_path)),`

 `ARG_IMM (params->dlopen_flags),`

 `ARG_IMM (params->open_impl));`

`#else`

 `EMIT_CALL_IMM (params->dlopen_impl,`

 `2,`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (so_path)),`

 `ARG_IMM (params->dlopen_flags));`

`#endif`

 `EMIT_MOVE (X19, X0);`

 `EMIT_STORE_FIELD (module_handle, X19);`

 `}`

 `EMIT_LABEL (skip_dlopen);`

 `EMIT_CALL_IMM (params->dlsym_impl,`

 `2,`

 `ARG_REG (X19),`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (entrypoint_name)));`

 `EMIT_MOVE (X5, X0);`

 `EMIT_LDR_U64 (X0, FRIDA_UNLOAD_POLICY_IMMEDIATE);`

 `EMIT_PUSH (X0, X21);`

 `EMIT_MOVE (X1, SP);`

 `EMIT_ADD (X2, SP, 8);`

 `EMIT_CALL_REG (X5,`

 `3,`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (entrypoint_data)),`

 `ARG_REG (X1),`

 `ARG_REG (X2));`

 `EMIT_LDR (W21, SP, 8);`

 `EMIT_LDR (W22, SP, 0);`

 `EMIT_LDR_U64 (X1, FRIDA_UNLOAD_POLICY_IMMEDIATE);`

 `EMIT_CMP (W22, W1);`

 `EMIT_B_COND (NE, skip_dlclose);`

 `{`

 `EMIT_CALL_IMM (params->dlclose_impl,`

 `1,`

 `ARG_REG (X19));`

 `}`

 `EMIT_LABEL (skip_dlclose);`

 `EMIT_LDR_U64 (X1, FRIDA_UNLOAD_POLICY_DEFERRED);`

 `EMIT_CMP (W22, W1);`

 `EMIT_B_COND (EQ, skip_detach);`

 `{`

`#ifdef HAVE_ANDROID`

 `EMIT_LDR_ADDRESS (X5, frida_resolve_libc_function (params->pid,` `"pthread_detach"``));`

`#else`

 `EMIT_LOAD_FIELD (X0, pthread_so);`

 `EMIT_CALL_IMM (params->dlsym_impl,`

 `2,`

 `ARG_REG (X0),`

 `ARG_IMM (FRIDA_REMOTE_DATA_FIELD (pthread_detach_string)));`

 `EMIT_MOVE (X5, X0);`

`#endif`

 `EMIT_LOAD_FIELD (X0, worker_thread);`

 `EMIT_CALL_REG (X5,`

 `1,`

 `ARG_REG (X0));`

 `}`

 `EMIT_LABEL (skip_detach);`

`#ifndef HAVE_ANDROID`

 `EMIT_LOAD_FIELD (X0, pthread_so);`

 `EMIT_CALL_IMM (params->dlclose_impl,`

 `1,`

 `ARG_REG (X0));`

`#endif`

 `EMIT_MOVE (X1, SP);`

 `EMIT_CALL_IMM (params->write_impl,`

 `3,`

 `ARG_REG (W21),`

 `ARG_REG (X1),`

 `ARG_IMM (1));`

 `EMIT_POP (X0, X1);`

 `EMIT_CALL_IMM (params->syscall_impl,`

 `1,`

 `ARG_IMM (__NR_gettid));`

 `EMIT_PUSH (X0, X1);`

 `EMIT_MOVE (X1, SP);`

 `EMIT_CALL_IMM (params->write_impl,`

 `3,`

 `ARG_REG (W21),`

 `ARG_REG (X1),`

 `ARG_IMM (4));`

 `EMIT_POP (X0, X1);`

 `EMIT_CALL_IMM (params->close_impl,`

 `1,`

 `ARG_REG (W21));`

 `EMIT_POP (X19, X20);`

 `EMIT_POP (X21, X22);`

 `EMIT_POP (FP, LR);`

 `EMIT_RET ();`

 `frida_inject_instance_commit_arm64_code (&cw, code);`

 `gum_arm64_writer_clear (&cw);`

`}`

可以看到`frida-helper`启动之后就开启了 Dbus 监听，将`Frida.ObjectPath.HELPER`绑定到`LinuxHelperService`类。在 frida-helper 中会通过 dbus 获取该类并调用相关方法。

在`main`函数中调用`REPLApplication`的`run`方法，实际是调用到了`application.py:375`的`ConsoleApplication`的`run`方法。而`run`方法中又调用了`self._try_start`方法。

`_try_start`首先是通过一系列的操作找到指定的设备，然后通过`frida.core.Device`的`spawn`方法启动目标程序（本次只分析`-f`方式启动的流程，其他方式大同小异）。

首先分析`frida`是如何找到设备的。点击`add_remote_device`跟踪进去发现又向下调用了一个`self._impl.add_remote_device`，最终定位到了`site-packages/_frida/__init__.pyi`

熟悉的读者可以发现这是一个`CPython`编写的模块，它实际的实现在`site-packages/_frida.abi3.so`（macos 环境）, 在代码仓库中是通过编译`frida-python/src/_frida.c`生成的。

在`_frida.c`中对应的方法是`PyDeviceManager_add_remote_device`，通过调用`frida_device_manager_add_remote_device_sync`来进行核心实现，而`frida_device_manager_add_remote_device_sync`实际是通过开头的`#include <frida-core.h>`导入的。  
![](https://bbs.kanxue.com/upload/attach/202308/923166_3CDZXH472AQYVVN.jpg)  
`frida-core.h`是在 meson 编译的过程中生成到`build/<target>/include/frida-1.0/frida-core.h`, 实际的代码实现是在`frida-core/src/frida.vala`。所以`frida_device_manager_add_remote_device_sync`是在`frida.vala:288`的`device_manager`类的`add_remote_device_sync`方法实现的。

核心实现是 244 行，new 了一个`Device`指定了 id，name，标记为`HostSessionProviderKind.REMOTE`, 并指定`HostSessionProvider`为`socket_device.provider`。现在我们已经得到了一个 Device，接下来我们看一下 frida 是如何 spawn 一个新的程序的。

`host_session`通过`provider`的`create`方法创建。这里的`provider`就是之前的 device 中指定的`socket_device.provider`。

然后通过 websocket 拿到的 connection 连接到远程的 DBus 最然后拿到`ObjectPath.HOST_SESSION`接口。通过之前对`frida-server`的分析我们知道这个接口对应的是`ControlChannel`类。

在实例化`ControlChannel`的时候`parent`为`ControlService`, 而`ControlService`的`host_session`为`LinuxHostSession`, 那么显然这里就是在调用`LinuxHostSession.spawn()`。

这里进行了判断，如果带了`/`的话就使用`helper`也就是`LinuxHelperProcess`来进行启动，我们这里分析的是 app 所以不带`/`。继续查看`RoboLauncher`。