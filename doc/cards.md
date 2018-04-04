# card2b — API для карточек

Карточки выпускаются по **шаблону**. Впоследствии id шаблона у карточки не меняется: меняются только данные карточки, либо внешний вид/данные шаблона.

У выпущенных карточек хранится полная история версий, как описано в [базовых концепциях](./basic-concepts.md#данные-и-версионирование).



<a name="api_card_issue"></a>
## POST /api/card/issue — выпуск карты

- *POST* **template_id** *number* — id шаблона
- *POST* **override_data** *object* *optional* — переопределение данных относительно дефолтных; может содержать спец. данные с [зарезервированными ключами](./override_data.md) 

Ответ:

``` js
{
  "card_id": 123,
  "url": "https://full.url.to/card/download"
}
```



<a name="api_card_get"></a>
## GET /api/card/{card_id} — данные конкретной карточки

- **card_id** *number* — id карточки

Ответ: [объект карточки](./working-with-api.md#card) — карточка как есть на данный момент, в том числе нумерация всех версий.

<a name="api_card_get_vnum"></a>
## GET /api/card/{card_id}?v_num={v_num} — получить версию карточки

- **card_id** *number* — id карточки
- **v_num** *number* — число от 1 до количества версий карточки

Ответ: [объект версии карточки](./working-with-api.md#card-version).

<a name="api_card_get_vtime"></a>
## GET /api/card/{card_id}?v_time={v_time} — получить состояние на момент времени

- **card_id** *number* — id карточки
- **v_time** *number* — unix timestamp, на который нужно вернуть состояние (не ранее, чем *create_time* карточки)

Ответ: [объект версии карточки](./working-with-api.md#card-version).

   

<a name="api_card_update"></a>
## POST /api/card/{card_id}/update — изменение данных карты

- **card_id** *number* — id карточки
- *POST* **override_data** *object* — переопределение данных относительно текущего состояния; может содержать спец. данные с [зарезервированными ключами](./override_data.md)

Подробнее про **override_data**, здесь это важно. Если значение null, то ключ возвращается к дефолтным данным шаблона. Если не null, то перезатирается. Если отсутствует, то не меняется.

Пример. 
1. У шаблона **default_data** = `{"bonus":0, "status":"Начальный"}`.
2. Выпуск (issue) карты **override_data** = `{"bonus":10}` ⇒ **full_data** = `{"bonus":10, "status":"Начальный"}`
3. update данных **override_data** = `{"status":"VIP"}` ⇒ **full_data** = `{"bonus":10, "status":"VIP"}`
4. update данных **override_data** = `{"bonus":20}` ⇒ **full_data** = `{"bonus":20, "status":"VIP"}`  
5. update данных **override_data** = `{"bonus":30, "status":null}` ⇒ **full_data** = `{"bonus":30, "status":"Начальный"}`

То есть при update нужно прислать только то, что нужно изменить, остальное состояние сохранится.

Ответ:

``` js
{
  "card_id": 123,
  "changed": true            // true, если породилась новая версия и изменилось update_time
}
```


<a name="api_card_notify"></a>
## POST /api/card/{card_id}/notify — отправка уведомления

- **card_id** *number* — id карточки
- *POST* **notify_txt** *string* — текст уведомления

Уведомления — это аналог SMS, только бесплатные и менее раздражающие. А также не нужно знать номер телефона, чтобы отправить. Уведомление придёт на все устройства, где установлена карточка.

Ответ:

``` js
{
  "card_id": 123,
  "changed": true            // true, если уведомление отправилось и изменилось update_time
}
```

Если карта не установлена ни на одном устройстве, будет ошибка.



<a name="api_card_deactivate"></a>
## POST /api/card/{card_id}/deactivate — блокировка

- **card_id** *number* — id карточки

После блокировки карту нельзя будет скачать и установить. Если она уже установлена, то продолжит функционировать, т.е. до устройства будут доходить обновления и уведомления.

Ответ:

``` js
{
  "card_id": 123,
  "changed": true            // true, если была активна, а стала заблокирована
}
```
  

<a name="api_card_activate"></a>
## POST /api/card/{card_id}/activate — разблокировка

- **card_id** *number* — id карточки

Обратное действие для deactivate. После этого карту можно вновь скачивать и устанавливать.

Ответ:

``` js
{
  "card_id": 123,
  "changed": true            // true, если была заблокирована, а стала активна
}
```


<a name="api_card_stat"></a>
## GET /api/card/{card_id}/stat?... — детальная статистика

> По конкретной карточке можно получить детальную статистику (и построить график, например), хотя чаще всего интересует 
[статистика по шаблону](./templates.md#api_template_stat) 
(т.е. кумулятивно по всем выпущенным карточкам сразу)

- **card_id** *number* — id карточки
- **timeFrom** *unix timestamp* — с какого момента времени
- **timeTo** *unix timestamp* — до какого момента времени
- **field** *string* — ключ из *full_data*; значения должны быть числовыми (т.е. можно получить статистику по bonus, но нельзя по status из примеров выше)
- **step** *number* — с каким шагом, в минутах

Пример: запросим, как менялся *bonus* у *123*-й карты с *28 февраля* по *11 марта* с шагом в *сутки*:

```
GET /api/card/123/stat?field=bonus&timeFrom=1519765200&timeTo=1520715600&step=1440
```  

Ответ:

``` js
{
  "card_id": 123,
  "field": "bonus",
  "stepPoints": [0,2,2,2,0,0,0,0,0,8,8,3]
}
```

