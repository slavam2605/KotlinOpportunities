1. Система типов: nullable-типы

    Одной из самых известных особенностей Kotlin является разделение типов на nullable (которые могут
    принимать `null` в качестве значения) и на not-null типы. Nullable типы объявляются со знаком вопроса,
    например `String?`, `Int?` или, например, `Array<Int?>?`. Основное преимущество -- явно разделять 
    типы, которые могут и не могут принимать `null`. По статистике большинство исключений, которые возникают
    в Java являются NullPointerException'ами. Новая система типов существенно помогает решить эту проблему.
2. Система типов: `Nothing`

    В джаве, как и в большинстве современных языков, явно проверяется, что функция по любому пути исполнения
    возвращает некоторое значение. Например, следующий код на Java:
    ```java
    int foo(int x) {
        if (x > 0)
            return 1;
        // Здесь будет ошибка компиляции -- не возвращается никакое значение
    }
    ```
    Однако, есть выражения, которые меняют поток исполнение, например:
    ```java
    int foo(int x) {
        if (x > 0)
            return 1;
        throw new RuntimeException("x must be positive");
        // Здесь не будет ошибки, потому что поток управление не может пройти через throw
    }
    ```
    Но что, если ты хочешь кидать исключения не с помощью throw, а с помощью специальной функции, например,
    `error`, которая перед исключением выводит что-то на консоль?
    ```java
    void error(String message) {
        System.err.println(message);
        throw new RuntimeException(message);
    }
    
    int foo(int x) {
        if (x > 0)
            return 1;
        error("x must be positive");
        // Снова ошибка, должен быть return
    }
    ```
    В таком случае Java никак не может проверить, что функция `error` никогда не выполнится, поэтому
    она требует, чтобы после `error` был `return` или `throw`. Однако в Kotlin для этого есть решение:
    специальный тип `Nothing`. Тип `Nothing` ненаселен, то есть не существует значения, которое имеет такой тип.
    Если какое-то значение имеет тип `Nothing`, то оно никогда не завершится. Тип `Nothing` имеют операторы
    `return`, `break`, `continue`, `throw`, а также его можно получить в некоторых других ситуациях.
    Используя его в Kotlin можно написать функцию `error` с правильным поведением:
    ```kotlin
    fun error(message: String): Nothing {
        System.err.println(message)
        throw RuntimeException(message)
    }
    
    fun foo(x: Int): Int {
        if (x > 0)
            return 1
        error("x must be positive")
        // Никакой ошибки, успешная компиляция
    }
    ```
    Также существует множество других красивых применений типа `Nothing`. Вот пример: Пусть есть список 
    `list: List<String?>`. Хочется в цикле перебрать все значения списка и вывести только те, которые не 
    `null`. Можно это сделать топорным кодом:
    ```kotlin
    for (s in list) {
        if (s != null)
            println(s)
    }
    ```
    Однако, гораздо более удобным и красивым будет следующий способ. Он использует тот факт, что `continue` имеет
    тип `Nothing`, что `Nothing` является подтипом любого типа, а также специальный оператор Элвиса `?:`, который
    работает следующим образом: `a ?: b` эквивалентно `a != null ? a : b`. Сам код:
    ```kotlin
    for (s in list) {
        val string = s ?: continue
        println(string)
    }
    ```
    Этот код работает, потому что `continue` можно положить в переменную любого типа, так как `Nothing` является
    подтипом любого типа. Однако эффективно при вычислении `continue` будет выполнен пропуск итерации цикла, таким 
    образом в `string` попадут только неnullовые значения списка. Существует еще тонна применений типу `Nothing`, 
    например `null` имеет тип `Nothing?`, но их слишком много, чтобы пытаться рассказать все.
3. Система типов: вариантность типов

    В Java есть способ указывать границы на параметры типов, например так:
    ```java
        void foo(Comparator<? super String> comparator) { ... }
    ```
    Такая фукнция принимает компаратор для любого типа `T`, для которого верно, что `String extends T`, то есть 
    супертип (надтип) `String`. Для `String` это `CharSequence`, `Comparable`, `Serializable` и `Object`. Такое 
    объявление типа называется `call-site variance`, то есть вариантность типа на месте вызова. Однако существуют 
    типы, которые имеют сами по себе ковариантную или контравариантную структуру. Ковариантность означает, что 
    функции типов зависят так же, как и аргументы, то есть если `A <: B`, то `F<A> <: F<B>`. Здесь оператор `<:` 
    означает подтип. В Kotlin есть синтаксис (как в C#) для указания вариантности типа:
    ```kotlin
        class ReadOnlyList<out T> { ... }
        // Ковариантный по T, можно писать list: ReadOnlyList<Object> = ReadOnlyList<String>()
        
        class Consumer<in T> { ... }
        // Контравариантный по T, можно писать consumer: Consumer<String> = Comsumer<Object>()
    ```
    
4. Inline лямбды: non-local return

    Анонимные функции (лямбды) и замыкания в Java достаточно слабые -- они имеют значительный вес в рантайме, 
    потому что представляются отдельным классом в памяти, а также они не могут изменять состояние внешних переменных
    и захватывать нефинальные переменные. Однако в Kotlin функции могут принимать не простые функции (например, 
    `fun foo(f: (String) -> Int)`), а inline функции (`fun foo(inline f: (String) -> Int`). Такие функции при 
    компиляции встраиваются в место вызова и могут менять окружающее состояние:
    ```kotlin
    var count = 0
    list.forEach { item -> // <-- начало лямбды
        if (item > 0)
            count++ // Модификация локального состояния
    }
    ```
    Однако, гораздо интереснее применение inline-лямбд состоит в возможности выйти из функции, находясь внутри лямбды.
    Например, следующий код:
    ```kotlin
    fun hasPositive(list: List<Int>): Boolean {
        list.forEach { 
            if (it > 0) // it -- обозначение для единственного аргумента лямбды, если не объявлено явно
                return true // Здесь return выходит из функции hasPositive, а не из лямбды
            // Если требуется выйти из лямбды, то нужно написать return@forEach
        }
        return false
    }
    ```
    Такое свойство inline-функций делает их незаменимыми, в сочетании с тем, что они не существуют в рантайме, 
    полностью встраиваясь в место вызова.
5. Inline-лямды: reified generics

    В Java информация о generic-типах стирается во время исполнения, поэтому подобный код на Java (ровно как и на Kotlin)
    работать не будет:
    ```java
    <T> T[] create(int size) {
        return new T[size]; // Generic array creation error
    }
    ```
    ```kotlin
    fun <T> create(size: Int): Array<T?> {
        return Array<T?>(size) { null } // Cannot use 'T' as reified type parameter error
    }
    ```
    Однако у inline функций есть информация о типе, так как они встраиваются в место вызова. Для таких функций
    параметр типа можно пометить ключевым словом `reified` и тогда он будет доступен в рантайме:
    ```kotlin
    inline fun <reified T> create(size: Int): Array<T?> {
        return Array<T?>(size) { null } // OK    
    }
    ```
