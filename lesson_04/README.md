# Lesson 04. План запроса и индексы

Коллекция `logs.events`, 1200 документов, MongoDB 8.0.4. Планы сняты через `explain()` в режиме executionStats.

## Сводная таблица

| Файл | Запрос | Индекс | nReturned | totalDocsExamined | totalKeysExamined | Шаги плана (снизу вверх) |
|---|---|---|---|---|---|---|
| plan-1.json | 1 | нет | 33 | 1200 | 0 | COLLSCAN, SORT |
| plan-2.json | 1 | `{service:1, level:1, ts:-1, duration_ms:1}` | 33 | 33 | 43 | IXSCAN, FETCH |
| plan-3a.json | 1 | `{duration_ms:1, service:1, level:1, ts:-1}` | 33 | 33 | 378 | IXSCAN, SORT, FETCH |
| plan-3b.json | 1 | `{service:1, level:1, duration_ms:1, ts:-1}` | 33 | 33 | 33 | IXSCAN, SORT, FETCH |
| plan-4-bez.json | 2 | нет | 10 | 1200 | 0 | COLLSCAN, SORT |
| plan-4.json | 2 | `{level:1, duration_ms:-1}` | 10 | 10 | 10 | IXSCAN, FETCH, LIMIT |

Запрос 1:

```js
db.events.find({ service: "payments", level: "error", duration_ms: { $gt: 500 } }).sort({ ts: -1 })
```

Запрос 2:

```js
db.events.find({ level: "error" }).sort({ duration_ms: -1 }).limit(10)
```

## Задача 1. Запрос без индекса

- Запрос вернул 33 записи, `countDocuments` с тем же фильтром даёт 33.
- nReturned = 33, totalDocsExamined = 1200, totalKeysExamined = 0.
- Шаги снизу вверх: COLLSCAN, SORT.
- Разбор: https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:zVZtj-I2EP4v1n3kVglJgPANArmuupS9zban6lRVJh6CS2LnbIeFQ_z3jp3lZfd6u6urVPUT4Hnmxc8z42FPYFuXlIvfQGkuBRkSn3TIlwbU7rakQoAiwz0RtAJd0xzQXspCX8EGhNGIrKnSwD5avAW-o4KR4ec9KRFRuhP4gk6glFTkcOjsiQa14TbS0VbTXeWiOTNrFDVYyZ-VdpDCkGHkeYfDH4cO4YLBNuWlAZUBGpa01IBFYKUJzVeQrWgNP1G9wrBRP-gGybhHLuw_A1ZJvEE69pORtcja8Ip_dRnveQUzXpYcE3sdUtHttU0HbK4yWTYWou_AhmGnzGfQSLCXUFlOhb6XU2RbMngOqFUjgGVYSklVG1CfjA9cCC4KK4dlhOvkqbM2tLDCZPO7e7ySlsrcUoMUOTjyOnzvI3cVVDeYAFnzvXAQ9XseXtLsauuqeVWXQCzBdWOyNuD-FDmZ39xkyegXBCwd-f-B1IwryE3bkkupHqhi1oso-AvPgVk60O2zxSJduWMeKzcu1vmkyXPQeGZUg1yJOzCNEpa9ILhwfKa9kYaW2Cx6uqUVd_Dj6UTmF6d-17J4mb8A_RaVXi9kqrE1qQGX-kGqtbbpgqhDKNtQkR9dBQCzXq4Yv_39O4eyrVnTDVhW2jAKtJHq4gDTzFP0_Jdt0zJDDc34V8gwki0u9gcYt8HnYcL1-kxEjddraXZf0WwdsS4kyPq3hf1vGvFHlbJafFcpv9f_YaX-aTLw9Ju-tNNCy9INyvRY9ePEuAf-8rUcjeJpGvl-5HteEgSTOBjFoyQKB90oTOJeMIgiLw2TaTAZh_7En_T8adRDVBoGQZKGE6wgl1XlpNijOPaTnLbEWayTEmcJOkfhHhXrfF-Ptk2f9Oc7tnhcSsSaMTzgC7qUFrSSGsFkAnptZP1-_jH7YAutXYxu3_NRhM1p8Q2uvKsQzQU35224yIOILsLAi1gcdz0WMy-M_dzPY1jGi343GAQs8JfxOfktVbgt8brtOyDsRNHSbcgUN6gZN8slri9s9PHO2MfiYq6-Rc_odt4YnAV8d172uZFy3dRuZK7tUQWMY_OgX2N5xkAv-x-BmWxUDh-UbGr0mUEl1e6VShE3LmW-xi1lh7_1-VVjJa843iq54gtujt4zUAXMxUyKQmaP78DTPCPG7iXu_rfeA6GfsBnlQ2oHTb_1RqnV0I5yIoVR0janwYFZ4ObWRnG7gMjzu7R_l64LgXPrdvgnblaJLEvXy6lUd1DAFnNiq0h8D_0r7_A3
- Сервер прочитал 1200 документов, потому что индекса нет и проверить фильтр можно только у каждого документа по очереди. Из них подошли 33.

## Задача 2. Индекс по правилу ESR

Разбор запроса:

- E (равенство): `service`, `level`
- S (сортировка): `ts`
- R (диапазон): `duration_ms`

Команда:

```js
db.events.createIndex({ service: 1, level: 1, ts: -1, duration_ms: 1 })
```

- nReturned = 33, totalDocsExamined = 33, totalKeysExamined = 43.
- Шаги снизу вверх: IXSCAN, FETCH.
- Разбор: https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7Vbfb9s2EP5XCqIPe3ACyZKcyG-2bLXB5jqN0nVDGhi0eLZZy6RKUoldw__7jpR_D0vyuAF7onT33fF49_F4awLLsqBc_A5KcylIm_ikQX5UoFa3BRUCFGmviaAL0CXNAfWFnOpLeAJhNCJLqjSwzxZvge-pYKT9sCYFIgongR9oBEpJRTabxppoUE_cetrpSrpaOG9OzSpFDUYyWmgHmRrSjjxvs3ncNAgXDJYpLwyoDFAxoYUGDAIjTWg-g2xGS_hI9QzdRldBM0i6LXKk_xUwSpJGsR-2fKuRpeEL_tPteM8XMOBFwXFjr0EWdHljtwM2VJksKgvRd2DdsP3OB1BHsJdQWU6Fvpd9zLZkcA4oVSWAZRhKQVXtUO-Vz1wILqa2HDYjXCenxtrQqS1M2r9PPhKbpLIyWS1c77U3f2RJ5xOq57C6pQYz6Lztq-E3djXDLyxG-wLXk2L4uwJ8Qjagx63pyB85Q1yNHl34oyOjkSUT14OqMNzlfpeQrQADmemTMB4e93HYTxuIXU8CeXBU0F8ER57ufXKdOTIeCW6pMpwWB4mNfs_0JrrlCnJT834i1TNVjGxhXVkJdhYbefi2Z-s30nh3_PdIjiK3QEf5GrX9dBB3IvIwoEs8f-PdgAtcneb0jOQXZH3jHReTSw_VG7wcRMF3DBeYpcIuD0iV3LEOK25cvAdJleegUWZUhacXd2AqJSxzguDI8Iz3RhpaYEy6v6QL7uBhsBX3ZH4kPvHiCKffRNDXA-lrvJbUgAvoWaq5roOg7ImKfGcoAJi1Ie24_vmTQ8GcjaZPYBNSe1CgjVRHAtxhmDqms7-diBYKKFthFxmOv9foly_Uv-k4_1_u_-TldoU7u3AawPLE96xxqe-x6FCzwf72lCxL928bAy0K1xP6O-5t8-je8eNHsdOJ-2nk-5HveUkQ9OKgE3eSKLxuRmESt4LrKPLSMOkHvW7o9_xey-9HLUSlYRAkadjDBOZysXCv_JpMuF3JfhiYuJf5JK2H132fwO0w0Pjnpx4PL5WxspqrKHjPxtvZg1g1ugd8KCfSgmZSI5j0QM-NLC-Gn7MPNtDS-Wheef5Vgzzt55vrS-8yRPWUm8PQM86DiI7DwItYHDc9FjMvjP3cz2OYxOOrZnAdsMCfxIfNkX54U_C4dc8T9s7Rwg1CKQ5KpltNJjil8J_QXRlwhQyvo6uW5xrKORoZM6wMthnssC_b_CblvCpdN7qxogUwjn0A7SqbZ3T0sv0OmMlK5fBByapEmwEspFq9EiniuoXM5ziMZJjb2uaLxkheMbxVcsbH3OysB6CmMBQDKaYy27bY0306jN1LHPHeeg6EfkUyyufU9kz91hOltoa2ISdSGCUtOQ1emDEOaNoobt9acn6Weiq-mQpswW5U-8rNLJFF4bicSnUHU1jWHVXOcb30Nn8B
- Просмотренных документов стало меньше в 1200 / 33 ≈ 36 раз.
- Шаг SORT пропал, потому что внутри участка индекса с одинаковыми service и level записи уже лежат по ts в нужном порядке. Серверу не нужно сортировать результат.

## Задача 3. Порядок полей

| Индекс | Порядок | Возвращено | Документов просмотрено | Ключей просмотрено | Шаги плана |
|---|---|---|---|---|---|
| нет | нет | 33 | 1200 | 0 | COLLSCAN, SORT |
| service, level, ts, duration_ms | E, S, R | 33 | 33 | 43 | IXSCAN, FETCH |
| duration_ms, service, level, ts | R, E, S | 33 | 33 | 378 | IXSCAN, SORT, FETCH |
| service, level, duration_ms, ts | E, R, S | 33 | 33 | 33 | IXSCAN, SORT, FETCH |

**Почему в варианте А просмотрено 378 ключей.** Первым полем стоит диапазон. Сервер проходит весь участок индекса с `duration_ms > 500`, а в него попадают события всех сервисов и всех уровней. Фильтр по service и level применяется уже внутри этого участка, поэтому ключей просматривается в девять раз больше, чем в варианте по ESR. Сортировка при этом всё равно остаётся в памяти.

**Почему в варианте Б остался шаг SORT.** Поле ts стоит после диапазона. Внутри участка, который отобрал диапазон по duration_ms, записи лежат по duration_ms, а не по ts. Готового порядка по ts нет, и сервер сортирует результат отдельным шагом. Число ключей (33) здесь самое маленькое, но стоимость сортировки оно не учитывает.

**Какой вариант оставить.** Вариант по ESR, `{service, level, ts, duration_ms}`. Сортировку в памяти ограничивает лимит 100 МБ. На маленькой коллекции этого не видно, а на настоящем журнале запрос с вариантом Б упадёт с ошибкой. Вариант по ESR просматривает на 10 ключей больше, зато отдаёт порядок сам и не зависит от размера результата.

## Задача 4. Свой индекс

Запрос 2: десять самых долгих ошибок.

Разбор по ESR: равенство `level`, сортировка `duration_ms`, диапазона нет. Значит, в индексе два поля, сначала E, потом S.

```js
db.events.createIndex({ level: 1, duration_ms: -1 })
```

| | Индекс | nReturned | totalDocsExamined | totalKeysExamined | Шаги |
|---|---|---|---|---|---|
| До | нет | 10 | 1200 | 0 | COLLSCAN, SORT |
| После | level, duration_ms | 10 | 10 | 10 | IXSCAN, FETCH, LIMIT |

- Разбор: https://dfrancour.dev/tools/mongodb-paste-the-plan#v1:7VZZcxo5EP4rKVUesWsGGGD8xjUJtSE4HrJJynG5xKgBBY00kTQY4uK_b0scxrjW9h4P-7Avc6i_Vl-fWn1PYFUIyuXvoA1XklyQkFTIzxL0-lJQKUGTi3siaQ6moBmgXKiZOYclSGsQWVBtgH1yeAcUKBDu4y38RCxorTTZbCqESwarhAsLOgVLLqZUGEB1tNGl2RzSOS3gPTVz1Go0e-1W0EvIkfw3wP1JO4gbSdisoUQVluf8F7Xo9ZjnMORCcEMuggrJ6WrgzAEb6VSJ0kHMFbht2MHyA6gt2XOoNKPSjFUf86QYnAIKXUpgKboiqN5uaA7COy4llzOXSJcTbrqPlY2lM5fSD4PhYIwxCdzGtnNVSsxQGLisFaVNt6j7Azzpj7vvyZ9JB1_TbvsjihewvqQWEy6PKhNWCCu1z9ptjp6ehfvifMQau_I63G14e4S6PXOc4GZYCst9IfbZ2S2gmbk5MnJ9c2Ll-sZZMZ8lR2Yd1LlJPX2OFi6ptpyKhxXn2YGbVdyWa8jslqlTpe-oZmQH62De2CM3yPX3LQW_k8qbw-cNeeIeuR7SFQZSeTPkEt8I2WwcbTX8QHPAXAn3cWCJM88WTL319h5WyiwDg2tWl-i9vAJbaukq7qp5gJ3w1SpLBVo1_RXN-QHul3sqO1k-Nj8D8ypivexI3-Bxoha8Q3dKL3DjEMlC2ZLKbK8oAZjT8Sj3842DYP7P0CW4hGxlGoxV-mgBLYwST7-_SPK_6Xrwr7uOX-xJMajQQNkaG9do8mOLfv5Q_pfC-b9B_KMG4RN4wgcD4Ovl9AszxtzDtijut6dVUfh_31uoEL6t9Pcc2KXBX77H92FSbbQaURx1mkm_k7RajTgOa_gMoyCMg14zakTNpF2vtaph1K52a61eVG-04mo_7Eb1auxqk6k8p5K52KfcvcnhBp_6S_koK_tbG6NR2jrBU0L4Q7wLGW84AR1qs_m-771lk92g4HcBvQS8G6fK7TVXBhVJD8zCquJs9Cl913MXvTdVbQZhs0KWh2GkdR6c11E84_ZhQplktYhO6rUgYnFcDVjMgnocZmEWwzSeNKu1Vo3Vwmn8YBxZg-TFMLftUjrOU-GnlgSnGtspp1McTPgv6Kwt-ANXb0XNRuAP9CkaqTAqLR5zbM7P63xQalEWvhsM3FIOjOM5RL0yx_TjRs_r74GpKnUG77QqC9QZQq70-gVPEdcRKltgdVLM7Vbns0FPXlC81GrOJ9zutYegZzCSQyVnKt21uMd22oyNFU51r40DoV-QhOoucT3LvDaixNXQNcSuklYrx1SLB2WCM5mxmrtrmpzGsh1hBzOJLdBPZ1-4nXeVEJ7PidJXMIMV2kSqqAW-z4PNHw
- В коллекции 180 ошибок, но сервер прочитал только 10 документов. Индекс по level уже отдаёт ошибки от самых долгих к коротким, поэтому шаг LIMIT остановил чтение после десятой записи.

## Ответы на вопросы приёмки

- **totalKeysExamined и totalDocsExamined.** Первое считает записи индекса, которые просмотрел сервер, второе считает документы коллекции, которые он прочитал.
- **Зачем FETCH, если индекс нашёл документы.** В индексе лежат только значения полей и ссылки на документы. Остальные поля документа сервер забирает из коллекции шагом FETCH.
- **Почему 43 ключа при 33 документах.** Диапазон `duration_ms > 500` стоит в индексе последним. Внутри участка с service = payments, level = error сервер просматривает и записи с duration_ms ниже 500, а потом отбрасывает их. Таких записей 10.
- **Что будет с индексом при вставке новой записи.** Сервер добавит ключ нового документа в каждый индекс коллекции. Чем больше индексов, тем дороже вставка.
- **Почему не ставят индексы на все поля.** Каждый индекс замедляет запись, занимает память и диск. Индексы нужны под конкретные запросы, а лишние только мешают.
