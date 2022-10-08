# Ассемблер (Скаков)

## Про курс

Все мы знаем, что ассемблер - просто мнемоническая запись какой-то isa (их много). Наш курс - asm x86. 

## Где используются знания?

1) Антивирусы и вирусы. 
2) Небольшие кусочки операционных систем для которых не существует функций вызова высокого порядка. 
3) Оптимизация C/C++. Ну и отладка...
4) Проще править бинарник, чем перекомпилировать.

## Чем асм лучше бинарника?
В двоичном коде используются ссылки на сколько-то байтиков -> при вставлении кода в середину программы будет коллапс.

## Команды

Несколько синтаксисов: "Нормальный" - Intel и "Гнусный". 

**nasm/yasm:** - с нормальным синтаксисом :)

То что выдаст компилятор - объектный файл. Это значит, что часть можно написать на C, а часть на asm, а потом линк.

## Регистры

Традиционные регистры **i8086** (название микросхем от интел) общего назначения (те, где можно что-то посчитать):
ax - аккумулятор
bx - базовый (учатвовал в адреации)
cx - counter (посчитать число итераций чего-то) 
dx - расширение аккумулятора
sp
bp
si
di

Первые 4 регистра делятся на 8битные: ax = ah + al.

## хронология intel
i8086 (i8088) : i8088 просто чуть слабее железо, никакого исторического вклада.
i80186
i80286
i80386
i80486
i80586

Гарантируется полная совместимость "баг в баг": если функция как-то странно работала, то можно не волноваться - так странно и работает :) . Такая 
совместимость выгодна и пользователю (не нужно переписывать программы) и производителю (можно убедить купить новые процессоры). Костыли, которые 
приходится громаздить замедляют компьютер, но не сильно.

### i80386
Появились 32ух битные процессоры.

eax 
ebx 
ecx  
edx 
esp
ebp
esi
edi

Пропала связь между регистрами и их назначением.

### eflags

zf - регитр нулевой
sf - флаг знака и фиктивно равен старшему разряду
cf - перенос из старшего разряда или заем из старшего заряда

## MOVE

```mov куда, что```
Пример: 
```
mov eax, ebx
mov cl, ch
mov ax, 0
```
Важно: размерности регистров совпадают лево-право

```
mov al, [eax] // записываем значение, находящееся по адресу eax.
```

Может быть не больше одних скобочек на команду.

### Адресация

16 бит: **[bx/bp + si/di + offset 16]**
32 бита: **[eax/ebx/ecx/edx/ebp/esp/esi/edi + любой кроме esp * 1/2/4/8 + offset 32]**

Размеры: byte, word, dword, qword [только в 64 битном мире], dqword = oword, tbyte 

``` move word [eax], 3 ```

## Команды

movezx/movesx - загрузить маленькое в большое

bswap - реверсит биты (типа конверсия между литл эндиан и биг эндиан)

xchg eax, ebx - свапает значения и гарантирует атамарность переменной, но свап по памяти - зло

cmov (cmovz/cmovnc...): выполняет копирование если флаг... Пример cmovz - копирует, если установлен флаг нуля. Работает средне быстро

lea reg, [] - для быстрой арифметики

push/pop - рабта со стеком. Стек растет вниз. Соответственно push уменьшает esp на 4 (в 32 битной)
```push eax```

pusha - загрузить все 8 регистров, popa - читает 7 (все кроме esp)

cwd/cdq: (ax/eax) -> dx:ax/edx:eax

cbw/cwde: (al->ax)/(ax->eax)

in/out - для драйверов (секретно)

Читать про команды множно в **instruction manual**

## Арифметика

```
ADD eax, ebx // +=
SUB eax, ebx // -=
```
ADC и SBB тоже самое, но еще добавляет/вычитает знак переноса

Сложение двух 64битных чисел:
```
edx:eax
ebx:ecx
ADD eax, ecx
ADC edx, ebx
```

MUL ecx - берется eax * op32 -> edx:eax

DIV ecx - edx:eax / op32 -> частное в eax, остаток в edx. Если частное не получается впихать, то падаем с той же ошибкой, что и при деленении на 0.

IMUL/IDIV:
```
IMUL op, op2 // *=
IMUL op, op2, const // op = op2 * const
```

INC/DEC - уменьшают/увеличивают на 1 (сохраняют флаг cf)

NEG - меняет знак

NOT - меняет битики

AND, XOR, OR - логические побитывые операции

CMP/TEST - SUB/AND юез порчи регистра

SHR/SHL - 
SAR/SAL - 

SHRD/SHLD - сдвиги двойной точности

SHLD eax, edx, 4 // двигется первый операнд, на пустые позиции вдвигается старший разряд

ROL/ROR и RCL/RCR - делает перенос в cf 

XOR eax, eax - патрн обнуления регистра (однако портит флаги)

## Команды перехода

```
JMP метка
...
метка:
```

Jxx: 
Пример, JNZ метка - если не ноль, прыгаем в метку.

CALL/RET/RET const - прыгнуть на метку, сохранив EIP на стеке/снять eip со стека и прыгнуть обратно/прибавить к снятому eip константу и потом уже прыгнуть.

INT3 - жесткий брейкоинт для отладчика, но если запститься без отладчика, то упадет на этой строчке.

NOP (xchg eax, eax)

UD2 - гарантированно несуществующая команда

## Написание программы

### Секции

код - можно исполнять
дата - можно только читать

### стартовый код

```
  section .text
  global main; делаем main видимым оттовсюду

main:
  ret
```

Если точка входа - не main, то ее стоит передать линкеру

### Hello, world :)

```
  section .text
  global main ; делаем main видимым оттовсюду
  extern printf

main:
    push hello
    call printf
    add esp, 4
  
    ret

  section .rdata
hello: db "Hello", 0
```

db - дальше пойдут байтики, но без 0 - терменированной строчка не будет :(

### Конвенции

*Хотим f(a, b, c):*

**cdecl:**
```
push c
push b
push a
call f
add esp, 12

f:
  mov eax, [esp + 4] ; по esp лежит код возврата
  add eax, [esp + 8]
  ret
```

для возвращения используют eax и edx (как продолжение eax)

**stdcall:** 
```
push c
push b
push a
call f

f:
  mov eax, [esp + 4] ; по esp лежит код возврата
  add eax, [esp + 8]
  ret 12
```
Минус - не умеет работать с vararg :(
Плюс - меньше кода

**pascal:**

```
push a
push b
push c
call f

f:
  mov eax, [esp + 4] ; по esp лежит код возврата
  add eax, [esp + 8]
  ret 12
```
В случае vararg - не можем найти начало. Одни минусы. Конвенция мертва.

**fastcall**
передавать аргументы через регистры

Сохраняемые регистры: ebx, ebp, esi, edi
Несохраняемые регистры: eax, ecx, edx. 

## Условия

```
cmp eax, ebx
J...  L1 ; хотим сравнить знаково/беззнаково
X
L1:
...
```

### Беззнаковые
A, B, AE, BE; больше, меньше, больше равно, меньше равно
NA, NBE, ... ; их отрицания

if(eax < ebx && ecx != 0) :
```
cmp eax, ebx
jae L1
test ecx, ecx
jz L1
X
L1:
...
```
### Знаковые
G,L,GE,LE;
Также есть отрицания

## Циклы

### do while
Хотим:
```
do
  x;
while(eax < ebx)
```
Пишем:
```
L1:
  X
  cmp eax, ebx
  jb L1
```
### while
Хотим:
```
while(eax < ebx)
  X;
```
Пишем:
```
L1:
  cmp eax, ebx
  jae L2
  X
  jmp L1
L2:
```

**ОДНАКО** лучше хотеть 
```
if
do
while;
```
Поэтому
```
  cmp eax, ebx
  jae L2
L1:
  X
  cmp eax, ebx
  jb L1
L2:
```

### for
```
for (uint eax = 0; eax < ebx; eax++)
  X;
```
Форматируем в do while:
```
  xor eax, eax
  cmp eax, ebx
  jnb L2
L1:
  X
  inc eax
  cmp eax, ebx
  jb L1
L2:
```