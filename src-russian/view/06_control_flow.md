# Порядок выполнения

В большинстве приложений вам порой нужно принять решение: Мне рендерить это как часть view или нет? 
Мне рендерить  `<ButtonA/>` или `<WidgetB/>`. Это и есть **порядок выполнения** (_англ. control flow_).

## Несколько советов

Думая о том, как сделать это в Leptos, важно помнить несколько вещей:

1. Rust это язык ориентированный на выражения: такие выражения порядка выполнения, как
   `if x() { y } else { z }` и `match x() { ... }` возвращают свои значения. Это делает их очень полезными для декларативных 
пользовательских интерфейсов.
2. Для любого `T` реализующего `IntoView` — другими словами, для любого типа, который Leptos может рендерить
— `Option<T>` и `Result<T, impl Error>` _тоже_ реализуют `IntoView`. И так же как `Fn() -> T` рендерит реактивный `T`, `Fn() -> Option<T>`
   и `Fn() -> Result<T, impl Error>` тоже реактивны.
3. В Rust есть множество полезных хелперов, таких как [Option::map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map),
   [Option::and_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then),
   [Option::ok_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or),
   [Result::map](https://doc.rust-lang.org/std/result/enum.Result.html#method.map),
   [Result::ok](https://doc.rust-lang.org/std/result/enum.Result.html#method.ok), и
   [bool::then](https://doc.rust-lang.org/std/primitive.bool.html#method.then), позволяющих вам декларативно конвертировать значения между несколькими стандартными типами, все из которых 
могут быть отрендерены. Потратить время на чтение документации к `Option` и `Result` это один из лучших способов
выйти на новый уровень в Rust.
4. И всегда помните: чтобы быть реактивными, значения должны быть функциями. Ниже вы увидите как я постоянно оборачиваю вещи в
`move ||` замыкания. Это чтобы убедиться, что они действительно выполняются повторно когда меняется сигнал, от которого они зависят,
поддерживая реактивность UI.

## Ну и что?

Чтобы немного собрать всё воедино: на практике это значит, что большую часть контроля над
порядком выполнения можно реализовать при помощи нативного Rust кода, без особых компонентов,
управляющих порядком выполнения и не обладая специальными знаниями.    

К примеру, давайте начнём с простого сигнала и производного сигнала:

```rust
let (value, set_value) = create_signal(0);
let is_odd = move || value() & 1 == 1;
```

> Если вы не понимаете что происходит с `is_odd`, то не волнуйтесь. Это лишь простой способ 
> проверить число на нечётность, используя побитовое «И» с 1.

Мы можем использовать эти сигналы и обычный Rust чтобы реализовать большую часть управления порядком выполнения.

### Оператор `if`

Предположим, я хочу выводить одну надпись если число нечётно и другую если оно чётно. Что скажете вот об этом?

```rust
view! {
    <p>
    {move || if is_odd() {
        "Odd"
    } else {
        "Even"
    }}
    </p>
}
```

Выражение `if` возвращает своё значение, а `&str` реализует `IntoView`, поэтому
`Fn() -> &str` реализует `IntoView`, так что это... просто работает!

### `Option<T>`

Предположим мы хотим выводить надпись если число нечётно и ничего не выводить, если чётно.

```rust
let message = move || {
    if is_odd() {
        Some("Ding ding ding!")
    } else {
        None
    }
};

view! {
    <p>{message}</p>
}
```

Это отлично работает. Мы можем, как вариант, немного сократить код, используя `bool::then()`.

```rust
let message = move || is_odd().then(|| "Ding ding ding!");
view! {
    <p>{message}</p>
}
```

Вы даже можете обойтись без переменной, если хотите. Хотя, лично я иногда предпочитаю улучшенную поддержку `cargo fmt` и `rust-analyzer`,
которую я получаю, вынося такое за пределы `view`.

### Конструкция `match`

Мы всё ещё пишем обычный код на Rust, ведь так? Это означает, у вас в распоряжении вся мощь сопоставления с образцом, доступного в Rust.

```rust
let message = move || {
    match value() {
        0 => "Zero",
        1 => "One",
        n if is_odd() => "Odd",
        _ => "Even"
    }
};
view! {
    <p>{message}</p>
}
```

А почему бы и нет? Живём только раз, не так ли?

## Предотвращение чрезмерного рендеринга

А вот и не только раз.

Всё что мы только что сделали это в целом нормально. Но один нюанс, который вам стоит запомнить и обходиться с ним аккуратно.
Каждая из функций управляющих порядком выполнения, из тех что мы создали к этому моменту, это в сущности производный сигнал:
они будут перезапускаться при каждом изменении значения. В примерах выше, где значение меняется
с четного на нечётное при каждом изменении, это нормально.

Но рассмотрим следующий пример:

```rust
let (value, set_value) = create_signal(0);

let message = move || if value() > 5 {
    "Big"
} else {
    "Small"
};

view! {
    <p>{message}</p>
}
```

Это конечно _работает_. Но если добавить вывод в лог, то можно удивиться

```rust
let message = move || if value() > 5 {
    logging::log!("{}: rendering Big", value());
    "Big"
} else {
    logging::log!("{}: rendering Small", value());
    "Small"
};
```

Когда пользователь нажимает на кнопку будет что-то вроде этого:

```
1: rendering Small
2: rendering Small
3: rendering Small
4: rendering Small
5: rendering Small
6: rendering Big
7: rendering Big
8: rendering Big
... и так до бесконечности
```

Каждый раз когда `value` меняется, `if` выполняется снова. Оно и понятно, учитывая как устроена реактивность. Но у этого есть недостаток.
Для простого текстового узла повторное выполнение `if` и повторный рендеринг это пустяки. Но представьте если бы было так:

```rust
let message = move || if value() > 5 {
    <Big/>
} else {
    <Small/>
};
```

`<Small/>` перезапускается пять раз, затем `<Big/>` бесконечно. Если они подгружают ресурсы, создают сигналы или даже просто создают узлы DOM,
это лишняя работа. 

### `<Show/>`

Компонент [`<Show/>`](https://docs.rs/leptos/latest/leptos/fn.Show.html) это ответ этим проблемам. Вы передаёте условную функцию `when` и `fallback` для показа в случае если `when` вернула  `false`,
и дочерние элементы, которые будут отображаться если `when` равно `true`.

```rust
let (value, set_value) = create_signal(0);

view! {
  <Show
    when=move || { value() > 5 }
    fallback=|| view! { <Small/> }
  >
    <Big/>
  </Show>
}
```

`<Show/>` мемоизирует условие `when`, так что он рендерит `<Small/>` лишь раз,
продолжая показывать тот же компонент пока  `value` не станет больше пяти;
затем он отрендерит `<Big/>` один раз, продолжая его показывать вечно или пока `value`
не станет меньше пяти, тогда он снова отрендерит `<Small/>`.

Это полезный инструмент чтобы избежать повторного рендеринга при использовании динамических `if` выражений.
Как обычно, есть и накладные расходы: для очень маленького узла (как обновление одного текстового узла, обновления класса или атрибута)
конструкция `move || if ...` будет менее затратна. Но если рендеринг какой-либо ветки хоть сколько-нибудь затратен,
используйте `<Show/>`.

## Заметка: конвертация типов

Это последняя вещь, о которой важно сказать в этом разделе.

Макрос `view` не возвращает самый обобщенный оборачивающий тип [`View`](https://docs.rs/leptos/latest/leptos/enum.View.html).
Вместо этого, он возвращает такие типы как `Fragment` или `HtmlElement<Input>`. Это может немного раздражать если 
вы возвращаете разные элементы HTML из разных ветвей условия.

```rust,compile_error
view! {
    <main>
        {move || match is_odd() {
            true if value() == 1 => {
                // returns HtmlElement<Pre>
                view! { <pre>"One"</pre> }
            },
            false if value() == 2 => {
                // returns HtmlElement<P>
                view! { <p>"Two"</p> }
            }
            // returns HtmlElement<Textarea>
            _ => view! { <textarea>{value()}</textarea> }
        }}
    </main>
}
```

Эта сильная типизация в самом деле мощная штука, поскольку [`HtmlElement`](https://docs.rs/leptos/0.1.3/leptos/struct.HtmlElement.html) кроме всего прочего является умным указателем:
каждый тип `HtmlElement<T>` реализует `Deref` для подходящего внутреннего типа `web_sys`. Другими словами, `view` в браузере
возвращает настоящие DOM элементы и у них можно вызывать нативные методы DOM.

Но это может немного раздражать в условной логике как здесь, поскольку в Rust вы не можете возвращать разные типы из разных ветвей условия.
Есть два выхода из положения:

1. Если у вас несколько `HtmlElement` типов, конвертируйте их в  `HtmlElement<AnyElement>` при помощи  [`.into_any()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.into_any)
2. Если у вас набор view-типов, не все из которых `HtmlElement`, конвертируйте их во 
   `View` через [`.into_view()`](https://docs.rs/leptos/latest/leptos/trait.IntoView.html#tymethod.into_view).

Вот тот же пример, с добавленной конвертацией:

```rust,compile_error
view! {
    <main>
        {move || match is_odd() {
            true if value() == 1 => {
                // returns HtmlElement<Pre>
                view! { <pre>"One"</pre> }.into_any()
            },
            false if value() == 2 => {
                // returns HtmlElement<P>
                view! { <p>"Two"</p> }.into_any()
            }
            // returns HtmlElement<Textarea>
            _ => view! { <textarea>{value()}</textarea> }.into_any()
        }}
    </main>
}
```

```admonish sandbox title="Live example" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/6-control-flow-0-5-4yn7qz?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Please enable JavaScript to view examples.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/6-control-flow-0-5-4yn7qz?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox Source</summary>

```rust
use leptos::*;

#[component]
fn App() -> impl IntoView {
    let (value, set_value) = create_signal(0);
    let is_odd = move || value() & 1 == 1;
    let odd_text = move || if is_odd() { Some("How odd!") } else { None };

    view! {
        <h1>"Control Flow"</h1>

        // Simple UI to update and show a value
        <button on:click=move |_| set_value.update(|n| *n += 1)>
            "+1"
        </button>
        <p>"Value is: " {value}</p>

        <hr/>

        <h2><code>"Option<T>"</code></h2>
        // For any `T` that implements `IntoView`,
        // so does `Option<T>`

        <p>{odd_text}</p>
        // This means you can use `Option` methods on it
        <p>{move || odd_text().map(|text| text.len())}</p>

        <h2>"Conditional Logic"</h2>
        // You can do dynamic conditional if-then-else
        // logic in several ways
        //
        // a. An "if" expression in a function
        //    This will simply re-render every time the value
        //    changes, which makes it good for lightweight UI
        <p>
            {move || if is_odd() {
                "Odd"
            } else {
                "Even"
            }}
        </p>

        // b. Toggling some kind of class
        //    This is smart for an element that's going to
        //    toggled often, because it doesn't destroy
        //    it in between states
        //    (you can find the `hidden` class in `index.html`)
        <p class:hidden=is_odd>"Appears if even."</p>

        // c. The <Show/> component
        //    This only renders the fallback and the child
        //    once, lazily, and toggles between them when
        //    needed. This makes it more efficient in many cases
        //    than a {move || if ...} block
        <Show when=is_odd
            fallback=|| view! { <p>"Even steven"</p> }
        >
            <p>"Oddment"</p>
        </Show>

        // d. Because `bool::then()` converts a `bool` to
        //    `Option`, you can use it to create a show/hide toggled
        {move || is_odd().then(|| view! { <p>"Oddity!"</p> })}

        <h2>"Converting between Types"</h2>
        // e. Note: if branches return different types,
        //    you can convert between them with
        //    `.into_any()` (for different HTML element types)
        //    or `.into_view()` (for all view types)
        {move || match is_odd() {
            true if value() == 1 => {
                // <pre> returns HtmlElement<Pre>
                view! { <pre>"One"</pre> }.into_any()
            },
            false if value() == 2 => {
                // <p> returns HtmlElement<P>
                // so we convert into a more generic type
                view! { <p>"Two"</p> }.into_any()
            }
            _ => view! { <textarea>{value()}</textarea> }.into_any()
        }}
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
