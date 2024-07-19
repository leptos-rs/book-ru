# Вложенная Маршрутизации

Мы только что задали следующий набор маршрутов:

```rust
<Routes>
  <Route path="/" view=Home/>
  <Route path="/users" view=Users/>
  <Route path="/users/:id" view=UserProfile/>
  <Route path="/*any" view=NotFound/>
</Routes>
```

Тут есть некоторое дублирование: `/users` и `/users/:id`. Это нормально для мелкого приложения, но как вы наверное уже поняли, это плохо масштабируется.
Вот было бы классно если бы маршруты могли быть вложенными?

Барабанная дробь... они могут!

```rust
<Routes>
  <Route path="/" view=Home/>
  <Route path="/users" view=Users>
    <Route path=":id" view=UserProfile/>
  </Route>
  <Route path="/*any" view=NotFound/>
</Routes>
```

Но подождите. Поведение приложения немного изменилось.

Следующий подраздел один из самых важных во всём руководстве по маршрутизации. 
Прочтите его внимательно и смело задавайте вопросы если чего-то не поняли.

# Вложенные маршруты как макет

Вложенные маршруты это одна из форм макета, а не способ задания маршрутов.

Скажем иначе: Цель задания вложенных маршрутов главным образом не в том, чтобы избежать повторений при наборе путей в
определениях маршрутов. Она в действительности в том, чтобы сказать маршрутизатору отображать несколько `<Route/>`
на странице одновременно, бок о бок.

Давайте оглянёмся на наш практический пример.

```rust
<Routes>
  <Route path="/users" view=Users/>
  <Route path="/users/:id" view=UserProfile/>
</Routes>
```

Это значит:

- Если зайдешь на `/users`, получишь компонент `<Users/>`.
- Если зайдёшь на `/users/3`, получишь компонент `<UserProfile/>` (с параметром `id` равным `3`; об этом позже)

Скажем мы используем вместо этого вложенные маршруты:

```rust
<Routes>
  <Route path="/users" view=Users>
    <Route path=":id" view=UserProfile/>
  </Route>
</Routes>
```

Это значит:

- Если зайти на `/users/3`, путь подходит к двум `<Route/>`: `<Users/>` и `<UserProfile/>`.
- Если зайти на `/users`, путь ни к чему не подходит.

Нужно добавить fallback-маршрут.

```rust
<Routes>
  <Route path="/users" view=Users>
    <Route path=":id" view=UserProfile/>
    <Route path="" view=NoUser/>
  </Route>
</Routes>
```

Теперь:

- Если зайти на `/users/3`, путь подходит к `<Users/>` и `<UserProfile/>`.
- Если зайти на `/users`, путь подходит к `<Users/>` and `<NoUser/>`.

Другими словами, использовании вложенных маршрутов, каждый  **путь** может подходить к нескольким **маршрутам**: 
каждый URL может рендерить `view` предоставленные разными компонентами `<Route/>` одновременно, на одной странице.

Это может показаться контринтуитивным, но этот подход очень силён, в силу причин, продемонстрированных далее.

## Зачем нужна Вложенная Маршрутизация?

Зачем этим заморачиваться?

Большинство Веб-приложений содержат уровни навигации, соответствующие разным частям макета.
К примеру, в почтовом приложении может быть URL типа `/contacts/greg`, показывающий список контактов в левой части экрана,
и детали контакта Greg в правой части. Список контактов и детали контакта должны всегда быть одновременно видны на экране.
Если контакт не выбран, возможно стоит показать небольшой текст с инструкцией.

Этого легко добиться с помощь вложенных маршрутов

```rust
<Routes>
  <Route path="/contacts" view=ContactList>
    <Route path=":id" view=ContactInfo/>
    <Route path="" view=|| view! {
      <p>"Select a contact to view more info."</p>
    }/>
  </Route>
</Routes>
```

Можно пойти дальше вглубь.
Предположим хочется, чтобы были вкладки для адреса контакта, почты/телефона, и переписки с ним.
Можно добавить _ещё один_ набор вложенных маршрутов внутри `:id`:

```rust
<Routes>
  <Route path="/contacts" view=ContactList>
    <Route path=":id" view=ContactInfo>
      <Route path="" view=EmailAndPhone/>
      <Route path="address" view=Address/>
      <Route path="messages" view=Messages/>
    </Route>
    <Route path="" view=|| view! {
      <p>"Select a contact to view more info."</p>
    }/>
  </Route>
</Routes>
```

> На главной страница [веб-сайта Remix](https://remix.run/), фреймворкa на React от создателей React Router, 
> есть прекрасный визуальный пример если промотать вниз, с тремя уровнями вложенной маршрутизации: Sales > Invoices > an invoice.

## `<Outlet/>`

Родительные маршруты автоматически не рендерят свои вложенные маршруты. В конце концов, это обычные компоненты;
они не знают точно где им следует рендерить дочерние элементы и "просто присобачим их в конец родительного компонента" это плохой ответ.

Вместо этого, вы говорите родительному компоненту где рендерить любые вложенные компоненты используя компонент `<Outlet/>`.
`<Outlet/>` попросту показывает одно из двух:

- если нет подходящего вложенного маршрута, он ничего не показывает
- если есть подходящий, он показывает его `view`

Вот и всё! Но важно знать и помнить, что это частая причина вопроса “Почему же это не работает?” 
Если не предоставить `<Outlet/>`, то вложенный маршрут не будет отображён.

```rust
#[component]
pub fn ContactList() -> impl IntoView {
  let contacts = todo!();

  view! {
    <div style="display: flex">
      // the contact list
      <For each=contacts
        key=|contact| contact.id
        children=|contact| todo!()
      />
      // the nested child, if any
      // don’t forget this!
      <Outlet/>
    </div>
  }
}
```

## Рефакторинг Определений Маршрутов

Можно не задавать все маршруты в одном месте если не хочется. Любой `<Route/>` и его дочерние элементы можно перенести в отдельный компонент.

Вышеуказанный пример можно разбить на два отдельных компонента:

```rust
#[component]
fn App() -> impl IntoView {
  view! {
    <Router>
      <Routes>
        <Route path="/contacts" view=ContactList>
          <ContactInfoRoutes/>
          <Route path="" view=|| view! {
            <p>"Select a contact to view more info."</p>
          }/>
        </Route>
      </Routes>
    </Router>
  }
}

#[component(transparent)]
fn ContactInfoRoutes() -> impl IntoView {
  view! {
    <Route path=":id" view=ContactInfo>
      <Route path="" view=EmailAndPhone/>
      <Route path="address" view=Address/>
      <Route path="messages" view=Messages/>
    </Route>
  }
}
```

Второй компонент имеет пометку `#[component(transparent)]`, это означит, что он просто возвращает данные, а не `view`:
в данном случае это структура [`RouteDefinition`](https://docs.rs/leptos_router/latest/leptos_router/struct.RouteDefinition.html),
которую возвращает `<Route/>`. Пока стоит пометка `#[component(transparent)]`, этот под-маршрут может быть объявлен где угодно,
и при этом вставлен как компонент в дерево маршрутов.

## Вложенная Маршрутизация и Производительность

Задумка хорошая, но всё же, в чём соль?

Производительность.

В библиотеке с мелкозернистой реактивностью, такой как Leptos, всегда важно делать как можно меньшее количества рендеринга.
Поскольку мы работаем с реальными узлами DOM, а не сравнивам изменения через виртуальный DOM, мы хотим "перерендривать"
компоненты как можно реже. Со вложенной маршрутизацией этого чрезвычайно просто достичь.

Представьте наш пример со списком контактов. Если перейти от Greg к Alice, затем к Bob и обратно к Greg, контактная информация
должна меняться при каждом переходе. Но `<ContactList/>` вовсе не должен повторно рендериться.
Это не только экономит производительность, но и поддерживает состояние UI. К примеру, если вверху `<ContactList/>` расположена поисковая строка,
то переход от Greg к Alice и к Bob не сбросит строку поиска.

Фактически, в данном случае даже не нужно повторно рендерить компонент `<Contact/>` при переходе между контактами.
Маршрутизатор будет лишь реактивно обновлять параметр `:id` по мере навигации, позволяя делать мелкозернистые обновления. 
При навигации по контактам одиночные текстовые узлы будут обновляться: имя контакта, адрес и так далее, без _какого-либо_
дополнительного ререндеринга.

> Эта песочница содержит пару This sandbox includes a couple features (like nested routing) discussed in this section and the previous one, and a couple we’ll cover in the rest of this chapter. The router is such an integrated system that it makes sense to provide a single example, so don’t be surprised if there’s anything you don’t understand.

```admonish sandbox title="Live example" collapsible=true

[Click to open CodeSandbox.](https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2)

<noscript>
  Please enable JavaScript to view examples.
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox Source</summary>

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
