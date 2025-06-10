______

## Урок 1

https://www.youtube.com/watch?v=9N_wJ7oIHDk&list=PL3BR09unfgcgf7R88ZQRQqWOdLy4pRW2h

Категории грамматических конструкций языка - функция, объект, литерал, оператор.
Литерал - константа времени компиляции.
u8"Hello" - utf8 литерал, U"Hello" - utf32 литерал, nullptr - ключевое слово.
R"h
e
";
вывод в столбик (Row литерал).
Категории значений, массив - это lvalue (т.к. есть локация в памяти)
const char* decayed = "Hellow"; // array-to-pointer convertion
As if rule - компилятор имеет право переупорядочивать программу, но с 1 условием, он должен сохранять порядок побочных эффектов
```cpp
volatile nullptr_t a = nullptr;
int* b;
b= a;
return b;
```

#TODO : Найти в чем проблема, и сказать кто прав clang, gcc или msvc

COW - copy on write (оптимизация для строк)
Нам нужен struingbuf в котором помимо базовых полей делаем refcount
и теперь все строки указывают на общий буфер
поэтому:
```cpp
string s1 = "Hellow";
string s2 = s1; // no allocations using COW
```

При изменении s2 мы дублируем буффер
Но в чем проблемы?
Проблема в инвалидации указателей, т.к. при разделении буфера мы не знаем у кого останется изначальный буффер.
А т.к. operator[] не имеет право менять объект, то COW is dead)
SSO - small string optimization
использовать указатель как указатель, а для больших строк использовать heap indirection

![[Pasted image 20250610214347.png]]

______