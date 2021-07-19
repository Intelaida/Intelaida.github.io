---
title: "Утечки не нарушают безопасность памяти"
author: Huon Wilson
original: https://huonw.github.io/blog/2016/04/memory-leaks-are-memory-safe/
translator: Станислав Ткач
categories: обучение
---

Ошибки доступа к памяти и утечки памяти представляют собой две категории ошибок, 
которые привлекают больше всего внимания, так что на предотвращение или хотя бы 
уменьшение их количества направлено много усилий. Хотя их название и предполагает 
схожесть, однако они в некотором роде диаметрально противоположны и решение одной 
из проблем не избавляет нас от второй. Широкое распространение управляемых языков 
подтверждает эту идею: они предотвращают некоторые ошибки доступа к памяти, беря 
на себя работу по освобождению памяти.

Проще говоря: **нарушение доступа к памяти - это какие-то действия с некорректными 
данными, а утечка памяти - это *отсутствие* определённых действий с корректными данными**. 
В табличной форме:

```
                    Корректные данные     Некорректные данные
Используются        OK                    Ошибка доступа к памяти
Не используются     Утечка памяти         OK
```

<!--cut-->

Лучшие программы выполняют только действия из ОК-ячеек: они манипулируют корректными 
данными и не манипулируют некорректными. Приемлемые программы могут также содержать 
некоторые корректные, но неиспользуемые данные (утечки памяти), а плохие пытаются 
использовать некорректные данные.

Когда язык обещает *безопасную* работу с памятью, как это делает Rust, это не гарантирует 
невозможность *утечек* памяти.

#### Последствия
Наиболее важная разница между ошибками доступа к памяти и утечками на практике проявляется 
в потенциальных последствиях: первые приводят к весьма серьёзным проблемам, в то время 
как вторые просто раздражают.

Безопасность работы с памятью является ключевым элементом любой другой формы безопасности/корректности. 
Если программа допускает ошибки с памятью, о ее поведении сложно давать какие-либо гарантии, 
поскольку не исключается возможность повреждения памяти. Злоумышленники смогут воспользоваться 
ошибками доступа к памяти в такой программе для [считывания конфиденциальных ключей непосредственно 
из памяти сервера](https://ru.wikipedia.org/wiki/Heartbleed) или выполнения произвольного кода 
на вашем компьютере.

С другой стороны, утечка памяти в худшем случае приведёт к отказу в обслуживании: полезная программа 
аварийно завершится по причине использования слишком большого объёма памяти, а компьютер из-за ее 
дефицита может перестать отвечать на запросы. Такая ситуация также может вызвана злоумышленником, 
однако методики по борьбе с этим давно выработаны. Конечно, отказ в обслуживании очень раздражает и 
в некоторых местах является критической проблемой, но потенциальные ошибки доступа к памяти представляют 
собой не меньшую, а может и большую, проблему. Кроме того, учитывая непредсказуемость возможных ошибок 
доступа к памяти, это может привести всё к тому же отказу в обслуживании.

В итоге, большинство языков программирования предпочитают мириться с утечками памяти (допуская отсутствие 
освобождения или очистки данных после последнего использования), но не с ошибками доступа к памяти. 
Таким образом, большинство "безопасных" языков гарантируют, что программы, написанные на них, не содержат 
таких ошибок, если только вы сознательно не решите обойти ограничения (например, используя модуль `ctypes ` 
в Python или ключевое слово `unsafe ` в Rust). Что касается утечек, то с ними пытаются (как правило, сильно) 
бороться, но не дают никаких гарантий.

#### `delete free`
Ошибки доступа к памяти возникают по нескольким причинам, но одна категория выделяется, когда мы 
обсуждаем управление памятью. Процитирую Википедию:

> Ошибки работы с динамической памятью - некорректное использование динамической памяти и указателей:

> - Висячие указатели — указатели, хранящие адрес объекта, который был удалён.

> - Двойное освобождение — повторный вызов `free` может преждевременно удалить новый объект, располагающийся 
  по тому же адресу. Если же адрес не был повторно использован, тогда могут возникнуть другие проблемы, 
  особенно в аллокаторах, которые используют свободные списки (free lists).
  
> - Некорректное освобождение — передача некорректного адреса в функцию `free` может испортить кучу.

> - Обращение по нулевому указателю приведёт к исключению или аварийному завершению программы в большинстве 
  окружений, но может и вызвать порчу данных в ядре операционной системы или в системах без защиты памяти 
  или при применении большого отрицательного смещения.

В этом списке только обращение по нулевому указателю не вызвано некорректным освобождением памяти 
(вызовом функции `free` для того, чтобы пометить память неиспользуемой и возврата операционной системе). 
Таким образом, можно избежать всех этих проблем, просто никогда не вызывая `free`: если память никогда 
не освобождается, то и проблем, связанных с этим, не возникнет. Обращаясь к вышеприведённой таблице: 
убрав освобождение памяти, мы убираем колонку "некорректные данные" — все данные всегда будут корректными.

Конечно, простой запрет на вызов `free` имеет свои недостатки (хотя имеет и преимущества, помимо отсутствия 
проблем с освобождением памяти: не возникает сложностей с пониманием времени жизни данных, что упрощает 
написание многих параллельных алгоритмов). В частности, становится проблематично писать программы так, 
чтобы они не исчерпали всю доступную память. Однако, компьютеры, в отличии от людей, не ошибаются, 
так что, возможно, мы можем переложить вызов `free` на их плечи...

#### Оптимизация утечек
Значительная часть современного кода написана на языках, направленных на обеспечение безопасности работы 
с памятью, таких как Java, Javascript, Python или Ruby. Они обходятся без явного вызова `free` и автоматически 
управляют памятью (отсюда и название "управляемые языки") при помощи "сборщика мусора" (garbage collector), 
встроенного в среду выполнения языка.

По сути, сборка мусора — это способ предоставить программисту и программе иллюзию бесконечной памяти и 
избавиться от необходимости тщательно отслеживать момент, когда память можно освободить. Вы сосредотачиваетесь 
на логике предметной области, а сборщик мусора автоматически освобождает гарантированно незадействованные 
участки памяти. Практически все сборщики мусора консервативно определяют данные, которые можно удалить, 
если на них нет ссылок (таким образом, сборщик мусора должен отслеживать или иметь доступ к всем выделениям 
памяти).

Стоит заметить, что качественные реализации сборщика мусора обеспечивают дополнительные преимущества: 
выделение памяти, как правило, реализуется в виде простого сдвига указателя при наличии поколений объектов, 
а возможности перемещающего сборщика мусора улучшают локальность кеша (что особенно полезно, учитывая то, 
что доступ к данным и так происходит по указателю в большинстве управляемых языков). Однако, эти особенности 
не относятся к теме статьи.

На практике программистам практически никогда не приходится задумываться о том, что память не бесконечна, но 
безопасность работы с памятью, как и хотелось, обеспечивается. В высокопроизводительном коде зачастую приходится 
прибегать ко всяким хитростям (например, объектным пулам, помогающим избежать выделения памяти при частом 
создании-удалении объектов), чтобы обойти неэффективность сборки мусора. Также возможна ситуация, когда 
данные из-за [забытых ссылок](https://en.wikipedia.org/wiki/Lapsed_listener_problem) живут дольше необходимого.

Тем не менее, цель достигается даже с учётом возникающих на практике проблем: отсутствие вызовов free 
гарантирует отсутствие (некоторых) проблем работы с памятью.

#### Меньше абстракций
Не могу не упомянуть альтернативу автоматическому управлению памятью: вместо того, чтобы пытаться избавиться 
от колонки "Некорректные данные" целиком, можно гарантировать только отсутствие проблем безопасности работы 
с памятью. [Язык программирования Rust](https://www.rust-lang.org/) делает именно это.

Такой подход избавляет от необходимости ручного освобождения ресурсов, хотя функция 
[`drop`](http://doc.rust-lang.org/std/mem/fn.drop.html) и позволяет вызвать деструктор раньше времени. 
В отличии от С и С++, на этапе компиляции Rust запрещает дальнейшее использование таких ставших некорректными 
данных и предотвращает ошибки.

Однако, эта модель не гарантирует отсутствие утечек: в пересмотренной таблице для Rust (и языков с 
аналогичным принципом) всё ещё имеется ячейка "Утечка памяти".

```
                    Корректные данные     Некорректные данные
Используются        OK                    Невозможно
Не используются     Утечка памяти         OK
```

Многие не видят разницы между "утечками памяти" и "безопасностью работы с памятью". Услышав, что 
Rust гарантирует безопасность работы с памятью, они просто ожидают защиту от утечек и не понимают, 
что этот язык умеет такого, что не умеет современный С++ в области низкоуровневого программирования. 
**Rust не допускает ошибок доступа к памяти, но не исключает утечек.**

#### `std::mem::forget`
И, наконец, в Rust присутствует функция [`forget`](https://doc.rust-lang.org/std/mem/fn.forget.html), 
которая помечает данные как освобождённые, предотвращая дальнейший доступ к ним, но не вызывает деструктор, 
что потенциально ведёт к утечке памяти. Долгое время эта функция была помечена как небезопасная (unsafe), 
то есть Rust неявно подразумевал, что утечки памяти - это то, что программист должен выбирать сознательно, 
аналогично безопасности работы с памятью. Однако, на практике указатели с подсчётом ссылок или взаимная 
блокировка потоков могут приводить к утечкам. В итоге функцию `forget` 
[сделали безопасной](https://github.com/rust-lang/rfcs/blob/master/text/1066-safe-mem-forget.md), 
сфокусировавшись на предотвращении ошибок доступа к памяти, хотя и прилагая все возможные усилия в борьбе 
с утечками, впрочем, как и все остальные языки.

Как и современный С++, Rust довольно неплохо справляется с задачей: управление ресурсами, основанное на 
RAII, а именно деструкторы, являются [мощным средством](http://blog.skylight.io/rust-means-never-having-to-close-a-socket/) 
для управления памятью ([и не только](http://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html#locks)), 
особенно в комбинации с семантикой перемещения по умолчанию, используемой в Rust. Отсутствие утечек не 
гарантируется по двум причинам:

- Дать формальное определение утечке памяти непросто (по меньшей мере, полезность такого 
  определения зависит от контекста).
- Существуют относительно редкие граничные случаи, которые, по всей видимости, невозможно 
  статически предотвратить без накладных расходов.

Стандартная библиотека Rust [ожидает](https://github.com/ruRust/rustonomicon/blob/master/src/leaking.md), 
что утечки безопасны, хотя и могут приводить к некорректной работе. Другими словами, вы можете получить 
нежелательное поведение, если данные не будут освобождены, но последствия будут менее разрушительны, 
чем ошибки сегментации (segmentation fault) или повреждения памяти.