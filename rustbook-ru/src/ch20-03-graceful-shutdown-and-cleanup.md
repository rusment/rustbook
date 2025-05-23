## Мягкое завершение работы и очистка

Листинг 20-20 асинхронно отвечает на запросы с помощью использования пула потоков, как мы и хотели. Мы получаем некоторые предупреждения про `workers`, `id` и поля `thread`, которые мы не используем напрямую, что напоминает нам о том, что мы не освобождаем все ресурсы. Когда мы используем менее элегантный метод остановки основного потока клавишной комбинацией <span class="keystroke">ctrl-c</span>, все остальные потоки также немедленно останавливаются, даже если они находятся в середине обработки запроса.

Далее, реализуем типаж  `Drop` для вызова `join` у каждого потока в пуле, чтобы они могли завершить запросы, над которыми они работают, перед закрытием. Затем мы реализуем способ сообщить потокам, что они должны перестать принимать новые запросы и завершить работу. Чтобы увидеть этот код в действии, мы изменим наш сервер так, чтобы он принимал только два запроса, после чего корректно завершал работу пула потоков.

### Реализация типажа `Drop` для `ThreadPool`

Давайте начнём с реализации `Drop` у нашего пула потоков. Когда пул удаляется, все наши потоки должны объединиться (join), чтобы убедиться, что они завершают свою работу. В листинге 20-22 показана первая попытка реализации `Drop`, код пока не будет работать.

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch21-web-server/listing-21-22/src/lib.rs:here}}
```

<span class="caption">Листинг 20-22: Присоединение (Joining) каждого потока, когда пул потоков выходит из области видимости</span>

Сначала мы пройдёмся по каждому `worker` из пула потоков. Для этого мы используем `&mut` с `self`, потому что нам нужно иметь возможность изменять `worker`. Для каждого обработчика мы выводим сообщение о том, что он завершает работу, а затем вызываем `join` у потока этого обработчика. Для случаев, когда вызов `join` не удался, мы используем `unwrap`, чтобы заставить Rust запаниковать и перейти в режим грубого завершения работы.

Ошибка получаемая при компиляции этого кода:

```console
{{#include ../listings/ch21-web-server/listing-21-22/output.txt}}
```

Ошибка говорит нам, что мы не можем вызвать `join`, потому что у нас есть только изменяемое заимствование каждого `worker`, а `join` забирает во владение свой аргумент. Чтобы решить эту проблему, нам нужно извлечь поток из экземпляра `Worker`, который владеет `thread`, чтобы `join` мог его использовать. Мы сделали это в листинге 17-15: теперь, когда `Worker` хранит в себе `Option<thread::JoinHandle<()>>`, мы можем воспользоваться методом `take` у `Option`, чтобы извлечь значение из варианта `Some`, тем самым оставляя на его месте `None`. Другими словами, в рабочем состоянии `Worker` будет использовать вариант `Some` содержащий `thread`, а когда мы захотим завершить `Worker`, мы заменим `Some` на `None`, чтобы у `Worker` не было потока для работы.

Итак, мы хотим обновить объявление `Worker` следующим образом:

<span class="filename">Файл: src/lib.rs</span>


```rust
{{#rustdoc_include ../listings/ch21-web-server/no-listing-04-update-drop-definition/src/lib.rs:here}}
```

Теперь давайте опираться на компилятор, чтобы найти другие места, которые нужно изменить. Проверяя код, мы получаем две ошибки:

```console
{{#include ../listings/ch21-web-server/listing-21-22/output.txt}}
```

Давайте обратимся ко второй ошибке, которая указывает на код в конце `Worker::new`; нам нужно обернуть значение `thread` в вариант `Some` при создании нового `Worker`. Внесите следующие изменения, чтобы исправить эту ошибку:

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch21-web-server/no-listing-05-fix-worker-new/src/lib.rs:here}}
```

Первая ошибка находится в нашей реализации `Drop`. Ранее мы упоминали, что намеревались вызвать `take` для параметра `Option`, чтобы забрать `thread` из процесса `worker`. Следующие изменения делают это:

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,not_desired_behavior
{{#rustdoc_include ../listings/ch21-web-server/no-listing-06-fix-threadpool-drop/src/lib.rs:here}}
```

Как уже говорилось в главе 17, метод `take` у типа `Option` забирает значение из варианта `Some` и оставляет вариант `None` в этом месте. Мы используем `if let`, чтобы деструктурировать `Some` и получить поток; затем вызываем `join` у потока. Если поток "работника" уже `None`, мы знаем, что этот "работник" уже очистил свой поток, поэтому в этом случае ничего не происходит.

### Сигнализация потокам прекратить прослушивание получения задач

Теперь, после всех внесённых нами изменений, код компилируется без каких-либо предупреждений. Но плохая новость в том, что этот код всё ещё не работает так, как мы этого хотим. Причина заключается в логике замыканий, запускаемых потоками экземпляров Worker: в данный момент мы вызываем join, но это не приводит к завершению потоков, так как они находятся в бесконечном цикле, ожидая новую задачу. Если мы попытаемся удалить ThreadPool в текущей реализации drop, основной поток навсегда заблокируется в ожидании завершения первого потока из пула.

Чтобы решить эту проблему, нам нужно будет изменить реализацию `drop` в `ThreadPool`, а затем внести изменения в цикл `Worker` .

Во-первых, изменим реализацию `drop` `ThreadPool` таким образом, чтобы явно удалять `sender` перед тем, как начнём ожидать завершения потоков. В листинге 20-23 показаны изменения в `ThreadPool` для явного удаления `sender` . Мы используем ту же технику `Option` и `take`, что и с потоком, чтобы переместить `sender` из `ThreadPool`:

<span class="filename">Файл: src/lib.rs</span>

```rust,noplayground,not_desired_behavior
{{#rustdoc_include ../listings/ch21-web-server/listing-21-23/src/lib.rs:here}}
```

<span class="caption">Листинг 20-23. Явное удаление <code>sender</code> перед ожиданием завершения рабочих потоков</span>

Удаление `sender` закрывает канал, что указывает на то, что сообщения больше не будут отправляться. Когда это произойдёт, все вызовы `recv`, выполняемые рабочими процессами в бесконечном цикле, вернут ошибку. В листинге 20-24 мы меняем цикл `Worker` для корректного выхода из него в этом случае, что означает, что потоки завершатся, когда реализация `drop` `ThreadPool` вызовет для них `join`.

<span class="filename">Файл: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch21-web-server/listing-21-24/src/lib.rs:here}}
```

<span class="caption">Листинг 20-24: Явный выход из цикла, когда <code>recv</code> возвращает ошибку</span>

Чтобы увидеть этот код в действии, давайте изменим `main`, чтобы принимать только два запроса, прежде чем корректно завершить работу сервера как показано в листинге 20-25.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch21-web-server/listing-21-25/src/main.rs:here}}
```

<span class="caption">Код 20-25. Выключение сервера после обслуживания двух запросов с помощью выхода из цикла</span>

Вы бы не хотели, чтобы реальный веб-сервер отключался после обслуживания только двух запросов. Этот код всего лишь демонстрирует, что корректное завершение работы и освобождение ресурсов находятся в рабочем состоянии.

Метод `take` определён в типаже `Iterator` и ограничивает итерацию максимум первыми двумя элементами. `ThreadPool` выйдет из области видимости в конце `main` и будет запущена его реализация `drop`.

Запустите сервер с `cargo run` и сделайте три запроса. Третий запрос должен выдать ошибку и в терминале вы должны увидеть вывод, подобный следующему:

<!-- manual-regeneration
cd listings/ch21-web-server/listing-21-25
cargo run
curl http://127.0.0.1:7878
curl http://127.0.0.1:7878
curl http://127.0.0.1:7878
third request will error because server will have shut down
copy output below
Can't automate because the output depends on making requests
-->

```console
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 1.0s
     Running `target/debug/hello`
Worker 0 got a job; executing.
Shutting down.
Shutting down worker 0
Worker 3 got a job; executing.
Worker 1 disconnected; shutting down.
Worker 2 disconnected; shutting down.
Worker 3 disconnected; shutting down.
Worker 0 disconnected; shutting down.
Shutting down worker 1
Shutting down worker 2
Shutting down worker 3
```

Вы возможно увидите другой порядок рабочих потоков и напечатанных сообщений. Мы можем увидеть, как этот код работает по сообщениям: "работники" номер 0 и 3 получили первые два запроса. Сервер прекратил принимать соединения после второго подключения, а реализация `Drop` для `ThreadPool` начинает выполняется ещё тогда, когда как работник 3 даже не приступил к выполнению своей работы. Удаление `sender` отключает все рабочие потоки от канала и просит их завершить работу. Каждый рабочий поток при отключении печатает сообщение, а затем пул потоков вызывает `join`, чтобы дождаться, пока каждый из рабочих потоков завершится.

Обратите внимание на один интересный аспект этого конкретного запуска: ThreadPool удалил `sender`, и прежде чем какой-либо из работников получил ошибку, мы попытались присоединить (join) рабочий поток с номером 0. Рабочий поток 0 ещё не получил ошибку от `recv`, поэтому основной поток заблокировался, ожидания завершения потока работника 0. Тем временем, работник 3 получил задание, а затем каждый из рабочих потоков получил ошибку. Когда рабочий поток 0 завершился, основной поток ждал окончания завершения выполнения остальных рабочих потоков. В этот момент все они вышли из своих циклов и остановились.

Примите поздравления! Теперь мы завершили проект; у нас есть базовый веб-сервер, использующий пул потоков для асинхронных ответов. Мы можем выполнить корректное завершение работы сервера, очистив все потоки в пуле.

Вот полный код для справки:

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch21-web-server/no-listing-07-final-code/src/main.rs}}
```

<span class="filename">Файл: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch21-web-server/no-listing-07-final-code/src/lib.rs}}
```

Мы могли бы сделать ещё больше! Если вы хотите продолжить совершенствование этого проекта, вот несколько идей:

- Добавьте больше документации в `ThreadPool` и его публичные методы.
- Добавьте тесты для функционала, реализуемого библиотекой.
- Замените вызовы `unwrap` на более устойчивую обработку ошибок.
- Используйте `ThreadPool` для выполнения некоторых других задач, помимо обслуживания веб-запросов.
- На [crates.io](https://crates.io/) найдите крейт для работы с пулами потоков и на его основе реализуйте аналогичный веб-сервер. Затем сравните его API и надёжность с реализованным нами пулом потоков.

## Итоги

Отличная работа! Вы сделали это к концу книги! Мы хотим поблагодарить вас за то, что присоединились к нам в этом путешествии по языку Rust. Теперь вы готовы реализовать свои собственные проекты на Rust и помочь с проектами другим людям. Имейте в виду, что сообщество Rust разработчиков довольно гостеприимно, они с удовольствием постараются помочь вам с любыми трудностями, с которыми вы можете столкнуться в своём путешествии по Rust.
