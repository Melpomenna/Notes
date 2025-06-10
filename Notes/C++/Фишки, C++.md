#Cpp 

______

для std::function есть правило, чтобы создавался объект используя placement new:
для msvc есть вот такой тип:
```cpp
using _Impl = _Func_impl_no_alloc<decay_t<_Fx>, _Ret, _Types...>;
```
        
Для него выстраиваются правила на аллокацию:

1) 
```cpp
   sizeof(_Impl) <= _Space_size, где _Space_size = (_Small_object_num_ptrs - 1) * sizeof(void*),  а _Small_object_num_ptrs = 6 + 16 / sizeof(void*)
```

2) 
```cpp
   alignof(_Impl) <= alignof(max_align_t), где max_align_t алиас на double
```

3) функция должна быть noexcept

Параметры шаблоны для _Func_impl_no_alloc можно посчитать следующим образом:
```cpp
_Fx - decltype(callable объект)
_Ret - возвращаемый тип лямбды
_Types... - набор параметров вызываемой функции
```


Если все 3 условия выполняются то будет сделан placement new в Storage, хранящий _Small_object_num_ptrs объектов. Если хотя бы 1 не выполняется, то будет произведена аллокация, соответственно это время на синхронизацию дополнительно.

Отсюда в любом случае следует проблемы с std::function:

1) Возвожная аллокация, а для вызова функции постоянное обращение к указателю, который не закеширован скорее всего, частое обращение к std::function, с другими вызовами может привести к потере производительности.

2) Данный вызов не инлайнится, т.к. имеет длинную цепочку вызова, а так же вызов виртуальной функции, в msvc базовый класс помечен __declspec(novtable), поэтому вызова через vtable не будет, нужно посмотреть реализацию в других компиляторах.

Стоит учесть, что _Fx выводиться из шаблона, а не через decltype, поэтому использовать самому может быть проблемой, если только не оборачивать в шаблон


______
#MSVC 

Чтобы отключить отладку контейнеров можно в msvc установить `_HAS_ITERATOR_DEBUGGING` в 0, может увеличить производительность кода

______
#Windows 

Подменяем методы vtable

```cpp

void** GetVtbl(void* instancePtr)
{
return *(void***)instancePtr;
}

void ChangeVtbl(void* instancePtr, void** newVtbl)
{
	*(void**)(instancePtr) = newVtbl;
}

template<class F>
void ReplaceVtblEntry(void** vtblm SizeT index, F f)
{
	// memcpy
	mem::Copy(&vtbl[index], &f, sizeof(void*));
}

SizeT GuessVtblSize(void** vtbl)
{
	for(SizeT i = 0; ; ++i)
	{
		MEMORY_BASIC_INFORMATION memInfo = {};
		VirtualQuery(vtbl[i], &memInfo, sizeof(MEMORY_BASIC_INFORMATION));
	
		DWORD TestFlags = PAGE_EXECUTE | PAGE_EXECUTE_READ | PAGE_EXECUTE_READ_WRITE;
		if (!(memInfo.Protect & TestFlags))
			return i;
	}
}

void Test()
{
	BaseClass* Instance = new TestClass();
	void** OrigVtbl = GetVtbl(Instance);
	SizeT VtblSize = GuessVtblSize(OrigVtbl);
	void** NewVtbl = reinterpret_cast<void**>(mem::Malloc(VtblSize*sizeof(void*)));
	mem::Copy(NewVtbl, OrigVtbl, VtblSize*sizeof(void*));

	ReplaceVtblEntry(NewVtbl, 1, &TestClass::DoBaz);

	ChangeVtbl(Instance, NewVtbl);
	Instance->DoFoo(123);


	ChangeVtbl(Instance, OrigVtbl);
	mem::Free(NewVtbl);
	delete Instance;
}

```

______

Для  методов класса можно ставить final:

```cpp

class Base
{
	virtual void foo();
};

class Derived :  Base
{
	void foo() override final;
};

```

Данный прием помогает компилятору делать девиртуализацию во фронтенде (индерект вызов метода foo); 

______

#Windows 

Получение call stack под Windows

```cpp
#include <dbghelp.h>

#pragma comment(lib, "Dbghelp.lib")
#pragma warning(disable: 4273)

    void CStackTrace::InitSymbols()    
    {
        if (!m_bIsInited) 
        {
            m_bIsInited = SymInitialize(::GetCurrentProcess(), NULL, TRUE);       
        }
    }
    
    void CStackTrace::DestroySymbols()    
    {
        if (m_bIsInited)        
        {
            SymCleanup(::GetCurrentProcess());            
            m_bIsInited = false;
        }    
    }
    
    std::string CStackTrace::GetStackTrace()
    {        
	    HANDLE hCurrentProcess = ::GetCurrentProcess();
        if (!m_bIsInited)
		    return "";
        if (SymRefreshModuleList(hCurrentProcess) == FALSE)
	        return "";
	        
        constexpr int nMaxFrames = 64;
        void* pStackFrames[nMaxFrames];
        memset(pStackFrames, 0, nMaxFrames);
        int nFrames = 0;
        
        nFrames = RtlCaptureStackBackTrace(0, nMaxFrames, pStackFrames, nullptr);
        SYMBOL_INFO* pSymbolInfo = (SYMBOL_INFO*)malloc(sizeof(SYMBOL_INFO) + 256 * sizeof(char));

        if (pSymbolInfo == nullptr)        
            return "";
            
        pSymbolInfo->SizeOfStruct = sizeof(*pSymbolInfo);        
        pSymbolInfo->MaxNameLen = 512;
        std::ostringstream oss;
        
        for (int i = 0; i < nFrames; ++i) {
            oss << "Frame addr:" << pStackFrames[i] << " ";   
                     
            if (SymFromAddr(hCurrentProcess, (DWORD64)(pStackFrames[i]), nullptr, pSymbolInfo) == TRUE) {
                DWORD displacement = 0;                
                IMAGEHLP_LINE line;
                line.SizeOfStruct = sizeof(IMAGEHLP_LINE);                
                if (SymGetLineFromAddr(hCurrentProcess, (DWORD64)(pStackFrames[i]), &displacement, &line) == TRUE) {
                    oss << "Frame " << i << ": " << pSymbolInfo->Name << " in "                        << line.FileName << " line " << line.LineNumber << "\n";
                }                
                else {
                    oss << "Frame " << i << ": " << pSymbolInfo->Name << " (line info not available)\n";                
                }
            }            
            else {
                oss << "Frame " << i << ": (symbol not found)\n";            
            }
        }
        free(pSymbolInfo);        
        return oss.str();
    }
```