---
title: Ревизии для IVersionedEntity
layout: default
tags: v9preview2 v9
---

В API V9Preview2 изменена работа ревизий для [`IVersionedEntity`](https://iiko.github.io/front.api.sdk/v9/html/T_Resto_Front_Api_Data_Common_IVersionedEntity.htm).

Ранее ревизия [`IVersionedEntity.Revision`](https://iiko.github.io/front.api.sdk/v9/html/P_Resto_Front_Api_Data_Common_IVersionedEntity_Revision.htm) была сквозная между терминалами одной группы. Ревизия назначалась на главной кассе и передавалась ведомым терминалам. В силу этого, для корректной работы синхронизации доступа к данным, плагины, использующие функции редактирования данных, предпочтительно было устанавливать на главную кассу. При установке на другие терминалы, в некоторых сценариях плагин мог разово получать отказы в применении изменений. 

Теперь каждый терминал фронта ведет свою ревизию объектов. Это означает, что отказы в применении изменений устранены. 
При каждом изменении объекта назначается новая ревизия, ревизия строго и монотонно растет.

Объектам может быть переназначена ревизия, в случае если произошло пересоздание фронтовой базы данных. Например, это может произойти при смене главного терминала. 
В связи с этим был добавлен новый метод [`GetHostDatabaseId`](https://iiko.github.io/front.api.sdk/v9/html/M_Resto_Front_Api_IOperationService_GetHostDatabaseId.htm), который возвращает идентификатор фронтовой базы данных. При пересоздании фронтовой БД, ей назначается новый идентификатор.

В  [`SamplePlugin`](https://github.com/iiko/front.api.sdk/tree/master/sample/v9preview2/Resto.Front.Api.SamplePlugin) был добавлен [`пример`](https://github.com/iiko/front.api.sdk/blob/master/sample/v9preview2/Resto.Front.Api.SamplePlugin/SamplePlugin.VersionedEntityRevisionHelper.cs) отслеживания сброса ревизий.