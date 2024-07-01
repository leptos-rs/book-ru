# `view`: Dynamic Classes, Styles and Attributes

Ранее мы рассмотрели как использовать макрос `view` для создания слушателей событий и для создания динамического текста
путём передачи функции (такой как сигнал) во view.

Но конечно же есть и другие вещи, которые вы возможно захотите обновлять в пользовательском интерфейсе.
В этом разделе мы рассмотрим способы изменения классов, стилей и атрибутов динамически, а также
познакомим вас с концепцией **произвольных сигналов**.

Давайте начнём с простого компонента, который должен быть вам знаком: нажатие на кнопку чтобы инкрементировать счетчик.

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            "Click me: "
            {move || count()}
        </button>
    }
}
```

Пока что это просто пример из предыдущей главы.

## Динамические Классы

Теперь предположим что я хочу динамически обновлять список CSS классов этого элемента.
К примеру, я хочу, скажем, добавлять класс `red` когда значение счетчика нечётно. Я могу добиться этого, используя
синтаксис вида `class:`.

```rust
class:red=move || count() % 2 == 1
```

`class:` атрибуты принимают

1. имя класса, следующее за двоеточием (`red`)
2. значение, которое может быть либо `bool` либо функцией возвращающей `bool`

Когда значение равно `true`, класс добавляется. Когда значение равно `false`, класс удаляется.
И если значение это функция, тогда класс будет реактивно обновляться когда сигнал изменяется.

Теперь каждый раз когда я нажимаю на кнопку цвет текст будет чередоваться между красным и черным, так как число становится
то чётным, то нечётным.

```rust
<button
    on:click=move |_| {
        set_count.update(|n| *n += 1);
    }
    // the class: syntax reactively updates a single class
    // here, we'll set the `red` class when `count` is odd
    class:red=move || count() % 2 == 1
>
    "Click me"
</button>
```

> Если запускаете примеры по ходу чтения, убедитесь, что открыли `index.html` и добавили что-то вроде:
>
> ```html
> <style>
>   .red {
>     color: red;
>   }
> </style>
> ```

Некоторые имена CSS классов не могут быть напрямую распознаны макросом `view`, особенно если они включают в себя набор из тире,
чисел и других символов. В таком случае можно использовать синтаксис кортежей: `class=("name", value)` всё так же будет обновлять единственный класс.

```rust
class=("button-20", move || count() % 2 == 1)
```

## Динамические Стили

Отдельные CSS свойства могут напрямую обновляться через схожий синтаксис вида `style:`.

```rust
    let (x, set_x) = create_signal(0);
        view! {
            <button
                on:click={move |_| {
                    set_x.update(|n| *n += 10);
                }}
                // set the `style` attribute
                style="position: absolute"
                // and toggle individual CSS properties with `style:`
                style:left=move || format!("{}px", x() + 100)
                style:background-color=move || format!("rgb({}, {}, 100)", x(), 100)
                style:max-width="400px"
                // Set a CSS variable for stylesheet use
                style=("--columns", x)
            >
                "Click to Move"
            </button>
    }
```

## Динамические Атрибуты

То же самое применимо и к простым атрибутам. Передача простой строки или значения примитива в качества значения
атрибута делает его статическим. Передача функции (включая сигнал) в качестве атрибута делает так, что значение будет обновляться реактивно.
Давайте добавим ещё один элемент в наш view:

```rust
<progress
    max="50"
    // signals are functions, so `value=count` and `value=move || count.get()`
    // are interchangeable.
    value=count
/>
```

Теперь каждый раз когда мы устанавливаем значение `count`, не только `class` элемента `<button>` будет чередоваться, но и 
`value` элемента `<progress>` будет увеличиваться, что означает, что и наш индикатор выполнения будет двигаться вперёд.

## Произвольные Сигналы

Давайте заглянем на уровень глубже, шутки ради.

Как вы уже знаете, реактивные интерфейсы можно создавать, просто передавая функции в `view`. Это означает, что мы можем 
легко менять наш индикатор выполнения. К примеру, предположим мы хотим, чтобы он двигался вдвое быстрее:

```rust
<progress
    max="50"
    value=move || count() * 2
/>
```

Но представьте, что мы хотим использовать это вычисление в более чем одном месте. Сделать это можно с помощью
**производного сигнала**: замыкания которое обращается к сигналу. 

```rust
let double_count = move || count() * 2;

/* insert the rest of the view */
<progress
    max="50"
    // we use it once here
    value=double_count
/>
<p>
    "Double Count: "
    // and again here
    {double_count}
</p>
```

Производные сигналы позволяют вам создавать реактивные вычисляемые значения, которые могут быть использованы сразу в нескольких местах
в приложении с минимальными накладными расходами.

Примечание: Использование производного сигнала означает что вычисление запускается по разу на каждый раз когда сигнал меняется
(когда `count()` меняется) и по разу на каждое место где мы обращаемся к `double_count`; другими словами, дважды. Данное вычисление очень дёшево, 
так что ничего страшного. Мы разберём мемоизацию в другой главе, она была придумана для решения этой проблемы при дорогих вычислениях.

> #### Продвинутая тема: Вставка Сырого HTML
>
> Макрос `view` поддерживает дополнительный атрибут, `inner_html`, который можно использовать для прямого задания
> тела любого элемента в виде HTML, затирая при этом любых дочерние элементы, которые могли в нём находиться.
> Обратите внимание, что он **не** экранирует HTML, который вы передаете. Вам следует убедиться в том, что передаваемый HTML
> собран из доверенных источников или что все HTML-сущности экранированы, дабы предотвратить cross-site scripting (XSS)  атаки.
>
> ```rust
> let html = "<p>This HTML will be injected.</p>";
> view! {
>   <div inner_html=html/>
> }
> ```
>
> [Click here for the full `view` macros docs](https://docs.rs/leptos/latest/leptos/macro.view.html).

```admonish sandbox title="Live example" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/2-dynamic-attributes-0-5-lwdrpm?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/2-dynamic-attributes-0-5-lwdrpm?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::*;

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    // a "derived signal" is a function that accesses other signals
    // we can use this to create reactive values that depend on the
    // values of one or more other signals
    let double_count = move || count() * 2;

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }

            // the class: syntax reactively updates a single class
            // here, we'll set the `red` class when `count` is odd
            class:red=move || count() % 2 == 1
        >
            "Click me"
        </button>
        // NOTE: self-closing tags like <br> need an explicit /
        <br/>

        // We'll update this progress bar every time `count` changes
        <progress
            // static attributes work as in HTML
            max="50"

            // passing a function to an attribute
            // reactively sets that attribute
            // signals are functions, so `value=count` and `value=move || count.get()`
            // are interchangeable.
            value=count
        ></progress>
        <br/>

        // This progress bar will use `double_count`
        // so it should move twice as fast!
        <progress
            max="50"
            // derived signals are functions, so they can also
            // reactively update the DOM
            value=double_count
        ></progress>
        <p>"Count: " {count}</p>
        <p>"Double Count: " {double_count}</p>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
