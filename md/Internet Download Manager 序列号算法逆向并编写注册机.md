> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1451438-1-1.html)

> [md]## 1. 前言 **idm version : 6.38 Build 23**## 2. 算法逆向 IDM 的序列号验证函数定位在：下面是在 IDA 下的代码分析:```assembly ... Inte......

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)kei07251. 前言
-----

**idm version : 6.38 Build 23**

2. 算法逆向
-------

IDM 的序列号验证函数定位在：

![](https://attach.52pojie.cn/forum/202106/01/160855gsxfrvfn5rbpqfzx.png)

**idman_20210514184509943.png** _(402.71 KB, 下载次数: 3)_

[下载附件](forum.php?mod=attachment&aid=MjI5ODYxM3w2N2FiMGQ3NXwxNjIyNzEwMzI2fDIxMzQzMXwxNDUxNDM4&nothumb=yes)

2021-6-1 16:08 上传

下面是在 IDA 下的代码分析:

```
.text:00510010                 push    ebp
.text:00510011                 lea     ebp, [esp-1FCh]
.text:00510018                 sub     esp, 1FCh
.text:0051001E                 push    0FFFFFFFFh
.text:00510020                 push    offset SEH_510010
.text:00510025                 mov     eax, large fs:0
.text:0051002B                 push    eax
.text:0051002C                 sub     esp, 160h
.text:00510032                 mov     eax, ___security_cookie
.text:00510037                 xor     eax, ebp
.text:00510039                 mov     [ebp+1FCh+var_4], eax
.text:0051003F                 push    ebx
.text:00510040                 push    esi
.text:00510041                 push    edi
.text:00510042                 push    eax
.text:00510043                 lea     eax, [ebp+1FCh+var_208]
.text:00510046                 mov     large fs:0, eax
.text:0051004C                 mov     [ebp+1FCh+var_20C], esp
.text:0051004F                 mov     edi, ecx
.text:00510051                 mov     [ebp+1FCh+var_234], edi
.text:00510054                 mov     encode36, 32h ; '2' ; char basecode36[] = {0x32, 0x59, 0x4F, 0x50, 0x42, 0x33, 0x41, 0x51, 0x43, 0x56, 0x55, 0x58, 0x4D, 0x4E, 0x52, 0x53,0x39, 0x37, 0x57, 0x45, 0x30, 0x49, 0x5A, 0x44, 0x34, 0x4B, 0x4C, 0x46, 0x47, 0x48, 0x4A, 0x38,0x31, 0x36, 0x35, 0x54};
.text:0051005B                 mov     encode36+1, 59h ; 'Y'
.text:00510062                 mov     encode36+2, 4Fh ; 'O'
.text:00510069                 mov     encode36+3, 50h ; 'P'
.text:00510070                 mov     encode36+4, 42h ; 'B'
.text:00510077                 mov     encode36+5, 33h ; '3'
.text:0051007E                 mov     encode36+6, 41h ; 'A'
.text:00510085                 mov     encode36+7, 51h ; 'Q'
.text:0051008C                 mov     encode36+8, 43h ; 'C'
.text:00510093                 mov     encode36+9, 56h ; 'V'
.text:0051009A                 mov     encode36+0Ah, 55h ; 'U'
.text:005100A1                 mov     encode36+0Bh, 58h ; 'X'
.text:005100A8                 mov     encode36+0Ch, 4Dh ; 'M'
.text:005100AF                 mov     encode36+0Dh, 4Eh ; 'N'
.text:005100B6                 mov     encode36+0Eh, 52h ; 'R'
.text:005100BD                 mov     encode36+0Fh, 53h ; 'S'
.text:005100C4                 mov     encode36+10h, 39h ; '9'
.text:005100CB                 mov     encode36+11h, 37h ; '7'
.text:005100D2                 mov     encode36+12h, 57h ; 'W'
.text:005100D9                 mov     encode36+13h, 45h ; 'E'
.text:005100E0                 mov     encode36+14h, 30h ; '0'
.text:005100E7                 mov     encode36+15h, 49h ; 'I'
.text:005100EE                 mov     encode36+16h, 5Ah ; 'Z'
.text:005100F5                 mov     encode36+17h, 44h ; 'D'
.text:005100FC                 mov     encode36+18h, 34h ; '4'
.text:00510103                 mov     encode36+19h, 4Bh ; 'K'
.text:0051010A                 mov     encode36+1Ah, 4Ch ; 'L'
.text:00510111                 mov     encode36+1Bh, 46h ; 'F'
.text:00510118                 mov     encode36+1Ch, 47h ; 'G'
.text:0051011F                 mov     encode36+1Dh, 48h ; 'H'
.text:00510126                 mov     encode36+1Eh, 4Ah ; 'J'
.text:0051012D                 mov     encode36+1Fh, 38h ; '8'
.text:00510134                 mov     encode36+20h, 31h ; '1'
.text:0051013B                 mov     encode36+21h, 36h ; '6'
.text:00510142                 mov     encode36+22h, 35h ; '5'
.text:00510149                 mov     encode36+23h, 54h ; 'T'
.text:00510150                 mov     [ebp+1FCh+var_200], 0
.text:00510157                 push    32h ; '2'       ; int
.text:00510159                 lea     eax, [ebp+1FCh+Data]
.text:0051015F                 push    eax             ; lpString
.text:00510160                 push    4B0h            ; nIDDlgItem
.text:00510165                 call    ?GetDlgItemTextA@CWnd@@QBEHHPADH@Z ; CWnd::GetDlgItemTextA(int,char *,int)
.text:0051016A                 test    eax, eax
.text:0051016C                 jnz     short loc_5101AC ; 获取注册名字并判断是否成功
.text:0051016E                 push    eax             ; uType
.text:0051016F                 push    offset Caption  ; "Internet Download Manager"
.text:00510174                 mov     ecx, dword_716978
.text:0051017A
.text:0051017A loc_51017A:                             ; CODE XREF: SerialCheck+1C1↓j
.text:0051017A                                         ; SerialCheck+1E8↓j ...
.text:0051017A                 push    ecx             ; lpMultiByteStr
.text:0051017B                 mov     edx, [edi+20h]
.text:0051017E                 push    edx             ; hWnd
.text:0051017F
.text:0051017F loc_51017F:                             ; CODE XREF: SerialCheck+271↓j
.text:0051017F                                         ; SerialCheck+3F7↓j
.text:0051017F                 call    MyMessageBox    ; 弹出假冒序列号窗口函数
.text:00510184                 add     esp, 10h
.text:00510187
.text:00510187 loc_510187:                             ; CODE XREF: SerialCheck+5CF↓j
.text:00510187                                         ; SerialCheck+63B↓j ...
.text:00510187                 mov     ecx, [ebp+1FCh+var_208]
.text:0051018A                 mov     large fs:0, ecx
.text:00510191                 pop     ecx
.text:00510192                 pop     edi
.text:00510193                 pop     esi
.text:00510194                 pop     ebx
.text:00510195                 mov     ecx, [ebp+1FCh+var_4]
.text:0051019B                 xor     ecx, ebp        ; StackCookie
.text:0051019D                 call    @__security_check_cookie@4 ; __security_check_cookie(x)
.text:005101A2                 add     ebp, 1FCh
.text:005101A8                 mov     esp, ebp
.text:005101AA                 pop     ebp
.text:005101AB                 retn
.text:005101AC ; ---------------------------------------------------------------------------
.text:005101AC
.text:005101AC loc_5101AC:                             ; CODE XREF: SerialCheck+15C↑j
.text:005101AC                 push    32h ; '2'       ; int
.text:005101AE                 lea     eax, [ebp+1FCh+var_CC]
.text:005101B4                 push    eax             ; lpString
.text:005101B5                 push    413h            ; nIDDlgItem
.text:005101BA                 mov     ecx, edi        ; this
.text:005101BC                 call    ?GetDlgItemTextA@CWnd@@QBEHHPADH@Z ; CWnd::GetDlgItemTextA(int,char *,int)
.text:005101C1                 test    eax, eax
.text:005101C3                 jnz     short loc_5101D3 ; 获取注册姓氏并判断是否成功
.text:005101C5                 push    eax
.text:005101C6                 push    offset Caption  ; "Internet Download Manager"
.text:005101CB                 mov     ecx, dword_71697C
.text:005101D1                 jmp     short loc_51017A
.text:005101D3 ; ---------------------------------------------------------------------------
.text:005101D3
.text:005101D3 loc_5101D3:                             ; CODE XREF: SerialCheck+1B3↑j
.text:005101D3                 push    32h ; '2'       ; int
.text:005101D5                 lea     eax, [ebp+1FCh+var_100]
.text:005101DB                 push    eax             ; lpString
.text:005101DC                 push    4A5h            ; nIDDlgItem
.text:005101E1                 mov     ecx, edi        ; this
.text:005101E3                 call    ?GetDlgItemTextA@CWnd@@QBEHHPADH@Z ; CWnd::GetDlgItemTextA(int,char *,int)
.text:005101E8                 test    eax, eax
.text:005101EA                 jnz     short loc_5101FA ; 获取注册邮箱并判断是否成功
.text:005101EC                 push    eax
.text:005101ED                 push    offset Caption  ; "Internet Download Manager"
.text:005101F2                 mov     ecx, dword_716980
.text:005101F8                 jmp     short loc_51017A
.text:005101FA ; ---------------------------------------------------------------------------
.text:005101FA
.text:005101FA loc_5101FA:                             ; CODE XREF: SerialCheck+1DA↑j
.text:005101FA                 push    32h ; '2'       ; int
.text:005101FC                 lea     eax, [ebp+1FCh+RegisterSerialNumber]
.text:00510202                 push    eax             ; lpString
.text:00510203                 push    4AAh            ; nIDDlgItem
.text:00510208                 mov     ecx, edi        ; this
.text:0051020A                 call    ?GetDlgItemTextA@CWnd@@QBEHHPADH@Z ; CWnd::GetDlgItemTextA(int,char *,int)
.text:0051020F                 test    eax, eax
.text:00510211                 jnz     short loc_510224 ; 获取序列号并判断是否成功
.text:00510213                 push    eax
.text:00510214                 push    offset Caption  ; "Internet Download Manager"
.text:00510219                 mov     ecx, dword_716984
.text:0051021F                 jmp     loc_51017A
.text:00510224 ; ---------------------------------------------------------------------------
.text:00510224
.text:00510224 loc_510224:                             ; CODE XREF: SerialCheck+201↑j
.text:00510224                 mov     bl, 20h ; ' '   ; bl = 0x20(' ')
.text:00510226
.text:00510226 loc_510226:                             ; CODE XREF: SerialCheck+24A↓j
.text:00510226                 cmp     [ebp+1FCh+RegisterSerialNumber], bl ; 比对序列号的第一位是否是空格
.text:0051022C                 jnz     short loc_51025C
.text:0051022E                 push    32h ; '2'       ; int
.text:00510230                 lea     eax, [ebp+1FCh+var_57]
.text:00510236                 push    eax             ; Source
.text:00510237                 lea     ecx, [ebp+1FCh+var_1FC]
.text:0051023A                 push    ecx             ; Destination
.text:0051023B                 call    sub_405380
.text:00510240                 add     esp, 0Ch
.text:00510243                 lea     ecx, [ebp+1FCh+var_1FC]
.text:00510246                 lea     edx, [ebp+1FCh+RegisterSerialNumber]
.text:0051024C                 lea     esp, [esp+0]
.text:00510250
.text:00510250 loc_510250:                             ; CODE XREF: SerialCheck+248↓j
.text:00510250                 mov     al, [ecx]
.text:00510252                 mov     [edx], al
.text:00510254                 inc     ecx
.text:00510255                 inc     edx
.text:00510256                 test    al, al
.text:00510258                 jnz     short loc_510250
.text:0051025A                 jmp     short loc_510226
.text:0051025C ; ---------------------------------------------------------------------------
.text:0051025C
.text:0051025C loc_51025C:                             ; CODE XREF: SerialCheck+21C↑j
.text:0051025C                 lea     eax, [ebp+1FCh+RegisterSerialNumber] ; EAX = 序列号的第一位地址
.text:00510262                 lea     edx, [eax+1]    ; EDX = 序列号的第二位的地址
.text:00510265
.text:00510265 loc_510265:                             ; CODE XREF: SerialCheck+25A↓j
.text:00510265                 mov     cl, [eax]
.text:00510267                 inc     eax
.text:00510268                 test    cl, cl
.text:0051026A                 jnz     short loc_510265
.text:0051026C                 sub     eax, edx        ; 计算序列号的个数
.text:0051026E                 jnz     short loc_510290
.text:00510270                 push    eax
.text:00510271                 push    offset Caption  ; "Internet Download Manager"
.text:00510276                 mov     edx, dword_716988
.text:0051027C                 push    edx
.text:0051027D                 mov     eax, [edi+20h]
.text:00510280                 push    eax
.text:00510281                 jmp     loc_51017F
.text:00510281 ; } // starts at 510010
.text:00510281 SerialCheck     endp
.text:00510281
.text:00510286 ; ---------------------------------------------------------------------------
.text:00510286                 jmp     short loc_510290
.text:00510286 ; ---------------------------------------------------------------------------
.text:00510288                 align 10h
.text:00510290 ; START OF FUNCTION CHUNK FOR SerialCheck
.text:00510290
.text:00510290 loc_510290:                             ; CODE XREF: SerialCheck+25E↑j
.text:00510290                                         ; .text:00510286↑j ...
.text:00510290 ; __unwind { // SEH_510010              ; EAX = 序列号的第一位地址
.text:00510290                 lea     eax, [ebp+1FCh+RegisterSerialNumber]
.text:00510296                 lea     edx, [eax+1]    ; EDX = 序列号的第二位地址
.text:00510299                 lea     esp, [esp+0]
.text:005102A0
.text:005102A0 loc_5102A0:                             ; CODE XREF: SerialCheck+295↓j
.text:005102A0                 mov     cl, [eax]
.text:005102A2                 inc     eax
.text:005102A3                 test    cl, cl
.text:005102A5                 jnz     short loc_5102A0
.text:005102A7                 sub     eax, edx        ; 计算序列号的长度
.text:005102A9                 cmp     [ebp+eax+1FCh+var_59], bl ; 比对序列号的最后一位是否等于0x20(' ')
.text:005102B0                 jnz     short loc_5102D2
.text:005102B2                 lea     eax, [ebp+1FCh+RegisterSerialNumber]
.text:005102B8                 lea     edx, [eax+1]
.text:005102BB                 jmp     short loc_5102C0
.text:005102BB ; } // starts at 510290
.text:005102BB ; END OF FUNCTION CHUNK FOR SerialCheck
.text:005102BB ; ---------------------------------------------------------------------------
.text:005102BD                 align 10h
.text:005102C0 ; START OF FUNCTION CHUNK FOR SerialCheck
.text:005102C0
.text:005102C0 loc_5102C0:                             ; CODE XREF: SerialCheck+2AB↑j
.text:005102C0                                         ; SerialCheck+2B5↓j
.text:005102C0 ; __unwind { // SEH_510010
.text:005102C0                 mov     cl, [eax]
.text:005102C2                 inc     eax
.text:005102C3                 test    cl, cl
.text:005102C5                 jnz     short loc_5102C0
.text:005102C7                 sub     eax, edx
.text:005102C9                 mov     [ebp+eax+1FCh+var_59], cl
.text:005102D0                 jmp     short loc_510290
.text:005102D2 ; ---------------------------------------------------------------------------
.text:005102D2
.text:005102D2 loc_5102D2:                             ; CODE XREF: SerialCheck+2A0↑j
.text:005102D2                 lea     ecx, [ebp+1FCh+RegisterSerialNumber]
.text:005102D8                 push    ecx             ; String
.text:005102D9                 call    __strupr        ; 把序列号全部为大写
.text:005102DE                 add     esp, 4
.text:005102E1                 lea     eax, [ebp+1FCh+RegisterSerialNumber] ; EAX = 序列号的第一位地址
.text:005102E7                 lea     edx, [eax+1]    ; EDX = 序列号的第二位地址
.text:005102EA                 lea     ebx, [ebx+0]
.text:005102F0
.text:005102F0 loc_5102F0:                             ; CODE XREF: SerialCheck+2E5↓j
.text:005102F0                 mov     cl, [eax]
.text:005102F2                 inc     eax
.text:005102F3                 test    cl, cl
.text:005102F5                 jnz     short loc_5102F0
.text:005102F7                 sub     eax, edx        ; 计算序列号的长度
.text:005102F9                 cmp     eax, 17h
.text:005102FC                 jnz     loc_5103F6      ; 判断序列号的长短是否等于23（0x17）
.text:00510302                 xor     bl, bl          ; bl = 0;
.text:00510304                 mov     al, 2Dh ; '-'   ; al = '-'(0x2d);
.text:00510306                 cmp     [ebp+1FCh+var_53], al ; 判断序列号的第六位是否等于'-'(0x2d)
.text:0051030C                 jnz     short loc_51031E
.text:0051030E                 cmp     [ebp+1FCh+var_4D], al ; 判断序列号的第12位是否等于'-'(0x2d)
.text:00510314                 jnz     short loc_51031E
.text:00510316                 cmp     [ebp+1FCh+var_47], al ; 判断序列号的第18位是否等于'-'(0x2d)
.text:0051031C                 jz      short loc_510320
.text:0051031E
.text:0051031E loc_51031E:                             ; CODE XREF: SerialCheck+2FC↑j
.text:0051031E                                         ; SerialCheck+304↑j
.text:0051031E                 mov     bl, 1
.text:00510320
.text:00510320 loc_510320:                             ; CODE XREF: SerialCheck+30C↑j
.text:00510320                 push    5               ; Count
.text:00510322                 lea     edx, [ebp+1FCh+RegisterSerialNumber]
.text:00510328                 push    edx             ; Source
.text:00510329                 lea     eax, [ebp+1FCh+RegisterSerialNumber_0_5]
.text:0051032F                 push    eax             ; Destination
.text:00510330                 call    _strncpy        ; 取出序列号的0-5位
.text:00510335                 push    5               ; Count
.text:00510337                 lea     ecx, [ebp+1FCh+Source]
.text:0051033D                 push    ecx             ; Source
.text:0051033E                 lea     edx, [ebp+1FCh+RegisterSerialNumber_6_11]
.text:00510344                 push    edx             ; Destination
.text:00510345                 call    _strncpy        ; 取出序列号的6-11位
.text:0051034A                 push    5               ; Count
.text:0051034C                 lea     eax, [ebp+1FCh+var_4C]
.text:00510352                 push    eax             ; Source
.text:00510353                 lea     ecx, [ebp+1FCh+RegisterSerialNumber_13_17]
.text:00510359                 push    ecx             ; Destination
.text:0051035A                 call    _strncpy        ; 取出序列号的13-17位
.text:0051035F                 push    5               ; Count
.text:00510361                 lea     edx, [ebp+1FCh+var_46]
.text:00510367                 push    edx             ; Source
.text:00510368                 lea     eax, [ebp+1FCh+RegisterSerialNumber_18_23]
.text:0051036E                 push    eax             ; Destination
.text:0051036F                 call    _strncpy        ; 取出序列号的18-23位
.text:00510374                 mov     [ebp+1FCh+var_7], 0
.text:0051037B                 mov     [ebp+1FCh+var_F], 0
.text:00510382                 mov     [ebp+1FCh+var_1F], 0
.text:00510389                 mov     [ebp+1FCh+var_17], 0 ; 在每个取出的5位序列号后面添加0x00
.text:00510390                 lea     ecx, [ebp+1FCh+encode_1]
.text:00510393                 push    ecx
.text:00510394                 lea     edx, [ebp+1FCh+RegisterSerialNumber_0_5]
.text:0051039A                 push    edx
.text:0051039B                 call    SerialEncrypt   ; 序列号加密函数
.text:005103A0                 add     esp, 38h
.text:005103A3                 test    eax, eax
.text:005103A5                 jnz     short loc_5103A9 ; 返回值等于1跳转,等于0则 bl = 0x01
.text:005103A7                 mov     bl, 1
.text:005103A9
.text:005103A9 loc_5103A9:                             ; CODE XREF: SerialCheck+395↑j
.text:005103A9                 lea     eax, [ebp+1FCh+encode_2+4]
.text:005103AC                 push    eax
.text:005103AD                 lea     ecx, [ebp+1FCh+RegisterSerialNumber_6_11]
.text:005103B3                 push    ecx
.text:005103B4                 call    SerialEncrypt   ; 序列号加密函数
.text:005103B9                 add     esp, 8
.text:005103BC                 test    eax, eax
.text:005103BE                 jnz     short loc_5103C2 ; 返回值等于1跳转,等于0则 bl = 0x01
.text:005103C0                 mov     bl, 1
.text:005103C2
.text:005103C2 loc_5103C2:                             ; CODE XREF: SerialCheck+3AE↑j
.text:005103C2                 lea     edx, [ebp+1FCh+encode_3]
.text:005103C5                 push    edx
.text:005103C6                 lea     eax, [ebp+1FCh+RegisterSerialNumber_13_17]
.text:005103CC                 push    eax
.text:005103CD                 call    SerialEncrypt   ; 序列号加密函数
.text:005103D2                 add     esp, 8
.text:005103D5                 test    eax, eax
.text:005103D7                 jnz     short loc_5103DB ; 返回值等于1跳转,等于0则 bl = 0x01
.text:005103D9                 mov     bl, 1
.text:005103DB
.text:005103DB loc_5103DB:                             ; CODE XREF: SerialCheck+3C7↑j
.text:005103DB                 lea     ecx, [ebp+1FCh+encode_4]
.text:005103DE                 push    ecx
.text:005103DF                 lea     edx, [ebp+1FCh+RegisterSerialNumber_18_23]
.text:005103E5                 push    edx
.text:005103E6                 call    SerialEncrypt   ; 序列号加密函数
.text:005103EB                 add     esp, 8
.text:005103EE                 test    eax, eax
.text:005103F0                 jz      short loc_5103F6
.text:005103F2                 test    bl, bl
.text:005103F4                 jz      short loc_51040C ; 判断上面的每个函数都返回1 则跳转
.text:005103F6
.text:005103F6 loc_5103F6:                             ; CODE XREF: SerialCheck+2EC↑j
.text:005103F6                                         ; SerialCheck+3E0↑j ...
.text:005103F6                 push    0
.text:005103F8                 push    offset Caption  ; "Internet Download Manager"
.text:005103FD                 mov     eax, dword_716988
.text:00510402                 push    eax
.text:00510403                 mov     ecx, [edi+20h]
.text:00510406                 push    ecx
.text:00510407                 jmp     loc_51017F
.text:0051040C ; ---------------------------------------------------------------------------
.text:0051040C
.text:0051040C loc_51040C:                             ; CODE XREF: SerialCheck+3E4↑j
.text:0051040C                 mov     ecx, [ebp+1FCh+encode_1]
.text:0051040F                 mov     eax, 2FA0BE83h
.text:00510414                 imul    ecx
.text:00510416                 sar     edx, 3
.text:00510419                 mov     eax, edx
.text:0051041B                 shr     eax, 1Fh
.text:0051041E                 add     eax, edx
.text:00510420                 imul    eax, 2Bh ; '+'
.text:00510423                 mov     edx, ecx
.text:00510425                 sub     edx, eax        ; edx = encode1 % 43;
.text:00510427                 jnz     short loc_51042D ; 判断序列号第一部分的加密的结果除于43取余是否等于0
.text:00510429                 test    ecx, ecx
.text:0051042B                 jnz     short loc_51042F
.text:0051042D
.text:0051042D loc_51042D:                             ; CODE XREF: SerialCheck+417↑j
.text:0051042D                 mov     bl, 1
.text:0051042F
.text:0051042F loc_51042F:                             ; CODE XREF: SerialCheck+41B↑j
.text:0051042F                 mov     ecx, dword ptr [ebp+1FCh+encode_2+4]
.text:00510432                 mov     eax, 0B21642C9h
.text:00510437                 imul    ecx
.text:00510439                 add     edx, ecx
.text:0051043B                 sar     edx, 4
.text:0051043E                 mov     eax, edx
.text:00510440                 shr     eax, 1Fh
.text:00510443                 add     eax, edx
.text:00510445                 imul    eax, 17h
.text:00510448                 mov     edx, ecx
.text:0051044A                 sub     edx, eax        ; edx = encode2 % 23;
.text:0051044C                 jnz     short loc_510452 ; 判断序列号第二部分的加密的结果除于23取余是否等于0
.text:0051044E                 test    ecx, ecx
.text:00510450                 jnz     short loc_510454
.text:00510452
.text:00510452 loc_510452:                             ; CODE XREF: SerialCheck+43C↑j
.text:00510452                 mov     bl, 1
.text:00510454
.text:00510454 loc_510454:                             ; CODE XREF: SerialCheck+440↑j
.text:00510454                 mov     ecx, [ebp+1FCh+encode_3]
.text:00510457                 mov     eax, 78787879h
.text:0051045C                 imul    ecx
.text:0051045E                 sar     edx, 3
.text:00510461                 mov     eax, edx
.text:00510463                 shr     eax, 1Fh
.text:00510466                 add     eax, edx
.text:00510468                 mov     edx, eax
.text:0051046A                 shl     edx, 4
.text:0051046D                 add     edx, eax
.text:0051046F                 mov     eax, ecx
.text:00510471                 sub     eax, edx        ; eax = encode_3 % 17;
.text:00510473                 jnz     short loc_510479 ; 判断序列号第三部分的加密的结果除于17取余是否等于0
.text:00510475                 test    ecx, ecx
.text:00510477                 jnz     short loc_51047B
.text:00510479
.text:00510479 loc_510479:                             ; CODE XREF: SerialCheck+463↑j
.text:00510479                 mov     bl, 1
.text:0051047B
.text:0051047B loc_51047B:                             ; CODE XREF: SerialCheck+467↑j
.text:0051047B                 mov     ecx, [ebp+1FCh+encode_4]
.text:0051047E                 mov     eax, 4D4873EDh
.text:00510483                 imul    ecx
.text:00510485                 sar     edx, 4
.text:00510488                 mov     eax, edx
.text:0051048A                 shr     eax, 1Fh
.text:0051048D                 add     eax, edx
.text:0051048F                 imul    eax, 35h ; '5'
.text:00510492                 mov     edx, ecx
.text:00510494                 sub     edx, eax        ; edx = encode_4 % 53;
.text:00510496                 jnz     loc_5103F6      ; 判断序列号第四部分的加密的结果除于53取余是否等于0
.text:0051049C                 test    ecx, ecx
.text:0051049E                 jz      loc_5103F6
.text:005104A4                 test    bl, bl
.text:005104A6                 jnz     loc_5103F6      ; 判断bl是否等于1 也就是判断上面的验证是否都是正确的
.text:005104AC                 push    8               ; Count
.text:005104AE                 lea     edx, [ebp+1FCh+RegisterSerialNumber]
.text:005104B4                 push    edx             ; Source
.text:005104B5                 lea     eax, [ebp+1FCh+ArgList]
.text:005104BB                 push    eax             ; Destination
.text:005104BC                 call    _strncpy        ; 取出序列号的前八位

```

​        根据上面的代码逆向出序列号验证流程：

*   1. 获取注册的 名字 姓氏 邮箱 序列号
    
*   2. 如果序列号的开头或结尾是空格的话就去掉空格
    
*   3. 把序列号转换成大写
    
*   4. 判断序列号的长度是否等于 23(0x17), 并序列号的第 6 位, 第 12 位，第 18 位是否等于‘-’(0x2d)
    
    *   这里就可得出序列号的格式是`xxxxx-xxxxx-xxxxx-xxxxx`
*   5. 分别取出 4 部分的 5 位序列号`xxxxx`进行加密得到 4 个无符号数
    
*   6. 判断序列号是否正确，条件如下：
    
    *   第一部分的 5 位序列号加密后的无符号数除于 43 取余是否等于 0
        
    *   第二部分的 5 位的序列号加密后的无符号数处于 23 取余是否等于 0
        
    *   第三部分的 5 位的序列号加密后的无符号数处于 17 取余是否等于 0
        
    *   第四部分的 5 位的序列号加密后的无符号数处于 53 取余是否等于 0
        
*   之后就是把注册信息写入到注册表和网络验证，这里不讨论
    

在`text:0051039B`,`005103B4`,`text:005103CD`,`text:005103E6` 这 4 处调用的加密函数 IDA 反汇编代码如下：

```
.text:0050F050 SerialEncrypt   proc near               ; CODE XREF: SerialCheck+38B↓p
.text:0050F050                                         ; SerialCheck+3A4↓p ...
.text:0050F050
.text:0050F050 arg_0           = dword ptr  4
.text:0050F050 arg_4           = dword ptr  8
.text:0050F050
.text:0050F050                 push    esi
.text:0050F051                 mov     esi, [esp+4+arg_4] ; esi = arg_4;
.text:0050F055                 push    edi
.text:0050F056                 mov     edi, [esp+8+arg_0] ; edi = arg_0;
.text:0050F05A                 mov     dword ptr [esi], 0 ; *esi = (4 byte)0x00;
.text:0050F060                 xor     ecx, ecx        ; ecx = 0
.text:0050F062
.text:0050F062 loc_50F062:                             ; CODE XREF: SerialEncrypt+3C↓j
.text:0050F062                 mov     dl, [ecx+edi]   ; dl =  arg_0[ecx];
.text:0050F065                 xor     eax, eax        ; eax = 0;
.text:0050F067
.text:0050F067 loc_50F067:                             ; CODE XREF: SerialEncrypt+23↓j
.text:0050F067                 cmp     encode36[eax], dl
.text:0050F06D                 jz      short loc_50F07A ; 判断encode36[eax]是否等于arg_0[ecx]
.text:0050F06F                 inc     eax             ; eax++;
.text:0050F070                 cmp     eax, 24h ; '$'
.text:0050F073                 jl      short loc_50F067 ; 判断eax是否小于36(0x24)
.text:0050F075
.text:0050F075 loc_50F075:                             ; CODE XREF: SerialEncrypt+2D↓j
.text:0050F075                 pop     edi
.text:0050F076                 xor     eax, eax        ; return 0;
.text:0050F078                 pop     esi
.text:0050F079                 retn
.text:0050F07A ; ---------------------------------------------------------------------------
.text:0050F07A
.text:0050F07A loc_50F07A:                             ; CODE XREF: SerialEncrypt+1D↑j
.text:0050F07A                 cmp     eax, 0FFFFFFFFh
.text:0050F07D                 jz      short loc_50F075 ; 判断eax是否等于-1(0xFFFFFFFF)
.text:0050F07F                 mov     edx, [esi]
.text:0050F081                 imul    edx, 25h ; '%'
.text:0050F084                 add     edx, eax
.text:0050F086                 inc     ecx
.text:0050F087                 cmp     ecx, 5
.text:0050F08A                 mov     [esi], edx      ; *arg_0 = *arg_0 * 37(0x25) + eax;
.text:0050F08C                 jl      short loc_50F062 ; 判断eax是否小于5
.text:0050F08E                 pop     edi
.text:0050F08F                 mov     eax, 1          ; return 1;
.text:0050F094                 pop     esi
.text:0050F095                 retn
.text:0050F095 SerialEncrypt   endp

```

对应的 C++ 代码

```
const char kEnCode36[] = { 0x32, 0x59, 0x4F, 0x50, 0x42, 0x33, 0x41, 0x51, 0x43, 0x56, 0x55, 0x58, 0x4D, 0x4E, 0x52, 0x53,0x39, 0x37, 0x57, 0x45, 0x30, 0x49, 0x5A, 0x44, 0x34, 0x4B, 0x4C, 0x46, 0x47, 0x48, 0x4A, 0x38,0x31, 0x36, 0x35, 0x54 };

int __cdecl sub_50e990(const char* serial_number, int* arg_4)
{
        *arg_4 = 0;
        for (int serial_number_count = 0; serial_number_count < 5; serial_number_count++) {
                int encode36_count = 0;
                while (kEnCode36[encode36_count] != serial_number[serial_number_count]) {
                        encode36_count++;
                        if (encode36_count >= 36) return 0;
                }
                if (encode36_count == 0XFFFFFFFF) return 0;
                *arg_4 = *arg_4 * 37 + encode36_count;
        }
        return 1;

}

```

​    序列号分为 4 部分, 每部分的 5 位序列号分别用 sub_50e990 加密. Encode36 的内容为 `2YOPB3AQCVUXMNRS97WE0IZD4KLFGHJ8165T`实际上就是所有的字母和数字, sub_50e990 函数的加密过程就是先把 arg_4 初始化为 0，之后计算 arg_4 乘以 37 + 当前序列号在 Encode36 的 Count 得到的结果赋值给 arg_4 在进行下一次计算. 循环 5 次得到一个无符号数之后在进行相对应的整除判断。现在知道了他的序列号验证过程就可以反向推导出序列号

```
const char kEnCode36[] = { 0x32, 0x59, 0x4F, 0x50, 0x42, 0x33, 0x41, 0x51, 0x43, 0x56, 0x55, 0x58, 0x4D, 0x4E, 0x52, 0x53,0x39, 0x37, 0x57, 0x45, 0x30, 0x49, 0x5A, 0x44, 0x34, 0x4B, 0x4C, 0x46, 0x47, 0x48, 0x4A, 0x38,0x31, 0x36, 0x35, 0x54 };

int SnGenerate(unsigned int base,char* serial_number,int  serial_size)
{
    if (serial_size <= 5)  return 0;

    for (int i = 0; i < 5; i++) {
        for (int encode36_count = 0; encode36_count <= 35; encode36_count++) {
            if ((base - encode36_count) % 37 == 0) {
                base = (base - encode36_count) / 37;
                serial_number[i] = kEnCode36[encode36_count];
                break;
            }

            if (encode36_count == 35) return 0;
        }

    }
    return 1;    
}

```

​    完整的计算序列号代码：

```
#include <iostream>
#include <Windows.h>

using namespace std;

#pragma warning(disable:4996)

// "2YOPB3AQCVUXMNRS97WE0IZD4KLFGHJ8165T"
const char kEnCode36[] = { 0x32, 0x59, 0x4F, 0x50, 0x42, 0x33, 0x41, 0x51, 0x43, 0x56, 0x55, 0x58, 0x4D, 0x4E, 0x52, 0x53,0x39, 0x37, 0x57, 0x45, 0x30, 0x49, 0x5A, 0x44, 0x34, 0x4B, 0x4C, 0x46, 0x47, 0x48, 0x4A, 0x38,0x31, 0x36, 0x35, 0x54 };

int SnGenerate(unsigned int base,char* serial_number,int  serial_size)
{
    if (serial_size <= 5)  return 0;

    for (int i = 0; i < 5; i++) {
        for (int encode36_count = 0; encode36_count <= 35; encode36_count++) {
            if ((base - encode36_count) % 37 == 0) {
                base = (base - encode36_count) / 37;
                serial_number[i] = kEnCode36[encode36_count];
                break;
            }

            if (encode36_count == 35) return 0;
        }

    }
    return 1;    
}

//  USTWQ-3I9R6-BBIGL-M8LE8
int main()
{
    unsigned int base = 454651;
    unsigned int number1 = 43 * base;
    unsigned int number2 = 23 * base;
    unsigned int number3 = 17 * base;
    unsigned int number4 = 53 * base;

    char register_sn_0_5[6] = {0};
    char register_sn_6_11[6] = {0};
    char register_sn_13_17[6] = {0};
    char register_sn_18_23[6] = {0};

    SnGenerate(number1,register_sn_0_5,6);
    SnGenerate(number2,register_sn_6_11,6);
    SnGenerate(number3,register_sn_13_17,6);
    SnGenerate(number4,register_sn_18_23,6);

    printf("Serial Number: %s-%s-%s-%s \r\n",strrev(register_sn_0_5),strrev(register_sn_6_11),
           strrev(register_sn_13_17),strrev(register_sn_18_23));
}

``` ![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg)Hmily [@kei0725](https://www.52pojie.cn/home.php?mod=space&uid=1114243) markdown 的语法这有些问题 ```c++ 可能不行需要换成 ```cpp，我帮你修改了。  
加精鼓励，另外 IDM 是不是有网络验证？光 keygen 可能不够，再追加下网络验证分析？ ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 膜拜大神【表示看不懂编程】![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)瓜皮的小游戏 学习了。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)小猫猫 写的很详细，收藏了慢慢学习![](https://static.52pojie.cn/static/image/smiley/default/42.gif) ![](https://avatar.52pojie.cn/data/avatar/000/50/77/62_avatar_middle.jpg) dustyangel 注释很详细，感谢分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) hxd97244 随便看看![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)老道创 学习知识![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)霏映 膜拜大神【表示看不懂编程】![](https://avatar.52pojie.cn/data/avatar/000/48/78/73_avatar_middle.jpg)难（男）人  
学习了。