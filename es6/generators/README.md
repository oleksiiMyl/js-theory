### Итераторы

**Итератором** (iterator) называется структирированный шаблон, предназначенный для последовательного извлечения
информации из источника. В программировании подобные шаблоны используются давно. Разумеется, и на JS с
незапамятных времен проектируются и реализуются итераторы, так что это далеко не новая тема.

В ES6 для итераторов появился неявный стандартизованный интерфейс. Многие встроенные структуры JavaScript
предоставляют итератор, реализующий этот стандарт. Вы можете сконструировать и собственные версии итераторов,
если будете придерживаться стандарта для обеспечения максимальной совместимости.

Например, можно реализовать сервис, который при каждом запросе будет генерировать новый уникальный идентификатор;
или формировать бесконечный ряд значений, прокручиваемых в фиксированном списке циклическим способом; или
присоединить итератор к результатам запроса к базе данных, чтобы по одной извлекать новые строки.

#### Интерфейсы

Интерфейс `Iterator` отвечает следующим требованиям:

    Iterator [обязательные параметры]
        next() { метод }: загружает следующий IteratorResult

Два дополнительных параметра для расширения некоторых итераторов:

    Iterator [обязательные параметры]
        return() { метод } останавливает итератор и возвращает IteratorResult
        throw() { метод } сообщает об ошибке и возвращает IteratorResult

Интерфейс `IteratorResult` определен следующим образом:

    IteratorResult
        value { свойство }: значение на текущей итерации или окончательное возвращаемое значение
                            (не обязательно, если это 'undefined')
        done { свойство }: тип boolean, показывает состояние выполнения

Существует также интерфейс `Iterable`, описывающий объект, который должен уметь генерировать итераторы:

    Iterable
        @@iterator() { метод }: генерирует Iterator

`@@iterator` - специальный встроенный символ, представляющий метод, который умеет генерировать итераторы для объекта.

#### IteratorResult

Интерфейс `IteratorResult` определяет, что возвращаемое значение любой из операций итератора будет представлять
собой объект следующего вида:

    { value: .. , done: true / false }

Встроенные итераторы всегда возвращают значения в таком виде, но если нужно, ничто не мешает добавить и другие свойства.

Например, пользовательский итератор может вставлять в результирующий объект дополнительные метаданные (например, откуда
взялись данные, сколько времени заняло их извлечение, каково время жизни кэша и частота для следующего запроса и т.п.).

#### Метод next()

Рассмотрим итерируемый массив и итератор, который он создает для работы со своими значениями:

    var arr = [1,2,3];
    
    var it = arr[Symbol.iterator]();
    
    it.next();		// { value: 1, done: false }
    it.next();		// { value: 2, done: false }
    it.next();		// { value: 3, done: false }
    
    it.next();		// { value: undefined, done: true }

Каждый раз, когда связанный со свойством `Symbol.iterator` метод вызывается со значением `arr`, он создает свежий
итератор. Большинство структур будут давать аналогичный результат, в том числе встроенные в JS структуры данных.

Но структура, которая, к примеру, является получателем очереди событий, может создать лишь единственный итератор
(шаблон одиночка). Еще бывают структуры, допускающие в каждый момент времени только один итератор. В этом случае
перед созданием нового итератора следует завершить уже существующий.

Итератор `it` из последнего фрагмента кода не сообщает о завершении своей работы: когда вы получаете значение 3,
`false` меняется на `true`. Но чтобы узнать об этом, вам приходится еще раз вызывать метод `next()`, фактически
выходя за границу значений массива.

Примитивные строковые значения также по умолчанию итерируемые:

    var greeting = "hello world";
    
    var it = greeting[Symbol.iterator]();
    
    it.next();		// { value: "h", done: false }
    it.next();		// { value: "e", done: false }
    ..

С технической точки зрения само по себе значение примитива итерируемым не является, но благодаря "упаковке" строка
`"hello world"` преобразуется в итерируемый объект-оболочку `String`.

В ES6 также появилось несколько новых структур данных, называемых коллекциями. Коллекции не только итерируемы,
но и предоставляют API для генерации итератора. Например:

    var m = new Map();
    m.set( "foo", 42 );
    m.set( { cool: true }, "hello world" );
    
    var it1 = m[Symbol.iterator]();
    var it2 = m.entries();
    
    it1.next();		// { value: [ "foo", 42 ], done: false }
    it2.next();		// { value: [ "foo", 42 ], done: false }
    ..

Метод итератора `next()` в зависимости от вашего желания принимает один или несколько аргументов. Встроенные
итераторы эту возможность в основном не используют, чего нельзя сказать об итераторе генератора.

По общему соглашению, вызов метода `next(..)` после завершения работы итератора (в том числе для встроенных
итераторов) не приводит к ошибке, а дает результат `{ value: undefined, done: true }`.

#### Необязательные методы: return(..) и throw(..)

Необязательные методы интерфейса итератора - `return(..)` и `throw(..)` - у большинства встроенных итераторов не
реализуются. Тем не менее они, безусловно, значимы в контексте генераторов.

Метод `return(..)` определен как отправляющий итератору сигнал о том, что код извлечения информации завершил
работу и больше не будет извлекать значения. Этот сигнал уведомляет итератор, отвечающий на вызов метода `next(..)`,
что пришло время приступить к очистке, например к освобождению/закрытию сетевого соединения, базы данных или
дескриптора файла.

Если у итератора есть метод `return(..)` и при этом возникает любое условие, которое можно интерпретировать как
аномальное или ранее прекращение использования итератора, этот метод вызывается автоматически. Впрочем это легко
сделать и вручную.

Метод `return(..)`, как и `next(..)`, возвращает объект `IteratorResult`. В общем случае необязательное значение,
отправленное в метод `return(..)`, будет возвращено как значение в объекте `IteratorResult`, хотя возможны и другие
варианты развития событий.

Метод `throw(..)` сообщает итератору об исключению или ошибке, причем эту информацию тот может использовать иначе,
нежели подаваемый методом `return(..)` сигнал завершения. Ведь, в отличие от последнего, исключение или ошибка
вовсе не подразумевает обязательного прекращения работы итератора.

Например, в случае с итераторами генератора метод `throw(..)`, по сути, вставляет в приостановленный контекст
выполнения генератора порожденное им исключение, которое может быть перехвачено оператором `try..catch`.
Необработанное исключение в конечном счете прерывает работу итератора.

По общему соглашению, после вызова методов `return(..)` или `throw(..)` итератор не должен больше генерировать
никаких результатов.

#### Цикл итератора

Появившийся в ES6 цикл `for..of` работает непосредственно с итерируемыми объектами.

Если итератор сам является итерируемым, его можно напрямую использовать с циклом `for..of`. Чтобы сделать итератор
таковым, его следует передать методу `Symbol.iterator`, который вернет нужный результат:

    var it = {
        // делаем итератор `it` итерируемым
        [Symbol.iterator]() { return this; },
    
        next() { .. },
        ..
    };
    
    it[Symbol.iterator]() === it;		// true

Теперь можно вставить итератор `it` в цикл `for..of`:

    for (var v of it) {
        console.log( v );
    }

Чтобы полносью понять, что тут происходит, вспомните, как работает эквивалент цикла `for..of` - цикл `for`:

    for (var v, res; (res = it.next()) && !res.done; ) {
        v = res.value;
        console.log( v );
    }

При ближайшем рассмотрении вы видим, что перед каждой итерацией вызывается метод `it.next()`, после чего происходит
сравнение с переменной `res.done`. Если она имеет значение `true`, выражение получает результат `false`, и
следующей итерации не происходит.

Напомню, что ранее мы уже говорили о принятой для итераторов тенденции не возвращать вместе с предполагаемым
окончательным значением `done: true`. Теперь вы видите почему так происходит.

Если итератор вернет `{ done: true, value: 42 }`, цикл `for..of` отбросит последнее значение `42`, и оно будет потеряно.
Именно из-за того, что итераторы могут использоваться такими шаблонами, как цикл `for..of` или его эквиваленты,
следует подождать с возвращением сигнализирующего о завершении работы значения `done: true`, пока не будут
возвращены все связанные с итерациями значения.

#### Применение итераторов

Вы уже видели, как итератор шаг за шагом работает в цикле `for..of`. Но им могут пользоваться и другие структуры.

Рассмотрим итератор, присоединенный к массиву (хотя аналогичное поведение будет демострировать любой выбранный
нами итератор):

    var a = [1,2,3,4,5];

Опратор разделения `...` исчерпывает итератор полносью. Например:

    function foo(x,y,z,w,p) {
        console.log( x + y + z + w + p );
    }
    
    foo( ...a );                // 15

Кроме того, можно распределить итератор внутри массива:

    var b = [ 0, ...a, 6 ];
    b;                          // [0,1,2,3,4,5,6]

При процедуре деструктуризации массива итератор может использоваться как частично, так и полностью (в сочетании с
*rest/gather-оператором* `...`).

    var it = a[Symbol.iterator]();
    
    var [x,y] = it;                 // берем из 'it' только первые два элемента
    var [z, ...w] = it;             // берем третий элемент, а затем сразу все остальные
    
    // истощен ли 'it' полносью? Да
    it.next();                      // { value: undefined, done: true }
    
    x;                              // 1
    y;                              // 2
    z;                              // 3
    w;                              // [4,5]

___

### Генераторы

Все функции работают до своего завершения, верно? Другими словами, запущенная функция заканчивает работу до того, как
начнет выполнятся другая операция.

По крайней мере, именно так обстояли дела на протяжении всей истории существования языка JavaScript. Но в ES6 появилась
новая, несколько необычная форма функции, названная генератором. Такая функция может остановиться во время выполнения,
а затем продолжить работу с прерванного места.

Более того, каждый цикл остановки и возобновления работы позволяет передавать сообщения в обе стороны. Как генератор
может вернуть значение, так восстанавливающий его работу управляющий код может переслать туда что-нибудь.

Так же, как в случае с итераторами, рассмотренными в предыдущем разделе, на генераторы, вернее на то, для чего они
в основном предназначены, можно смотреть с разных сторон. Единственной верной точки зрения не существует.

#### Синтаксис

Функция-генератор объявляется следующим образом:

    function *foo() {
        // ..
    }

Положение звездочки `*` функционально несущественно. То же самое объявление можно написать любым из следующих способов:

    function *foo()  { .. }
    function* foo()  { .. }
    function * foo() { .. }
    function*foo()   { .. }
    ..

Кроме того, существует краткая форма генератора в объектных литералах:

    var a = {
        *foo() { .. }
    };

#### Выполнение генератора

Хотя генератор объявляется с символом `*`, вызывается он как обычная функция:

    foo();

В него можно передавать аргументы:

    function *foo(x,y) {
        // ..
    }
    
    foo( 5, 10 );

Основное отличие состоит в том, что запуск генератора, например `foo(5,10)`, не приводит к исполнению его кода.
Вместо этого создается итератор, контролирующий то, как генератор исполняет свой код.

    function *foo() {
        // ..
    }
    
    var it = foo();
    
    // чтобы начать/продолжить выполнение '*foo()',
    // вызываем 'it.next(..)'

#### Ключевое слово yield

Внутри генераторов используется ключевое слово, сигнализирующее о прерывании работы: `yield`. Рассмотрим пример:

    function *foo() {
        var x = 10;
        var y = 20;
    
        yield;
    
        var z = x + y;
    }

В этом генераторе `*foo()` сначала запускаются операции из первых двух строк, а затем ключевое слово `yield`
останавливает работу. После ее возобновления запускается последняя строчка генератора `*foo()`. Ключевое слово
`yield` может появляться в генераторе произвольное количество раз (или вообще отсутствовать).

Ничто не мешает поместить ключевое слово `yield` в тело цикла, создав повторяющуюся точку останова. При этом в
случае с бесконечным циклом вы получите генератор, работа которого никогда не завершается, что иногда бывает
вполне допустимо и даже необходимо.

Ключевое слово `yield` не просто прерывает работу генератора. В момент остановки оно посылает наружу значение.
Вот пример цикла `while..true` внутри генератора, который на каждой итерации получает новое случайное число:

    function *foo() {
        while (true) {
            yield Math.random();
        }
    }

Выражение `yield..` позволяет не только передать значение (ключевое слово `yield`, не сопровождающее ничем, означает
`yield undefined`), но и получить - то есть заместить - конечное значение при возобновлении работы генератора.
Рассмотрим пример:

    function *foo() {
        var x = yield 10;
        console.log( x );
    }

Этот генератор в момент остановки сначала получит значение 10. После возобновления работы - методом
`it.next(..)` - любое восстановленное значение (если такое вообще существует) полностью заместит выражение
`yield 10`, то есть именно оно будет присвоено переменной `x`.

Выражение `yield..` может появляться во всех тех местах, куда вставляют обычные выражения. Например:

    function *foo() {
        var arr = [ yield 1, yield 2, yield 3 ];
        console.log( arr, yield 4 );
    }

В данном случае генератор `*foo()` содержит четыре записи `yield..`. Каждая из них останавливает работу генератора и
ожидает некого нового значения, которое затем вставляется в различные контексты выражения.

Строго говоря, слово `yield` - не оператор, хотя выражения вида `yield 1` выглядят именно как операторы. Тем не менее
оно используется и само по себе, например `var x = yield;`, поэтому его представление в качестве оператора может
привести к путанице.

С технической точки зрения выражение `yield..` имеет такой же приоритет (концептуально в данном случае это
соответствует приоритету операторов), как и выражение присваивания, например `a = 3`. Таким образом, `yield..`
может появляться везде, где допустимо выражение `a = 3`.

Проиллюстрируем эту аналогию:

    var a, b;
    
    a = 3;                      // valid
    b = 2 + a = 3;              // invalid
    b = 2 + (a = 3);            // valid
    
    yield 3;                    // valid
    a = 2 + yield 3;            // invalid
    a = 2 + (yield 3);          // valid

Если как следует плдумать, то в одинаковом поведении выражения `yield..` и выражения присваивания есть концептуальный
смысл. Когда после остановки генератор возобновляет работу, выражение с ключевым словом `yield`
завершается/замещается значением восстановления, что фактически аналогично присваиванию последнего.

Если нужно вставить `yield..` в место, где недопустимы такие выражения, как `a = 3`, его помещяют в скобки `()`.

Из-за низкого приоритета ключевого слова `yield` практически все выражения, следующие за `yield..`, будут
вычисляться раньше. Более низким приоритетом обладают только оператор распределения `...` и запятая `,`, до них
дело доходит после вычисления выражения с `yield`.

Как и в случае с набором обычных операторов, есть еще один вариант применения скобок `()`. Они позволяют переопределить
(поднять) низкий приоритет ключевого слова `yield`, как в случае с этими двумя выражениями:

    yield 2 + 3;			// то же самое, что и `yield (2 + 3)`
    
    (yield 2) + 3;			// сначала `yield 2`, затем `+ 3`

Подобно оператору присваивания `=`, ключевое слово `yield` имеет правую ассоциативность. Это означает, что набор
последовательных выражений `yield` рассматривается как сгруппированный с помощью скобок `(..)` справа налево.
То есть `yield yield yield 3` трактируется как `yield (yield (yield 3))`. Трактовка с левой ассоциативностью,
то есть `((yield) yield) yield 3` не имеет смысла.

При комбинации ключевого слова `yield` с какими-либо операторами или с другими ключевыми словами `yield` разумно
прибегнуть к группировке с помощью скобок `(..)`, чтобы четко обозначить свои намерения (как вы помните, с другими
операторами дело обстоит так же).

#### Выражение yield*

Аналогично тому, как символ `*` превращает объявление функции в объявление генератора, в случае использования `*` с
ключевым словом `yield` образуется совершенно другой механизм, называемый *yield-делегированием*. Грамматичестки
выражение `yield *..` будет вести себя так же, как и рассмотренное в предыдущем разделе `yield ..`.

Выражение `yield *..` требуется итерируемый объект; оно вызывает его итератор и передает ему управление собственным
генератором. Рассмотрим пример:

    function *foo() {
        yield *[1,2,3];
    }

Значение `[1,2,3]` даст нам итератор, который будет пошагово передавать свои значения генератору `*foo()`. Другой
способ продемонстрировать такое поведение - сделать yield-делегирование другому генератору:

    function *foo() {
        yield 1;
        yield 2;
        yield 3;
    }
    
    function *bar() {
        yield *foo();
    }

Итератор, появившийся при вызове генератором `*bar()` генератора `*foo()`, делегируется при помощи выражения `yield *`.
Это означает, что все значения, порождаемые генератором `*foo()`, будут выводиться генератором `*bar()`.

Если заверщающее значение выражения `yield ..` появляется при возобновлении работы генератора методом `it.next(..)`,
то завершающее значение выражения `yield *..` представляет собой значение, возвращаемое итератором, которому были
делегированы полномочия (если таковое существует).

У встроенных итераторов возвращаемое значение, как правило, отсутствует, как было показано в конце раздела
"Цикл итератора". Но собственноручно созданный вами итератор (или генератор) можно заставить возвращать значение,
которым в конечном итоге и воспользуется выражение `yield *..`:

    function *foo() {
        yield 1;
        yield 2;
        yield 3;
        return 4;
    }
    
    function *bar() {
        var x = yield *foo();
        console.log( "x:", x );
    }
    
    for (var v of bar()) {
        console.log( v );
    }
    // 1 2 3
    // x: { value: 4, done: true }

Если значения 1, 2 и 3 отданы генератором `*foo()`, а затем генератором `*bar()`, то возвращенное генератором `*foo()`
значение 4 является завершающим для выражения `yield *foo()` и присваивается переменной `x`.

Выражение `yield *` может не только вызвать еще один генератор (путем делегирования его итератору), но и породить
рекурсию генератора, вызывая само себя:

    function *foo(x) {
        if (x < 3) {
            x = yield *foo( x + 1 );
        }
        return x * 2;
    }
    
    foo( 1 );

В результате `foo(1)` и последующего вызова метода `next()` итератора для прохода через этапы рекурсии мы получим 24.
При первом запуске генератора `*foo()` переменная `x` имеет значение 1, то есть соблюдается условие `x < 3`.
Выражение `x + 1` рекурсивно передается генератору `*foo(..)`, соответственно, `x` обретает значение 2. Следующий
рекурсивный вызов дает переменной `x` значение 3.

После этого условие `x < 3` перестает выполняться, и рекурсия прекращается, а оператор `return 3 * 2` возвращает
выражению `yield *..` из предыдущего вызова значение 6, которое в свою очередь, присваивается переменной `x`.
Следующий оператор `return 6 * 2` возвращает предыдущему вызову `x` значение 12. Наконец, после завершающего этапа
работы генератора `*foo(..)` возвращается значение `12 * 2`, то есть 24.

#### Контроль со стороны итератора

Выше уже было сказано, что генераторы управляются итераторами. Давайте подробно разберем этот процесс.

Для примера рассмотрим рекурсивную форму генератора `*foo(..)` из предыдущего раздела. Вот как она действует:

    function *foo(x) {
        if (x < 3) {
            x = yield *foo( x + 1 );
        }
        return x * 2;
    }
    
    var it = foo( 1 );
    it.next();				// { value: 24, done: true }

В этом случае генератор вообще не останавливает свою работу, так как выражение `yield ..` отсутствует. Вместо него
в коде имеется выражение `yield *`, которое обеспечивает прохождение каждой итерации путем рекурсивного вызова.
Так что работа генератора осуществляется исключительно вызовами функции `next()` итератора.

Теперь рассмотрим генератор, имеющий несколько этапов и соответственно дающий несколько значений:

    function *foo() {
        yield 1;
        yield 2;
        yield 3;
    }

Мы уже знаем, что воспользоваться итератором, даже если тот присоединен к генератору `*foo()`, можно с помощью
цикла `for..of`:

    for (var v of foo()) {
        console.log( v );
    }
    // 1 2 3

Для цикла `for..of` необходим итерируемый объект. Сама по себе ссылка на функцию генератора (например, `foo`)
таковым считаться не может; для получения итератора ее следует выполнить как `foo()` (при этом, как говорилось выше,
такой итератор сам является итерируемым). Теоретически можно расширить `GeneratorPrototype` (прототип всех функций
генератора) функцией `Symbol.iterator`, которая, по сути, всего лишь возвращает `this()`. Это сделает ссылку `foo`,
итерируемой, что обеспечит работу цикла `for (var v of foo) { .. }` (обратите внимание на отсутствие скобок `()` у
функуии `foo`).

Попробуем выполнить итерации генератора вручную:

    function *foo() {
        yield 1;
        yield 2;
        yield 3;
    }
    
    var it = foo();
    
    it.next();				// { value: 1, done: false }
    it.next();				// { value: 2, done: false }
    it.next();				// { value: 3, done: false }
    
    it.next();				// { value: undefined, done: true }

Мы видим, что на три оператора `yield` приходится четыре вызова метода `next()`. Такое несовпадение может показаться
странным. Однако количество вызовов метода `next()` всегда на единицу превышает количество выражений `yield`, тем
самым давая возможность сделать все вычисления и позволить генератору отработать до конца.

Впрочем, если смотреть с другой стороны (изнутри, а не снаружи), более осмысленным кажется равное число `yield` и
`next()`.

Напомню, что выражение `yield ..` завершается значением, которое используется в момент возобновления работы генератора.
Таким образом, передаваемый в метод `next(..)` аргумент завершает выражение, приостановленное в текущий момент.

Проиллюстрируем это следующим фрагментом кода:

    function *foo() {
        var x = yield 1;
        var y = yield 2;
        var z = yield 3;
        console.log( x, y, z );
    }

Здесь каждое выражение `yield ..` посылает значение (1, 2, 3), точнее, останавливает генератор, чтобы он дождался
значения. Это напоминает обращение: "Дай знать, что мне тут следует использовать, я пока подожду".

Вот таким способом мы заставляем генератор `*foo()` начать работу:

    var it = foo();
    
    it.next();				// { value: 1, done: false }

Первый вызов метода `next()` пробуждает генератор из начального состояния и доводит его до первого ключевого слова
`yield`. На этот момент у нас еще нет ожидающего завершения выражения `yield ..`. Если передать методу `next()`
какое-либо значение, оно будет выброшено, так как никто не ждет его.

Попробуем ответить на пока еще открытый вопрос: "Какое значение следует присвоить переменной `x`?" Для этого отправим
значение следующему вызываемому методу `next(..)`:

    it.next( "foo" );		// { value: 2, done: false }

Теперь переменная `x` имеет значение `"foo"`, но возникает новый вопрос: "Что присвоить переменной `y`?"
Вот что мы ответим:

    it.next( "bar" );		// { value: 3, done: false }

Ответ дан, следующий вопрос сформулирован. Окончательный ответ будет таким:

    it.next( "baz" );       // "foo" "bar" "baz"
                            // { value: undefined, done: true }

Теперь, когда стало видно, что на каждый вопрос выражения `yield ..` отвечает следующий вызов метода `next(..)`,
легко понять, что наблюдаемый нами "лишний" вызов метода `next()` - это самая первая операция, которая запускает
работу генератора.

Объеденим все шаги:

    var it = foo();
    
    // запускает генератор
    it.next();                      // { value: 1, done: false }
    
    // отвечает на первый вопрос
    it.next( "foo" );               // { value: 2, done: false }
    
    // отвечает на второй вопрос
    it.next( "bar" );               // { value: 3, done: false }
    
    // отвечает на третий вопрос
    it.next( "baz" );               // "foo" "bar" "baz"
                                    // { value: undefined, done: true }

Генератор можно представить как поставщик значений, в этом случае каждая итерация просто создает значение для
дальнейшего использования.

Но в более общем смысле, возможно, корректнее будет рассматривать генераторы как управляемое поэтапное выполнение
кода, во многом напоминающее очередь задач.

#### Раннее завершение

Как уже упоминалось, присоединенный к генератору итератор поддерживает необязательные методы `return(..)` и
`throw(..)` - оба немедленно прерывают работу приостановленного генератора.

Рассмотрим пример:

    function *foo() {
        yield 1;
        yield 2;
        yield 3;
    }
    
    var it = foo();
    
    it.next();		        // { value: 1, done: false }
    
    it.return( 42 );		// { value: 42, done: true }
    
    it.next();		        // { value: undefined, done: true }

Метод `return(x)` как бы вынуждает немедленно выполнить оператор `return x`, и вы сразу получаете указанное значение.
Генератор, чья работа была завершена обычным образом или преждевременно, как показано в примере, больше ничего не
делает ни с каким кодом и не возвращает никаких значений.

Метод `return(..)` может вызываться не только вручную, но и автоматически в конце итераций. В последнем случае его
вызывает компонент, использующий итератор, например цикл `for..of` или оператор разделения `...`.

Эта функциональная особенность позволяет уведомить генератор, что контролирующий код прекратил выполнять перебор и
можно приступить к задачам, связанным с очисткой (освобождением ресурсов, сбросом состояния и т.п.). Данная задача,
аналогично обычному шаблону очистки функции, в основном решается с помощью оператора `finally`:

    function *foo() {
        try {
            yield 1;
            yield 2;
            yield 3;
        }
        finally {
            console.log( "cleanup!" );
        }
    }
    
    for (var v of foo()) {
        console.log( v );
    }
    // 1 2 3
    // очистка!
    
    var it = foo();
    
    it.next();              // { value: 1, done: false }
    it.return( 42 );        // очистка!
                            // { value: 42, done: true }

Помещать оператор `yield` внутрь оператора `finally` нельзя! Формально такое допустимо, но делать этого не стоит. В
определенном смысле `finally` действует как отложенное завершение вызванного вами метода `return(..)`, потому что
любое выражение `yield ..` в операторе `finally` будет прерывать его работу и отправлять сообщения; вы не получите
немедленно завершающийся генератор, как ожидаете. Нет разумных причин использовать настолько сумасшедший вариант,
поэтому избегайте его!

Представленный выше фрагмент не только показывает, как метод `return(..)` прерывает работу генератора, запуская
оператор `finally`, но и демонстрирует, что при каждом вызове генератор формирует новый итератор. Более того, вы
можете использовать несколько итераторов, одновременно присоедененных к одному и тому же генератору:

    function *foo() {
        yield 1;
        yield 2;
        yield 3;
    }
    
    var it1 = foo();
    it1.next();				// { value: 1, done: false }
    it1.next();				// { value: 2, done: false }
    
    var it2 = foo();
    it2.next();				// { value: 1, done: false }
    
    it1.next();				// { value: 3, done: false }
    
    it2.next();				// { value: 2, done: false }
    it2.next();				// { value: 3, done: false }
    
    it2.next();				// { value: undefined, done: true }
    it1.next();				// { value: undefined, done: true }

#### Раннее прерывание

Вместо метода `return(..)` можно вызвать метод `throw(..)`. Аналогично тому, как метод `return(x)`, по сути,
представляет собой оператор `return x`, вставленный в текущую точку останова генератора, вызов метода `throw(x)`
вставляет туда оператор `throw x`.

Помимо генерации исключений, метод `throw(..)` выполняет раннее завершение, прерывая работу генератора в текущей
точке останова. Например:

    function *foo() {
        yield 1;
        yield 2;
        yield 3;
    }
    
    var it = foo();
    
    it.next();                      // { value: 1, done: false }
    
    try {
        it.throw( "Oops!" );
    }
    catch (err) {
        console.log( err );         // Исключение: Ой!
    }
    
    it.next();                      // { value: undefined, done: true }

Так как метод `throw(..)` по сути, вставляет вместо строчки `yield 1` оператор `throw ..`, и при этом обработчик
исключения отсутствует, исключение немедленно возвращается к вызывающему коду, который тспользует блок `try..catch`.

В отличии от метода `return(..)`, метод итератора `throw(..)` автоматически никогда не вызывается.

В представленном фрагменте кода этого не показано, но если блок `try..finally` в момент вызова метода `throw(..)`
ожидал внутри генератора, оператор `finally` получает шанс завершить работу до момента, когда исключение вернется
в вызывающий код.

#### Обработка ошибок

Я уже упоминал, что в случае с генераторами обработка ошибок реализуется при помощи блока `try..catch`, который
работает как во входящем, так и в исходящем направлении.

    function *foo() {
        try {
            yield 1;
        }
        catch (err) {
            console.log( err );
        }
    
        yield 2;
    
        throw "Hello!";
    }
    
    var it = foo();
    
    it.next();                  // { value: 1, done: false }
    
    try {
        it.throw( "Hi!" );      // Hi!
                                // { value: 2, done: false }
        it.next();
    
        console.log( "never gets here" );
    }
    catch (err) {
        console.log( err );	    // Hello!
    }

Ошибки также могут распространяться в обоих направлениях через делегат `yield *`:

    function *foo() {
        try {
            yield 1;
        }
        catch (err) {
            console.log( err );
        }
    
        yield 2;
    
        throw "foo: e2";
    }
    
    function *bar() {
        try {
            yield *foo();
    
            console.log( "never gets here" );
        }
        catch (err) {
            console.log( err );
        }
    }
    
    var it = bar();
    
    try {
        it.next();              // { value: 1, done: false }
    
        it.throw( "e1" );       // e1
                                // { value: 2, done: false }
    
        it.next();              // foo: e2
                                // { value: undefined, done: true }
    }
    catch (err) {
        console.log( "never gets here" );
    }
    
    it.next();                  // { value: undefined, done: true }

Вы уже видели, что когда генератор `*foo()` вызывает оператор `yield 1`, значение 1 проходит через генератор
`*bar()` без изменений.

Впрочем, самое интересное в этом фрагменте то, что при вызове генератором `*foo()` строчки `throw "foo: e2"` ошибка
переходит в генератор `*bar()` и немедленно перехватывается вставленным туда блоком `try..catch`. Ей не удается
беспрепятственно пройти через `*bar()`, как это делает значение 1.

Оператор `catch` внутри генератора `*bar()` выполняет обычный вывод `err ("foo: e2")`, после чего генератор обычным
образом завершает свою работу. Вот почему от метода `it.next()` приходит результат итератора
`{ value: undefined, done: true }`.

Если бы в `*bar()` возле выражения `yield *..` отсутствовал блок `try..catch`, ошибка, разумеется, проходила бы
генератор насквозь, и при этом все равно требовалось бы завершить его работу.

#### Применение генераторов

Теперь, когда вы хорошо понимаете, как работают генераторы, поговорим о том, зачем они нужны.

Как правило они используются в одном из двух следующих случаев.

##### `Создание набора значений`

Этот вариант применения может быть как простым (например, случайные строки или возрастающие числа), так и дающим
более структурированный доступ к данным (например, итерации по строкам, которые вернул запрос к базе данных).

В любом случае мы контролируем генератор с помощью итератора, что дает возможность реализовывать некую логическую
схему при каждом вызове метода `next(..)`. Обычные итераторы, работающие со структурами данных, просто
извлекают оттуда значения, не добавляя никакой управляющей логической схемы.

##### `Очередь задач для последовательного выполнения`

Этот вариант применения часто представляет собой управление шагами какого-нибудь алгоритма, причем на каждом из них
требуется извлекать данные из некого внешнего источника - немедленно или с асинхронной задержкой.

С точки зрения кода внутри генератора детали реализации синхронного или асинхронного извлечения данных в точке
оператора `yield` совершенно непрозрачны. Более того, они намеренно сделаны абстрактными, чтобы не прятать
естественное последовательное выражение шагов за сложностями реализации. Кроме того, абстрагирование дает
возможность часто менять или перерабатывать реализацию, никак не затрагивая код внутри генератора.

Если смотреть на генераторы с точки зрения их применения, они перестают быть всего лишь красивым синтаксисом для
конечного автомата с ручным управлением. Генераторы - это мощный инструмент абстрагирования для систематизации и
контроля за упорядоченным созданием и использованием данных.