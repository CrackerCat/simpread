> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [yougame.biz](https://yougame.biz/threads/277698/)

> Зимний шалом. Статья делится на две части: первая часть была сделана ещё в августе, вторая - сегодня......

Зимний шалом.

Статья делится на две части: первая часть была сделана ещё в августе, вторая - сегодня вечером :)  
Причина такому долгому простою послужила учёба и много-много прочей рутины, даже сейчас я не обладаю каким-либо большим запасом времени, поэтому я просто дополнил материал в статье, не могу сказать, что это полностью завершённая статья.

**- Благодарности**  
За помощь в дополнительной и полезной информации я обязан нескольким источникам информации

```
back.engineering/17/05/2021/
t.me/battleyeblog (Сеня давал информацию о хендлерах и Control Flow)
katyscode.wordpress.com/2021/01/23/reverse-engineering-adventures-vmprotect-control-flow-obfuscation-case-study-string-algorithm-cryptanalysis-in-honkai-impact-3rd/

```

**- Предисловие**  
Во многих P2C проектах я видел, что люди в основном абузят слитую лицензию VMProtect 3.5 с билдом 1213, поэтому в некоторых моментах, где есть отличия я буду сравнивать две версии - 3.5 и 3.6.  
Также весь дизассемблированный код, который я прикладываю будет в основном очищен от джанк-кода, исключениями выступают демонстрации джанк-кода в виртуальной машине.

**- Информация о протекторах**  
VMProtect 3.5 - билд 1213 (Слитая лицензия)  
VMProtect 3.6 - билд 1406 (Слитая лицензия)

**Первая часть**

**- Обработчики виртуальной машины**

После того как юзер накрывает программу протектором, в секции .text появляются входы в VM, которые ведут уже в секцию протектора, где лежит исполняемый код, данные и контекст виртуальной машины.

Входы в VM были статичны в том плане, что их можно было найти по паттерну и начиная с VMProtect 3.6 этот паттерн частично ломается. Частично, потому что в 3.6 можно всё ещё составить правильный паттерн для нахождения входов в VM Entry.

Сравним вход в VM у 3.5 и 3.6 версии:

**VMProtect 3.5:**

```
push 2A95E159
call 7FF6EA991D85

```

**VMProtect 3.6:**

```
00007FF760C7106B  | PUSH R14                                               |
00007FF760C7106D  | PUSH R15                                               |
00007FF760C7106F  | CALL vmprotect36.vmp.7FF760C7570C                      |

00007FF760C7570C  | PUSHFQ                                                 |
00007FF760C7570D  | MOV R14,58AC28AC3F68476C                               |
00007FF760C75717  | ADD QWORD PTR SS:[RSP+8],FFFFFFFFBFFFEF8C              |
00007FF760C75727  | MOV R15,6E740BAA45855371                               |
00007FF760C75731  | JGE vmprotect36.vmp.7FF760C752E2                       |

00007FF760C752E6  | MOV R14,QWORD PTR SS:[RSP+18]                          |
00007FF760C752EB  | MOV QWORD PTR SS:[RSP+18],FFFFFFFFF60D74C5             |
00007FF760C752FD  | CALL vmprotect36.vmp.7FF760C7108B                      |

00007FF760C7108B  | CALL vmprotect36.vmp.7FF760C75302                      |

00007FF760C75302  | PUSH R11                                               |
00007FF760C75304  | MOV R11,2D4D16365288609E                               |
00007FF760C7530E  | MOVSX R11D,R11W                                        |
00007FF760C75312  | MOV R15,QWORD PTR SS:[RSP+28]                          |
00007FF760C75317  | TEST R11D,E1508A5                                      |
00007FF760C7531E  | JL vmprotect36.vmp.7FF760C7532C                        |
00007FF760C75333  | MOV R11,QWORD PTR SS:[RSP]                             |
00007FF760C75338  | CALL vmprotect36.vmp.7FF760C75881                      |

```

**Второй метод входа в VM для 3.6:**

```
00007FF650C151F5  | MOV QWORD PTR SS:[RSP],FFFFFFFFF44BB4F8                |
00007FF650C151FE  | CALL vmprotect36.vmp.7FF650C15269                      |

00007FF650C15269  | PUSH 36241C98                                          |
00007FF650C1526E  | CALL vmprotect36.vmp.7FF650C158BE                      |

00007FF650C158BE  | MOV WORD PTR SS:[RSP+8],231E                           |
00007FF650C158C5  | PUSH R13                                               |
00007FF650C158C7  | CALL vmprotect36.vmp.7FF650C150D4                      |

00007FF650C150D4  | PUSHFQ                                                 |
00007FF650C150D5  | MOV R13,QWORD PTR SS:[RSP+20]                          |
00007FF650C150DA  | MOV R13,QWORD PTR SS:[RSP+10]                          |
00007FF650C150DF  | AND WORD PTR SS:[RSP+20],289E                          |
00007FF650C150E6  | SUB WORD PTR SS:[RSP+20],4E                            |
00007FF650C150ED  | JL vmprotect36.vmp.7FF650C150FC                        |
00007FF650C150F3  | ADD QWORD PTR SS:[RSP+8],FFFFFFFFBFFFA734              |
00007FF650C150FC  | CMP QWORD PTR SS:[RSP+20],8B23C7D                      |
00007FF650C15105  | NOT WORD PTR SS:[RSP+20]                               |
00007FF650C1510A  | PUSH QWORD PTR SS:[RSP]                                |
00007FF650C1510E  | POPFQ                                                  |
00007FF650C1510F  | LEA RSP,QWORD PTR SS:[RSP+30]                          |
00007FF650C15114  | CALL vmprotect36.vmp.7FF650CD2288                      |

```

После перехода в VM Entry нас встречает декрипт адреса следующего блока с кодом, которому программа передаст управление. Естественно, всё это также будет окутано джанк-кодом.

```
00007FF6D1341DED  | mov rdx, 7FF5912A0000 ; Вшивает результат (Base Address - Image Base)
00007FF6D1341E09  | mov r8, qword ptr ss:[rsp+90]

```

Для х64 двоичных файлов зашифрованный относительный виртуальный адрес всегда будет находится по адресу RSP+90, это так же относится и к 3.6 версии.

```
00007FF6D1341E16  | bswap r8d                                              | Начало расшифровки адреса
00007FF6D1341E19  | sar al,cl                                              |
00007FF6D1341E1B  | sar di,58                                              |
00007FF6D1341E1F  | xadd cx,bp                                             |
00007FF6D1341E23  | neg r8d                                                |
00007FF6D1341E26  | dec r8d                                                |
00007FF6D1341E29  | rcl r10,E3                                             |
00007FF6D1341E2D  | movsx di,sil                                           |
00007FF6D1341E32  | sbb bpl,FA                                             |
00007FF6D1341E36  | xor r8d,261F7AAB                                       |
00007FF6D1341E3D  | cmc                                                    |
00007FF6D1341E3E  | ror r8d,1                                              |

```

После инструкции ror относительный виртуальный адрес уже будет расшифрованным, однако помимо ксора для преобразования может использоваться и обыкновенный add.

Пример взят с перекомпиленного двоичного файла, поэтому и регистр соответственно тоже используется другой.

```
00007FF6D1341E16  | add ebp, 2594586D ; Второй метод расшифровки адреса
00007FF6D1341E19  | neg ebp
00007FF6D1341E1B  | inc ebp
00007FF6D1341E1F  | rol ebp,1
00007FF6D1341E2D  | add rbp, r10 ; Конец расшифровки

00007FF6D1341E45  | lea r8,qword ptr ds:[r8+rdx]                           |
00007FF6D1341E49  | bts ecx,r12d                                           |
00007FF6D1341E52  | mov rax,100000000                                      |
00007FF6D1341E5F  | lea r8,qword ptr ds:[r8+rax]                           |

00007FF6D1341E66  | sub rsp,180                                            | Выравнивание стека
00007FF6D1341E74  | and rsp,FFFFFFFFFFFFFFF0                               |

```

У основного алгоритма тоже есть два метода расшифровки следующего хендлера

Первый метод:

```
00007FF6D1341E7B  | mov rdi,r8                                             |
00007FF6D1341E83  | mov r9,7FF5912A0000                                    |
00007FF6D1341E8D  | ror r10b,26                                            |
00007FF6D1341E91  | sub rdi,r9                                             |
00007FF6D1341E94  | sbb ecx,50BC75E8                                       |
00007FF6D1341E9A  | cmp r14b,69                                            |
00007FF6D1341E9E  | sub r10b,cl                                            |
00007FF6D1341EA1  | lea r10,qword ptr ds:[7FF6D1341EA1]                    |
00007FF6D1341EAF  | mov ecx,dword ptr ds:[r8]                              |
00007FF6D1341EB6  | add r8,4                                               |
00007FF6D1341EBD  | xor ecx,edi                                            |
00007FF6D135082C  | inc ecx                                                |
00007FF6D1350833  | not ecx                                                |
00007FF6D1350835  | bswap ecx                                              |
00007FF6D1350837  | add ecx,7E0005C8                                       |
00007FF6D135083E  | push rdi                                               |
00007FF6D135083F  | xor dword ptr ss:[rsp],ecx                             |
00007FF6D135084F  | pop rdi                                                |
00007FF6D1350850  | movsxd rcx,ecx                                         |

```

Второй метод:

```
00007FF6EF52103D  | mov r10d, dword ptr ds:[rdi]                            |
00007FF6EF58C876  | xor r10d, r11d                                          |
00007FF6EF58C879  | neg r10d                                               |
00007FF6EF58C87C  | rol r10d, 1                                             |
00007FF6EF4CBDDF  | add r10d,47BC214D                                      |
00007FF6EF4CBDE7  | neg r10d                                               |
00007FF6EF4C064A  | not r10d                                               |
00007FF6EF530B38  | dec r10d                                               |
00007FF6EF530B3B  | xor r10d,433320D7                                      |
00007FF6EF530B46  | sub r10d,29F1797C                                      |
00007FF6EF530B55  | push r11                                               |
00007FF6EF530B62  | xor dword ptr ss:[rsp],r10d                            |
00007FF6EF530B6A  | pop r11                                                |
00007FF6EF530B6C  | movsxd r10,r10d                                        |

```

В регистре уже есть расшифрованный адрес для последующего перехода к нему, для передачи управления VMProtect может использовать как и подмену адреса возврата, так и прямой джамп по адресу в регистре:

**Первый метод:**

```
add r8, r10 ; Вычисляем адрес
jmp r8 ; Прыгаем по адресу в R8

```

**Второй метод:**

```
add r10, rcx ; Вычисляем адрес
push r10 ; Помещаем адрес в стек
retп ; Передаем управление адресу, который положили в стек

```

- **Обфускация смешанной булевой арифметики**

Обфускация смешанной булевой арифметики - это метод для выполнения сохраняющего семантику преобразования из простого выражения в представление, которое в дальнейшем тяжело представить во время анализа. В дальнейшем я буду использовать аббревиатуру - **MBA (Mixed-Boolean-Arithmetic Obfuscation).**

Рассмотрим обфускацию по подробнее как раз на примере таргета под VMProtect.

**- Инструкция Sub**

Объясню вкратце начало обфускации: Берется первый операнд и инвертируется инструкцией NOT.

Думаю стоит разъяснить как именно работает эта инструкция на очень простом примере. Зачем? Дело в том, что информация об этой инструкции поможет нам понять принцип работы обфусцированного SUB и других ребилднутых по ходу статьи.

Инструкция NOT - это логическое отрицание одного единственного операнда, он инвертирует каждый бит операнда.

Представим число в RDX - **0x1337**, в битах это будет представлено как 0001001100110111. И затем инструкция NOT инвертирует каждый бит, т.е. 0 в 1, а 1 в 0. И получим в результате мы  
1110110011001000. А если перевести полученное значение в hex, то у нас получится - **0xECC8**.

Возьмем за пример: 4183 - 2000, чтобы не путаться: 4183 - в hex это 0x1057, а 2000 - 0x7D0

Как я уже и сказал выше операнд, в котором хранится число 4183 будет инвертировано в 0xEFA8, но протектор за собой в регистр тащит и 0xFFFF, поэтому в результате получается 0xFFFFEFA8, к этому числу плюсуется уже нетронутая 2000.  
После операции добавления мы получим 0x00000000FFFFF778, думаю вы уже догадались, что дальше будет делать протектор :)

Это почти, что конец ребилда. Число 0xFFFFF778 попадет под инструкцию NOT и мы получим в результате наше заветное число - 2183, в hex это будет 0x883.

Теперь попробуем представить эту функцию без всякого джанк-кода виртуальной машины. Будем использовать тот же алгоритм исполнения MBA обфусцированной инструкции, что и в виртуальной машине.

```
mov r10, dwFirstOp
mov rcx, dwSecondOp
not r10
add r10, rcx
not r10d
mov dwFirstOp, r10

```

Во всех представленных инструкциях я так же буду показывать и сишный код, и вот код Sub:

```
DWORD VMProtect::Arithmetic::ManualSub( DWORD dwFirstOp, DWORD dwSecondOp )
{
    return ~( dwSecondOp + ~dwFirstOp );
}

```

**- Инструкция Xor**

Вспоминаем основы дискретной математики. Xor - операция исключающего ИЛИ. Сравниваются все биты двух операндов между собой, в результате 1 бит может записаться только в том случае, если сравниваемые биты не были равны друг другу. Пример:

```
Первый операнд            Второй      Результат

            0     XOR     0     =     0

            0     XOR     1     =     1

            1     XOR     0     =     1

            1     XOR     1     =     0

```

Теперь посмотри как VMProtect смог ребилднуть эту инструкцию.

Сам конец операции выглядит таким образом:

```
not edx
not r11d
and edx, r11d

```

Но каково было мое удивление, когда вместо изначальных значений операндов я получил совершенно разные числа, но по итогу вычислений в конце все равно был верный результат.

[![](https://yougame.biz/data/attachments/232/232446-5e238ec4af29549677f4d95c0c24127c.jpg)](https://yougame.biz/attachments/232581/)

До выполнения операции NOT на EDX и R11D хранились значения 0xFFFFE828 и 0x50. Если еще с первым числом понятно, что до этого над ним повторно выполнялась инструкция NOT, то с 0x50 ничего не понятно. Начинаем снова просматривать дизасм код.

И попадаем на следующий код:

[![](https://yougame.biz/data/attachments/232/232447-5ba73fad9709599d7e7f260a512f9f78.jpg)](https://yougame.biz/attachments/232582/)

Опять наши два операнда инвертируются, и после того как мы прыгаем на следующий вм блок, то нас сразу же встречает инструкция and.

[![](https://yougame.biz/data/attachments/232/232448-43a3da057ed194dc0065de4bd1dc2bb8.jpg)](https://yougame.biz/attachments/232583/)

После and edx, r11d в регистр edx будет записано число FFFFE828, то самое число, которое мы встретили в конце операции ксора, в дальнейшем это значение будет хранится в указателе.

Теперь попробуем понять откуда взялось 0x50, первой мыслью у меня было, что это число было преобразовано после операции первого и второго операнда, оставалось только понять где именно это происходит. Чуть позже я нашел этот код и оказался прав, 0x50 получился в результате инструкции AND EDX, R11D. Где в EDX хранилось число 0x1057, а в R11D - 0x7D0.

Попробуем сгенерировать чистый алгоритм без джанк кода виртуальной машины.

```
mov edx, dwFirstOp
mov r11, dwSecondOp

mov eax, edx ; Резервируем первый операнд
and edx, r11d ; Получаем третье число
mov ecx, edx ; Пихаем третье число в ecx
mov edx, eax ; Восстанавливаем первый операнд

```

Инвертируем операнды

Получаем итоговые константы

```
and edx, r11d
mov r11d, ecx

```

Проворачиваем ту же операцию с этими константами

Получаем наше заветное число после ксора

```
and edx, r11d
mov dwFirstOp, edx

```

Так же можно этот вариант представить и без ассемблерного кода, попытавшись его декомпилировать.  
На выходе мы получаем чистый алгоритм XOR инструкции VMProtect :)

![](https://yougame.biz/proxy.php?image=https%3A%2F%2Fi.ibb.co%2F7yKVptR%2Fcleared-native-xor.png&hash=28b27d91539f6582ffade3e7929ee8fa)

Разницы особой нет в выполнении, результат будет один, а главное - что он является рабочим.

![](https://yougame.biz/proxy.php?image=https%3A%2F%2Fi.ibb.co%2FnL6PsrL%2Fresult-of-cleared-xor.png&hash=1a348637fc6e4b94edac9efa5fe414d4)

**- Инструкции And и Or**

Я объединил эти две инструкции в один подпункт, т.к. их реализация различается в одной инструкции.  
Для начала объясню вкратце реализацию AND:

```
mov eax, dwFirstOp
mov r10, dwSecondOp

```

Инвертируем операнды

Используем на них операцию ИЛИ и снова инвертируем, получая верный результат

Возвращаем полученный результат в первый операнд

Алгоритм ребилда OR у VMProtect почти 1 в 1, единственная разница между ними в том, что вместо операции ИЛИ в OR используется операция И.

```
mov eax, dwFirstOp
mov r10, dwSecondOp

```

Инвертируем операнды

Используем на них операцию ИЛИ и снова инвертируем, получая верный результат

Возвращаем полученный результат в первый операнд

**- Остальные инструкции и обфускация константных значений и вычисления адреса хендлеров**

Нужно отметить еще один интересный вариант трансляции инструкции, допустим у нас есть инструкция: add r9, rax. VMProtect может транслировать её как:

push r9 ; Указатель стека теперь указывает на адрес регистра r9  
add qword ptr ds[rsp], rax ; Добавляем rax к указателю стека  
pop r9 ; После выполнения этой инструкции в r9 будет результат инструкции add r9, rax

Такую работу со стеком VMProtect практикует и на других инструкциях.

Я объединил эти две части воедино, поскольку во время анализа виртуальной машины я заметил, что хендлер, в котором происходит расшифровка переменной затем использует её для вычисления следующего адреса хендлера. Это легко проверить поменяв значение переменной сразу же после её расшифровки.

Допустим переменную типа DWORD64 со значением 0xDEADC0DEDEADBEEF, теперь посмотрим как она расшифруется и в дальнейшем на вычисления адреса.

```
00007FF7F8CB142B  | sub r9,8                                               |
00007FF7F8CB1433  | rcr r10b,5                                             |
00007FF7F8CB1437  | mov r10,qword ptr ds:[r9]                              | Зашифрованная переменная -> 0x4404498584F7A556
00007FF7F8CB143A  | xor r10,r11                                            | Начало расшифровки
00007FF7F8CB143D  | btr esi,eax                                            |
00007FF7F8CB1440  | btc si,FE                                              |
00007FF7F8C195AC  | bswap r10                                              |
00007FF7F8C195AF  | ror r10,1                                              |
00007FF7F8C195B2  | bts rsi,r9                                             |
00007FF7F8C195B6  | neg r10                                                |
00007FF7F8C195B9  | movzx si,r8b                                           |
00007FF7F8C195BE  | movsx si,bpl                                           |
00007FF7F8C195C3  | cmova rsi,r13                                          |
00007FF7F8C195C7  | inc r10                                                |
00007FF7F8C195CA  | bts rsi,rcx                                            |
00007FF7F8C195CE  | ror r10,1                                              | Дешифрованная переменная -> 0xDEADC0DEDEADBEEF

```

Теперь хендлер начинает вычислять адрес следующего хендлера, используя во всю нашу расшифрованную переменную

```
00007FF7F8C195D1  | xor r11,r10                                            | ; Начинает использовать нашу переменную
00007FF7F8C195E3  | mov qword ptr ds:[rdi],r10                             | ; Сохраняет расшифрованную переменную для следующего хендлера
00007FF7F8C195E9  | sub r9,4                                               |
00007FF7F8C195F3  | sbb si,17C8                                            |
00007FF7F8C195F8  | bsf si,bp                                              |
00007FF7F8C195FC  | mov esi,dword ptr ds:[r9]                              |
00007FF7F8C19602  | xor esi,r11d                                           |
00007FF7F8BBDC26  | add esi,60E00069                                       |
00007FF7F8BBDC2C  | neg esi                                                |
00007FF7F8BBDC2E  | add esi,2DAA5513                                       |
00007FF7F8BBDC3B  | xor esi,B596E8B                                        |
00007FF7F8C0A0CE  | inc esi                                                |
00007FF7F8C0A0D0  | rol esi,1                                              |
00007FF7F8C0A0D2  | neg esi                                                |
00007FF7F8C0A0D5  | not esi                                                |
00007FF7F8C0A0D8  | push r11                                               |
00007FF7F8C0A0E8  | xor dword ptr ss:[rsp],esi                             | Значение в rsp ксорим на адрес esi
00007FF7F8C0A0F4  | pop r11                                                | Результат ксора запихиваем в r11
00007FF7F8C0A0F6  | movsxd rsi,esi                                         |
00007FF7F8C0A0FB  | add rbx,rsi                                            | Расшифрованный адрес следующего хендлера
00007FF7F8C13F05  | push rbx                                               |
00007FF7F8C13F06  | ret                                                    |

```

**- Мутация**

Однако, хочу сказать сразу, что нет смысла сравнивать мутацию в VMP 3.x, она попросту не улучшается уже долго время, но мы специально это сделаем для наглядности :)

Мутация в VMProtect 3.5 и 3.6 не представляет особой сложности, оригинальный асм код вполне читабелен, в основном мутация состоит из добавления ненужного мусора, таковой является jmp $+5, инструкция, которая делает джамп на следующую инструкцию.

Сравнение мутации с 3.5 и 3.6, 3.7

Возьмем как пример простой код, который будет сравнивать две переменные, будем смотреть поменялся ли как-то Control Flow.

![](https://yougame.biz/proxy.php?image=https%3A%2F%2Fi.ibb.co%2FdWYFj97%2Fcfg.png&hash=0fd7bcf20d53fd929b6b2485a8f0ff21)

Но несмотря на слабый джанк-код, сама мутация со связкой виртуализации (Ультра) является довольно мощной штукой, хоть виртуализация даже без галочки "Мутации" использует джанк-код в хендлерах.  
P.S. По слухам от одного человека разработчик VMProtect планировал переделать мутацию, но, видимо в 2022 году такого обновления пока не стоит ждать.

**Вторая часть**  
[Анализ от 23.12.2022]

**- Сравнение VMProtect 3.x с VMProtect 2.x**

Для сравнения двух разных версии возьмем VMProtect 3.6 и VMProtect 2.13.8.

Наше сравнение будет состоять из следующих пунктов:  
- Вызов обыкновенного Native Api  
- Вызов Native Api syscall

Для тестирования будет использоваться вот такой семпл:

![](https://yougame.biz/proxy.php?image=https%3A%2F%2Fi.ibb.co%2FXSDqPJG%2FCode.png&hash=45604a06d19cfd2c20bb7d944799de84)

**- Загрузка аргумента для вызова функции в VMProtect 2**

Загрузка аргумента происходит посредством дешифрования через инструкции sub, xor, neg и других нестандартных инструкции, которые берут значение только из определенных регистров x64.

```
00007FF7A1E26D3D    | 8A46 FF                          | mov al,byte ptr ds:[rsi-1]              | al = 0x14
00007FF7A1E26D43    | 28D8                             | sub al,bl                               | al - 0x70 = 0xA4
00007FF7A1E257B2    | 2C 5A                            | sub al,5A                               | al - 0x5A = 0x4A

00007FF7A1E257B6    | F6D8                             | neg al                                  | Инвертация всех битов нашего аргумента, получим из 0xA4 -> B6

```

Дальше над al будет произведена операция XOR со специальным ключем, чтобы получить 0xFF.

```
00007FF7A1E289A6    | 34 49                            | xor al,49                               | al ^ 0x49 = 0xFF

```

Как уже было выше сказано про нестандартные инструкции, не имеющих операнда при вызове, то таковыми являются: **cbw, cwde, cdqe**.  
Все эти инструкции для своего операнда используют только регистр RAX и никак не влияют на RFlags.

Каждая из этих инструкции работает только с определенным размером регистра, если вы используете Cbw, то соответственно Cbw будет брать только al значение, а Cwde - eax значение.

*   Cbw - конвертирует byte в word, т.е. 0xFF будет преобразован в 0xFFFF.
*   Cwde - конвертирует word в dword, т.е. 0xFFFF будет преобразован в 0xFFFFFFFF.
*   Cdqe - как вы могли уже догадаться конвертирует dword в qword, т.е. 0xFFFFFFFF будет преобразован в 0xFFFFFFFFFFFFFFFF.

Далее, преобразованный аргумент виртуальная машина сохранит в виртуальный стек, перед этим сместив стек на 0x8.

```
00007FF7A1E29E3E    | 48:83ED 08                       | sub rbp,8                               | Смещаем виртуальный стек машины на 0x8, чтобы пушнуть туда qword число
00007FF7A1E288C8    | 48:8945 00                       | mov qword ptr ss:[rbp],rax              | Делаем своего рода push 8-байтного числа, в нашем случае это -1

```

В последующем виртуальная машина будет запрашивать этот аргумент из виртуального стека для своих манипуляции.

```
00007FF7A1E27029    | 48:8B55 00                       | mov rdx,qword ptr ss:[rbp]              | Выгружаем обратно для наших хитрых махинации

```

**- Загрузка аргумента для вызова функции в VMProtect 3**

Декрипт -1 выглядит следующим способом:

```
00007FF68A68E7D5    | 48:FFCE                          | dec rsi                                 | rsi = 0
00007FF68A68E7D5    | 48:FFCE                          | dec rsi                                 | rsi = -1
00007FF68A68E81F    | 49:8931                          | mov qword ptr ds:[r9],rsi               | Загружаем в виртуальный стек

```

Да, преобразование действительно выглядит так, и для достоверности я решил поменять значение rsi до его загрузки в вирт. стек, после F9 в RAX появился статус ошибки **STATUS_INVALID_HANDLE.**  
Конечно мы рассматриваем случай, связанный только с -1, где есть множество способов преобразовать его. Для каких-нибудь других больших чисел используется совсем другой метод дешифрования.

**- Вызов функции в VMProtect 2 и 3**

Разницы между ними особой и нет, в любом случае используется RET для передачи управления адресу, на который указывает стек.  
По соглашениям вызовов первый аргумент всегда будет лежать в регистре RCX, что и делает VMProtect.

```
00007FF7A1E258B5    | 6641:87C8                        | xchg r8w,cx                             |
00007FF7A1E258B9    | 59                               | pop rcx                                 |
00007FF7A1E258BA    | 6641:89C8                        | mov r8w,cx                              |
00007FF7A1E258BE    | 6644:0FBEC2                      | movsx r8w,dl                            |
00007FF7A1E258C3    | 4C:8D80 9902A751                 | lea r8,qword ptr ds:[rax+51A70299]      |
00007FF7A1E258CA    | 41:58                            | pop r8                                  |
00007FF7A1E258CC    | C3                               | ret                                     |

```

4 инструкции тут являются обыкновенным джанком, поэтому их можно вырезать, и получим такой вызов:

```
00007FF7A1E258B9    | 59                               | pop rcx                                 | ; 0xFFFFFFFFFFFFFFFF
00007FF7A1E258CA    | 41:58                            | pop r8                                  | ; NtSuspendProcess
00007FF7A1E258CC    | C3                               | ret                                     |

```

**- Вызов syscall в VMProtect 2 и 3**

Разобрались с обычной функцией, а что на счёт системного вызова? Дадим протектору новый семпл:

![](https://yougame.biz/proxy.php?image=https%3A%2F%2Fi.ibb.co%2F3zJXVG6%2FSyscall.png&hash=ef64136fc0f5835a2fa4e2138edbd08a)

В VMProtect 2 инициализация системного вызова выглядит следующим образом:

```
00007FF7E3FE9B3F    | 66:0FB746 FE                     | movzx ax,word ptr ds:[rsi-2]            |
00007FF7E3FE9B49    | 86E0                             | xchg al,ah                              |
00007FF7E3FE9B50    | 66:31D8                          | xor ax,bx                               |
00007FF7E3FE9B54    | 66:F7D8                          | neg ax                                  |
00007FF7E3FE9B58    | 66:F7D0                          | not ax                                  |
00007FF7E3FE9B61    | 66:05 3F1E                       | add ax,1E3F                             | ax = 0x1BB - системный номер NtSuspendProcess

00007FF7E3FE9F3D    | 41:5A                            | pop r10                                 | r10 = 0x1BB
00007FF7E3FE9F41    | 5A                               | pop rdx                                 | Адрес с сисколлом
00007FF7E3FE9F42    | C3                               | ret                                     |

00007FF7E3FEAD1A    | 0F05                             | syscall                                 |

```

VMProtect 3.x:

```
00007FF7093BE5DC    | 8B2F                             | mov ebp,dword ptr ds:[rdi]              | ebp = 6B1ED59D
00007FF7093BE5EC    | 41:33EB                          | xor ebp,r11d                            | ebp ^ 4016D008 = 2B080595
00007FF7093BE5F4    | F7D5                             | not ebp                                 | ebp = D4F7FA6A
00007FF7093BE5F6    | F7DD                             | neg ebp                                 | ebp = 2B080596
00007FF7093BE600    | 81ED 5207082B                    | sub ebp,2B080752                        | ebp = FFFFFE44
00007FF7093BE60D    | F7D5                             | not ebp                                 | ebp = 1BB

00007FF70934233C    | 41:5A                            | pop rax                                 | rax = 0x1BB
00007FF70934233D    | 41:5A                            | pop rcx                                 | r10 = -1
00007FF709397B77    | 41:5A                            | pop r10                                 |
00007FF709397B80    | 5D                               | pop rbp                                 |
00007FF709397B81    | 41:5B                            | pop r11                                 |
00007FF709397B88    | 41:59                            | pop r9                                  |
00007FF709397B8A    | 41:5E                            | pop r14                                 |
00007FF709397B8C    | 5E                               | pop rsi                                 |
00007FF709397B8D    | E9 31CCF7FF                      | jmp easton_gay.vmp36.7FF7093147C3       |
00007FF7093147C3    | C3                               | ret                                     |

00007FF7E3FEAD1A    | 0F05                             | syscall                                 |

```

**- Разница между вызовами для защиты памяти в VMProtect 2 и VMProtect 3**

Анти-отладку обозревать, думаю, смысла нет никакого, но для тех, кто не видел - [Ahora57](https://yougame.biz/members/381942/) в своём треде расписал работу анти-дебага и темиды и вмпрота. [https://yougame.biz/threads/264690/](https://yougame.biz/threads/264690/)

Перед считыванием CRC32 VMProtect 2 вызывает **GetModuleFileNameW**, чтобы получить полный путь к таргету. Не тяжело догадаться, что после этого будет вызван CreateFileW с аргументом, где лежит наш путь к файлу.

Последовательность вызовов выглядит таким образом: **GetModuleFileNameW -> CreateFileW -> GetFileSize -> CreateFileMappingW -> MapViewOfFile -> UnmapViewOfFile -> CloseHandle**, и всё это вызывается в одном месте, просто call rax.  
Только после этой чреды вызовов протектор начнёт сканировать байты абсолютно всех секции, чем-то напоминает метод SecuROM :)

Обойти подобный чек возможно подменив путь к оригинальному файлу в **CreateFileW**, однако, тот же SecuROM после вызова **CloseHandle** специально себя патчил, и не стоит забывать о том, что он сканирует самого себя.

В защите памяти для VMProtect 3 из WinAPI используется только **GetModuleFileNameW**, всё остальное это NtApi.  
Список NtApi функции, которые вызываются для чтения файла и дальнейшего взаимодействия с ним:

```
NtOpenFile
NtCreateSection
NtMapViewOfSection
NtQueryVirtualMemory
NtUnmapViewOfSection

```

Так же вместо **CloseHandle** теперь используется **NtClose**.

Что ещё примечательно, так это то, что считывание байтов секции у VMProtect 2 происходит только в одном месте:

```
xor al, byte ptr ds:[rdx]

```

Когда у VMProtect 3 вычисления разбросаны повсюду.

**Заключение**  
Вот и подошла к концу моя короткая статья про VMProtect. Добавить мне особого нечего к заключению, как я и говорил выше в предисловии - у меня больше не хватает времени пилить контент для этого форума. Не буду заглядывать в будущее, всякое может приключиться, ведь саму статью я вообще планировал выпускать в сентябре.  
Не будем о плохом, ведь скоро новый год, поэтому желаю вам всех благ!

Мой бложик: