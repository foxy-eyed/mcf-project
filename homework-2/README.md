# ДЗ#2

Содержание
- [Business domain](#business-domain)
  - [Поддомены](#поддомены)
  - [Core Domain Chart](#core-domain-chart)
- [Bounded contexts](#bounded-contexts)
  - [Отличия от ES-модели](#отличия-от-es-модели)
  - [Переделанная ES-модель](#переделанная-es-модель)
- [Архитектурные характеристики](#архитектурные-характеристики)
- [Выбор архитектуры](#выбор-архитектуры)
- [Структура и коммуникации](#структура-и-коммуникации)

## Business domain
**Цель** компании Make Cats Free — повышение продуктивности котов-тестировщиков за счет делегирования рутинных задач
внешним наемным исполнителям.

**Конкурентные преимущества:**
- передовые алгоритмы отсева воркеров;
- самый совершенный матчинг.

### Поддомены
- Найм воркеров;
- Матчинг воркеров под задачи;
- Управление делегированием задач;
- Контроль качества исполнения задач;
- Оснащение воркеров;
- Поддержание мотивации менеджеров.

| Вид поддомена                      | Конкурентное преимущество | Сложность                        | Изменчивость |
|------------------------------------|---------------------------|----------------------------------|--------------|
| Найм воркеров                      | да                        | высокая                          | высокая      |
| Матчинг воркеров под задачи        | да                        | низкая, но в перспективе высокая | высокая      |
| Управление делегированием задач    | нет                       | высокая                          | высокая      |
| Контроль качества исполнения задач | нет                       | низкая                           | низкая       |
| Оснащение воркеров                 | нет                       | низкая                           | низкая       |
| Поддержание мотивации менеджеров   | нет                       | низкая                           | низкая       |

### Core Domain Chart

![Core domain chart](https://github.com/foxy-eyed/mcf-project/blob/main/homework-2/img/core_domain_chart.jpg)

[Открыть в Miro](https://miro.com/app/board/uXjVNJgr-44=/?moveToWidget=3458764571280237351&cot=14)

## Bounded contexts

| Вид поддомена                      | Предполагаемый вид поддомена | Выделенный боундед-контекст                                                                                |
|------------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------|
| Найм воркеров                      | Core                         | - Прием заявок от кандидатов<br>- Управление вакансиями и тестами                                          |
| Матчинг воркеров под задачи        | Core                         | Инновационный матчер                                                                                       |
| Управление делегированием задач    | Supporting                   | - Заказ и трекинг выполнения задач<br>- Менеджмент услуг<br>- Расчет и выплата вознаграждений исполнителям |
| Контроль качества исполнения задач | Generic                      | Экспертиза услуг                                                                                           |
| Оснащение воркеров                 | Generic                      | - Комплектация заказов<br>- Работа с поставщиками печенья                                                  |
| Поддержание мотивации менеджеров   | Generic                      | Тотализатор                                                                                                |

![Bounded contexts scheme](https://github.com/foxy-eyed/mcf-project/blob/main/homework-2/img/bounded_contexts.jpg)

[Открыть в Miro](https://miro.com/app/board/uXjVNJgr-44=/?moveToWidget=3458764571330933669&cot=14)

### Отличия от ES-модели
1. Для начала, очевидно, что при построении ES-модели я использовала для контекстов не очень корректный нейминг.
Я мыслила в контексте реализации: как именно это будет поделено на конкретные пользовательские интерфейсы по ролям.
Отсюда «кабинеты», «бэкофисы» и т.п.
2. В домене «Управление делегированием задач» у меня схлопнулось два контекста — «Кабинет клиента» (заказ услуг)
и «Кабинет воркера» (исполнение услуг), т.к. они решают одну бизнес-задачу, но со стороны разных участников процесса.
Один без другого, получается, бессмысленный.
3. Появился новый контекст, которого не было в ES-диаграмме, — «Экспертиза услуг». Он у меня прятался в бэкофисе для
менеджеров в куче с управлением услугами. Если предположить, что конроль качества является зоной ответственности
отдельного подразделения компании, то выделить его будет правильным. Т.к. иначе у нас в одном контексте будут две
разные роли менеджеров, чьи интересы могут в будущем разойтись, как это было у Ибрагима.

### Переделанная ES-модель

![ES model](https://github.com/foxy-eyed/mcf-project/blob/main/homework-2/img/es_diagram.jpg)

[Открыть в Miro](https://miro.com/app/board/uXjVNJgr-44=/?moveToWidget=3458764571431659496&cot=14)

1. Переименовала и перегруппировала контексты;
2. Схлопнулись технические шаги:
   - Выставление инвойсов и транзакции стали одной командой;
   - Нотификации убрала как отдельные команды, теперь просто есть стикеры с комментариями там, где важно о них не забыть;
3. Добавила контекст «Тотализатор» (в предыдущей версии был отмечен как внешняя система);
4. Расположила команды в хронологическом порядке, где возможно.
5. Я оставила стикеры с read-моделями там, где они не особо значимы с т.з. Event Storming, но важны для отображения
стриминговых событий (пунктирные стрелки), чтобы укрупнённо видеть связи на уровне данных. 
Т.о. я себе срезала немного костов на перерисовку модели данных (сорри T_T)

## Архитектурные характеристики

#### Система в целом
> Клиенты пока только тестировщики happy cat box, но компания планирует расширяться в будущем

Прогнозируется рост, значит: *Modifiability*, *Scalability*, *Agility*

> Бизнесу необходим низкий ТТМ (Time To Market), чтобы конкурировать на рынке

*Agility*, *Testability*, *Deployability*

Если вернуться к цели системы — повышение мотивации и лояльности котов-тестировщиков — то стоит включить:

*Availability*, *Usability*

> MCF планирует реализовать проект с нуля по заданным требованиям

Бизнес-проект в самом начале жизненного цикла => много изменений => опять же,
*Modifiability*, *Agility*, *Testability*, *Deployability*

#### Матчинг
> MCF планирует использовать самые передовые алгоритмы отсева воркеров и самый совершенный матчинг

> Исследователи MCF пока думают о том, как должна работать инновационная система матчинга

Скорее всего, будет много изменений — *Modifiability*, *Scalability*, *Agility*, *Testability*, *Deployability*

#### Найм
> Мы ожидаем 1к заявок в день от рандомных котов, также, судя по отзывам, наши конкуренты могут попытаться нас заддосить в этом месте

*Scalability*, *Elasticity*

> Менеджеры должны иметь возможность конфигурировать набор тестов под каждого отдельного кота или вакансию котов

Вроде тут речь про гибкость работы интерфейса управления, но можно предположить, что для совершенствования
системы отбора потребуется и частая модификация кода, поэтому — *Modifiability*, *Agility*, *Testability*, *Deployability*

> По результатам тестов кот либо добавляется в общий пул рабочих с полученными характеристиками (сильными/слабыми сторонами) и прочей информацией о коте (имя, возраст, порода, цвет шерсти, фото), либо бракуется

*Securability* — потому что храним персональные данные котов-воркеров

#### Тотализатор

> Ставки могут делать только менеджеры, которых будет не больше 15 голов. Мы не ожидаем большого количества ставок из-за общего количества заказов.

*Scalability*, *Elasticity*

## Выбор архитектуры

Выбрала **микросервисную архитектуру**, потому что ранее были выявлены требования к таким характеристикам, как
*Agility, Testability, Deployability, Modifiability, Maintainability, Scalability, Elasticity*, и этот стиль им удовлетворяет.

При этом по сложности и стоимости явных ограничений нет:

> Деньги на данный момент не критичны, happy cat box готовы потратить столько, сколько потребуется. Команда разработки будет собрана после нашего анализа требуемой системы.

## Структура и коммуникации

![System structure](https://github.com/foxy-eyed/mcf-project/blob/main/homework-2/img/structure.jpg)

Разделила контексты на **5 сервисов в соответствии с выделенными поддоменами** (на схеме — жёлтые блоки).
Единственное исключение — поддомен «Контроль качества исполнения задач» я внесла в тот же сервис,
который реализует управление делегированием задач.

Сделала я это из соображений сильной связанности на уровне данных, при этом к ф-лу экспертизы нет особенных требований,
которые могли бы обосновать вынесение его в отдельный сервис (типа независимого деплоя, повышенной нагрузки или
изменяемости и т.п.) Единственное сомнение — то, что в системе появятся две роли менеджеров, которые потенциально в
будущем могут транслировать конкурирующие запросы. Но пока об этом ничего не известно.

Каждый из сервисов имеет собственную реляционную БД.

Каждый сервис, который объединяет в себе более одного контекста, должен реализовывать архитектуру **Modular Monolith**.

Все коммуникации — **синхронные**.
