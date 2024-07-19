# Параметры и поля запроса

Статические пути полезны чтобы различать разные страницы, но почти в каждом приложении рано или поздно потребуется передавать
данные в URL.

Есть два способа это сделать:

1. именованные **параметры** запроса как `id` в `/users/:id`
2. именованные **поля** запроса как `q` inв `/search?q=Foo`

В силу того как устроен URL, поля запроса доступны во `view` _любого_ `<Route/>`. Параметры запроса доступны в том `<Route/>`, 
где они объявлены и во всех потомках.

Обратиться к параметрам и полям легко с помощью пары хуков:

- [`use_query`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_query.html) или [`use_query_map`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_query_map.html)
- [`use_params`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_params.html) или [`use_params_map`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_params_map.html)

Каждый из них представлен в типизированным вариантом (`use_query` и `use_params`) и нетипизированным (`use_query_map` и `use_params_map`).

Нетипизированные версии функций возвращают `Map<String, String>`. Чтобы использовать типизированные версии,
достаточно объявить структуру и добавить `derive` для типажа [`Params`](https://docs.rs/leptos_router/0.2.3/leptos_router/trait.Params.html).

> `Params` это очень легковесный типаж, конвертирующий `Map<String, String>` в структуру, применяя `FromStr` к каждому из полей.
> Поскольку параметры и поля запроса имеют плоскую структуру, этот типаж намного проще чем, например, `serde`; он также добавляет куда меньше веса бинарнику приложения.

```rust
use leptos::*;
use leptos_router::*;

#[derive(Params, PartialEq)]
struct ContactParams {
	id: usize
}

#[derive(Params, PartialEq)]
struct ContactSearch {
	q: String
}
```

> Примечание: Макрос `Params` находится в `leptos::Params`, а типаж `Params` в `leptos_router::Params`. 
> Если не использовать `use leptos::*;`, то надо убедиться, что импортируется именно макрос.
>
> Если особенность `nightly` не включена, будет ошибка:
>
> ```
> no function or associated item named `into_param` found for struct `std::string::String` in the current scope
> ```
>
> На данный момент поддержка и `T: FromStr` и `Option<T>` для типизированных параметров требует особенности `nightly`.
> Это можно исправить просто поменяв в структуре `q: Option<String>` на `q: String`.

Теперь их можно использовать в компоненте. Представим URL, имеющий и параметры и поля: `/contacts/:id?q=Search`.

Типизированные версии возвращают `Memo<Result<T, _>>`. Это мемоизированное значение, реагирующее на изменение URL.
`Result` используется потому, что параметры или поля берутся из URL, который может быть как корректным, так и некорректным.

```rust
let params = use_params::<ContactParams>();
let query = use_query::<ContactSearch>();

// id: || -> usize
let id = move || {
	params.with(|params| {
		params.as_ref()
			.map(|params| params.id)
			.unwrap_or_default()
	})
};
```

Нетипизированные версии возвращают `Memo<ParamsMap>`. И снова, это мемоизированное значение, реагирующее на изменение URL.
[`ParamsMap`](https://docs.rs/leptos_router/0.2.3/leptos_router/struct.ParamsMap.html) ведёт себя как любой другой тип map,
его метод `.get()` возвращает `Option<&String>`.

```rust
let params = use_params_map();
let query = use_query_map();

// id: || -> Option<String>
let id = move || {
	params.with(|params| params.get("id").cloned())
};
```

Тут может возникнуть небольшая путаница: производный сигнал вокруг `Option<_>` или `Result<_>` может включать пару шагов.
Но его стоит сделать по двум причинам:

1. Это корректно, поскольку мы вынуждены рассматривать случаи “Что если пользователь не передает значение этого поля? Что если значение недопустимо?”
2. Это производительно. В особенности, при навигации между разными путями, соответствующими одному и тому же `<Route/>`, когда меняются только параметры или поля запроса, 
можно добиться мелкозернистых обновлений разных частей приложения без повторного рендеринга.
Например, в нашем примере с контакт-листом, навигация между контактами приводит к прицельному обновлению поля
Name (и, со временем, контактной информации) без необходимости замены или повторного рендеринга всего `<Contact/>`. 
Это то, зачем нужна мелкозернистая реактивность.

> Это тот же самый пример из предыдущего раздела. Маршрутизатор это настолько тесно интегрированная система, что имеет смысл 
> рассмотреть единый пример демонстрирующий сразу несколько возможностей, включая те, которые ещё не были затронуты этим руководством.

```admonish sandbox title="Живой пример" collapsible=true

[Нажмите, чтобы открыть CodeSandbox.](https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2)

<noscript>
  Пожалуйста, включите Javascript для просмотра примеров.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>Код примера CodeSandbox</summary>

```rust
use leptos::*;
use leptos_router::*;

#[component]
fn App() -> impl IntoView {
    view! {
        <Router>
            <h1>"Contact App"</h1>
            // this <nav> will show on every routes,
            // because it's outside the <Routes/>
            // note: we can just use normal <a> tags
            // and the router will use client-side navigation
            <nav>
                <h2>"Navigation"</h2>
                <a href="/">"Home"</a>
                <a href="/contacts">"Contacts"</a>
            </nav>
            <main>
                <Routes>
                    // / just has an un-nested "Home"
                    <Route path="/" view=|| view! {
                        <h3>"Home"</h3>
                    }/>
                    // /contacts has nested routes
                    <Route
                        path="/contacts"
                        view=ContactList
                      >
                        // if no id specified, fall back
                        <Route path=":id" view=ContactInfo>
                            <Route path="" view=|| view! {
                                <div class="tab">
                                    "(Contact Info)"
                                </div>
                            }/>
                            <Route path="conversations" view=|| view! {
                                <div class="tab">
                                    "(Conversations)"
                                </div>
                            }/>
                        </Route>
                        // if no id specified, fall back
                        <Route path="" view=|| view! {
                            <div class="select-user">
                                "Select a user to view contact info."
                            </div>
                        }/>
                    </Route>
                </Routes>
            </main>
        </Router>
    }
}

#[component]
fn ContactList() -> impl IntoView {
    view! {
        <div class="contact-list">
            // here's our contact list component itself
            <div class="contact-list-contacts">
                <h3>"Contacts"</h3>
                <A href="alice">"Alice"</A>
                <A href="bob">"Bob"</A>
                <A href="steve">"Steve"</A>
            </div>

            // <Outlet/> will show the nested child route
            // we can position this outlet wherever we want
            // within the layout
            <Outlet/>
        </div>
    }
}

#[component]
fn ContactInfo() -> impl IntoView {
    // we can access the :id param reactively with `use_params_map`
    let params = use_params_map();
    let id = move || params.with(|params| params.get("id").cloned().unwrap_or_default());

    // imagine we're loading data from an API here
    let name = move || match id().as_str() {
        "alice" => "Alice",
        "bob" => "Bob",
        "steve" => "Steve",
        _ => "User not found.",
    };

    view! {
        <div class="contact-info">
            <h4>{name}</h4>
            <div class="tabs">
                <A href="" exact=true>"Contact Info"</A>
                <A href="conversations">"Conversations"</A>
            </div>

            // <Outlet/> here is the tabs that are nested
            // underneath the /contacts/:id route
            <Outlet/>
        </div>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
