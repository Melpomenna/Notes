#Windows
[[Полезные статьи и ссылки, Windows|Полезные статьи и ссылки, Windows]]
[[Книги]]

______
#Процессоры #Asm #x86-64 #Cpp #Компиляторы 
[Документация](https://learn.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-prefetchcacheline)
Макрос:  ***PreFetchCacheLine***

Указывает процессору, что строка кеша понадобится в ближайшем будующем, схожа с \__builtin_prefetch, который выстраивается в prefetch0-prefetch{n} инструкцию под x86-64

Параметры:
1) Указывает как часто будет использоваться строка кеша:
	1) **PF_TEMPORAL_LEVEL_1** - строка кэша должна быть загружена во все кэши и, скорее всего, будет использоваться несколько раз.
	2) **PF_NON_TEMPORAL_LEVEL_ALL** - строка кеша врядли понадобиться снова, после 1 обращения к ней
2) Указатель на адрес памяти, не обязательно должен находиться в кеше
   
Если 1 параметр имеет значение PF_NON_TEMPORAL_LEVEL_ALL, то используется инструкция:
***prefetchnta***  - эта инструкция предназначена для предзагрузки данных в кэш процессора с минимальным эффектом на кэш

[пример на godbolt](https://godbolt.org/z/1qYKM1T5s)

x64 msvc 19 latest

```cpp
#define WIN32_LEAN_AND_MEAN

#include <Windows.h>

#include <winnt.h>

  

void test(void* ptr)

{

    PreFetchCacheLine(PF_TEMPORAL_LEVEL_1, ptr);

  

    PreFetchCacheLine(PF_NON_TEMPORAL_LEVEL_ALL, ptr);

}
```

```asm
ptr$ = 8

void test(void *) PROC                             ; test

        mov     QWORD PTR [rsp+8], rcx

        mov     rax, QWORD PTR ptr$[rsp]

        prefetcht0 BYTE PTR [rax]

        mov     rax, QWORD PTR ptr$[rsp]

        prefetchnta BYTE PTR [rax]

        ret     0

void test(void *) ENDP                             ; test

```

```preprocessor

void test(void* ptr)

{

    _mm_prefetch((CHAR const *) ptr, 1);

  

    _mm_prefetch((CHAR const *) ptr, 0);

}

```

Макрос можно расширяться не на всех процессорах, см. документацию

Подробнее про загрузку в кеши можно прочитать тут:
https://stackoverflow.com/questions/46521694/what-are-mm-prefetch-locality-hints

______

#Процессоры #Asm #MultiThreading #x86-64 #Компиляторы #Cpp 

[Документация](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setthreadaffinitymask)

***SetThreadAffinityMask***
Указывает маску привязки процессора для потока, указывает, на каком ядре/ядрах должен выполняться поток

Параметры:
1) Дескриптор потока, поток должен обладать следующие права доступа:
   **THREAD_SET_INFORMATION** или **THREAD_SET_LIMITED_INFORMATION**, а так же право доступа **THREAD_QUERY_INFORMATION**, **THREAD_QUERY_LIMITED_INFORMATION**
   Доп. информацию можно посмотреть здесь: [Безопасность потоков и права](https://learn.microsoft.com/en-us/windows/desktop/ProcThread/thread-security-and-access-rights)
2) Маска привязки, подробнее можно прочитать тут: https://learn.microsoft.com/en-us/windows/win32/procthread/processor-groups (относиться к  системе с более чем 64 процессорами)
Возвращает:
Если успешно получилось установить маску, то возвращает предыдущую маску, если не работает, то 0
   
   
[godbolt](https://godbolt.org/z/ar1WvTxMd)

```cpp
#define WIN32_LEAN_AND_MEAN

#include <Windows.h>

void SetFirstCoreThread(HANDLE hThread)

{

    DWORD_PTR mask = 0x00000001;

    SetThreadAffinityMask(hThread, mask);

}
```

```asm
mask$ = 32

hThread$ = 64

void SetFirstCoreThread(void *) PROC           ; SetFirstCoreThread

$LN3:

        mov     QWORD PTR [rsp+8], rcx

        sub     rsp, 56                             ; 00000038H

        mov     QWORD PTR mask$[rsp], 1

        mov     rdx, QWORD PTR mask$[rsp]

        mov     rcx, QWORD PTR hThread$[rsp]

        call    QWORD PTR __imp_SetThreadAffinityMask

        npad    1

        add     rsp, 56                             ; 00000038H

        ret     0

void SetFirstCoreThread(void *) ENDP           ; SetFirstCoreThread
```

______

[Безопасность потоков и права доступа](https://learn.microsoft.com/en-us/windows/win32/procthread/thread-security-and-access-rights)

## Безопасность потоков и права доступа: 
***Безопасность потока*** — это права, которые можно установить для потока, чтобы указать, какие операции можно выполнять над потоком, здесь есть набор понятий, например SACL и DACL, которые нужно изучить ***Права доступа*** — набор определенных состояний, которые можно установить для потока, дабы работать над объектом потока, например чтение определенной информации, установка контекста потока

#TODO SACL, DACL

______

[Группы процессоров](https://learn.microsoft.com/en-us/windows/win32/procthread/processor-groups)

## Процессорные группы 
Начнем с понятий:
 ***Логический процессор*** — один единый вычислительный механизм, с точки зрения операционной системы, приложения, драйвера. 
 ***Ядро*** — это единый блок, состоящий из 1 или нескольких логических процессоров. ***Физический процессор*** — может состоять из 1 или нескольких ядер, может являться центральным процессором 
 
 В 64 разрядной версии Windows для 1 ПК поддерживает более 64 логических процессоров. Поддержка систем, основанная на 64 логических процессоров, основана на процессорных группах (концепции групп процессоров) — статический набор из 64 логических процессоров, рассматриваемых как единое планирующее устройство.
 Данная концепция позволяет более эффективно выполнять программу, т.к. группы имеют общую локальность

______