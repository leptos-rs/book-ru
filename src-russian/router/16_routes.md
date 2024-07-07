# Описывание маршрутов

## Начало работы

Начать работу с маршрутизатором легко.

Перво-наперво нужно убедиться, что пакет  `leptos_router` добавлен в зависимости проекта.
Как и `leptos`, маршрутизатор полагается на активацию особенности `csr`, `hydrate` или `ssr`. 
К примеру, для добавления маршрутизатора в приложение с рендерингом на стороне клиента (CSR), можно вызвать: 
```sh 
cargo add leptos_router --features=csr 
```

> Важно отметить, что `leptos_router` это пакет отдельный от самого `leptos`. Это означает, что всё, что есть в маршрутизаторе может
> быть задано в пользовательском пространстве _(англ. userland)_. Можно без проблем создать свой собственный маршрутизатор или
> вовсе обойтись без оного.

И импортировать сопутствующие типы из `leptos_router`, или как-то так:

```rust
use leptos_router::{Route, RouteProps, Router, RouterProps, Routes, RoutesProps};
```

или просто

```rust
use leptos_router::*;
```

## Объявление `<Router/>`

Поведение маршрутизатора определяется компонентом [`<Router/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.Router.html). Обычно он вызывается где-то рядом с корнем приложения.

> Не следует вызывать `<Router/>` более одного раза. Помните, что маршрутизатор управляет глобальным состоянием:
> если у вас несколько маршрутизаторов, то какой из них будет решать когда менять URL?

Начнём с простого компонента `<App/>`, использующего маршрутизатор:

```rust
use leptos::*;
use leptos_router::*;

#[component]
pub fn App() -> impl IntoView {
  view! {
    <Router>
      <nav>
        /* ... */
      </nav>
      <main>
        /* ... */
      </main>
    </Router>
  }
}
```

## Объявление `<Routes/>`

Компонент  [`<Routes/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.Routes.html) это то, где задаются все маршруты по которым может переходить пользователь приложения.
Каждый из доступных маршрутов задается компонентом [`<Route/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.Route.html).

Компонент `<Routes/>` должен находиться там, где содержимое должно меняться в зависимости от маршрута. 
Всё что вне `<Routes/>` будет отображаться всегда, так что такие вещи как панель навигации или меню можно оставить за пределами `<Routes/>`.

```rust
use leptos::*;
use leptos_router::*;

#[component]
pub fn App() -> impl IntoView {
  view! {
    <Router>
      <nav>
        /* ... */
      </nav>
      <main>
        // all our routes will appear inside <main>
        <Routes>
          /* ... */
        </Routes>
      </main>
    </Router>
  }
}
```

Отдельные маршруты задаются добавлением в `<Routes/>` дочерних компонентов `<Route/>`. `<Route/>` принимает свойства `path` и `view`. 
Когда текущий URL удовлетворяет `path`, `view` будет создано и показано.

Свойство `path` может быть:

- статическим путём (`/users`),
- динамическим путём, именованные параметры начинаются с двоеточия (`/:id`),
- и/или шаблоном начинающийся со звёздочки (`/user/*any`)

Свойство `view` это функция, возвращающая представление. Тут подходит любой компонент без свойств, так же как и замыкание, возвращающее какой-то view.

```rust
<Routes>
  <Route path="/" view=Home/>
  <Route path="/users" view=Users/>
  <Route path="/users/:id" view=UserProfile/>
  <Route path="/*any" view=|| view! { <h1>"Not Found"</h1> }/>
</Routes>
```

> `view` принимает в качестве аргумента `Fn() -> impl IntoView`. Если у компонента нет свойств, его можно напрямую передать в свойство `view`. В данном примере, `view=Home` это сокращенная форма `|| view! { <Home/> }`.

Если перейти по адресу  `/` или `/users`,  то можно увидеть главную страницу или `<Users/>`.
Если перейти на  `/users/3` или `/blahblah`, то будет профиль пользователя or страница ошибки 404 (`<NotFound/>`). 
При каждом переходе, маршрутизатор определяет какой `<Route/>` подходит, и как следствие, какое содержимое должно быть
отображено там, где объявлен компонент `<Routes/>`.

Следует отметить, что маршруты можно задавать в любом порядке. Маршрутизатор использует систему баллов, чтобы определить
какой маршрут лучше подходит, а не просто берёт первый подходящий идя по списку сверху вниз.

Достаточно просто?

## Маршруты с условиями

`leptos_router` основан на том допущении, что в приложении один и только один компонент `<Routes/>`.
Он использует это, чтобы сгенерировать маршруты на стороне сервера, оптимизировать сопоставление маршрутов путём кеширование
вычисленных ветвей и отрендерить приложение.

Нельзя рендерить `<Routes/>` внутри условий используя другие компоненты как `<Show/>` или `<Suspense/>`.

```rust
// ❌ не делайте так!
view! {
  <Show when=|| is_loaded() fallback=|| view! { <p>"Loading"</p> }>
    <Routes>
      <Route path="/" view=Home/>
    </Routes>
  </Show>
}
```

Вместо этого можно использовать вложенные маршруты, чтобы отрендерить `<Routes/>` единожды, и условно рендерить `<Outlet/>`:

```rust
// ✅ делайте так!
view! {
  <Routes>
    // parent route
    <Route path="/" view=move || {
      view! {
        // only show the outlet if data have loaded
        <Show when=|| is_loaded() fallback=|| view! { <p>"Loading"</p> }>
          <Outlet/>
        </Show>
      }
    }>
      // nested child route
      <Route path="/" view=Home/>
    </Route>
  </Routes>
}
```

Если это выглядит причудливо, не стоит беспокоиться! Следующий раздел этой книги посвящен такой вложенной маршрутизации.
