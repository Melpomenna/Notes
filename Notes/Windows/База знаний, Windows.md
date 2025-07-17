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

#Cpp 

https://learn.microsoft.com/ru-ru/cpp/intrinsics/rdtsc?view=msvc-170

\_\_rdtsc - Создает инструкцию, которая возвращает метку времени процессора. Метка времени процессора записывает количество циклов часов с момента последнего сброса.

______

#Cpp 

Быстрый подсчет времени:

```cpp

class PerformanceClockCounter
{

public:

[[nodiscard]] unsigned GetTickCount() const noexcept
{
	return __rdtsc();
}

[[nodiscard]] unsigned GetTickPerSec() const noexcept
{
	return tickPerSec_;
}

private:

void RecalcTimePerSec() noexcept
{
	LARGE_INTEGER freq;
	QueryPerformanceFrequency(&freq);
	LARGE_INTEGER startQpc;
	LARGE_INTEGER endQpc;

	MemoryBarrier();
	QueryPerformanceCounter(&startQpc);
	const unsigned startTsc = __rdtsc();
	MemoryBarrier();

	do
	{
	  QueryPerformanceCounter(&endQpc);
	} while((endQpc.QuadPart - startQpc.QuadPart) > freq.QuadPart / 32);

	MemoryBarrier();
	const unsigned endTsc = __rdtsc();
	tickPerSec_ = ((endTsc - startTsc)*freq.QuadPart) / (endQpc.QuadPart - startQpc.QuadPart);
}

unsigned tickPerSec_{0};
};

```

______

#Cpp 

Функция для именования потока

```cpp

void SetThreadName(DWORD dwThreadID, const char* threadName)
{
	const DWORD MS_VC_EXCEPTION = 0x406D1388;
	#pragma pack(push, 8)
		typedef struct tagTHREADNAME_INFO
		{
		  DWORD dwType;
		  LPCSTR szName;
		  DWORD dwThreadID;
		  DWROD dwFlags;
		} THREADNAME_INFO;
	#pragma pack(pop)

	THREADNAME_INFO info;
	info.dwType = 0x1000; // must be 0x1000
	info.szName = threadName;
	info.dwThreadID = dwThreadID;
	info.dwFlags = 0;
	#pragma warning(push)
	#pragma warning(disable: 6320, 6322)

	__try
	{
		RaiseException(MS_VC_EXCEPTION, 0, sizeof(info)/sizeof(ULONG_PTR), (ULONG_PTR*)&info);
	}
	__except(EXCEPTION_EXCLUDE_HANDLER)
	{
	}
	#pragma warning(pop)
}
```

______

Event Tracing for Windows (ETW) - механизм для трассирования и логирования событий произошедших в user-mode приложениях и kernel-mode драйверах.

События собираемые через ETW:
- Process Start/End
- Thread Start/End
- CSwitch / ReadyThread
- Image Load/Unload
- File Create/Destroy/Rename/Map/Unmap/Read/Write
- Virtual Alloc/Free + Page Faults
Пример (thx CppRussia & LestaGames):

```cpp

void ETWTraceCallback(PEVENT_RECORD recod)
{

// https://learn.microsoft.com/en-us/windows/win32/etw/cswitch
struct CSwitch
{
	// ...
};

static constexpr UCHAR CONSTEXT_SWITCH_OPCODE = 36;

if (record->EventHeader.EventDescriptor.OpCode != CONSTEXT_SWITCH_OPCODE) 
	return;
const CSwitch* cs = reinterpret_cast<const CSwitch*>(record->UserData);

ContextSwitchReccord res;
res.timeStamp = record->EventHeader.TimeStamp.QuadPart;
res.newThreadID = cs->NewThreadId;
res.oldThreadID = cs->OldThreadId;
res.reason = cs->OldThreadWaitReason;
res.processor = static_cast<UInt8>(record->BufferContext.ProcessorIndex);
reinterpret_cast<ETWTracer*>(record->UserContext)->m_callback(res);

}

ETWTracer* CreateETWTracer(ETWEventCallback&& callback)
{


TracerInfo info = {};
TracerInfo tmpInfo = info;
ControlTrace(NULL, KERNEL_LOGGER_NAME, &tempInfo.base, EVENT_TRACE_CONTROL_STOP);

TRACEHANDLE hTrace;
ULONG nRes = StartTrace(&hTrace, KERNEL_LOGGER_NAME, &info.base);
ETWTracer* tracer = new ETWTracer();

EVENT_TRACE_LOGFILE traceLog = {};


traceLog.EventRecordCallback = ETWTraceCallback;
traceLog.Context = tracer;
tracer->m_handle = OpenTrace(&traceLog);

struct Internal
{
	static void TraceThreadEntryPoint(void* userData)
	{
		ETWTrace* tracer = reinterpret_cast<ETWTracer*>(userData);
		ProcessTrace(&tracer->m_handle, 1, NULL, NULL);
	}
};

tracer->m_thread.Start(&Internal::TraceThreadEntryPoint, tracer, "ETW Trace Thread");

}

```

______

#Cpp 

Подменяем вызовы функций для Windows
Немного теории:
в каждом модуле есть таблиц импорта (список всех импортируемых модулей)
Есть функция GetModuleHandle - она возвращает адрес начала где загружен блок (image модуля).
Если посмотреть строение PE (Portable Executable) там есть NT headers, там есть Data Directories (все что есть в dll или exe) и там есть список функций, которые модуль берет за собой
#TODO прочитать про PE и ImageLoader
ImageLoader пропатчит и слинкует все

Вот как можно заменить исходную функцию на свою:

```cpp

#define WIN32_LEAN_AND_MEAN
#include <Windows.h>
#include <DbgHelp.h>
#include <iostream>

#pragma comment(lib, "Dbghelp.lib")

typedef LPVOID(WINAPI* PFN_VirtualAlloc)(
    _In_opt_ LPVOID,
    _In_ SIZE_T,
    _In_ DWORD,
    _In_ DWORD);

static PFN_VirtualAlloc g_OriginalVirtualAlloc = nullptr;

extern "C" LPVOID WINAPI OurVirtualAlloc(
    _In_opt_ LPVOID lpAddress,
    _In_ SIZE_T dwSize,
    _In_ DWORD flAllocationType,
    _In_ DWORD flProtect
)
{
    std::cout << "Alloc " << dwSize << '\n';
    return g_OriginalVirtualAlloc(lpAddress, dwSize, flAllocationType, flProtect);
}

void HookIAT(HMODULE module, const char* importModuleName, const char* functionName, void* newFunction, void** originalFunction)
{
    ULONG size = 0;
    auto* importDesc = (PIMAGE_IMPORT_DESCRIPTOR)ImageDirectoryEntryToData(module, TRUE, IMAGE_DIRECTORY_ENTRY_IMPORT, &size);
    if (!importDesc) return;

    for (; importDesc->Name; ++importDesc)
    {
        const char* modName = (const char*)((PBYTE)module + importDesc->Name);
        if (_stricmp(modName, importModuleName) != 0)
            continue;

        auto* thunk = (PIMAGE_THUNK_DATA)((PBYTE)module + importDesc->FirstThunk);
        auto* origThunk = (PIMAGE_THUNK_DATA)((PBYTE)module + importDesc->OriginalFirstThunk);
        for (; thunk->u1.Function; ++thunk, ++origThunk)
        {
            if (origThunk->u1.Ordinal & IMAGE_ORDINAL_FLAG)
                continue;

            auto* importByName = (PIMAGE_IMPORT_BY_NAME)((PBYTE)module + origThunk->u1.AddressOfData);
            if (strcmp((char*)importByName->Name, functionName) == 0)
            {
                DWORD oldProtect;
                VirtualProtect(&thunk->u1.Function, sizeof(void*), PAGE_READWRITE, &oldProtect);
                if (originalFunction && !*originalFunction)
                    *originalFunction = (void*)thunk->u1.Function;
                thunk->u1.Function = (ULONGLONG)newFunction;
                VirtualProtect(&thunk->u1.Function, sizeof(void*), oldProtect, &oldProtect);
                return;
            }
        }
    }
}

struct HookInitializer
{
    HookInitializer()
    {
        HMODULE exeModule = GetModuleHandle(nullptr);
        g_OriginalVirtualAlloc = &VirtualAlloc;
        HookIAT(exeModule, "KERNEL32.dll", "VirtualAlloc", (void*)&OurVirtualAlloc, (void**)&g_OriginalVirtualAlloc);
    }
};

#pragma init_seg(lib)
static HookInitializer g_HookInit;

int main()
{
    VirtualAlloc(nullptr, 1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    return 0;
}

```


_____

Реализация Winapi в Wine (ориг см. в gitlab)
https://github.com/wine-mirror/

______

Обзор от ИИ
      
 In the context of operating systems and synchronization, a mutant is a synchronization primitive, similar to a mutex, that allows exclusive access to a resource. It's a kernel object that can be in a signaled or non-signaled state, controlling access to a shared resource. When a thread wants to access the resource, it attempts to "acquire" the mutant. If the mutant is already acquired (non-signaled), the thread waits until it becomes available (signaled). 

  
       Here's a more detailed explanation:   
  
Exclusive Access: A
 mutant is designed to ensure that only one thread can access a specific resource at a time, preventing race conditions and data corruption. 


Signaled/Non-Signaled: 
A mutant has a state that can be either signaled or non-signaled. When signaled, it indicates the resource is available. When non-signaled, it means the resource is currently in use. 


Acquisition and Release: 
Threads acquire (wait for) and release (signal) the mutant. When a thread acquires it, the mutant transitions to the non-signaled state,
 and the thread gains exclusive access to the resource. When the thread is done, it releases the mutant, making it available again (signaled). 

NtQueryMutant: 
This is a Windows API function (ntdll.dll) that allows querying information about a mutant object. Specifically, it's used to retrieve basic information about the mutant, such as its 
current state (signaled or non-signaled) and other attributes. The MUTANT_BASIC_INFORMATION structure is used to store the retrieved information. 

Relationship to Mutex: 
In many ways, a mutant is very similar to a mutex (mutual exclusion object). The key difference is that a mutant can be used to create a hierarchy of 
synchronization objects in some operating systems, allowing for more complex synchronization scenarios. 

NtCreateMutant: This is another Windows API function (also from ntdll.dll) used to create a mutant object, which is the first step in using it for synchronization.

______

Если вдруг появилась проблема в сети wifi: "Без подключения к интернету"
Если ничего не помогло из интернетика, то через poweshell запущенный от имени администратора нужно ввести:
```
ipconfig /flushdns
ipconfig /registerdns
ipconfig /release
ipconfig /renew
netsh winsock reset catalog
netsh int ip reset
netsh ipv4 reset reset.log
netsh ipv6 reset reset.log
```