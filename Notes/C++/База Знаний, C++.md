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