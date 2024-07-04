# Мутация данных через действия (Actions)

Мы обсудили как подгружать асинхронные данные с помощью ресурсов. Ресурсы немедленно загружают данные, они тесно связаны с 
компонентами `<Suspense/>` и `<Transition/>`, которые показывают пользователю загружаются ли данные.
Но что если хочется просто вызвать произвольную `async` функцию и следить за тем, что происходит?

Ну, всегда можно использовать [`spawn_local`](https://docs.rs/leptos/latest/leptos/fn.spawn_local.html). Это позволяет породить `async` задачу в синхронном окружении,
передав `Future` в браузер (или, на сервере, в Tokio или в другую используемую асинхронную среду выполнения).
Но как узнать выполняется ли она всё ещё или нет? Ну, можно сделать сигнал, показывающий идёт ли загрузка, и ещё один
для показа результата...

Всё это правда. Или можно просто последний `async` примитив: [`create_action`](https://docs.rs/leptos/latest/leptos/fn.create_action.html).

Действия и ресурсы с виду похожи, но они представляют фундаментально разные вещи. Если нужно загрузить данные через
вызов `async` функции, единожды или когда какое-то другое значение меняется, то наверно стоит использовать `create_resource`.
Если же нужно время от времени запускать `async` функцию в ответ на, например, нажатие на кнопку, наверно стоит использовать
`create_action`.

Предположим есть  `async` функцию, которую нужно выполнить.

```rust
async fn add_todo_request(new_title: &str) -> Uuid {
    /* do some stuff on the server to add a new todo */
}
```

`create_action` принимает  в качестве аргумента `async` функцию, которая в качестве аргумента принимает ссылку на 
то, что можно назвать “вводный тип”.

> Вводный тип всегда один. Если нужно передать несколько аргументов, это можно сделать через структуру или кортеж.
>
> ```rust
> // if there's a single argument, just use that
> let action1 = create_action(|input: &String| {
>    let input = input.clone();
>    async move { todo!() }
> });
>
> // if there are no arguments, use the unit type `()`
> let action2 = create_action(|input: &()| async { todo!() });
>
> // if there are multiple arguments, use a tuple
> let action3 = create_action(
>   |input: &(usize, String)| async { todo!() }
> );
> ```
>
> Поскольку функция действия принимает ссылку, а футуре нужно время жизни `'static`,
> обычно требуется клонировать значение, чтобы передать его в футуру. Это, признаться, неуклюже, но это открывает
> такие мощные возможности, как оптимистичный UI. Подробнее об этом в будущих главах.

Так что в данном случае всё что нужно сделать это создать действие:

```rust
let add_todo_action = create_action(|input: &String| {
    let input = input.to_owned();
    async move { add_todo_request(&input).await }
});
```

Вместо прямого вызова `add_todo_action`, вызовём его через `.dispatch()`

```rust
add_todo_action.dispatch("Some value".to_string());
```

Вы можете сделать из слушателя событий, таймаута, или откуда угодно; поскольку `.dispatch()` это не `async` функция, 
она может быть вызвана из синхронного контекста.

Действия дают доступ к нескольким сигналам, синхронизирующим асинхронное действия и синхронную реактивную систему:

```rust
let submitted = add_todo_action.input(); // RwSignal<Option<String>>
let pending = add_todo_action.pending(); // ReadSignal<bool>
let todo_id = add_todo_action.value(); // RwSignal<Option<Uuid>>
```

Это позволяет легко отслеживать текущее состояние вашего запроса, показать индикатор загрузки, или сделать “оптимистичный UI”,
основанный на допущении, что отправка увенчается успехом.

```rust
let input_ref = create_node_ref::<Input>();

view! {
    <form
        on:submit=move |ev| {
            ev.prevent_default(); // don't reload the page...
            let input = input_ref.get().expect("input to exist");
            add_todo_action.dispatch(input.value());
        }
    >
        <label>
            "What do you need to do?"
            <input type="text"
                node_ref=input_ref
            />
        </label>
        <button type="submit">"Add Todo"</button>
    </form>
    // use our loading state
    <p>{move || pending().then("Loading...")}</p>
}
```

Что ж, всё это может показаться чуточку переусложненным или, быть может, слишком ограничивающим.
Я захотел включить сюда действия, вместе с сигналами, как недостающий пазл. В реальном приложении на Leptos,
вы действительно чаще всего будете использовать действия вместе с серверными функциями, компоненты [`create_server_action`](https://docs.rs/leptos/latest/leptos/fn.create_server_action.html),
и [`<ActionForm/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.ActionForm.html) для создания действительно мощных форм с прогрессивным улучшением.
Так что, если этот примитив кажется бесполезным, не переживайте! Возможно понимание придёт позже. (Или ознакомьтесь с нашим [`todo_app_sqlite`](https://github.com/leptos-rs/leptos/blob/main/examples/todo_app_sqlite/src/todo.rs) примером прямо сейчас.)

```admonish sandbox title="Живой пример" collapsible=true

[Нажмите, чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/13-actions-0-5-8xk35v?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/13-actions-0-5-8xk35v?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::{html::Input, *};
use uuid::Uuid;

// Here we define an async function
// This could be anything: a network request, database read, etc.
// Think of it as a mutation: some imperative async action you run,
// whereas a resource would be some async data you load
async fn add_todo(text: &str) -> Uuid {
    _ = text;
    // fake a one-second delay
    TimeoutFuture::new(1_000).await;
    // pretend this is a post ID or something
    Uuid::new_v4()
}

#[component]
fn App() -> impl IntoView {
    // an action takes an async function with single argument
    // it can be a simple type, a struct, or ()
    let add_todo = create_action(|input: &String| {
        // the input is a reference, but we need the Future to own it
        // this is important: we need to clone and move into the Future
        // so it has a 'static lifetime
        let input = input.to_owned();
        async move { add_todo(&input).await }
    });

    // actions provide a bunch of synchronous, reactive variables
    // that tell us different things about the state of the action
    let submitted = add_todo.input();
    let pending = add_todo.pending();
    let todo_id = add_todo.value();

    let input_ref = create_node_ref::<Input>();

    view! {
        <form
            on:submit=move |ev| {
                ev.prevent_default(); // don't reload the page...
                let input = input_ref.get().expect("input to exist");
                add_todo.dispatch(input.value());
            }
        >
            <label>
                "What do you need to do?"
                <input type="text"
                    node_ref=input_ref
                />
            </label>
            <button type="submit">"Add Todo"</button>
        </form>
        <p>{move || pending().then(|| "Loading...")}</p>
        <p>
            "Submitted: "
            <code>{move || format!("{:#?}", submitted())}</code>
        </p>
        <p>
            "Pending: "
            <code>{move || format!("{:#?}", pending())}</code>
        </p>
        <p>
            "Todo ID: "
            <code>{move || format!("{:#?}", todo_id())}</code>
        </p>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
