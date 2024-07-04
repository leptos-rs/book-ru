# `<Transition/>`

Как можно заметить из примера с `<Suspense/>`, если вы будете перезагружать данные, будет снова и снова мигать  `"Loading..."`.
Иногда это нормально. Для остальных случаев есть [`<Transition/>`](https://docs.rs/leptos/latest/leptos/fn.Transition.html).

`<Transition/>` ведёт себя точно так же как `<Suspense/>`, но вместо показа `fallback` каждый раз,
`fallback` покажется только при первой загрузке. При всех последующих, старые данные будут продолжать показываться,
до тех пор пока новые данные не будут готовы. Это может быть очень полезно, чтобы избежать мигающего эффекта 
и позволить пользователям продолжить взаимодействие с приложением.

Данный пример показывает как создать простой список контактов со вкладками с помощью `<Transition/>`.
Когда вы выбираете новую вкладку, текущий контакт продолжает показываться пока не загрузятся новые данные.
Так пользовательский опыт получается намного лучше чем с постоянным откатом к экрану загрузки.

```admonish sandbox title="Live example" collapsible=true

[Нажмите, чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/12-transition-0-5-2jg5lz?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/12-transition-0-5-2jg5lz?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::*;

async fn important_api_call(id: usize) -> String {
    TimeoutFuture::new(1_000).await;
    match id {
        0 => "Alice",
        1 => "Bob",
        2 => "Carol",
        _ => "User not found",
    }
    .to_string()
}

#[component]
fn App() -> impl IntoView {
    let (tab, set_tab) = create_signal(0);

    // this will reload every time `tab` changes
    let user_data = create_resource(tab, |tab| async move { important_api_call(tab).await });

    view! {
        <div class="buttons">
            <button
                on:click=move |_| set_tab(0)
                class:selected=move || tab() == 0
            >
                "Tab A"
            </button>
            <button
                on:click=move |_| set_tab(1)
                class:selected=move || tab() == 1
            >
                "Tab B"
            </button>
            <button
                on:click=move |_| set_tab(2)
                class:selected=move || tab() == 2
            >
                "Tab C"
            </button>
            {move || if user_data.loading().get() {
                "Loading..."
            } else {
                ""
            }}
        </div>
        <Transition
            // the fallback will show initially
            // on subsequent reloads, the current child will
            // continue showing
            fallback=move || view! { <p>"Loading..."</p> }
        >
            <p>
                {move || user_data.read()}
            </p>
        </Transition>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
