# Обработка ошибок

[В предыдущей главе](./06_control_flow.md), мы рассмотрели, что можно рендерить `Option<T>`:
в случае `None`, ничего не будет выведено, а в случае `Some(T)`, будет выведен `T`
(если `T` реализует `IntoView`). С `Result<T, E>` можно обойтись весьма схожим образом. 
В случае `Err(_)`, ничего не будет выведено. В случае `Ok(T)` будет выведен `T`.

Давайте начнем с простого компонента, осуществляющего захват числового поля ввода.

```rust
#[component]
fn NumericInput() -> impl IntoView {
    let (value, set_value) = create_signal(Ok(0));

    // when input changes, try to parse a number from the input
    let on_input = move |ev| set_value(event_target_value(&ev).parse::<i32>());

    view! {
        <label>
            "Type an integer (or not!)"
            <input type="number" on:input=on_input/>
            <p>
                "You entered "
                <strong>{value}</strong>
            </p>
        </label>
    }
}
```

Каждый раз когда значение поля ввода меняется, `on_input` попытается превратить это значение в 32-битное число (`i32`)
и сохранить его в наш сигнал `value` с типом `Result<i32, _>`. Если ввести число `42`, UI отобразит

```
You entered 42
```

Но если введете строку `foo`, он отобразит

```
You entered
```

Выглядит так себе. Это экономит нам вызов `.unwrap_or_default()` или чего-то подобного, но было бы намного лучше, если
мы могли бы поймать эту ошибку и что-нибудь с ней сделать.

Это можно сделать, используя компонент [`<ErrorBoundary/>`](https://docs.rs/leptos/latest/leptos/fn.ErrorBoundary.html)

```admonish note
Люди часто пытаются указать на то, что `<input type="number>` не даст вам написать строку как `foo` или что-либо ещё,
что не является числом. Это справедливо в каких-то браузерах, но не во всех! Более того, есть множество вещей, которые
можно напечатать в обычный числовое поле ввода и которые не являются `i32`: число с плавающей точкой, число больше
 чем позволяют 32 бита, буква `e` и так далее. Браузеру можно сказать чтоб он поддерживал некоторые из этих инвариантов,
 однако поведение браузера всё же вариативно: Важно парсить самостоятельно!
```

## `<ErrorBoundary/>`

`<ErrorBoundary/>` немного сродни компоненту `<Show/>`, рассмотренному нами в предыдущей главе.
Если всё окей —точнее сказать, если всё `Ok(_)`— он выводит дочерние элементы.
Но если среди потомков будет выведен `Err(_)`, это вызовет отображение `fallback` в `<ErrorBoundary/>`.

Давайте добавим `<ErrorBoundary/>` в наш пример.

```rust
#[component]
fn NumericInput() -> impl IntoView {
    let (value, set_value) = create_signal(Ok(0));

    let on_input = move |ev| set_value(event_target_value(&ev).parse::<i32>());

    view! {
        <h1>"Error Handling"</h1>
        <label>
            "Type a number (or something that's not a number!)"
            <input type="number" on:input=on_input/>
            <ErrorBoundary
                // the fallback receives a signal containing current errors
                fallback=|errors| view! {
                    <div class="error">
                        <p>"Not a number! Errors: "</p>
                        // we can render a list of errors as strings, if we'd like
                        <ul>
                            {move || errors.get()
                                .into_iter()
                                .map(|(_, e)| view! { <li>{e.to_string()}</li>})
                                .collect_view()
                            }
                        </ul>
                    </div>
                }
            >
                <p>"You entered " <strong>{value}</strong></p>
            </ErrorBoundary>
        </label>
    }
}
```

Теперь если ввести `42`, `value` примет значение `Ok(42)` и вы увидите

```
You entered 42
```

Если ввести `foo`, `value` будет `Err(_)` и отобразится `fallback`.
Мы вывели список ошибок в виде `String`, так что вы увидите что-то вроде

```
Not a number! Errors:
- cannot parse integer from empty string
```

Если исправить эту ошибку, сообщение об ошибке исчезнет и контент обёрнутый в `<ErrorBoundary/>` появится снова.

```admonish sandbox title="Live example" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/7-errors-0-5-5mptv9?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/7-errors-0-5-5mptv9?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>
```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::*;

#[component]
fn App() -> impl IntoView {
    let (value, set_value) = create_signal(Ok(0));

    // when input changes, try to parse a number from the input
    let on_input = move |ev| set_value(event_target_value(&ev).parse::<i32>());

    view! {
        <h1>"Error Handling"</h1>
        <label>
            "Type a number (or something that's not a number!)"
            <input type="number" on:input=on_input/>
            // If an `Err(_) had been rendered inside the <ErrorBoundary/>,
            // the fallback will be displayed. Otherwise, the children of the
            // <ErrorBoundary/> will be displayed.
            <ErrorBoundary
                // the fallback receives a signal containing current errors
                fallback=|errors| view! {
                    <div class="error">
                        <p>"Not a number! Errors: "</p>
                        // we can render a list of errors
                        // as strings, if we'd like
                        <ul>
                            {move || errors.get()
                                .into_iter()
                                .map(|(_, e)| view! { <li>{e.to_string()}</li>})
                                .collect::<Vec<_>>()
                            }
                        </ul>
                    </div>
                }
            >
                <p>
                    "You entered "
                    // because `value` is `Result<i32, _>`,
                    // it will render the `i32` if it is `Ok`,
                    // and render nothing and trigger the error boundary
                    // if it is `Err`. It's a signal, so this will dynamically
                    // update when `value` changes
                    <strong>{value}</strong>
                </p>
            </ErrorBoundary>
        </label>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
