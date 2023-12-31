# Инструменты разработки

## Этапы компиляции программ на Си

У большинства современных языков есть свои системы сборки, которые занимаются
импортом библиотек и компиляцией проекта из исходников. Проекты на го собираются
просто командой `go build`, на расте &mdash; `cargo build`. Си отличается тем,
что его процесс сборки &mdash; кишками наружу, за счет чего можно смотреть на
артефакты с отдельных этапов компиляции.

Мы рассмотрим этапы сборки на примере программы `Hello, World!`. Компиляция
библиотек работает аналогично.
```c
// main.c

#include <stdio.h>

int main() {
  printf("Hello, World!\n");
  return 0;
}
```

Я буду использовать компилятор [`clang`](https://clang.llvm.org/). Он может
примерно то же самое, что `gcc`, и имеет практически такие же флаги командной
строки, но выдает более понятные варниги/ошибки. Плюс вокруг кланга написано
много хорошего тулинга (например,
[clang-format](https://clang.llvm.org/docs/ClangFormat.html) и
[clang-tidy](https://clang.llvm.org/extra/clang-tidy/)), поэтому я team clang.
Чтобы получить exe-шник &mdash; исполняемый файл, &mdash; нужно выполнить
команду `clang main.c -o main`.

Компиляция делится на несколько этапов, и ее можно "прервать" посередине с
помощью специальных флагов компилятора.
1. Разворачиваются директивы препроцессора (`#include`, `#define`, `#ifdef` и
   другие). В нашем случае в текст программы будет целиком скопирован хедер
   `stdio.h`, и получится огромная паста. Можно на нее посмотреть с помощью
   команды `clang -E main.c -o main_pp.c`. Флаг `-E` говорит компилятору,
   что нужно сделать только препроцессинг и на этом остановиться. В результате у
   меня получилось [вот что](main_pp.c).
2. После происходит преобразование Си в инструкции ассемблера целевой платформы.
   Пока что достаточно знать, что ассемблер &mdash; просто более низкоуровневое
   описание инструкций, которые должен исполнить процессор. Посмотреть на них
   можно c помощью флага `-S`, команда `clang -S main.c -o main.S`. В этот раз
   получается файл поменьше, т.к. все неиспользуемые определения были вырезаны.
   ```x86asm
   // main.S

       .text
       .file	"main.c"
       .globl	main                            # -- Begin function main
       .p2align	4, 0x90
       .type	main,@function
   main:                                   # @main
       .cfi_startproc
   # %bb.0:
       pushq	%rbp
       .cfi_def_cfa_offset 16
       .cfi_offset %rbp, -16
       movq	%rsp, %rbp
       .cfi_def_cfa_register %rbp
       subq	$16, %rsp
       movl	$0, -4(%rbp)
       leaq	.L.str(%rip), %rdi
       movb	$0, %al
       callq	printf@PLT
       xorl	%eax, %eax
       addq	$16, %rsp
       popq	%rbp
       .cfi_def_cfa %rsp, 8
       retq
   .Lfunc_end0:
       .size	main, .Lfunc_end0-main
       .cfi_endproc
                                           # -- End function
       .type	.L.str,@object                  # @.str
       .section	.rodata.str1.1,"aMS",@progbits,1
   .L.str:
       .asciz	"Hello, World!"
       .size	.L.str, 14

       .ident	"clang version 15.0.7"
       .section	".note.GNU-stack","",@progbits
       .addrsig
       .addrsig_sym printf
   ```
3. Следующий этап &mdash; компиляция ассемблера в так называемый объектный файл,
   для этого есть флаг `-c`. После выполнения команды `clang main.c -c main.o`
   появится объектник `main.o`, он содержит настоящий машинный код. Файл
   бинарный, поэтому просто прочитать его не получится. Преобразование
   ассемблера в объектник в целом обратимо. Можно дизассемблировать `main.o` с
   помощью утилиты
   [`objdump`](https://man7.org/linux/man-pages/man1/objdump.1.html). Команда
   `objdump -d` выведет что-то такое
   ```x86asm
   main.o:     file format elf64-x86-64


   Disassembly of section .text:

   0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	c7 45 fc 00 00 00 00 	movl   $0x0,-0x4(%rbp)
   f:	48 8d 3d 00 00 00 00 	lea    0x0(%rip),%rdi        # 16 <main+0x16>
   16:	b0 00                	mov    $0x0,%al
   18:	e8 00 00 00 00       	call   1d <main+0x1d>
   1d:	31 c0                	xor    %eax,%eax
   1f:	48 83 c4 10          	add    $0x10,%rsp
   23:	5d                   	pop    %rbp
   24:	c3                   	ret
   ```
   Слева мы видим машинные коды, а справа &mdash; их мнемоники, т.е.
   ассемблерные инструкции с предыдущего шага. Инструкций стало меньше, потому
   что мы дизассемблировали только секцию `.text` &mdash; сам исполняемый код. В
   объектнике еще есть другие секции, например, секция данных, в которой
   хранится строка `"Hello, World!"`. О них мы подробнее поговорим позже, когда
   будем изучать формат исполняемых бинарников
   [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format).
4. Последний этап &mdash; линковка. Команда `clang main.o -o main` превращает
   объектный файл в итоговый exe-шник. Линковка нужна по двум причинам.
   Во-первых мог быть не один файл `main.c`, а несколько единиц трансляции, т.е.
   исходников, которые преобразуются в объектные файлы. Их все нужно совместить
   в один exe-шник, чем и занимается линкер. Во-вторых программа могла
   использовать внешние библиотеки, как минимум, стандартную библиотеку Си. Их
   тоже нужно включить в итоговый бинарник. Линковкой обычно занимается не сам
   компилятор, а отдельная программа-линковщик. В нашем случае под капотом кланг
   запустит [`ld`](https://man7.org/linux/man-pages/man1/ld.1.html), чтобы
   получить исполняемый файл.

Код становится платформо-зависимым уже на втором шаге, когда получили ассемблер.
Ассемблерные инструкции будут разными в зависимости от архитектуры процессора и
операционной системы, для которых собираем exe-шник. Например, если
компилировать под процессоры ARM (конкретно
[AArch64](https://en.wikipedia.org/wiki/AArch64)), получится вот такой
ассемблер.
```armasm
        .arch armv8-a
        .file   "main.c"
        .text
        .section        .rodata
        .align  3
.LC0:
        .string "Hello, World!"
        .text
        .align  2
        .global main
        .type   main, %function
main:
        stp     x29, x30, [sp, -16]!
        add     x29, sp, 0
        adrp    x0, .LC0
        add     x0, x0, :lo12:.LC0
        bl      printf
        mov     w0, 0
        ldp     x29, x30, [sp], 16
        ret
        .size   main, .-main
        .ident  "GCC: (Linaro GCC 7.5-2019.12) 7.5.0"
```

Хорошая новость в том, что машинные коды практически один в один соответствуют
инструкциям ассемблера, поэтому можно считать ассемблер наиболее низкоуровневым
описанием программы. Если уметь его читать, можно рассуждать об исполнении кода
на уровне инструкций процессора.

> Такой процесс компиляции &mdash; чудовищное легаси, которое нам досталось от
> Си и перешло в C++. У современных языков этапы компиляции другие, потому что
> нет разделения кода на хедеры и исходники, а есть просто модули. Даже если
> компилировать C++20 с
> [модулями](https://en.cppreference.com/w/cpp/language/modules), уже будет
> чуть-чуть иначе. Но несмотря на это формат итогового бинарника и представление
> в ассемблере всегда одни и те же.

> Еще могут быть другие этапы сборки, если включить [link-time
> оптимизации](https://llvm.org/docs/LinkTimeOptimization.html). Вместе с LTO,
> оптимайзер одновременно видит несколько единиц трансляции и, например, может
> заинлайнить функцию из одного объектника в другой &mdash; без LTO так не
> получится. В таком случае формат объектников отличается: в них будет
> содержаться не машинный код, а специальное промежуточное представление, с
> которым работает оптимайзер, &mdash; например, [биткод
> LLVM](https://llvm.org/docs/BitCodeFormat.html).

## Статические и динамические библиотеки

Разберемся, как линкуются библиотеки. Концептуально все библиотеки делятся на
статические и динамические. Статические во время линковки просто копируются в
итоговый бинарник, а в случае с динамическими в бинарник вставляется только путь
до файла библиотеки. Перед запуском программы операционная система ищет
динамические библиотеки по этим путям и подгружает их в память. Плюс
динамических либ в том, что несколько разных программ могут использовать одну и ту
же библиотеку, и ее не нужно каждый раз копировать. Например, libc используют
почти все программы, и было бы расточительно иметь копию внутри каждого
исполняемого файла. Поэтому динамические библиотеки еще называются разделяемыми.

Попробуем собрать оба вида библиотек и заиспользовать их.

```c
// aplusb.h

int sum(int x, int y);
```

```c
// aplusb.c

int sum(int x, int y) {
  return x + y;
}
```

```c
// main.c

#include <stdio.h>

#include "aplusb.h"

int main() {
  printf("%d\n", sum(1, 2));
  return 0;
}
```

1. Статические библиотеки &mdash; просто архивы из объектных файлов. Создать
   архив можно с помощью тулов
   [`ar`](https://man7.org/linux/man-pages/man1/ar.1.html) либо
   [`llvm-ar`](https://llvm.org/docs/CommandGuide/llvm-ar.html). Стандартное
   расширение для статических библиотек &mdash; `.a` (означает "archive").
   ```bash
   clang aplusb.c -c -o aplusb.o
   llvm-ar -qc libaplusb.a aplusb.o
   ```
   После этого использовать библиотеку можно так.
   ```bash
   clang main.c -c -o main.o
   clang libaplusb.a main.o -o main
   ```
   Процесс компиляции будет такой же, как если бы мы просто передали клангу
   объектники `main.o` и `aplusb.o`.

   Разархивировать статическую библиотеку тоже можно. Команда `llvm-ar -x
   libaplusb.a` извлекает обратно `aplusb.o`.
2. Динамические библиотеки больше похожи на исполняемые файлы, потому что они
   точно так же компилируются и линкуются. Из них уже не получится извлечь
   отдельные объектники. Стандартное расширение для динамических библиотек
   &mdash; `.so` (от названия "shared object"). Для сборки надо передать
   компилятору флаг `-shared`.
   ```bash
   clang aplusb.c -shared -o aplusb.so
   ```
   Прилинковать библиотеку можно так.
   ```bash
   clang main.c -c -o main.o
   clang main.o aplusb.so -rpath . -o main
   ```
   Флаг `-rpath` указывает директории, в которых нужно искать динамические
   библиотеки при старте программы. В `rpath` по умолчанию есть дефолтные пути
   типа `/usr/lib`, у нас будет еще один кастомный путь &mdash; текущая
   директория. `rpath` нужен, потому что при линковке динамическая библиотека
   может лежать в одном месте, а при запуске программы &mdash; в другом
   (например, вы собираете `abplusb.so` в папке проекта, а при установке
   копируете в `/usr/lib`).


Есть утилита [`ldd`](https://man7.org/linux/man-pages/man1/ldd.1.html), которая
позволяет смотреть, от каких динамических библиотек зависит файл (название
&mdash; сокращение от "ld dump"). Если прилинковать `aplusb.so` и выполнить `ldd
main`, увидим такой вывод.
```
linux-vdso.so.1 (0x00007ffe0574d000)
libaplusb.so => ./libaplusb.so (0x00007f17b0f31000)
libc.so.6 => /usr/lib/libc.so.6 (0x00007f17b0c00000)
/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f17b0f3d000)
```
Здесь есть две особенные зависимости.
1. [`linux-vdso.so.1`](https://man7.org/linux/man-pages/man7/vdso.7.html)
   &mdash; это такая штука которая ускоряет сисколлы &mdash; обращения к API
   линукса. Про нее мы поговорим позже.
2. [`/lib64/ld-linux-x86-64.so.2`](https://man7.org/linux/man-pages/man8/ld.so.8.html)
   &mdash; рантаймовый линковщик, который перед стартом программы подгружает все
   остальные динамические библиотеки (это не то же самое, что линковщик `ld`,
   который вызывается во время сборки). Сам `ld-linux.so` shared object-ом не
   является. `ldd`  его показывает, потому что без рантаймового линковщика
   невозможен запуск exe-шников, зависящих от динамических библиотек, а суффикс
   `.so` нужен, чтобы отличать его от обычного compile-time `ld`.

## Статическая и динамическая линковка

Если исполняемый файл линкуется к динамическим библиотекам, это не всегда
удобно, т.к. библиотеки надо везде носить с собой и подкладывать в папки,
которые прописаны в `rpath`. Даже libc на разных системах может лежать в разных
директориях и иметь несовместимые версии. Поэтому если вы скомпилируете
программу у себя на компьютере а потом перекопируете на другую машину, не факт,
что она запустится, т.к. могут не найтись нужные `.so`.

Плюс некоторые программы, например линковщик `linux-ld.so` не могут зависеть от
динамических библиотек, т.к. сами их реализуют. Поэтому бывают статически
слинкованные бинарники, т.е. не зависящие ни от каких динамических либ.

Попробуем избавиться от рантаймовых зависимостей в нашем `Hello, World!`. Для
статической линковки надо передать клангу флаг `-static`.
```bash
clang main.c -static -o main
```
При сборке кланг использовал статические варианты библиотек, от которых зависел
`Hello, World!`, например `libc.a` (библиотеки могут быть установлены сразу в
двух вариантах &mdash; и `.so`, и `.a`). Теперь если попробовать `ldd main`,
получим сообщение `not a dynamic executable`. Статически слинкованная программа
не использует `ld-linux.so`. Она будет работать на любой машине с совместимой
архитектурой процессора и операционной системой.

## Санитайзеры

Программировать на Cи и C++ весело, потому что можно допускать утечки,
корраптить память, переполнять инты и еще много всего. Чтобы проверять
корректность программ, люди придумали санитайзеры &mdash; специальные
инструменты, которые в рантайме следят, что делает приложение, и сообщают, когда
произошло что-то плохое.

Санитайзеры работают за счет
1. Compile-time преобразований кода программы.
2. Рантаймовых библиотек, в которых прописана логика проверок на корректность.

Про алгоритмы работы санитайзеров можно прочитать
[здесь](https://github.com/google/sanitizers/wiki/).

Есть более-менее стандартный набор санитайзеров, которые поддерживаются в gcc и
clang, я оставлю ссылки на clang. Наиболее полезные &mdash;
[AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) (ASan),
[UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
(UBSan) и [ThreadSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)
(TSan). ASan проверяет, что нет утечек и обращений к освобожденной памяти.
UBSan, &mdash; что нет UB (например, переполнения знаковых интов, невыровненных
обращений к памяти и т.д.). TSan проверяет корректность многопоточных программ,
он нам пока что не пригодится.

Посмотрим, как работает ASan. Сначала напишем простую утечку.
```c
#include <stdlib.h>

int main() {
  malloc(1024);
  return 0;
}
```

Чтобы подключить ASan, нужно передать компилятору флаг `-fsanitize=address`.
```
clang main.c -fsanitize=address -o main
```
При компиляции кланг заменит стандартные `malloc()` и `free()` на специальные
реализации из библиотеки ASan, которые следят, чтобы вся аллоцированная память
была освобождена.

Теперь если запустить `main`, получим сообщение от LeakSanitizer-а, который
входит в комплект AddressSanitizer-а.
```
=================================================================
==46288==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 1024 byte(s) in 1 object(s) allocated from:
    #0 0x55fb8ec43c59 in __interceptor_malloc (/home/dgdh/main+0xd6c59) (BuildId: f88b8c36603f7685cc8f7ad771b219c3dd50e4d0)
    #1 0x55fb8ec8a038 in main (/home/dgdh/main+0x11d038) (BuildId: f88b8c36603f7685cc8f7ad771b219c3dd50e4d0)
    #2 0x7fcf3ce27ccf  (/usr/lib/libc.so.6+0x27ccf) (BuildId: 316d0d3666387f0e8fb98773f51aa1801027c5ab)

SUMMARY: AddressSanitizer: 1024 byte(s) leaked in 1 allocation(s).
```

Некоторые санитайзеры можно комбинировать. Чтобы включить одновременно ASan и
UBSan, нужно передать клангу флаг `-fsanitize=address,undefined`.

Код с санитайзерами работает медленее. Например, ASan своими проверками
замедляет программу примерно в 2 раза, а TSan &mdash; в 5-10 раз. Тем не менее,
это очень полезные инструменты, без которых тяжело жить.

Еще есть [MemorySanitizer](https://clang.llvm.org/docs/MemorySanitizer.html). Он
проверяет, что программа читает только инициализированную память. ASan тоже
проверяет обращения к памяти, но не все, MSan в этом плане сильнее.