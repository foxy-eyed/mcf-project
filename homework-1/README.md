# ДЗ#1

Содержание
 - [ES модель](#es-модель)
 - [Модель данных](#модель-данных)
 - [Общая модель коммуникаций](#общая-модель-коммуникаций)
 - [Реализация](#реализация)
 - [Про сторонние решения](#про-сторонние-решения)

## ES модель

![ES модель](https://github.com/foxy-eyed/mcf-project/blob/main/homework-1/img/ES_model.jpg)
[Открыть в Miro](https://miro.com/app/board/uXjVNNeyOQc=/?share_link_id=321736457182)

Выделенные контексты:
  - Всё, что связано с заказами и их выполнением:
    - Кабинет клиента (создание и управление заказами);
    - Кабинет воркера (исполнение заказов);
    - BackOffice для менеджеров (управление услугами, контроль качества, ручные вознаграждения исполнителям);
    - Биллинг;
    - Матчер (подбор воркера и оценка стоимости);
  - Всё связанное с наймом:
    - Кабинет кандидата (отклик и тестирование);
    - BackOffice для HR-менеджеров (работа с откликами, управление тестами);
  - Склад.
     
### Заметки к ES модели
  - История [US-010] в схеме не отражена, потому что я сделала вывод, что отдельной регистрации тестировщиков в системе
не будет (предполагаю SSO с существующей системой Happy Cat Box).
  - В [US-030] неявно упоминается, что клиент может отменить заказ, добавила соотв. событие.
  - Обнаружено противоречение между [US-030] (статусы задачи может менять только клиент) и [US-110]
(заказ автоматически проваливается). Я посчитала логичным проваливать заказ автоматически по крону: 
проверяем все текущие заказы, у которых наступило время выполнения, но они до сих пор не взяты в работу 
(кот-воркер на нажал соотв. кнопку в приложении).
  - История со скидками [US-040] не влияет на ES модель.
  - [US-070] — эта история выглядит избыточно на данный момент, потому что в конечном итоге статус выполнения заказа
зависит от команды заказчика. Возможно, тут в будущем потребуется выделить отдельный статус заказа, по которому можно 
будет понять, что в реальности клиент заказ принял, а в системе по какой-то причине не отметил.
  - [US-101] — детали реализации, которые не влияют на модель.
  - [US-081] — не пользовательская история, а скорее ограничение, которое стоит учитывать при выборе решения. 
В частности, ф-л приема заявок вместе с процессом их рассмотрения имеет смысл сразу же вынести в отдельный микросервис,
чтобы его падение в случае DDOS не зааффектило остальные части системы.
  - [US-250] и [US-260] со всеми вложенными историями отображена как внешняя система, ниже есть пояснение.

## Модель данных

![Data model](https://github.com/foxy-eyed/mcf-project/blob/main/homework-1/img/data_model.jpg)

[Открыть в Miro](https://miro.com/app/board/uXjVNMS8xzo=/?share_link_id=43973897252)

### Заметки к модели данных

Так вышло, что при построении ES-модели я выделила много небольших контекстов, и уже в процессе рисования на ней связей
между событиями и read-моделями стало понятно, что ряд контекстов очень тесно связан на уровне данных.
Поэтому к моменту построения модели данных я уже решила объединить их в более крупные: заказы, склад и найм. 
Таким образом, большинство стриминговых событий исчезло со схемы.

## Общая модель коммуникаций

![Communication model](https://github.com/foxy-eyed/mcf-project/blob/main/homework-1/img/system_schema.jpg)

[Открыть в Miro](https://miro.com/app/board/uXjVNLnCn5E=/?share_link_id=938097208728)

### Заметки к модели коммуникаций

В итоге получается, что полученные три больших контекста связаны минимумом **стриминговых событий**:
  - стримить данные о принятых воркерах из **Найма** в **Заказы** после приёма;
  - стримить данные о заказах, взятых в работу, из **Заказов** на **Склад** для комплектации.

**Бизнес-события**, которые как-то торчат наружу — изменение статуса заказов для тотализатора.

## Реализация

Предлагаю выделить на данный момент **три приложения**:
  - **Заказ услуг** — core-монолит, закрывающий ключевую функциольность системы — заказ услуг, их исполнение, расчёты. Сюда войдут следующие контексты:
    - Кабинет клиента;
    - Кабинет воркера;
    - BackOffice для менеджеров;
    - Биллинг;
    - Инновационный матчер;
  - **Найм воркеров**;
  - **Склад**.

Микроконтексты внутри каждого сервиса требуется реализовать в виде отдельных модулей, минимизировав связанность
(для этого можно зафиксировать публичные интерфейсы и контролировать только их вызовы с помощью фитнес-функций).

Каждый их трёх сервисов имеет **собственную реляционную БД**.

Немногочисленные **коммуникации** между сервисами я предлагаю реализовать с помощью православных HTTP запросов к API консьюмеров.
Поскольку нам не нужно получать ничего в ответ, эти запросы имеет смысл делать **асинхронно в бэкграунд-процессе**.

**Нотификации** являются частью бизнес-процессов и правила их генерации и рассылки очень сильно зависят от контекста.
Поэтому у каждого сервиса будет собственный модуль для генерации писем под конкретный процесс, а сами письма должны
генерироваться отправляться **в бэкграунд-процессе**.

### Заметки к реализации
Распиливать core-монолит **Заказов** на микросервисы сразу я бы не стала:
  - видение продукта со стороны заказчика пока сырое, а будущее продукта и всего бизнес-направления туманно,
  - у компании отсутствует опыт в автоматизации процессов,
  - есть пожелание быстрого старта и проверки гипотез.

Вкладывать много времени и ресурсов на разработку микросервисной архитектуры скорее всего нерационально, и пока
неочевидна ценность. Нужен MVP, для этих целей больше подойдет монолит — проще и дешевле.

**Склад** и **Найм** — вспомогательные контексты, не особенно связанные с основной функциональностью и легко выносятся,
поэтому выделяем их сразу. Кроме того, нам не нужно, чтобы основной сервис завалили заявками злобные конкуренты.

Неплохой кандидат на выделение — **Матчер**, но с текущей мизерной функциональностью смысла в вынесении нет.

**Нотификации** я сначала вытащила в отдельный контекст и потенцильно рассматривала вариант вынесения в отдельный сервис.
Затем поняла, что если делегировать ему генерацию нотификаций, то нужно будет передавать ему слишком много контекста,
и это очень усложнит и сам сервис, и коммуникации с ним. Делегировать только отправку писем не имеет смысла при текущих
требованиях. Поэтому склоняюсь к тому, что логика генерации писем и адресатов должна быть частью бизнес-транзакции.

## Про сторонние решения
  - Я бы склонила заказчика вынести тотализатор за границы системы и реализовать идею no-code инструментами (нужен рисёч). 
Мы можем отсылать хуки наружу на события «Заказ выполнен» и «Заказ провален», на которые будет завязан пересчёт
вознаграждений в какой-нибудь условной google sheets.
Аргументы:
    - Это побочный ф-л, который не является приоритетным по словам самого заказчика;
    - Простая логика, для которой не потребуется хитрого кода и дальнейшей поддержки;
    - Серая зона — лучше держать отдельно;
    - Озвучена готовность часть процессов вести в условной «тетрадке».
  - Чисто теоретически можно было бы рассмотреть вариант использования внешних решений для процессов найма. Но я решила,
что стоит оставить эту функциональность в своих руках, потому что озвучено требование «Для бизнеса критично проверять
новые гипотезы по отсеву котов и изменять уже существующие с максимальной скоростью и надёжностью».
Это значит, что нам потребуется гибкость в настройке тестовых испытаний.
