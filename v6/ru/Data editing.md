---
title: Редактирование данных
layout: default
---
Под редактированием данных подразумеваются такие действия как создание заказов, манипуляция с их составом, резервирование столов, регистрация гостей и т. п. Каждое отдельно взятое действие вносит небольшое точечное изменение — например, `AddOrderItemProduct` добавляет в заказ блюдо, а `AddOrderItemModifier` добавляет к блюду модификатор. Многие действия сами по себе могут приводить данные к несогласованному состоянию — к примеру, если блюдо имеет обязательные модификаторы, то добавление блюда без модификаторов нарушило бы соответствующее правило предметной области. Комбинируя эти действия, можно добавить в заказ блюдо с модификаторами. При этом важно, чтобы набор действий выполнялся с соблюдением принципа «всё или ничего» и переводил данные из одного согласованного состояния в другое согласованное состояние. Для обеспечения транзакционности при изменении данных вводится понятие *сессии редактирования*. 

## Сессии редактирования ##
Сессия редактирования — это некое подобие транзакций в базах данных. Все действия, даже одиночные, выполняются в рамках сессий следующим образом:

1. Создаётся сессия [`IEditSession`](https://iiko.github.io/front.api.sdk/v6/html/T_Resto_Front_Api_Editors_IEditSession.htm) с помощью вызова [`PluginContext.Operations.CreateEditSession`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_IOperationService_CreateEditSession.htm).
2. В рамках сессии выполняется одно или несколько действий.
3. Сделанные изменения сохраняются методом [`PluginContext.Operations.SubmitChanges`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_IOperationService_SubmitChanges.htm).

Сделанные на втором шаге изменения не видны до их успешного сохранения. На третьем шаге либо все изменения успешно записываются, либо все откатываются и генерируется исключение. 

## Синхронизация ##
Доступ к данным синхронизирован с помощью блокировок и ревизий. На время редактирования объекты блокируются, но через API напрямую управлять блокировками невозможно — объекты блокируются автоматически при сохранении изменений (`SubmitChanges`). Это соответствует концепции *оптимистических блокировок*: на этапах создания сессии редактирования и выполнения действий объекты не блокируются, а при сохранении сделанных изменений происходит проверка, не изменились ли они в это же время кем-то другим. При каждом изменении объектам назначается новая ревизия, что позволяет различать разные версии одного и того же объекта. С учётом низкой конкуренции за параллельное редактирование одних и тех же объектов такой подход позволяет упростить программный интерфейс (нет методов управления блокировками, не надо заботиться об их корректном освобождении) и обеспечить доступность данных (невозможно надолго заблокировать объект, невозможно «забыть» отпустить блокировку, в случае аварийного завершения работы плагина объект не зависнет в заблокированном состоянии).

Следует учитывать некоторые особенности реализации:

- Само приложение iikoFront может блокировать объекты на длительное время (*пессимистические блокировки*). Например, при переходе на экран редактирования заказа соответствующий заказ будет заблокирован как минимум на всё время нахождения на этом экране (дальше зависит от того, на какой экран перейдёт пользователь). В это время редактировать заблокированный заказ через API невозможно. Имеет смысл предупредить официантов, чтобы они, отходя от стационарного терминала, блокировали экран, либо указать небольшой интервал времени для автоблокировки (по умолчанию 10 минут). Кроме того, объекты могут быть заблокированы ненадолго без участия пользователя (например, при внесении изменений по таймеру).
- При блокировании какого-либо объекта вместе с ним автоматически блокируются и связанные объекты. Так, например, при блокировании банкетного заказа также будет заблокирован соответствующий резерв. Если хоть один из объектов не удалось заблокировать, операция завершается неуспешно.
- Для корректной работы синхронизации доступа к данным необходимо с помощью iikoOffice в настройках группы назначить главную кассу. Плагин, использующий функции редактирования данных, предпочтительно устанавливать на главную кассу. Установка на другие терминалы допускается, но в некоторых сценариях возможны разовые отказы в применении изменений, вызванные деталями реализации. Начиная с [V9Preview3](https://www.nuget.org/packages/Resto.Front.Api.V9Preview3), данное ограничение [`снято`]({{ site.baseurl }}/2024/07/05/versioned-entity-revision.html).

## Выполнение операций непрерываемой серией ##
Операции ([`IOperationService`](https://iiko.github.io/front.api.sdk/v6/html/Methods_T_Resto_Front_Api_IOperationService.htm)), требующие синхронизации для редактирования данных, блокируют и затем разблокируют объекты при каждом вызове.
Соответственно, при последовательном вызове нескольких операций каждая из них будет независимо от других блокировать данные, вносить изменения и затем разблокировать, между вызовами операций могут «вклиниться» другие желающие редактировать эти же данные, в результате часть наших операций выполнится успешно, а часть может завершиться ошибками [`EntityAlreadyInUseException`](https://iiko.github.io/front.api.sdk/v6/html/T_Resto_Front_Api_Exceptions_EntityAlreadyInUseException.htm), [`EntityModifiedException`](https://iiko.github.io/front.api.sdk/v6/html/T_Resto_Front_Api_Exceptions_EntityModifiedException.htm), некоторые операции могут стать неприменимыми с учётом чужих правок.

Например, в плагине, принимающем из внешнего источника (веб-сайт, агрегатор) заказы на доставку, понадобилось создать доставку с проведённой внешней предоплатой.
Выполнить всё это атомарно в рамках одной сессии редактирования невозможно, поскольку проведение платежа — необратимая операция, выполняется отдельно, поэтому сначала создаём доставку и заполняем её поля, включая добавление непроведённой внешней предоплаты ([`CreateEditSession`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_IOperationService_CreateEditSession.htm), [`CreateDeliveryOrder`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_Editors_IEditSession_CreateDeliveryOrder_1.htm), [`AddExternalPaymentItem`](https://iiko.github.io/front.api.sdk/v6/html/Overload_Resto_Front_Api_Editors_IEditSession_AddExternalPaymentItem.htm) и пр.), сохраняем эти изменения ([`SubmitChanges`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_IOperationService_SubmitChanges.htm)), а затем пытаемся провести предоплату ([`ProcessPrepay`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_IOperationService_ProcessPrepay.htm)).
Между этими операциями ([`SubmitChanges`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_IOperationService_SubmitChanges.htm) и [`ProcessPrepay`](https://iiko.github.io/front.api.sdk/v6/html/M_Resto_Front_Api_IOperationService_ProcessPrepay.htm)) данные разблокированы и доступны для редактирования любому желающему.
Те, кто подписан на изменения доставок, могут успеть получить уведомление о создании нами новой доставки и внести в неё изменения прежде, чем мы проведём предоплату.
Писать код, который не боится быть прерванным и способен продолжать работать, подстраиваясь под чужие правки, трудоёмко.
Для удобства реализации подобных сценариев добавлена возможность выполнения нескольких операций одной непрерываемой серией.

[`ExecuteContinuousOperation`](https://iiko.github.io/front.api.sdk/v6/html/Overload_Resto_Front_Api_Extensions_OperationServiceExtensions_ExecuteContinuousOperation.htm) — специальная операция, внутри которой можно последовательно выполнить несколько других операций одной непрерываемой серией.
В коде плагина нужно собрать серию операций в одну функцию или лямбду и передать её как callback в метод `ExecuteContinuousOperation`, который вызовет этот callback, передав ему на вход специальный экземпляр сервиса `IOperationService`, предназначенный для непрерываемого выполнения операций:

```cs
PluginContext.Operations.ExecuteContinuousOperation(
    operations =>
    {
        ...
        operations.SubmitChanges(...);
        ...
        operations.ProcessPrepay(...);
        ... 
    });
```

Следует обратить внимание, что корневая операция `ExecuteContinuousOperation` вызывается через общий сервис `PluginContext.Operations`, а вложенные операции — через полученный лямбдой на вход экземпляр сервиса (названный в примере выше `operations`).
Технически, вызываемые через специальный экземпляр сервиса операции работают точно так же, но не разблокируют после себя данные, то есть каждая операция, требующая синхронизации, блокирует данные, если они ещё не были заблокированы предыдущими операциями, вносит изменения и оставляет данные заблокированными для последующих операций.
Это гарантирует, что никто другой не сможет «угнать» блокировку и «вклиниться», и наши последующие операции операции над этими же объектами не столкнутся с `EntityAlreadyInUseException`.
Данные разблокируются при возврате управления из лямбды.

Следует учесть следующие ограничения:

- Данная функция не имеет никакого отношения к атомарности или транзакционности, каждая вложенная операция выполняется сама по себе и сохраняет изменения немедленно.
Если любая из операций завершится ошибкой (сгенерирует исключение), предыдущие операции не отменятся.
- Непрерываемость операции не означает, что никто другой не может вообще ничего делать.
Имеется в виду, что нельзя прервать последовательное редактирование нами какого-то определённого объекта.
Другие плагины и само приложение iikoFront могут параллельно с нами редактировать какие-то другие объекты, которые мы не трогали и которые нами не заблокированы.
Данные блокируются по мере необходимости, поэтому если посреди непрерываемой серии после успешного выполнения операций над одним объектом мы решим внести изменения в другой объект, мы вполне можем получить `EntityAlreadyInUseException`.
- Поскольку затронутые данные остаются заблокированными в течение всего времени работы переданной в `ExecuteContinuousOperation` лямбды и это может ограничивать работу пользователя, других плагинов и функций приложения, необходимо выполнить серию операцию как можно быстрее.
В рамках непрерываемой сессии следует осторожно выполнять запросы к внешним сервисам, к оборудованию, не допуская возможности зависнуть надолго, а лучше вообще избегать внешнего I/O.
По возможности, стоит всю подготовительную часть выполнить заранее, вне непрерываемой сессии. Если такой возможности нет, необходимо, по крайней мере, убедиться в наличии разумных таймаутов ожидания.


## Заглушки ##
Так как результаты действий невозможно получить до сохранения всей сессии, при выполнении последовательности действий в рамках одной сессии бывает необходимо сослаться на создаваемый, но ещё не существующий объект. Например, вслед за созданием заказа нужна возможность добавить в него гостя, этому гостю — блюдо, а блюду — модификатор, хотя при этом ещё нет ни заказа, ни гостя, ни блюда. Для этой цели вводится понятие заглушек объектов — неких фиктивных, однако, однозначных указателей на объекты. Действия создания объектов, такие как `CreateOrder` или `AddOrderGuest`, возвращают заглушки вида `INew...Stub`, которые в рамках той же сессии можно использовать вместо будущих объектов.

Большинство методов редактирования принимают в качестве аргументов подобные заглушки, что позволяет передавать в них как существующие, так и новые объекты. Например, метод `SetOrderType` принимает `IOrderStub`, поэтому можно задать тип и уже существующему заказу (`IOrder : IOrderStub`), и только создаваемому (`INewOrderStub : IOrderStub`). 

Впрочем, некоторые действия могут требовать строго одного из двух — нового или существующего объекта, в таком случае в сигнатуре метода будет использован не базовый тип, а один из наследников.

## Ожидаемые исключения ##
При попытке сохранить изменения могут возникать различные исключения. Некоторые из них могут свидетельствовать об ошибке в коде плагина (например, `ArgumentNullException` или `ArgumentOutOfRangeException`), подавлять такие исключения не рекомендуется (лучше исправить ошибку в коде). Однако, некоторые исключения предугадать или предотвратить невозможно, их следует перехватывать и корректно обрабатывать: 

- `EntityAlreadyInUseException` — попытка применить изменения к объекту, который в этот момент заблокирован. Можно повторить попытку позднее.
- `EntityModifiedException` — попытка применить изменения к старой версии объекта. Это означает, что после того, как плагин прочитал объект, этот объект был кем-то изменён. Необходимо повторно прочитать объект и, если запланированные изменения всё ещё актуальны, повторно применить их.
- `PermissionDeniedException` — попытка выполнить действия, не имея достаточных прав. Если пользователь хочет, чтобы плагин мог выполнять эти действия, ему следует выдать соответствующие права с помощью iikoOffice.
- ...
 
## Синтаксический сахар ##
Иногда требуется выполнить всего одно действие, при этом явное создание сессии редактирование выглядит громоздким. Для таких случаев к `IOperationService` реализованы вспомогательные extension-методы, которые создают сессию редактирования, выполняют единственное действие, сохраняют изменения и возвращают результат действия. В принципе всё то же самое можно было написать вручную. Не рекомендуется использовать эти обёртки, если предполагается одновременное выполнение нескольких действий.
