# Подгрузка данных с помощью ресурсов

[Ресурс (Resource)](https://docs.rs/leptos/latest/leptos/struct.Resource.html) это реактивная структура данных, отражающая текущее состояние асинхронной задачи, и 
позволяющая интегрировать асинхронные футуры (`Future`) в синхронную реактивную систему. Вместо ожидания данных через `.await`,
мы превращаем футуру в сигнал, возвращающий `Some(T)` если она уже разрешилась и `None` если она ещё в ожидании.

Это делается с помощью функции [`create_resource`](https://docs.rs/leptos/latest/leptos/fn.create_resource.html). Она принимает два аргумента:

1. сигнал-источник, порождающий новую футуру при любом изменении
2. функция, принимающая как аргумент данные из сигнала и возвращающая футуру

Вот пример:

```rust
// our source signal: some synchronous, local state
let (count, set_count) = create_signal(0);

// our resource
let async_data = create_resource(
    count,
    // every time `count` changes, this will run
    |value| async move {
        logging::log!("loading data from API");
        load_data(value).await
    },
);
```

Для создания ресурса, который выполняется единожды, можно передать нереактивный, пустой сигнал-источник:

```rust
let once = create_resource(|| (), |_| async move { load_data().await });
```

Для доступа к значению можно использовать  `.get()` или `.with(|data| /* */)`. Они работают так же как `.get()` и `.with()`
у сигнала: `get` возвращает клонированное значение, а `with` применяет замыкание. Однако для всякого `Resource<_, T>`, 
они возвращают `Option<T>`, а не `T`, поскольку всегда возможно, что ресурс всё ещё грузится.

Так можно отобразить текущее состояние ресурса во `view`:

```rust
let once = create_resource(|| (), |_| async move { load_data().await });
view! {
    <h1>"My Data"</h1>
    {move || match once.get() {
        None => view! { <p>"Loading..."</p> }.into_view(),
        Some(data) => view! { <ShowData data/> }.into_view()
    }}
}
```

Ресурсы также предоставляют метод `refetch()`, позволяющий вручную перезапросить данные (например, в ответ на нажатие кнопки) 
и метод `loading()`, возвращающий `ReadSignal<bool>` — индикатор того загружается ли в данный момент сигнал.

```admonish sandbox title="Live example" collapsible=true

[Нажмите чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/10-resources-0-5-x6h5j6?file=%2Fsrc%2Fmain.rs%3A2%2C3)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/10-resources-0-5-9jq86q?file=%2Fsrc%2Fmain.rs%3A2%2C3" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::*;

// Here we define an async function
// This could be anything: a network request, database read, etc.
// Here, we just multiply a number by 10
async fn load_data(value: i32) -> i32 {
    // fake a one-second delay
    TimeoutFuture::new(1_000).await;
    value * 10
}

#[component]
fn App() -> impl IntoView {
    // this count is our synchronous, local state
    let (count, set_count) = create_signal(0);

    // create_resource takes two arguments after its scope
    let async_data = create_resource(
        // the first is the "source signal"
        count,
        // the second is the loader
        // it takes the source signal's value as its argument
        // and does some async work
        |value| async move { load_data(value).await },
    );
    // whenever the source signal changes, the loader reloads

    // you can also create resources that only load once
    // just return the unit type () from the source signal
    // that doesn't depend on anything: we just load it once
    let stable = create_resource(|| (), |_| async move { load_data(1).await });

    // we can access the resource values with .get()
    // this will reactively return None before the Future has resolved
    // and update to Some(T) when it has resolved
    let async_result = move || {
        async_data
            .get()
            .map(|value| format!("Server returned {value:?}"))
            // This loading state will only show before the first load
            .unwrap_or_else(|| "Loading...".into())
    };

    // the resource's loading() method gives us a
    // signal to indicate whether it's currently loading
    let loading = async_data.loading();
    let is_loading = move || if loading() { "Loading..." } else { "Idle." };

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            "Click me"
        </button>
        <p>
            <code>"stable"</code>": " {move || stable.get()}
        </p>
        <p>
            <code>"count"</code>": " {count}
        </p>
        <p>
            <code>"async_value"</code>": "
            {async_result}
            <br/>
            {is_loading}
        </p>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
