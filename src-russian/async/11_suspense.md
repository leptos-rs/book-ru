# `<Suspense/>`

В предыдущей главе было рассмотрено создание простого экрана загрузки, чтобы показывать некий `fallback` пока ресурс грузится.

```rust
let (count, set_count) = create_signal(0);
let once = create_resource(count, |count| async move { load_a(count).await });

view! {
    <h1>"My Data"</h1>
    {move || match once.get() {
        None => view! { <p>"Loading..."</p> }.into_view(),
        Some(data) => view! { <ShowData data/> }.into_view()
    }}
}
```

Но что если есть два ресурса и хочется ожидать оба?

```rust
let (count, set_count) = create_signal(0);
let (count2, set_count2) = create_signal(0);
let a = create_resource(count, |count| async move { load_a(count).await });
let b = create_resource(count2, |count| async move { load_b(count).await });

view! {
    <h1>"My Data"</h1>
    {move || match (a.get(), b.get()) {
        (Some(a), Some(b)) => view! {
            <ShowA a/>
            <ShowA b/>
        }.into_view(),
        _ => view! { <p>"Loading..."</p> }.into_view()
    }}
}
```

Выглядит не _очень_ плохо, но всё же несколько раздражает. Что если использовать инверсию управления (IoC)?

Компонент [`<Suspense/>`](https://docs.rs/leptos/latest/leptos/fn.Suspense.html) позволяет сделать именно это. Вы передаёте ему свойство  `fallback` и дочерние элементы,
один или более из которых обычно включает в себя чтение из ресурса. Чтение из ресурса “внутри” a `<Suspense/>`
(т.е. в одном из элементов-потомков) регистрирует ресурс в `<Suspense/>`. Если загрузка хотя бы одного из ресурсов всё ещё идет, 
отображается `fallback`. Когда они все загружены, отображаются дочерние элементы.

```rust
let (count, set_count) = create_signal(0);
let (count2, set_count2) = create_signal(0);
let a = create_resource(count, |count| async move { load_a(count).await });
let b = create_resource(count2, |count| async move { load_b(count).await });

view! {
    <h1>"My Data"</h1>
    <Suspense
        fallback=move || view! { <p>"Loading..."</p> }
    >
        <h2>"My Data"</h2>
        <h3>"A"</h3>
        {move || {
            a.get()
                .map(|a| view! { <ShowA a/> })
        }}
        <h3>"B"</h3>
        {move || {
            b.get()
                .map(|b| view! { <ShowB b/> })
        }}
    </Suspense>
}
```

Каждый раз когда любой из ресурсов перезагружается, `fallback` с "Loading..." отображается снова.

Инверсия управления упрощает добавление и удаление отдельных ресурсов, избавляя от необходимости самостоятельно делать сопоставление с шаблоном.
Это также открывает значительные оптимизации производительности во время серверного рендеринга (SSR), о которых мы поговорим в одной из следующих глав.

## `<Await/>`

Если хочется просто дождаться какого-то футуры перед рендерингом, компонент `<Await/>` может быть полезен в уменьшении шаблонного кода.
`<Await/>` по сути сочетает в себе ресурс с источником `|| ()` и `<Suspense/>` без `fallback`.

Другими словами:

1. Он poll'ит `Future` лишь раз и не реагирует ни на какие реактивные изменения.
2. Он ничего не рендерит пока `Future` не разрешится.
3. После разрешения футуры, он кладёт её данные в выбранную переменную и рендерит дочерние элементы с этой переменной в области видимости.

```rust
async fn fetch_monkeys(monkey: i32) -> i32 {
    // maybe this didn't need to be async
    monkey * 2
}
view! {
    <Await
        // `future` provides the `Future` to be resolved
        future=|| fetch_monkeys(3)
        // the data is bound to whatever variable name you provide
        let:data
    >
        // you receive the data by reference and can use it in your view here
        <p>{*data} " little monkeys, jumping on the bed."</p>
    </Await>
}
```

```admonish sandbox title="Live example" collapsible=true

[Нажмите, чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/11-suspense-0-5-qzpgqs?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/11-suspense-0-5-qzpgqs?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::*;

async fn important_api_call(name: String) -> String {
    TimeoutFuture::new(1_000).await;
    name.to_ascii_uppercase()
}

#[component]
fn App() -> impl IntoView {
    let (name, set_name) = create_signal("Bill".to_string());

    // this will reload every time `name` changes
    let async_data = create_resource(

        name,
        |name| async move { important_api_call(name).await },
    );

    view! {
        <input
            on:input=move |ev| {
                set_name(event_target_value(&ev));
            }
            prop:value=name
        />
        <p><code>"name:"</code> {name}</p>
        <Suspense
            // the fallback will show whenever a resource
            // read "under" the suspense is loading
            fallback=move || view! { <p>"Loading..."</p> }
        >
            // the children will be rendered once initially,
            // and then whenever any resources has been resolved
            <p>
                "Your shouting name is "
                {move || async_data.get()}
            </p>
        </Suspense>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
