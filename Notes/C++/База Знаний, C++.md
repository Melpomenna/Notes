#Cpp 
[[Фишки, C++]]
[[Полезные статьи и ссылки, C++|Полезные статьи и ссылки, C++]]
[[Книги]]

______
## RVO и NRVO

***RVO*** - Return Value Optimization, ныне обязательная оптимизация, которую выполняет компилятор, аналогично и с NRVO - Named Return Value Optimization - в случае возвращаемого значения, объекта из функции, метода класса, оператор копирования/перемещения не вызывается, а объект конструируется на месте вызова функции, из которой возвращается объект
Если простым языком, при выходе из функции, возвращаемый объект не вызывает конструктор,. оператор копирования/перемещения, а создает объект на месте вызова функции
1) ***RVO*** работает только с prvalue типами и тип возвращаемого значения функции должен совпадать с типом возвращаемого объекта без учета cv-квалификаторов
2) ***NRVO*** работает аналогично с RVO, но работает с lvalue типами
   

______

#Алгоритмы #Компиляторы #Процессоры #Asm

[Itanium ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html)

______

[https://clang.llvm.org/docs/ConstantInterpreter.html#bytecode-compilation](https://clang.llvm.org/docs/ConstantInterpreter.html#bytecode-compilation)

- constexpr вычисления преобразуются в байт код и выполняются интерпретатором:  
    У компилятора есть 2 бекэнда:

1. Для создания байт кода - ByteCodeEmitter
2. Для выполнения выражений без генерации байт кода EvalEmitter

Все функции компилируются в байт код, выражения верхнего уровня, используемые в константных контекстах выполняются напрямую, т.к. их байт код не нужен повторно  
Побайтовый код выполняется на основе стека  
Контекст выполнения состоит из InterpStack и цепочки InterpFrame объектов, хранящих кадры вызовов. Кадры создаются с помощью инструкций вызова и уничтожаются с помощью инструкций возврата.  
Структура памяти:  
Управление памятью в интерпретаторе состоит основано на 3 структурах:

1. Block - хранит данные и встроенные с ним метаданные
2. Pointer - объект, которые ссылаются на Block или входят в них
3. Descriptor - описывает структуры и вложенные в них объекты
______

внутреннее устройство std::pmr::allocator и реализаций memory_resource

реализация на основе msvc:

шаблонный класс polymorphic_allocator имеет следующие особенности:
- alias на value_type:   
```
        using value_type = _Ty;
```
- шаблонный friend класс:
```cpp
        template <class>
        friend class polymorphic_allocator;
```
- класс содержит приватным полем указатель на интерфейс: memory_resource
```cpp
memory_resource* _Resource = _STD pmr::get_default_resource();
```
Об этом интерфейсе чуть позже
- класс имеет следующие методы:
- select_on_container_copy_construction - всегда возвращает дефолтный аллокатор:
```cpp
        _NODISCARD polymorphic_allocator select_on_container_copy_construction() const noexcept /* strengthened */ {
            // don't propagate on copy
            return {};
        }	
```
- метод destroy, для вызова деструктора у указателя:
```cpp
        template <class _Uty>
        void destroy(_Uty* const _Ptr) noexcept /* strengthened */ {
            _Ptr->~_Uty();
        }
```
- метод construct, для конструирования объекта:
```cpp
        template <class _Uty, class... _Types>
        void construct(_Uty* const _Ptr, _Types&&... _Args) {
            // propagate allocator *this if uses_allocator_v<remove_cv_t<_Uty>, polymorphic_allocator>
#if _HAS_CXX20
            // equivalent to calling uninitialized_construct_using_allocator except for handling of cv-qualification
            _STD apply(
                [_Ptr](auto&&... _Construct_args) {
                    return ::new (const_cast<void*>(static_cast<const volatile void*>(_Ptr)))
                        _Uty(_STD forward<decltype(_Construct_args)>(_Construct_args)...);
                },
                _STD uses_allocator_construction_args<_Uty>(*this, _STD forward<_Types>(_Args)...));
#else // ^^^ _HAS_CXX20 / !_HAS_CXX20 vvv
            allocator<char> _Al{};
            if constexpr (_Is_cv_pair<_Uty>) {
                _STD _Uses_alloc_construct_pair(_Ptr, _Al, *this, _STD forward<_Types>(_Args)...);
            } else {
                _STD _Uses_alloc_construct_non_pair(_Ptr, _Al, *this, _STD forward<_Types>(_Args)...);
            }
#endif // ^^^ !_HAS_CXX20 ^^^
        }
```

функция uses_allocator_construction_args возвращает typle из аргументов

- дальше есть ряд функций выделения и освобождения памяти, но нам интересны следующие:
```cpp
        _NODISCARD_RAW_PTR_ALLOC __declspec(allocator) _Ty* allocate(_CRT_GUARDOVERFLOW const size_t _Count) {
            // get space for _Count objects of type _Ty from _Resource
            void* const _Vp = _Resource->allocate(_Get_size_of_n<sizeof(_Ty)>(_Count), alignof(_Ty));
            return static_cast<_Ty*>(_Vp);
        }

        void deallocate(_Ty* const _Ptr, const size_t _Count) noexcept /* strengthened */ {
            // return space for _Count objects of type _Ty to _Resource
            // No need to verify that size_t can represent the size of _Ty[_Count].
            _Resource->deallocate(_Ptr, _Count * sizeof(_Ty), alignof(_Ty));
        }
```

И того получается, что в зависимости от ресурса, по-определенному будет выделена память:
Вот мое небольшое исследование зависимости времени выполнения от количества аллокаций:
![[graph.png]]

в среднем все ресурсы кроме std::default_resource, полученный с помощью
```cpp
std::get_default_resource
```

уменьшает количество аллокаций в разы (с 100000 до 5 для std::set при такой конфигурации):
```cpp
    constexpr int nSizeof = sizeof(int);
    constexpr int nAlign = alignof(int);
    constexpr int maxBlock = std::hardware_constructive_interference_size < nSizeof
        ? (nSizeof + nAlign) << 2
        : std::hardware_constructive_interference_size;
    std::pmr::pool_options options;
    options.largest_required_pool_block = maxBlock;
    options.max_blocks_per_chunk = std::numeric_limits<std::size_t>::max() - 1;
    std::pmr::unsynchronized_pool_resource resource(options);
    std::pmr::polymorphic_allocator<int> alloc(&resource);
```
а скорость работы сильно не увеличивается
unsynchronized_pool_resource: allocations=5 duration=963[1/2688000000]s
std::allocator: allocations=10000004 duration=3679[1/2688000000]s
на самом деле в среднем для 10млн итераций std::set работает лучше, но занимает в раз больше kernel времени (от случая к случаю)
для int
при одновременной вставке 10млн объектов
рассмотрим структуру std::pmr::unsynchronized_pool_resource, но для начала посмотрим каким должен быть интерфейс std::memory_resource:
```cpp
        virtual void* do_allocate(size_t _Bytes, size_t _Align)               = 0;
        virtual void do_deallocate(void* _Ptr, size_t _Bytes, size_t _Align)  = 0;
        virtual bool do_is_equal(const memory_resource& _That) const noexcept = 0;
```

обладать он должен данными методами, теперь к std::pmr::unsynchronized_pool_resource
начнем с полей класса, имеет следующие приватные поля:
```cpp
        pool_options _Options{}; // parameters that control the behavior of this pool resource
        _Intrusive_list<_Oversized_header> _Chunks{}; // list of oversized allocations obtained directly from upstream
        pmr::vector<_Pool> _Pools{}; // pools in order of increasing block size
```

начнем по порядку и для чего все это нужно.
\_Options имеет тип pool_options который устанавливается через конструктор объекта и имеет следующий набор полей:
```cpp
    _EXPORT_STD struct pool_options {
        size_t max_blocks_per_chunk        = 0;
        size_t largest_required_pool_block = 0;
    };
```
В каждом конструкторе класса unsynchronized_pool_resource (далее usync_pool), вызывается функция _Setup_options:
```cpp
        void _Setup_options() noexcept { // configure pool options
            constexpr auto _Max_blocks_per_chunk_limit = static_cast<size_t>(PTRDIFF_MAX);
            constexpr auto _Largest_required_pool_block_limit =
                static_cast<size_t>((PTRDIFF_MAX >> 4) + 1); // somewhat arbitrary power of 2
            static_assert(_Is_pow_2(_Largest_required_pool_block_limit));

            if (_Options.max_blocks_per_chunk - 1 >= _Max_blocks_per_chunk_limit) {
                _Options.max_blocks_per_chunk = _Max_blocks_per_chunk_limit;
            }

            if (_Options.largest_required_pool_block - 1 < sizeof(void*)) {
                _Options.largest_required_pool_block = sizeof(void*);
            } else if (_Options.largest_required_pool_block - 1 >= _Largest_required_pool_block_limit) {
                _Options.largest_required_pool_block = _Largest_required_pool_block_limit;
            } else {
                _Options.largest_required_pool_block = static_cast<size_t>(1)
                                                    << _Ceiling_of_log_2(_Options.largest_required_pool_block);
            }
        }
```

алгоритм ее работы следующий:
1) если max_blocks_per_chunk -1 >= \_Max_blocks_per_chunk_limit, равная PTRDIFF_MAX (9223372036854775807i64), то max_blocks_per_chunk = PTRDIFF_MAX
2) Если largest_required_pool_block-1 < sizeof(void*) (8 или 4 байта), то largest_required_pool_block = sizeof(void*)
3) если largest_required_pool_block-1 >= \_Largest_required_pool_block_limit - это static_cast<size_t>((PTRDIFF_MAX >> 4) + 1) (576460752303423488Ui64), то largest_required_pool_block = \_Largest_required_pool_block_limit
4) в другом случае largest_required_pool_block высчитывается по формуле:
	 1 << ceil(log_2(largest_required_pool_block)), например для largest_required_pool_block=64, largest_required_pool_block будет равен 64, а для 80 - 128
Об ее использовании поговорим чуть позже.
\_Chunks имеет тип \_Intrusive_list<\_Oversized_header>, начнем с \_Oversized_header:
простоя структура имеющая следующее:
```cpp
        struct _Oversized_header : _Double_link<> {
            // tracks an allocation that was obtained directly from the upstream resource
            size_t _Size;
            size_t _Align;

            void* _Base_address() const { // headers are stored at the end of the allocated memory block
                return const_cast<char*>(reinterpret_cast<const char*>(this + 1) - _Size);
            }
        };
```
где \_Double_link<> - 
```cpp
    template <class _Tag = void>
    struct _Double_link { // base class for intrusive doubly-linked structures
        _Double_link* _Next;
        _Double_link* _Prev;
    };
```
сам \_Intrusive_list: имеет приватным полем \_Link_type \_Head{&\_Head, &\_Head}
и работает как обычный список, но без выделения памяти.
Теперь рассмотрим \_Pools
```cpp
            _Chunk* _Unfull_chunk = nullptr; // largest _Chunk with free blocks
            _Intrusive_stack<_Chunk> _All_chunks{}; // all chunks (ordered by decreasing _Id)
            size_t _Next_capacity = _Default_next_capacity; // # of blocks to allocate in next _Chunk
                                                            // in (1, (PTRDIFF_MAX - sizeof(_Chunk)) >> _Log_of_size]
            size_t _Block_size; // size of allocated blocks
            size_t _Log_of_size; // _Block_size == 1 << _Log_of_size
            _Chunk* _Empty_chunk = nullptr; // only _Chunk with all free blocks

            static constexpr size_t _Default_next_capacity = 4;
```
о каждом поле поговорим по мере событий, начнем со структуры \_Chunk:
```cpp
            struct _Chunk : _Single_link<> {
                // a memory allocation consisting of a number of fixed-size blocks to be parceled out
                _Intrusive_stack<_Single_link<>> _Free_blocks{}; // list of free blocks
                size_t _Free_count; // # of unallocated blocks
                size_t _Capacity; // total # of blocks
                char* _Base; // address of first block
                size_t _Next_available = 0; // index of first never-allocated block
                size_t _Id; // unique identifier; increasing order of allocation

                _Chunk(_Pool& _Al, void* const _Base_, const size_t _Capacity_) noexcept
                    : _Free_count{_Capacity_}, _Capacity{_Capacity_}, _Base{static_cast<char*>(_Base_)},
                      _Id{_Al._All_chunks._Empty() ? 0 : _Al._All_chunks._Top()->_Id + 1} {
                    // initialize a chunk of _Capacity blocks, all initially free
                }

                _Chunk(const _Chunk&)            = delete;
                _Chunk& operator=(const _Chunk&) = delete;
            };
```
Структура имеет 1 связь (\_Single_link), \_Intrusive_stack аналог списка выше, но не выделяет память.
Перейдем к методам структуры \_Pools:
метод \_Clear, очищает все чанки памяти:
```cpp
            void _Clear(unsynchronized_pool_resource& _Pool_resource) noexcept {
                // release all chunks in the pool back upstream
                _Intrusive_stack<_Chunk> _Tmp{};
                _STD swap(_Tmp, _All_chunks);
                memory_resource* const _Resource = _Pool_resource.upstream_resource();
                while (!_Tmp._Empty()) {
                    const auto _Ptr = _Tmp._Pop();
                    _Resource->deallocate(_Ptr->_Base, _Size_for_capacity(_Ptr->_Capacity), _Block_size);
                }

                _Unfull_chunk  = nullptr;
                _Next_capacity = _Default_next_capacity;
                _Empty_chunk   = nullptr;
            }
```
сложность O(n)
Выделение памяти:
```cpp
void* _Allocate(unsynchronized_pool_resource& _Pool_resource) { // allocate a block from this pool
    for (;; _Unfull_chunk = _All_chunks._As_item(_Unfull_chunk->_Next)) {
        if (!_Unfull_chunk) {
            _Increase_capacity(_Pool_resource);
        } else if (!_Unfull_chunk->_Free_blocks._Empty()) {
            if (_Unfull_chunk == _Empty_chunk) { // this chunk is no longer empty
                _Empty_chunk = nullptr;
            }
            --_Unfull_chunk->_Free_count;
            return _Unfull_chunk->_Free_blocks._Pop();
        }

        if (_Unfull_chunk->_Next_available < _Unfull_chunk->_Capacity) {
            if (_Unfull_chunk == _Empty_chunk) { // this chunk is no longer empty
                _Empty_chunk = nullptr;
            }
            --_Unfull_chunk->_Free_count;
            char* const _Block = _Unfull_chunk->_Base + _Unfull_chunk->_Next_available * _Block_size;
            ++_Unfull_chunk->_Next_available;
            *(reinterpret_cast<_Chunk**>(_Block + _Block_size) - 1) = _Unfull_chunk;
            return _Block;
        }
    }
}
```
алгоритм следующий:
1) Для каждого чанка (\_Unfull_chunk - наибольший чанк имеющий свободный блок)
   Если текущий чанк пуст (nullptr):
	1) увеличиваем capacity используя \_Increase_capacity:
	2) в \_Increase_capacity алгоритм следующий:
		1) Если текущее capacity больше максимального, то оно становится максимальным
		2) Вычисляется размер для выделения памяти:
		3) Выделяется память с помощью ресурса (об алгоритме позже)
		4) Проверяется выравнивание
		5) Далее в часть указателя вставляем чанк через placement new
		6) пустой чанк становится текущим
		7) \_Unfull_chunk отправляется к пулу всех чанков
		8) увеличиваем capacity в 2 раза, если это возможно
	3) если существует чанк и количество свободных блоков у него не пусто, то
		1) если текущий чанк равен пустому, то зануляем пустой, т.к. он больше не пустой
		2) уменьшаем количество свободных блоков
		3) возвращаем указатель на свободны блок
	4) если количество свободных блоков у чанка пусто, то:
		1) если индекс следующего не выделенного блока меньше чем capacity, то:
			1) если текущий чанк равен пустому, то зануляем пустойm т.к. больше пустым он не будет
			2) уменьшаем количество свободных блоков (по умолчанию их число равно \_Capacity, см. конструктор выше)
			3) берем свободны блок
			4) увеличиваем следующий возможный индекс для выделения памяти на 1
			5) в блок, в конец (заголовок), помещаем \_Unfull_chunk
			6) возвращаем указатель на блок памяти
	```cpp
            size_t _Size_for_capacity(const size_t _Capacity) const noexcept {
                // return the size of a chunk that holds _Capacity blocks
                return (_Capacity << _Log_of_size) + sizeof(_Chunk);
            } 
	```
		   
	   ```cpp
	void _Increase_capacity(unsynchronized_pool_resource& _Pool_resource) {
     // this pool has no free blocks; get a new chunk from upstream
     if (_Next_capacity > _Pool_resource._Options.max_blocks_per_chunk) {
         // This is a fresh pool, _Next_capacity hasn't yet been bounded by max_blocks_per_chunk:
         _Next_capacity = _Pool_resource._Options.max_blocks_per_chunk;
     }
     const size_t _Size               = _Size_for_capacity(_Next_capacity);
     memory_resource* const _Resource = _Pool_resource.upstream_resource();
     void* const _Ptr                 = _Resource->allocate(_Size, _Block_size);
     _Check_alignment(_Ptr, _Block_size);

     void* const _Tmp = static_cast<char*>(_Ptr) + _Size - sizeof(_Chunk);
     _Unfull_chunk    = ::new (_Tmp) _Chunk{*this, _Ptr, _Next_capacity};
     _Empty_chunk     = _Unfull_chunk;
     _All_chunks._Push(_Unfull_chunk);

     // scale _Next_capacity by 2, saturating so that _Size_for_capacity(_Next_capacity) cannot overflow
     _Next_capacity =
         (_STD min)(_Next_capacity << 1, (_STD min)((PTRDIFF_MAX - sizeof(_Chunk)) >> _Log_of_size,
                                             _Pool_resource._Options.max_blocks_per_chunk));
 }

Теперь поговорим о методе \_Deallocate:
1) Забираем указатель на \_Chunk в указателя (заголовке)
2) к чанку среди свободных блоков добавляем указатель
3) увеличиваем количество свободных блоков у чанка
4) Если количество свободных блоков у чанка равно 0:
	1) если \_Unfull чанк пуст и его id меньше id текущего чанка, то \_Unfull становится текущим чанков, функция завершается
5) Если количество свободных блоков у чанка меньше чем capacity у чанка, то функция завершается
6) Если пустой чанк пуст, то он становится текущим чанком, программа завершается
7) Если id пустого чанка меньше чем id текущего чанка, то происходит swap
8) среди всех чанков удаляется текущий
9) очищаем память для указателя у чанка
```cpp
void _Deallocate(unsynchronized_pool_resource& _Pool_resource, void* const _Ptr) noexcept {
    // return a block to this pool
    _Chunk* _Current = *(reinterpret_cast<_Chunk**>(static_cast<char*>(_Ptr) + _Block_size) - 1);

    _Current->_Free_blocks._Push(::new (_Ptr) _Single_link<>);

    if (_Current->_Free_count++ == 0) {
        // prefer to allocate from newer/larger chunks...
        if (!_Unfull_chunk || _Unfull_chunk->_Id < _Current->_Id) {
            _Unfull_chunk = _Current;
        }

        return;
    }

    if (_Current->_Free_count < _Current->_Capacity) {
        return;
    }

    if (!_Empty_chunk) {
        _Empty_chunk = _Current;
        return;
    }

    // ...and release older/smaller chunks to keep the list lengths short.
    if (_Empty_chunk->_Id < _Current->_Id) {
        _STD swap(_Current, _Empty_chunk);
    }

    _All_chunks._Remove(_Current);
    _Pool_resource.upstream_resource()->deallocate(
        _Current->_Base, _Size_for_capacity(_Current->_Capacity), _Block_size);
}
```

теперь можно поговорить о самом аллокаторе:
К слову, 
\_Pool_resource.upstream_resource берется аллокатор у поля \_Pools, который явл. std:::pmr::vector, а там memory_resource по умолчанию.
И прежде чем говорить о методе выделения памяти, опишем вспомогательную функцию поиска пула:

Алгоритм следующий:
1) Берем максимальное значение между выравниванием и количеством байт, которые нужно выделить + sizeof(void*) - \_Size
2) вычисляем ceil(log2(\_Size)) - \_Log_of_size
3) С помощью двоичного поиска ищется (lower_bound указывает на элемент после нужного, или end, если такого нет) пул, \_Log_of_size которого меньше чем вычесленный \_Log_of_size

```cpp
        pair<pmr::vector<_Pool>::iterator, unsigned char> _Find_pool(
            const size_t _Bytes, const size_t _Align) noexcept {
            // find the pool from which to allocate a block with size _Bytes and alignment _Align
            const size_t _Size      = (_STD max)(_Bytes + sizeof(void*), _Align);
            const auto _Log_of_size = static_cast<unsigned char>(_Ceiling_of_log_2(_Size));
            return {
                _STD lower_bound(_Pools.begin(), _Pools.end(), _Log_of_size,
                    [](const _Pool& _Al, const unsigned char _Log) _STATIC_LAMBDA { return _Al._Log_of_size < _Log; }),
                _Log_of_size};
        }
```
Теперь к методу do_allocate у ресурса:
1) Если количество байт для выделения памяти меньше или равно чем largest_required_pool_block у опций, то:
	1) Находим нужный пул
	2) Если такого пула нет или его \_Log_of_size не равен \_Log_of_size найденного пула, то:
		1) Добавляем новый пул среди пулов
	3) С помощью пула (метод \_Allocate см. выше), выделяем память и возвращаем указатель на выделенный блок памяти
	4) Если количество байт для выделения памяти больше чем largest_required_pool_block, то:
		1) Вызываем функцию \_Allocate_oversized:
			1) Подготавливаем интрузивный список из чанков памяти (\_Overisized_header)
			2) выделяем память
			3) проверяем выравниванием
			4) в указатель вставляем структуру на header
			5) к пулу чанков добавляем текущий
			6) возвращаем блок памяти
			   
```cpp
        void* _Allocate_oversized(size_t _Bytes, size_t _Align) {
            // allocate a block directly from the upstream resource
            if (!_Prepare_oversized(_Bytes, _Align)) { // no room for header + alignment padding
                _Xbad_alloc();
            }

            memory_resource* const _Resource = upstream_resource();
            void* const _Ptr                 = _Resource->allocate(_Bytes, _Align);
            _Check_alignment(_Ptr, _Align);

            _Oversized_header* const _Hdr = reinterpret_cast<_Oversized_header*>(static_cast<char*>(_Ptr) + _Bytes) - 1;

            _Hdr->_Size  = _Bytes;
            _Hdr->_Align = _Align;
            _Chunks._Push_front(_Hdr);

            return _Ptr;
        }
```

```cpp
        static constexpr bool _Prepare_oversized(size_t& _Bytes, size_t& _Align) noexcept {
            // adjust size and alignment to allow for an _Oversized_header
            _Align = (_STD max)(_Align, alignof(_Oversized_header));

            if (_Bytes > SIZE_MAX - sizeof(_Oversized_header) - alignof(_Oversized_header) + 1) {
                // no room for header + alignment padding
                return false;
            }

            // adjust _Bytes to the smallest multiple of alignof(_Oversized_header) that is >=
            // _Bytes + sizeof(_Oversized_header), which guarantees that the end of the allocated space
            // is properly aligned for an _Oversized_header.
            _Bytes = (_Bytes + sizeof(_Oversized_header) + alignof(_Oversized_header) - 1)
                   & ~(alignof(_Oversized_header) - 1);

            return true;
        }
```

```cpp
        void* do_allocate(size_t _Bytes, const size_t _Align) override {
            // allocate a block from the appropriate pool, or directly from upstream if too large
            if (_Bytes <= _Options.largest_required_pool_block) {
                auto _Result = _Find_pool(_Bytes, _Align);
                if (_Result.first == _Pools.end() || _Result.first->_Log_of_size != _Result.second) {
                    _Result.first = _Pools.emplace(_Result.first, _Result.second);
                }

                return _Result.first->_Allocate(*this);
            }

            return _Allocate_oversized(_Bytes, _Align);
        }
```

В целом алгоритм не сложный, главное проследить, чтобы \_Allocate_oversized не вызывался по-хорошему
Теперь к очистке:
Работает он в целом аналогично выделению памяти - происходит проверка на largest_required_pool_block, находится нужный пул и удаление происходит через него или с использованием дочернего метода \_Deallocate_overised.
Собственно вот так работает unsynchronized_pool_resource

synchronized_pool_resource работает аналогично, просто наследуется от unsynchronized_pool_resource и при каждом вызове использует мьютекс|


monotonic_memory_resource

______

[ODR](https://translated.turbopages.org/proxy_u/en-ru.ru.b673f5f6-685ce60a-7908486e-74722d776562/https/en.cppreference.com/w/cpp/language/definition.html#One_Definition_Rule)
ODR - One defition rule в c++
Определения — это [декларации](https://translated.turbopages.org/proxy_u/en-ru.ru.b673f5f6-685ce60a-7908486e-74722d776562/https/en.cppreference.com/w/cpp/language/declarations.html), которые полностью описывают сущность, представленную декларацией. Каждая декларация является определением, за исключением следующих: 
Объявления (declaration)— это способ введения (или повторного введения) имён в программу на C++. Не все объявления на самом деле что-либо объявляют, и каждый тип сущностей объявляется по-разному. [Определения](https://translated.turbopages.org/proxy_u/en-ru.ru.b673f5f6-685ce60a-7908486e-74722d776562/https/en.cppreference.com/w/cpp/language/definition.html) — это объявления, которых достаточно для использования сущности, обозначенной именем.  
int f(int); // объявляет, но не определяет f
extern параметр
static поля классов
typedef, using
определение (definition) - реализация.
декларация (declaration) - объявление.
ODR - В любой единице трансляции допускается только одно определение любой переменной, функции, типа класса, типа перечисления, [концепции](https://translated.turbopages.org/proxy_u/en-ru.ru.b673f5f6-685ce60a-7908486e-74722d776562/https/en.cppreference.com/w/cpp/language/constraints.html)(начиная с C++20) или шаблона (некоторые из них могут иметь несколько объявлений, но допускается только одно определение). 
Важно!!!
Одно и только одно определение каждой не [встроенной](https://translated.turbopages.org/proxy_u/en-ru.ru.b673f5f6-685ce60a-7908486e-74722d776562/https/en.cppreference.com/w/cpp/language/inline.html) функции или переменной, которая используется odr (см. Ниже), должно присутствовать во всей программе (включая любые стандартные и определяемые пользователем библиотеки). Компилятор не обязан диагностировать это нарушение, но поведение программы, которая его нарушает, не определено. 
Для встроенной (inline) функции или встроенной переменной(начиная с C++17) требуется определение в каждой единице трансляции, где она используется .