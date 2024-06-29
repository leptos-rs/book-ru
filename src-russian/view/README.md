# Часть 1: Построение Пользовательского Интерфейса

В первой части этой книги мы рассмотрим построение пользовательских интерфейсов на стороне клиента используя Leptos.

"Под капотом" Leptos и Trunk добавляют на страницу небольшой код на Javascript, который подзагружает Leptos UI, 
скомпилированный в WebAssembly для обеспечения реактивности вашего CSR (client-side rendered) веб-сайта.

Часть 1 познакомит вас с простыми инструментами, которые вам понадобятся, чтобы построить реактивный пользовательский 
интерфейс с помощью Leptos и Rust.
К концу первой части вы наверняка сможете создать шустрый синхронный веб-сайт, который рендерится в браузере и 
который можно будет развернуть на любом хостинге статических сайтов, таком как Github Pages или Vercel.

```admonish info
Чтобы извлечь максимум из этой книги мы советуем вам запускать код примеров по ходу чтения.  
В главах [Начало работы](https://book-ru.leptos.dev/getting_started/) и [Leptos DX](https://book-ru.leptos.dev/getting_started/leptos_dx.html), 
мы разобрали настройку простого проекта с Leptos и Trunk, включая обработку ошибок WASM в браузере.
Этого простого сетапа вам будет достаточно для начала разработки с Leptos.

Если вы предпочитаете начать используя более полнофункциональный шаблон, демонстрирующий как настроить ряд обыденных
вещей, которые можно встретить в реальных Leptos проектах, например: маршрутизация (она описана далее в книге), 
вставка тегов `<Title>` и `<Meta>` в `<head>`, и несколько других фишек, тогда смело используйте репозиторий
 [шаблона leptos-rs `start-trunk`](https://github.com/leptos-rs/start-trunk) чтобы начать работу.

Шаблон `start-trunk` требует установленных `Trunk` и `cargo-generate`, которые можете поставить выполнив `cargo install trunk` и `cargo install cargo-generate`.

Чтобы создать проект, используя этот шаблон, просто выполните

`cargo generate --git https://github.com/leptos-community/start-csr`

затем выполните

`trunk serve --port 3000 --open`

в директории созданного приложения, чтобы начать разработку вашего приложения. 
Сервер Trunk будет перезагружать страницу с вашим приложением всякий раз, когда вы что-то меняете в исходных файлах,
делая разработку сравнительно плавной.  
```