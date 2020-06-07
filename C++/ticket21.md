## Билет 21
Автор: Венедиктов Роман
### Проблемы
1. Много копирований не по делу:
   ```c++
   struct Foo {  // Например, сотрудник.
       string a, b;  // Например, имя и фамилия.
       Foo(string /*const&*/_a, string _b) : a(_a), b(_b) {}
   };
   // ...
   string readString() {
       string result;
       cin >> result;
       return result;
   }
   // ...
   vector<Foo> v = ...;
   std::string a = readString();
   std::string b = readString();
   v.push_back(Foo(a, b)); // Переменные хочу, чтобы явно указать порядок.
   ```
   Копируем и в конструктор `Foo` (хотя там можно ссылку вставить), и в поле,
   а потом ещё и сам `Foo`.
2. `unique_ptr` нельзя копировать, но надо возвращать из функций.
   Надо "гарантированное" RVO или что-то похожее.
3. Возвращение вектора из функции, зачем-то копируем, хотя потом всё равно удалять

### Категории значений
* Ссылки
  * https://en.cppreference.com/w/cpp/language/value_category
  * https://habr.com/ru/post/441742/
* Чистые категории:
  * lvalue — у этого есть имя, нельзя разрушать.
    * Имя переменной: `var`
    * Обращения к полям: `a.m` (тут `a` — lvalue)
    * Обращение по указателю: `p->m` (тут `p` — что угодно)
    * То, что возвращает `T&`: `*p`, `a[i]` (если `a` lvalue для встроенных массивов), `getLval()` (если `int& getLval()`), `static_cast<int&>(x)`
    * Свойства:
      * Можно взять адрес
      * Можно написать "слева"
  * prvalue (как бы были раньше) — имени нет, сейчас помрёт, можно разрушить.
    * Литералы: `42`, `nullptr`
    * То, что возвращает по значению: `a + b`, `getVal()` (если `int getVal();`)
    * Обращения к полям: `a.m` (где `a` — prvalue) (**примечание от читающих ночью (Юра): судя по [cpprefrence](https://en.cppreference.com/w/cpp/language/value_category), a.m, где a - prvalue, - это xvalue**)
    * `this` (именно сам указатель)
    * Лямбда
    * Свойства:
      * Не полиморфное, мы точно знаем тип.
  * xvalue — новая категория — имя есть, но можно разрушить.
    * То, что возвращает `T&&`: `static_cast<int&&>(x)`, `std::move(x)` (это просто `static_cast` внутри).
    * Обращения к полям: `a.m` (тут `a` — xvalue). (**примечание от читающих ночью (Юра): судя по [cpprefrence](https://en.cppreference.com/w/cpp/language/value_category), a.m, где a - prvalue, - это тоже xvalue**)
* Смешанные категории:
  * glvalue — то, у чего есть имя. lvalue или xvalue.
  * rvalue — то, из чего можно мувать. prvalue или xvalue.
* Тонкость: `return v;` из функции — в стандарте стоит костыль, чтобы тут вызывался
  move, а не copy, несмотря на то, что `v` — lvalue.

### Где происходит копирование/перемещение в зависимости от категории и куда пытаемся передать
Если объект — rvalue, то везде move
Если — lvalue, то везде копирование
(( Исключение — конструктор от `initializer_list` в виду реализации в языке 
(https://stackoverflow.com/questions/8193102/initializer-list-and-move-semantics) ))

### `std::move` для смены категории значения, реализация

Это все из коробки уже работает, но если мы знаем, что испольуем объект последний раз, то можем перевести его из lvalue в xvalue с помощью `std::move`

Реализация: 

```c++
template<typename _Tp>
constexpr typename std::remove_reference<_Tp>::type&& move(_Tp&& __t) noexcept{ 
	return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); 
}
```
### Где можно/нужно/нельзя ставить std::move, примеры, в том числе unique_ptr
Можно:
* В конструкторе муваем дальше в переменную
* Последний раз используем объект

Нельзя:
* Использум объект не в последний раз
* return std::move(x) -  так делать нельзя, выключается RVO. Но если x - unique_ptr, то до 17 плюсов нужно, а с 17 там copy-elision (с разбора 1 теста, который про мувы: (тест)[https://www.youtube.com/watch?v=7h-1R8L2cu4&list=PLxMpIvWUjaJuKqgCKqnaCSgn7fHSE2clf&index=20&t=0s]
* auto x = std::move(foo()) - так делать нельзя, так как мувать из rvalue (справа именно оно) бессмысленно

Нужно:
* Для `unique_ptr` при передаче в функцию/конструктор/другую переменную

Хорошие примеры можно взять из теста, (ссылка)[https://docs.google.com/forms/d/e/1FAIpQLSdw3gFQ3aUHfZDUD3BwjiG_L6sguoXCSbW47S2edDg42N9qkQ/viewform]
