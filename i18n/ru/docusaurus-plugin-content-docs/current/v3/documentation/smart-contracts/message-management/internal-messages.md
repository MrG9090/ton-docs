# Внутренние сообщения

:::warning
Эта страница переведена сообществом на русский язык, но нуждается в улучшениях. Если вы хотите принять участие в переводе свяжитесь с [@alexgton](https://t.me/alexgton).
:::

## Общие сведения

Смарт-контракты взаимодействуют друг с другом, отправляя так называемые **внутренние сообщения**. Когда внутреннее сообщение достигает своего адресата, создается обычная транзакция от имени акккаунта получателя, а внутреннее сообщение обрабатывается в соответствии с кодом и постоянными данными этого аккаунта (смарт-контракт).

:::info
В частности, транзакция обработки может создавать одно или несколько исходящих внутренних сообщений, некоторые из которых могут быть адресованы исходному адресу обрабатываемого внутреннего сообщения. Это можно использовать для создания простых "клиент-серверных приложений", когда запрос инкапсулируется во внутреннее сообщение и отправляется другому смарт-контракту, который обрабатывает запрос и снова отправляет ответ в качестве внутреннего сообщения.
:::

Этот подход приводит к необходимости различать, предназначено ли внутреннее сообщение как "запрос", "ответ" или не требует никакой дополнительной обработки (например, "простой денежный перевод"). Кроме того, при получении ответа должен быть способ определить, какому запросу он соответствует.

Для достижения этой цели можно использовать следующие подходы к внутреннему макету сообщения (обратите внимание, что блокчейн TON не накладывает никаких ограничений на тело сообщения, поэтому это всего лишь рекомендации).

### Внутренняя структура сообщения

Тело сообщения может быть встроено в само сообщение или сохранено в отдельной ячейке, на которую ссылается сообщение, как указано во фрагменте схемы TL-B:

```tlb
message$_ {X:Type} ... body:(Either X ^X) = Message X;
```

Принимающий смарт-контракт должен принимать как минимум внутренние сообщения со встроенными телами сообщений (всякий раз, когда они помещаются в ячейку, содержащую сообщение). Если он принимает тела сообщений в отдельных ячейках (используя конструктор `right` из `(Either X ^X)`), обработка входящего сообщения не должна зависеть от конкретного варианта встраивания, выбранного для тела сообщения. С другой стороны, совершенно допустимо не поддерживать тела сообщений в отдельных ячейках для более простых запросов и ответов.

### Внутреннее тело сообщения

Тело сообщения обычно начинается со следующих полей:

```
* A 32-bit (big-endian) unsigned integer `op`, identifying the `operation` to be performed, or the `method` of the smart contract to be invoked.
* A 64-bit (big-endian) unsigned integer `query_id`, used in all query-response internal messages to indicate that a response is related to a query (the `query_id` of a response must be equal to the `query_id` of the corresponding query). If `op` is not a query-response method (e.g., it invokes a method that is not expected to send an answer), then `query_id` may be omitted.
* The remainder of the message body is specific for each supported value of `op`.
```

### Простое сообщение с комментарием

Если `op` равен нулю, то сообщение является "простым сообщением о передаче с комментарием". Комментарий содержится в оставшейся части тела сообщения (без поля `query_id`, т. е. начиная с пятого байта). Если он не начинается с байта `0xff`, комментарий является текстовым; он может быть отображен "как есть" конечному пользователю кошелька (после фильтрации недопустимых и контрольных символов и проверки того, что это допустимая строка UTF-8).

Когда комментарий настолько длинный, что не помещается в ячейку, не вмещающийся конец строки помещается в первую ссылку ячейки. Этот процесс продолжается рекурсивно для описания комментариев, которые не помещаются в две или более ячеек:

```
root_cell("0x00000000" - 32 bit, "string" up to 123 bytes)
         ↳1st_ref("string continuation" up to 127 bytes)
                 ↳1st_ref("string continuation" up to 127 bytes)
                         ↳....
```

Тот же формат используется для комментариев переводов NFT и [жетонов](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md#forward_payload-format).

Например, пользователи могут указать цель простого перевода со своего кошелька на кошелек другого пользователя в этом текстовом поле. С другой стороны, если комментарий начинается с байта `0xff`, то остаток представляет собой "двоичный комментарий", который не должен отображаться конечному пользователю в виде текста (только в виде шестнадцатеричного дампа, если необходимо). Предполагаемое использование "двоичных комментариев" заключается, например, в том, чтобы содержать идентификатор покупки для платежей в магазине, который автоматически генерируется и обрабатывается программным обеспечением магазина.

Большинство смарт-контрактов не должны выполнять нетривиальные действия или отклонять входящее сообщение при получении "простого сообщения о переводе". Таким образом, как только `op` оказывается равным нулю, функция смарт-контракта для обработки входящих внутренних сообщений (обычно называемая `recv_internal()`) должна немедленно завершиться с нулевым кодом выхода, указывающим на успех (например, путем выдачи исключения `0`, если смарт-контрактом не был установлен пользовательский обработчик исключений). Это приведет к тому, что на принимающий аккаунт будет зачислено значение, переданное сообщением, без каких-либо дальнейших последствий.

### Сообщения с зашифрованными комментариями

Если `op` равен `0x2167da4b`, то сообщение является "сообщением передачи с зашифрованным комментарием". Это сообщение сериализуется следующим образом:

Ввод:

- `pub_1` и `priv_1` - Ed25519 открытый и закрытый ключи отправителя, по 32 байта каждый.
- `pub_2` - Ed25519 открытый ключ получателя, 32 байта.
- `msg` - сообщение для шифрования, произвольная строка байтов. `len(msg) <= 960`.

Алгоритм шифрования следующий:

1. Вычислить `shared_secret` с помощью `priv_1` и `pub_2`.
2. Пусть `salt` будет [представлением bas64url](/v3/documentation/smart-contracts/addresses#user-friendly-address) адреса кошелька отправителя с `isBounceable=1` и `isTestnetOnly=0`.
3. Выберите байтовую строку `prefix` длиной от 16 до 31, так что `len(prefix+msg)` делится на 16. Первый байт `prefix` равен `len(prefix)`, остальные байты случайны. Пусть `data = prefix + msg`.
4. Пусть `msg_key` будет первыми 16 байтами `hmac_sha512(salt, data)`.
5. Вычислите `x = hmac_sha512(shared_secret, msg_key)`. Пусть `key=x[0:32]` и `iv=x[32:48]`.
6. Зашифруйте `data` с помощью AES-256 в режиме CBC с `key` и `iv`.
7. Создайте зашифрованный комментарий:
  1. `pub_xor = pub_1 ^ pub_2` - 32 байта. Это позволяет каждой стороне расшифровать сообщение, не заглядывая в открытый ключ другой стороны.
  2. `msg_key` - 16 байт.
  3. Зашифрованные `data`.
8. Тело сообщения начинается с 4-байтового тега `0x2167da4b`. Затем этот зашифрованный комментарий сохраняется:
  1. Строка байтов делится на сегменты и сохраняется в цепочке ячеек `c_1,...,c_k` (`c_1` является корнем тела). Каждая ячейка (кроме последней) имеет ссылку на следующую.
  2. `c_1` содержит до 35 байт (не включая 4-байтовый тег), все остальные ячейки содержат до 127 байт.
  3. Этот формат имеет следующие ограничения: `k <= 16`, максимальная длина строки 1024.

Тот же формат используется для комментариев при переводе NFT и жетонов, обратите внимание, что следует использовать открытый ключ адреса отправителя и адреса получателя (не адреса jetton-wallet).

:::info
Learn from examples of the message encryption algorithm:

- [encryption.js](https://github.com/toncenter/ton-wallet/blob/master/src/js/util/encryption.js)
- [SimpleEncryption.cpp](https://github.com/ton-blockchain/ton/blob/master/tonlib/tonlib/keys/SimpleEncryption.cpp)
  :::

### Простые сообщения о передаче без комментариев

"Простое сообщение о передаче без комментариев" имеет пустое тело (даже без поля `op`). Вышеизложенные соображения применимы и к таким сообщениям. Обратите внимание, что такие сообщения должны иметь свои тела, встроенные в ячейку сообщения.

### Различие между сообщениями запроса и ответа

Мы ожидаем, что сообщения "запроса" будут иметь `op` с очищенным старшим битом, т. е. в диапазоне `1 .. 2^31-1`, а сообщения "ответа" будут иметь `op` с установленным старшим битом, т. е. в диапазоне `2^31 .. 2^32-1`. Если метод не является ни запросом, ни ответом (так что соответствующее тело сообщения не содержит поля `query_id`), он должен использовать `op` в диапазоне "запроса" `1 .. 2^31 - 1`.

### Обработка стандартных сообщений ответа

Существуют некоторые "стандартные" сообщения ответа с `op`, равным `0xffffffff` и `0xfffffffe`. В общем случае значения `op` от `0xffffffff0` до `0xffffffff` зарезервированы для таких стандартных ответов.

```
* `op` = `0xffffffff` means "operation not supported". It is followed by the 64-bit `query_id` extracted from the original query, and the 32-bit `op` of the original query. All but the simplest smart contracts should return this error when they receive a query with an unknown `op` in the range `1 .. 2^31-1`.
* `op` = `0xfffffffe` means "operation not allowed". It is followed by the 64-bit `query_id` of the original query, followed by the 32-bit `op` extracted from the original query.
```

Обратите внимание, что неизвестные "ответы" (с `op` в диапазоне `2^31 .. 2^32-1`) следует игнорировать (в частности, в ответ на них не следует генерировать ответ с `op`, равным `0xffffffff`), так же как и неожиданные возвращенные сообщения (с установленным флагом "bounced").

## Известные коды операций

:::info
Также op-code, op::code и операционный код
:::

| Тип контракта | Шестнадцатеричный код | OP::Code                                                                                                   |
| ------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Общий         | 0x00000000            | Текстовый комментарий                                                                                                                      |
| Общий         | 0xffffffff            | Отклонение                                                                                                                                 |
| Общий         | 0x2167da4b            | [Зашифрованный комментарий](/v3/documentation/smart-contracts/message-management/internal-messages#messages-with-encrypted-comments)       |
| Общий         | 0xd53276db            | Избыток                                                                                                                                    |
| Электор       | 0x4e73744b            | Новая ставка                                                                                                                               |
| Электор       | 0xf374484c            | Подтверждение новой ставки                                                                                                                 |
| Электор       | 0x47657424            | Запрос на восстановление ставки                                                                                                            |
| Электор       | 0x47657424            | Ответ на восстановление ставки                                                                                                             |
| Кошелек       | 0x0f8a7ea5            | Перевод жетонов                                                                                                                            |
| Кошелек       | 0x235caf52            | [Вызов жетона](https://testnet.tonviewer.com/transaction/1567b14ad43be6416e37de56af198ced5b1201bb652f02bc302911174e826ef7)                 |
| Жетон         | 0x178d4519            | Внутренняя передача жетона                                                                                                                 |
| Жетон         | 0x7362d09c            | Уведомление жетона                                                                                                                         |
| Жетон         | 0x595f07bc            | Сжигание жетона                                                                                                                            |
| Жетон         | 0x7bdd97de            | Уведомление о сжигании жетона                                                                                                              |
| Жетон         | 0xeed236d3            | Установка статуса жетона                                                                                                                   |
| Выпуск жетона | 0x642b7d07            | Выпуск жетона                                                                                                                              |
| Выпуск жетона | 0x6501f354            | Изменение администратора жетона                                                                                                            |
| Выпуск жетона | 0xfb88e119            | Запрос администратора жетона                                                                                                               |
| Выпуск жетона | 0x7431f221            | Сброс администратора жетона                                                                                                                |
| Выпуск жетона | 0xcb862902            | Изменение метаданных жетона                                                                                                                |
| Выпуск жетона | 0x2508d66a            | Обновление жетона                                                                                                                          |
| Вестинг       | 0xd372158c            | [Пополнение](https://github.com/ton-blockchain/liquid-staking-contract/blob/be2ee6d1e746bd2bb0f13f7b21537fb30ef0bc3b/PoolConstants.ts#L28) |
| Вестинг       | 0x7258a69b            | Добавление белого списка                                                                                                                   |
| Вестинг       | 0xf258a69b            | Добавление ответа в белый список                                                                                                           |
| Вестинг       | 0xa7733acd            | Отправка                                                                                                                                   |
| Вестинг       | 0xf7733acd            | Отправка ответа                                                                                                                            |
| Dedust        | 0x9c610de3            | Обмен ExtOut на Dedust                                                                                                                     |
| Dedust        | 0xe3a0d482            | Обмен жетонов на Dedust                                                                                                                    |
| Dedust        | 0xea06185d            | Внутренний обмен Dedust                                                                                                                    |
| Dedust        | 0x61ee542d            | Внешний обмен                                                                                                                              |
| Dedust        | 0x72aca8aa            | Обмен пирами                                                                                                                               |
| Dedust        | 0xd55e4686            | Внутренний депозит ликвидности                                                                                                             |
| Dedust        | 0x40e108d6            | Депозит ликвидности жетона                                                                                                                 |
| Dedust        | 0xb56b9598            | Ликвидность всех депозитов                                                                                                                 |
| Dedust        | 0xad4eb6f5            | Выплаты из пула                                                                                                                            |
| Dedust        | 0x474а86са            | Выплата                                                                                                                                    |
| Dedust        | 0xb544f4a4            | Депозит                                                                                                                                    |
| Dedust        | 0x3aa870a6            | Вывод                                                                                                                                      |
| Dedust        | 0x21cfe02b            | Создать хранилище                                                                                                                          |
| Dedust        | 0x97d51f2f            | Создать непостоянный пул                                                                                                                   |
| Dedust        | 0x166cedee            | Отмена депозита                                                                                                                            |
| StonFi        | 0x25938561            | Внутренний обмен                                                                                                                           |
| StonFi        | 0xf93bb43f            | Запрос платежа                                                                                                                             |
| StonFi        | 0xfcf9e58f            | Обеспечение ликвидности                                                                                                                    |
| StonFi        | 0xc64370e5            | Успешный обмен                                                                                                                             |
| StonFi        | 0x45078540            | Ссылка на успешный обмен                                                                                                                   |

:::info
[DeDust docs](https://docs.dedust.io/docs/swaps)

[Документация StonFi](https://docs.ston.fi/docs/developer-section/architecture#calls-descriptions)
:::
