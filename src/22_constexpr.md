# Инициализация, constexpr

- [Запись лекции №1](https://youtu.be/Dp2AIvae27M?t=1377)
- [Запись лекции №2](https://www.youtube.com/watch?v=N7P81pzup6w)

## Статическая и динамическая инициализация

```c++
int a = 42;
int f() {
    int result;
    std::cin >> result;
    return result;
}

int a = 42; // инициализируется в момент компиляции
int b = f(); // инициализируется
```

Переменные `a` и `b` устроены по разному: `a` инициализируется на этапе компиляции, а под `b` просто резевируются 4 байта, а её инициализация происходит в момент старта программы.  Говорят, что переменная `a` инициализируется *статически*, a `b` - динамически.

Динамическая инициализация глобальных переменных происходит в момент запуска программы до входа в функцию `main`, соответственно, разрушаются они после выхода из `main`.

Есть несколько причин, почему динамическая инициализация может быть нежелательна:

- Тратит время при запуске программы
- Гарантируется, что в одной единице трансляции переменные инициализируются сверху вниз, а между разными порядок не гарантирован. Это может стать проблемой, если инициализация переменной в одной трансляции обращается к переменной в другой, а та ещё не инициализирована.
- Исключение в динамической инициализации завершает программу, так как до входа в `main`  его негде поймать.

## constexpr функции и переменные

Понятно, что использовать статическую инициализацию приятнее, чем динамическую, но в C++ все функции по дефолту исполняются в рантайме, но иногда мы хотим получить значение на этапе компиляции. Например, это может быть функция `max` для выбора максимального размера классов из шаблонных параметров. В принципе, это можно сделать так:

```c++
template <size_t a, size_t b>
struct max {
    static size_t const value = a < b ? b : a;
};

template <typename A, typename B>
struct variant {
    char stg[max<sizeof(A), sizeof(B)>::value];
};
```

Минус такого подхода, кроме неприятного синтаксиса, в том, что функцию `max` для рантайм-вычислений и структуру `max` приходится писать дважды.

В C++11 разрешили делать функции, которые могут исполняться во время компиляции. Если функция со спецификатором `constexpr` принимает аргументы, значения которых - компайл-тайм константы, то её значение будет вычислено в момент компиляции, иначе в рантайме.

```c++
template <typename T>
constexpr T const& max(T const& a, T const& b) {
    return a < b ? b : a;
}
```

В C++11 были жёсткие ограничения для таких функций - их тело должно было состоять только из одного `return`. В C++14, 17 ~~и 20~~ эти ограничения были сильно ослаблены. 

Если функция `constexpr`, то она должна внутри вызывать только `constexpr` функции.

С появлением `constexpr` функций обнаружилась нужда, например, в `constexpr` переменных. Отличие в том, что функция со спецификатором `constexpr` МОЖЕТ быть посчитана в компайл-тайме, а такая переменная ОБЯЗАНА быть компайл-тайм константой.

```c++
constexpr int f() {
    // ...
}
int const a = 42; // статическая инициализация
int const b = f(); // динамическая инициализация
constexpr int c = f(); // статическая инициализация
```

Переменная с модификатором `const` считается компайл-тайм константой, если она инициализируется статически. Кроме того,`const` для глобальных неявно означает, что они `static` (внутренняя линковка). Предположительно, так было сделано, потому что они подразумевались как замена дефайнов, которые обычно пишутся в хедерах.

Конструкторы и деструкторы так же, как и обычные функции, могут быть constexpr. Например, это позволяет инициализировать объект пользовательского типа статически (например, использовать его в constexpr-контексте).

Начиная с C++20, в constexpr-функциях можно аллоцировать и освобождать память с помощью `new`. К сожалению, `placement new`, так же стал constexpr-выражением только в C++20. Как в C++17 сделать `variant` с constexpr-конструктором? Можно вместо `std::aligned_storage` применить `union`:

```c++
template <typename A, typename B>
union storage_t {
    constexpr storage_t(A a)
        : a(std::move(a)) {}
    A a;
    B b;
};

template <typename A, typename B>
struct variant {
    constexpr variant (A a)
        : index(0),
          storage_t<A, B>(std::move(a)) {}
    A* get_first() {
        return stg.a;
    }
  private:
    size_t index;
    storage_t<A, B> stg;
};
```

`constexpr` может быть полезен, чтобы гарантировать, что static-переменные внутри функции инициализируются статически. Обычная static-переменная инициализируется в тот момент, когда исполнение программы первый раз доходит до её объявления (например, при вызове функции, если переменная не внутри каких-то ветвлений). В таком случае каждый следующий раз происходила бы проверка, инициализирована ли эта переменная. Если же эта переменная объявлена как `constexpr`, то никакие проверки не нужны, так как она инициализируется статически.

```c++
type_descriptor const* empty_type_descriptor {
    static constexpr type_descriptor instance = {
        //...
    };
}
```

## if constexpr

Расмотрим реализацию [function](https://github.com/sorokin/function), которую мы делали на практике.

В его имплементации использовался класс `type_descriptor` со SFINAE. Нам повезло, что критерий для объекта был только один (`fits_small_storage`), но если бы их было несколько, то это выглядело бы сильно хуже. Но даже в нашем случае это выглядит громоздко, например, мы  заводим функцию `initialize_storage`, которая используется только один раз.

Мы могли бы сделать это через `if`, но так получится не всегда:

```c++
struct mytype1 {
    static constexpr bool has_foo = true;
    void foo();
};

struct mytype2 {
    static constexpr bool has_foo = false;
    void bar();
};

template <typename T>
void f(T obj) {
    if (T::has_foo) {
        obj.foo();
    } else {
        obj.bar();
    }
}
```

Проблема в том, что при компиляции `if` оставляет обе ветки, но одна из них не скомпилируется, если у объекта нет какой-то из функций.

В языке для этого появилась конструкция `if constexpr`. Тогда код будет выглядеть следующим образом:

```c++
template <typename T>
void f(T obj) {
    if constexpr (T::has_foo) {
        obj.foo();
    } else {
        obj.bar();
    }
}
```

`if constexpr` работает следующим образом: он требует, чтобы условие было `constexpr` выражением, и делает бранчинг на этапе компиляции, не подставляя ту ветку, которая не подходит условию.

## Дедупликация функций

```c++
template <typename X>
struct A {
    template <typename Y>
    struct B {
        template <typename Z>
        void foo();
    }
}
```

Функция `foo` инстанцируется для каждого набора параметров `X, Y, Z`, даже если какой-нибудь из них не используется. Может ли компилятор посмотреть, что она не зависит, например, от `X` и "склеить" их?

Проблема в том, что у функции есть адрес и кажется логичным, что у разных функций должны быть разные адреса.

```c++
assert(&foo<int> != &foo<float>);
```

В стандарте (`7.6.10`) есть следующие слова:  *Otherwise, if the pointers are both null, both point to the same function, or both [represent the same address](https://eel.is/c++draft/basic.compound#def:represents_the_address), they compare equal[.](https://eel.is/c++draft/expr.eq#3.2.sentence-1)* Некоторые люди считают, что слова про *same address* могут относиться и к функциям тоже. В таком случае "склеивание" инстанциаций шаблонной функции валидно.

Некоторые линковщики умеют дедуплицировать функции, например, у `MSVC` функции склеиваются в одну. Линковщик *gold*, например, работает с этим аккуратнее - к примеру, не склеивает конструктор и деструктор, а так же может не склеивать функции, у которых берутся адреса.

Есть возможность уменьшить размер кода после инстанцирования шаблонов без нарушения инварианта на равенство адресов - можно оставить тело одной функции, а из остальных сделать на неё `jmp`. 

В GCC есть оптимизация `-fipa-icf`, которая "склеивает" функции - она работает эффективнее, если включена линк-тайм оптимизация, потому что на этапе линковки больше информации про функции из других единиц трансляции. Кроме того, на этапе компиляции оптимизируется код в каком-то внутреннем представлении компилятора, но машинный код для разных представлений может быть одинаковым, что становится понятно на этапе линковки.

Забавный факт - профилирование и отладка программы усложняются при "склеивании" функций (вызываем `foo`, а исполнение прыгает в функции `bar`).  В *gold* это обошли, анализируя цепочку вызовов и выводя из неё, какая функция была вызвана.

## if constexpr и дедупликация

Возвращаясь к реализации `function`: там для каждого типа у `empty_type_descriptor` создавалась пустая лямбда-функция деструктора. В реализации `function` у этих лямбд берётся адрес, хоть нам и не важно, чтобы они не совпадали, но компилятор их не склеивает. Как это пофиксить?

Заметим, что вообще в реализации `storage` был шаблонным, но шаблонный параметр использовался только в типе указателя, который он хранит, поэтому можно убрать шаблон и хранить просто `void*`. Тогда можно вынести функцию из пустого дескриптора наружу:

```c++
void destroy_trivial(storage*) {}
```

А в дескрипторе взять адрес этой функции, который будет одинаковый для всех пустых дескрипторов. 

Аналогично можно оптимизировать не только для пустого дескриптора - для всех тривиальных копирований, перемещений и уничтожений, можно объединить функции. Например:

```c++
template <typename T, typename R, typename... Args>
constexpr type_descriptor<R, Args...> compute_type_descriptor() {
    // if constexpr на свойства типа и т.д.
}

template <template typename T, typename R, typename... Args> 
inline constexpr type_descriptor<R, Args...> type_descr_instance = compute_type_descriptor<T, R, Args...>(); 
```

*inline нужен для того, чтобы во всех единицах трансляции эта глобальная переменная была общей, потому что по умолчанию у глобальных константных переменных внутренняя линковка*

Тогда внутри `compute_type_descriptor` может быть следующая конструкция:

```c++
template <typename T, typename R, typename... Args>
constexpr type_descriptor<R, Args...> compute_type_descriptor() {
    if constexpr (fits_small_storage<T>) {
        type_descriptor<R, Args...> result;
        if constexpr (std::is_trivially_destructible_v<T>) {
            result.destroy = &destroy_trivial;
        } else {
            result.destroy = [](storage* src) noexcept {
                src->template get_static<T>().~T();
            };
        }
        // copy, move, etc
    } else {
        // ...
    }
}
```

Такой код некорректен с точки зрения C++17, но корректен для C++20, потому что в 20-м стандарте разрешили оставлять неинициализированными переменные в constexpr выражениях (в данном случае мемберы `result`). Для C++17 можно сделать костыль и сначала проинициализировать всё нулями, а потом уже нужными значениями.

К чему вообще этот пример? Чтобы продемонстрировать, насколько `if constexpr` помогает упростить и сделать читабельной логику того, как работает функция в зависимости от каких-то компайл-тайм констант (в данном случае - свойств типов) - сделать такое через SFINAE было бы труднее.