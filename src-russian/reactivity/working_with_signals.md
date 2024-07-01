# Работа с Сигналами

Пока что мы использовали простые примеры с [`create_signal`](https://docs.rs/leptos/latest/leptos/fn.create_signal.html), возвращающие геттер [`ReadSignal`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html) и сеттер [`WriteSignal`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html).

## Чтение и запись данных

Есть четыре простые операции с сигналами:

1. [`.get()`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html#impl-SignalGet%3CT%3E-for-ReadSignal%3CT%3E) клонирует текущее значение сигнала и реактивно отслеживает будущие изменения.
2. [`.with()`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html#impl-SignalWith%3CT%3E-for-ReadSignal%3CT%3E) принимает функцию, получающую текущее значеие сигнала по ссылке (`&T`) и отслеживает будущие изменения.
3. [`.set()`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html#impl-SignalSet%3CT%3E-for-WriteSignal%3CT%3E) заменяет текущее значение сигнала и уведомляет об этом подписчиков.
4. [`.update()`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html#impl-SignalUpdate%3CT%3E-for-WriteSignal%3CT%3E) принимает функцию, которая принимает мутабельную ссылку на текущее значение сигнала (`&mut T`) 
и уведомляет подписчиков. (`.update()` не возвращает значение возвращённое замыканием;
для этого можно использовать [`.try_update()`](https://docs.rs/leptos/latest/leptos/trait.SignalUpdate.html#tymethod.try_update), если, например, вы удаляете элемент из `Vec<_>` и вам нужен удалённый элемент.)

Вызов `ReadSignal` как функции это синтаксический сахар для`.get()`. Вызов `WriteSignal`как функции это синтаксический сахар для `.set()`.
Так что

```rust
let (count, set_count) = create_signal(0);
set_count(1);
logging::log!(count());
```

это то же самое, что

```rust
let (count, set_count) = create_signal(0);
set_count.set(1);
logging::log!(count.get());
```

Можно заметить, что и `.get()` и `.set()` могут быть реализованы при помощи  `.with()` и `.update()`.
Другими словами, `count.get()` идентично `count.with(|n| n.clone())`, а `count.set(1)` реализуется посредством `count.update(|n| *n = 1)`.

Но конечно же, `.get()` и `.set()` (или простые вызовы функций!) это намного более приятный синтаксис.

Однако, и для `.with()` с `.update()` есть очень хорошее применение.

Например, возьмём сигнала с типом `Vec<String>`.

```rust
let (names, set_names) = create_signal(Vec::new());
if names().is_empty() {
	set_names(vec!["Alice".to_string()]);
}
```

С точки зрения логики, этот пример достаточно прост, но в нём скрываются существенные недостатки, влияющие на производительности.    
Помните, что `names().is_empty()` это сахар для `names.get().is_empty()`, который клонирует значение (it’s `names.with(|n| n.clone()).is_empty()`).
Это значит мы клонируем значение  `Vec<String>` целиком, выполняем `is_empty()`, и тут же выбрасываем клонированное значение.

Аналогично и `set_names` заменяет значение новым  `Vec<_>`. В общем-то ничего страшного, но мы можем просто мутировать исходный `Vec<_>` там где он лежит.

```rust
let (names, set_names) = create_signal(Vec::new());
if names.with(|names| names.is_empty()) {
	set_names.update(|names| names.push("Alice".to_string()));
}
```

Теперь наша функция просто принимает `names` по ссылке и запускает `is_empty()`, избегая клонирования.

Если вы включили Clippy или у вас острый глаз, мы могли заметить, что мы можем сделать ещё лучше:

```rust
if names.with(Vec::is_empty) {
	// ...
}
```

В конце концов, `.with()` просто принимает функцию, которая принимает значение по ссылке.
Поскольку `Vec::is_empty` принимает `&self`, мы можем передать её напрямую и избавиться от ненужного замыкания.

Есть вспомогательные макросы, упрощающие использование  `.with()` и `.update()`, особенно при работе с несколькими сигналами сразу.

```rust
let (first, _) = create_signal("Bob".to_string());
let (middle, _) = create_signal("J.".to_string());
let (last, _) = create_signal("Smith".to_string());
```

Если вы хотите конкатенировать эти 3 сигнала вместе без ненужного клонирования, вам пришлось бы написать что-то вроде:

```rust
let name = move || {
	first.with(|first| {
		middle.with(|middle| last.with(|last| format!("{first} {middle} {last}")))
	})
};
```

Очень длинно и писать неприятно.

Вместо этого можно использовать макрос  `with!`, чтобы получить ссылки на все эти сигналы сразу.

```rust
let name = move || with!(|first, middle, last| format!("{first} {middle} {last}"));
```

Это превращается в то, что было выше. Посмотрите документацию к [`with!`](https://docs.rs/leptos/latest/leptos/macro.with.html) для дополнительной информации, 
и макросам [`update!`](https://docs.rs/leptos/latest/leptos/macro.update.html), [`with_value!`](https://docs.rs/leptos/latest/leptos/macro.with_value.html) и [`update_value!`](https://docs.rs/leptos/latest/leptos/macro.update_value.html).

## Делаем так, чтобы сигналы зависели друг от друга

Люди часто спрашивают о ситуациях, когда какой-то сигнал должен меняться в зависимости от значения другого сигнала.
Для этого есть три хороших способа и один неидеальный, но сносный в контролируемых обстоятельства.

### Хорошие варианты

**1) Б это функция от А.** Создадим сигнал для А и производный сигнал или memo для Б.

```rust
let (count, set_count) = create_signal(1);
let derived_signal_double_count = move || count() * 2;
let memoized_double_count = create_memo(move |_| count() * 2);
```

> Рекомендации о том как выбирать между производным сигналом и memo можно найти в документации к [`create_memo`](https://docs.rs/leptos/latest/leptos/fn.create_memo.html)

**2) В это функция от А и Б.** Создадим сигналы для А и Б, а также производный сигнал или memo для В.

```rust
let (first_name, set_first_name) = create_signal("Bridget".to_string());
let (last_name, set_last_name) = create_signal("Jones".to_string());
let full_name = move || with!(|first_name, last_name| format!("{first_name} {last_name}"));
```

**3) А и Б — независимые сигналы, но иногда они обновляются одновременно..** Когда вы обновляете A, отдельно вызывайте и обновление Б.

```rust
let (age, set_age) = create_signal(32);
let (favorite_number, set_favorite_number) = create_signal(42);
// use this to handle a click on a `Clear` button
let clear_handler = move |_| {
  set_age(0);
  set_favorite_number(0);
};
```

### Если без этого никак...

**4) Создайте эффект, чтобы писать в Б каждый раз когда А меняется.** Так делать официально не рекомендуется по нескольким причинам:
a) Это всегда будет более затратно, так как это значит, что каждый раз, когда А обновляется, будет два полных прохода через реактивный процесс.  
(Вы устанавливаете значение А, это вызывает запуск этого эффекта, наряду с остальными, зависящими от А. Затем вы меняете Б, что вызовет выполнение всех зависящих от Б эффектов.)
b) Это повышает вероятность нечаянно сделать бесконечный цикл или эффекты, выполняющиеся слишком часто.
Это тот самый пинг-понг, происходивший в реактивном спагетти-коде в начале 2010-х годов, которого мы стараемся избежать
посредством разделения чтения и записи, а также не рекомендуя писать в сигналы из эффектов. 

В большинстве ситуаций лучше переписать всё таким образом, чтобы был чёткий поток данных сверху вниз,
основанный на производных сигналах или мемоизированных значениях. Но это не конец света.

> Я намеренно не привожу здесь пример кода. Прочтите документацию к [`create_effect`](https://docs.rs/leptos/latest/leptos/fn.create_effect.html) 
> понять как это должно работать.
