# Без Макросов: синтаксис строителя View

> Если вы и так счастливы с синтаксисом макросом `view!` описываемым до сих пор, мы можете пропустить эту главу.
>  синтаксис описанный в этом разделе всегда доступен, но никогда не обязателен.

По тем или иным причинам, многие разработчики предпочитают избегать макросов. Быть может вам не нравится ограниченная
поддержка со стороны `rustfmt`. (Хотя. вам стоит посмотреть на [`leptosfmt`](https://github.com/bram209/leptosfmt), это прекрасный инструмент!)
А может, вы беспокоитесь по поводу влияния макросов на время компиляции. Может быть, вы предпочитаете эстетику 
чистого синтаксиса Rust, или переключение контекста между HTML-подобным синтаксисом и вашим Rust кодом вызывает у вас трудности.
Или может вы хотите больше гибкости в том как вы создаёте и манипулируете HTML элементами, чем даёт макрос `view`.

Если вы относитесь к одной из этих категорий, синтаксис строителя может вам подойти.

Макрос `view` преобразует HTML-подобный синтаксис в набор Rust функций и вызовов методов. Если вы бы предпочли
не использовать макрос `view`, вы можете просто использовать этот выходной синтаксис сами. А он весьма хорош!

Во-первых, если хотите, вы можете даже убрать макрос `#[component]`: компонент это просто установочная функция,
создающая ваш `view`, так что вы можете просто объявить компонент как простой вызов функции:

```rust
pub fn counter(initial_value: i32, step: u32) -> impl IntoView { }
```

Элементы создаются путём вызова функции одноименной с создаваемым элементом HTML:

```rust
p()
```

Вы можете добавить дочерние элементы с помощью [`.child()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.child), который принимает одиночный элемент, массив 
или кортеж с типами, реализующими [`IntoView`](https://docs.rs/leptos/latest/leptos/trait.IntoView.html).

```rust
p().child((em().child("Big, "), strong().child("bold "), "text"))
```

Атрибуты добавляются при помощи метода [`.attr()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.attr). Он может принимать любые типы, которые можно передавать в 
качестве атрибута внутри макроса `view` (типы, реализующие [`IntoAttribute`](https://docs.rs/leptos/latest/leptos/trait.IntoAttribute.html)).

```rust
p().attr("id", "foo").attr("data-count", move || count().to_string())
```

Аналогично, `class:`, `prop:`, и `style:` синтаксисы соотносятся с метлдами [`.class()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.class), [`.prop()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.prop), и [`.style()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.style).

Слушатели событий могут быть добавлены через [`.on()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.on). Типизированные события из [`leptos::ev`](https://docs.rs/leptos/latest/leptos/ev/index.html) предотвращают опечатки
в названиях событий и обеспечивают правильный вывод типов _(англ. type inference)_ в функции обратного вызова.

```rust
button()
    .on(ev::click, move |_| set_count.update(|count| *count = 0))
    .child("Clear")
```

> Много дополнительных методов можно найти в документации к [`HtmlElement`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.child),
> включая некоторые методы, недоступные напрямую в макросе `view`.

Всё это складывается в очень Rust'овый синтаксис для построения полнофункциональных представлений, если вы предпочитаете этот стиль.

```rust
/// A simple counter view.
// A component is really just a function call: it runs once to create the DOM and reactive system
pub fn counter(initial_value: i32, step: u32) -> impl IntoView {
    let (count, set_count) = create_signal(0);
    div().child((
        button()
            // typed events found in leptos::ev
            // 1) prevent typos in event names
            // 2) allow for correct type inference in callbacks
            .on(ev::click, move |_| set_count.update(|count| *count = 0))
            .child("Clear"),
        button()
            .on(ev::click, move |_| set_count.update(|count| *count -= 1))
            .child("-1"),
        span().child(("Value: ", move || count.get(), "!")),
        button()
            .on(ev::click, move |_| set_count.update(|count| *count += 1))
            .child("+1"),
    ))
}
```

У этого также есть преимущество большей гибкости: поскольку всё это простые Rust функции и методы, 
их проще использовать в таких вещах, как адаптеры итераторов без какой-либо дополнительной "магии':
 it’s easier to use them in things like iterator adapters without any additional “magic”:

```rust
// take some set of attribute names and values
let attrs: Vec<(&str, AttributeValue)> = todo!();
// you can use the builder syntax to “spread” these onto the
// element in a way that’s not possible with the view macro
let p = attrs
    .into_iter()
    .fold(p(), |el, (name, value)| el.attr(name, value));

```

> ## Примечание о производительности

>
> Одно предостережение: макрос `view` применяет значительные оптимизации в режиме сервере рендеринга (SSR), чтобы
> значительно улучшить производительность рендеринга HTML (в 2-4 раза быстрее, зависит от характеристик приложения).
> Он делает это путём анализа вашего `view` во время компиляции и превращения статических частей в простые HTML строки,
> вместо синтаксиса строителя.
>
> Из этого следуют две вещи:
>
> 1. Синтаксис строителя и макрос `view` не стоит мешать вовсе или мешать, но очень осторожно: по меньшей мере в режиме SSR
> c выводом макроса `view` стоит обходиться как с "черным ящиком", к которому нельзя применять дополнительные методы строителя 
> не вызывая несоответствий.
> 2. Использование синтаксиса строителя выльется в производительность SSR ниже оптимальной. 
> Медленно не будет ни в коем случае (всё равно стоит сделать собственные замеры производительные), но медленнее
> чем версия оптимизированная макросом `view`.
