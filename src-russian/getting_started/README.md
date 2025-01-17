# Начало работы

Есть два пути чтобы начать работу с Leptos:
1. **Рендеринг на клиенте (CSR) с [Trunk](https://trunkrs.dev/)** - отличный вариант если просто хочется сделать шустрый сайт на Leptos,
или работать с уже существующим сервером или API.
В режиме CSR, Trunk компилирует Leptos приложение в WebAssembly (WASM) и запускает его в браузере как обычное 
одностраничное Javascript приложение (SPA). Примущества Leptos CSR включают более быструю сборку и ускоренный
итеративный цикл разработки, а также более простую ментальную модель и больше вариантов развёртывания вашего приложения.
CSR приложения имеют и некоторые недоставки: время первоначальной загрузки для пользователей будет дольше
в сравнении с подходом серверного рендеринга (SSR), а ещё пресловутые проблемы SEO, которыми сопровождаются 
одностраничники на JS, применимы и к приложениям на Leptos CSR. Также стоит заметить, что "под капотом" используется
 автоматически генерируемый скрипт на JS, подгружающий бинарник WASM с Leptos, так что JS *должен* быть включен
на устройстве пользователя, чтобы ваше CSR приложение отображалось нормально. Как и во всей программной инженерии,
здесь есть компромиссы, которые вам нужно учитывать.

2. **Full-stack, серверный рендеринг (SSR) c [`cargo-leptos`](https://github.com/leptos-rs/cargo-leptos)** 
— SSR это отличный вариант для построения веб-сайтов в стиле CRUD и кастомных веб-приложений если вы хотите,
чтобы Rust был и на клиенте и на сервере. При использовании варианта Leptos SSR, приложение рендерится 
в HTML на сервере и отправляется в браузер; затем в дело вступает WebAssembly, вооружая HTML, чтобы ваше приложение
стало интерактивным — этот процесс называется "гидратация". На стороне сервера, Leptos SSR приложения тесно интегрируются
либо с фреймворком на выбор — [Actix-web](https://docs.rs/leptos_actix/latest/leptos_actix/index.html)
или [Axum](https://docs.rs/leptos_axum/latest/leptos_axum/index.html), 
так что можно использовать крейты этих сообществ при построении сервера Leptos.
Преимущества выбора Leptos SSR включают в себя помощь в получении наилучшего времени первоначальной загрузки и оптимальные
очки SEO для вашего веб-приложения. SSR приложения могут также кардинально упростить клиент-серверное взаимодействие
с помощью Leptos-фичи под названием "серверные функции", которая позволяет прозрачно вызывать функции на сервере
из клиентского кода (об этом позже). Full-stack SSR, впрочем, это не только пони, питающиеся радугой, 
есть и недоставки, они включают в себя более медленный цикл разработки (потому что вам нужно перекомпилировать
и сервер и клиент, когда вы меняете Rust код), а также некоторую дополнительную сложность, которую вносит гидратация.

К концу этой книги у вас будет ясное представление о том, на какие компромиссы идти и какой путь избирать — CSR или SSR
 — в зависимости от требований вашего проекта.

В Части 1 этой книги мы начнём с клиентского рендеринга Leptos сайтов и построения реактивных UI используя `Trunk`
для отдачи нашего JS и WASM бандла в браузер.

Мы познакомим вас с `cargo leptos` во Части 2 этой книги, которая посвящена работе со всей мощью
Leptos в его full-stack SSR режиме.

```admonish note
Если вы пришли из мира Javascript и такие термины как клиентский рендеринг (CSR) и серверный рендеринг (SSR) вам незнакомы,
самый простой способ понять разницу между ними — это по аналогии:

CSR режим в Leptos похож на работу с React (или с фреймворком основаным на сигналах, таким как SolidJS) и сфокусирован
на создании UI на клиентской стороне, который можно использовать с любым серверным стеком.  

Использование режима SSR в Leptos похоже на работу с full-stack фреймворком как Next.js в мире React
(или "SolidStart" фреймворком в SolidJS) — SSR помогает строить сайты и приложения, которые рендрятся на сервере и затем отправляются клиенту.
SSR может помочь улучшить производительность загрузки и доступность сайта, 
а также упростить работу одному человеку *сразу* и над клиентом и на сервером без необходимости переключать контекст
 между разными языками для frontend и backend.       

Фреймворк Leptos может быть использовать либо в режиме CSR, чтобы просто сделать UI (как React), а может в
 full-stack SSR режиме (как Next.js), так чтобы вы могли писать и UI и серверную часть на одном языке: на Rust.   
```

## Привет, Мир! Подготовка к Leptos CSR разработке

Первым делом убедитесь что Rust установлен и обновлен ([здесь инструкции, если нужны](https://www.rust-lang.org/tools/install))

Если инструмент `Trunk` для запуска сайтов на Leptos CSR ещё не установлен, его можно установить так:

```bash
cargo install trunk
```

А затем создайте простой Rust проект

```bash
cargo init leptos-tutorial
```

`cd` в директорию только что созданного проекта `leptos-tutorial` и добавьте `leptos` в качестве зависимости

```bash
cargo add leptos --features=csr,nightly
```

Или без  `nightly` если вы на стабильной версии Rust

```bash
cargo add leptos --features=csr
```

> Использование `nightly` Rust, и `nightly` feature в Leptos включает синтаксис вызова функции для геттеров и сеттеров сигналов, который использован в большей части этой книги.
>
> Чтобы использовать nightly Rust, можно либо выбрать nightly для всех Rust проектов, выполнив
>
> ```bash
> rustup toolchain install nightly
> rustup default nightly
> ```
>
> либо только для этого проекта
>
> ```bash
> rustup toolchain install nightly
> cd <into your project>
> rustup override set nightly
> ```
>
> [Здесь больше подробностей](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html)
>
> Если хотите использовать стабильную версию Rust с Leptos, то так тоже можно. В этом руководстве и примерах
> просто используйте методы
> [`ReadSignal::get()`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html#impl-SignalGet%3CT%3E-for-ReadSignal%3CT%3E) 
> и [`WriteSignal::set()`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html#impl-SignalGet%3CT%3E-for-ReadSignal%3CT%3E) 
> вместо вызова геттеров и сеттеров как будто они функции.

Убедитесь, что добавили цель сборки `wasm32-unknown-unknown`, чтобы Rust мог компилировать код в WebAssembly для запуска в браузере.

```bash
rustup target add wasm32-unknown-unknown
```

Создайте простой файл `index.html` в корне директории `leptos-tutorial`

```html
<!DOCTYPE html>
<html>
  <head></head>
  <body></body>
</html>
```

И добавьте простое “Привет, Мир!” в ваш `main.rs`

```rust
use leptos::*;

fn main() {
    mount_to_body(|| view! { <p>"Привет, Мир!"</p> })
}
```

Структура директорий должна выглядеть как-то так

```
leptos_tutorial
├── src
│   └── main.rs
├── Cargo.toml
├── index.html
```

Теперь запустите  `trunk serve --open` из корня вашей директории `leptos-tutorial`.
Trunk должен автоматически скомпилировать приложение и открыть браузер по-умолчанию.
При внесении правок в `main.rs`, Trunk перекомпилирует исходный код и перезагрузит страницу.

Добро пожаловать в мир разработки UI с помощью Rust и WebAssembly (WASM), приводимый в действие Leptos и Trunk!

```admonish note
Под Windows, нужно учитывать, что `trunk server --open` может не работать. При проблемах с `--open` просто
используйте `trunk serve` и откройте вкладку браузера вручную. 
```

---

Теперь прежде чем мы начнем создавать ваш первый реальный UI c Leptos, есть пара вещей, о которых стоит знать, 
чтобы сделать вашу разработку с Leptos чуточку проще.
