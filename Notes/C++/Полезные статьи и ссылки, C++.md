#Cpp 

[Хеш-таблица и C++20](https://habr.com/p/897924/)

1) Использование концептов для более удобного анализа ошибок во время компиляции
2) Использование корутин, для LazyValue
3) применение с rangeы, без циклов (на мой взгляд сомнительно)
4) if constexpr, не совсем удачный, можно просто сделать emplace
5) удаление cv ref cpec через std::remove_cvref_t,  есть CleanKey, в то же время сам key имеет тип Key&&, который может измениться, не совсем понятно зачем
6) std::span для безопасной передачи в insert_range
7) \[\[no_unique_address]\] для аллокатора, для уменьшения размера памяти для объектов без состояния
8) thread-safety через std::atomic_ref<T> (аля lock-free, без mutex)
    
______
   
   [Cpp Standart draft](https://eel.is/c++draft/)
   
______
   
   [Безопасная работа с массивами? Нет, не слышали](https://habr.com/p/895208/)
   
   В примере рассказывалось о примере с использованием двумерного массива и выписки из стандарта, что массив лежит последовательно в памяти и обращение за границу - UB
Соответственно a[0][n] где n > size - UB, хоть и элементы в памяти и должны располагаться последовательно, соответственно результат может быть не самым приятным
Выводы:
Использовать безопасные обертки на типе std::mdspan
Не использовать двумерный массив как одномерный
______

## Как проверить к какому языку относиться h файл?

Нужно проверить существование макроса:  ***__cplusplus***

______

[Делаем собственный анализатор C++ кода в виде плагина для Clang](https://habr.com/p/900224/)

Небольшой гайд по созданию плагина для Clang
1) Рассказывается о преимуществе RecursiveASTVisitor по сравнению с AST matcher (более производительный и удобный по сравнению с AST matcher)
2) Как логировать код плагина: llvm::outs, llvm::errs
В целом больше говорилось о кастомных атрибутах (основная видная мысль)
Но в целом можно взять небольшие выдержки при написании своих плагинов под LLVM

______

[16 байт вместо 32: управляем layout'ом в C++](https://habr.com/p/899784/)
Не плохая статья, рассказывающая основы упаковки данных в структурах, выравнивании, битовых полях, а так же о паддинге
Из интересно подмеченного:
1) Проверка размера структуры во время компиляции, а так же офсета полей в constexpr, можно на этом сделать концепт
2)вывода размера структуры через clang:
clang++ -Xclang -fdump-record-layouts
Хотелось бы услышать про cache-friendly, но там этого не было
На примере показали как упаковать структуру с 32 байт до 16
2) pragma pack(push,1) - отключение выравнивания 
3) Стандартно выравнивание - 8 байт для 64 битных приложений
______

[RAII 2.0: RAII как архитектурный инструмент в C++](https://habr.com/p/901092/)

Представлены примеры идеомы RAII не для ресурсов
А как архитектурный стиль, например для транзакций или отмены асинхронный операций через деструктор
В целом и то и то ресурсы, но если не думать только в рамках ресурсах, а о задачах, то получается безопасное решение
______

#MSVC 

Чтобы использовать cl.exe нужно настроить следующие переменные:
INCLUDE, LIB:
INCLUDE:
```
C:\Program Files (x86)\Microsoft Visual Studio\<версия>\VC\include
 C:\Program Files (x86)\Windows Kits\10\Include\<версия>\ucrt 
 C:\Program Files (x86)\Windows Kits\10\Include\<версия>\shared 
 C:\Program Files (x86)\Windows Kits\10\Include\<версия>\um 
 C:\Program Files (x86)\Windows Kits\10\Include\<версия>\winrt
```

LIB:
```
C:\Program Files (x86)\Microsoft Visual Studio\<версия>\VC\lib
C:\Program Files (x86)\Windows Kits\10\Lib\<версия>\ucrt\x64 
C:\Program Files (x86)\Windows Kits\10\Lib\<версия>\um\x64
```
______

[Senders/Receivers в C++26: от теории к практике](https://habr.com/p/904134/)

В статье рассказываются о будущей возможности к асинхронному выполнению - это концепция Sender/Receiver

Помогает выстраивать более архитектурно, нежели с использование std::future/std::async std::thread и текущей Concurency Thread Support Library

Выглядит достаточно компактно, есть свои недостатки, так же есть поддержка корутин

Ждем чтобы опробовать

______

[WG21](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/)

Список пропозлов на год и месяц по c++
______

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3665r0.html

я не знаю как это оценить, но пропозл говорит о вертикально написании текста

______
#MSVC #Microsoft  #Компиляторы #Алгоритмы 

Microsoft dash board for c++26 features: https://github.com/orgs/microsoft/projects/1143

Microsoft dash board for c++23 features: https://github.com/orgs/microsoft/projects/1142

Microsoft STL official source code: https://github.com/microsoft/STL/

______

## Intel VTune

[Documentation](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-0/overview.html?language=en?language=en)

Программа для  профилирования кода и просмотра метрик

   
______
#CMake

В benchmarks от Google, чтобы не собирались для данной библиотеки тесты нужно установить следующие макросы:

```cmake
set(BENCHMARK_ENABLE_GTEST_TESTS  OFF CACHE BOOL "Whether to install benchmark")
set(BENCHMARK_ENABLE_TESTING OFF CACHE BOOL "Disable tests")
set(BENCHMARK_INSTALL_DOCS OFF CACHE BOOL "Disable docs")
```

______
#CMake

Чтобы clang-tidy работал каждый раз при компиляции можно сделать следующую функцию:

```CMake
function(UseClangTidy target)
    find_program(CLANG_TIDY_EXE NAMES "clang-tidy")

    if (NOT CLANG_TIDY_EXE)
        message(WARNING "no command clang-tidy")
    else ()
        message(STATUS "Set clang-tidy for ${target}")
        set(CLANG_TIDY_COMMAND "${CLANG_TIDY_EXE}" "--config-file=${CMAKE_SOURCE_DIR}/.clang-tidy")
        set_target_properties(${target} PROPERTIES CXX_CLANG_TIDY "${CLANG_TIDY_COMMAND}")
    endif ()
endfunction()
```


______

#MSVC #Microsoft 

Компания выпустила библиотеку GSL - Guideine Support Library, которая помогает писать более безопасный код.

Вот основные классы/функции, который в ней есть:
[GitHub](https://github.com/microsoft/GSL)
[YouTube](https://www.youtube.com/watch?v=eFJd3JNUEIs&t=232s)

gsl::finally - работает на основе RAII, принимает лямбду и вызывает ее в деструкторе

gsl::narrow - вызывает исключение при не корректном преобразовании типов, например преобразование int в char, а значение int больше диапазона char

gsl::not_null - проверяет, что указатель не является null. Обертка на типом. Во время компиляции проверяет, чтобы указатель не был null, во время runtime вызывает ошибку и программа прекращает работать

gsl::owner - синтаксическая аннотация, обертка над указателем - означает что указатель в данном скоупе должен быть освобожден.

gsl::span - обертка над последовательностью, для проверки границ

______

#CMake 

 Функция для проверки кода на cppcheck из CMake

```CMake
function(use_cppcheck target)
	 find_program(CPPCHECK_COMMAND cppcheck) 
	 if (CPPCHECK_COMMAND) 
		 message("CPPCheck founded!\n") 
		 set_target_properties(${target} PROPERTIES C_CPPCHECK "${CPPCHECK_COMMAND};-q;-j4;--enable=information,performance,portability,warning;--error-exitcode=1;}") 
	endif() 
endfunction()
```
_____

#Windows #Компиляторы

https://www.orderofsixangles.com/ru/2020/03/20/win-def-1-ru.html

Статья рассказывает о механизме защиты от переполнения буффера с помощью securiy cookie
Смысл заключается в том, что на стеке размещается значение secuiry cookie и в конце проверяется
Если происходит выход за границы буффера, то мы затрем secuirty cookie и программа, при проверке security cookie, сделает принудительное завершение
Включается по умолчанию с помощью ключа компилятора /GS
Если переопределить точку входа /ENTRY, то security cookie нужно будет самому инициализировать, т.к. перед вызовом main инициализируется C Runtime, где и инициализируется security cookie
____