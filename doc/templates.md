# card2b — API для шаблонов

Любая карточка выпускается на основе шаблона, поэтому нужно сначала создать шаблон в кабинете на https://{domain}.com. 

Внутренняя структура шаблона весьма громоздкая, так что создание и редактирование шаблонов проще делать визуально, а не по API. 
А вот по API как раз выпуск карточек, просмотр всех выпущенных, поиск, статистика и другое.

Шаблоны, как и карточки, версионируются. Версии появляются, когда в интерфейсе меняешь свойства шаблона или дефолтные данные, и сохраняешь. 

У каждого шаблона есть **template_id**, который можно легко найти на вкладке "API" в интерфейсе.



<a name="api_template_get"></a>
## GET /api/template/{template_id} — получение шаблона

- **template_id** *number* — id шаблона

Ответ: [объект шаблона](./working-with-api.md#template) — шаблон как есть на данный момент, в том числе нумерация всех версий.

<a name="api_template_get_vnum"></a>
## GET /api/template/{template_id}?v_num={v_num} — получить версию шаблона

- **template_id** *number* — id шаблона
- **v_num** *number* — число от 1 до количества версий шаблона

Ответ: [объект версии шаблона](./working-with-api.md#template-version).

<a name="api_template_vtime"></a>
## GET /api/template/{template_id}?v_time={v_time} — получить состояние на момент времени

- **template_id** *number* — id шаблона
- **v_time** *number* — unix timestamp, на который нужно вернуть состояние (не ранее, чем *create_time* шаблона)

Ответ: [объект версии шаблона](./working-with-api.md#template-version).



<a name="api_template_cards"></a>
## GET /api/template/{template_id}/cards?limit={limit} — выпущенные карточки

- **template_id** *number* — id шаблона
- **limit** *number* *optional, по умолчанию 100* — сколько карточек вернуть; сортировка — обратная хронологическая, т.е. возвращаются *limit* последних выпущенных карточек

Ответ: 

``` js
{
  "cards_issued": 569,
  "cards_installed": 301,
  "cards": [
    { /* карточка */ },         
    { /* карточка */ }
  ]
}
```


<a name="api_template_update"></a>
## POST /api/template/{template_id}/update — изменение шаблона

- **template_id** *number* — id шаблона
- *POST* **title** *string* *optional* — название для отображения в кабинете; не версионируется
- *POST* **description** *string* *optional* — подробное описание для отображения в кабинете; не версионируется
- *POST* **default_data** *object* *optional* — дефолтные данные для карточек (см. [концепцию](./basic-concepts.md#дефолтные-данные-и-переопределение))

> Если **default_data** меняется, то не только создаётся новая версия шаблона, но и создаётся новая версия всех выпущенных карточек, что логично. 
При этом, для всех карточек, которые установлены на телефоны, нужно послать пуши, чтобы форсировать их обновление у клиентов.

Ответ:

``` js
{
  "template_id": 123,
  "changed": true,              // true, если были изменены хоть какие-то свойства (даже title)
  "cards_reissue_needed": true  // true, если породилась новая версия шаблона и карточек, и нужно форсировать
}
```



<a name="api_template_reissue"></a>
## POST /api/template/{template_id}/reissue — перевыпуск всех карточек

- **template_id** *number* — id шаблона

Форсированно рассылает пуши всем клиентам, чтобы доставить обновлённые карточки на устройства.

В частности, если *update* выше вернул **cards_reissue_needed = true**, то значит, нужно вызвать этот метод (если этого не сделать, карточки у клиентов будут обновляться при изменении данных у самих карточек, индивидуально, а не массово).

Ответ:

``` js
{
  "template_id": 123,
  "changed": true,              // true, если есть (и была перевыпущена) хоть одна карточка 
  "cards_reissue_needed": false
}
```



<a name="api_template_notify"></a>
## POST /api/template/{template_id}/notify — отправка уведомления

- **template_id** *number* — id шаблона
- *POST* **notify_txt** *string* — текст уведомления

Аналог [/api/card/notify](./cards.md#api_card_notify), только сразу на все установленные карточки.

> Сейчас notify_text статичен, т.е. всем клиентам одинаковый. Так, можно разослать "У нас новая акция", но "Здравствуйте, {owner_name}!" пока не получится.
Есть предложения? Пишите issues ;)

Ответ:

``` js
{
  "template_id": 123,
  "changed": true,              // true, если уведомления были отправлены 
  "cards_reissue_needed": false
}
```

 
 
<a name="api_template_deactivate"></a>
## POST /api/template/{template_id}/deactivate — блокировка

- **template_id** *number* — id шаблона

После блокировки нельзя будет выпустить новую карточку по шаблону. Все существующие продолжат работать, как и раньше.

Ответ:

``` js
{
  "template_id": 123,
  "changed": true,              // true, если шаблон был активен, а стал заблокирован 
  "cards_reissue_needed": false
}
```
  

<a name="api_template_activate"></a>
## POST /api/template/{template_id}/activate — разблокировка

- **template_id** *number* — id шаблона

Обратное действие для deactivate. После этого карточки по шаблону можно вновь выпускать.

Ответ:

``` js
{
  "template_id": 123,
  "changed": true,              // true, если шаблон был заблокирован, а стал активен 
  "cards_reissue_needed": false
}
```


<a name="api_template_stat"></a>
## GET /api/template/{template_id}/stat?... — детальная статистика

- **template_id** *number* — id шаблона
- **timeFrom** *unix timestamp* — с какого момента времени
- **timeTo** *unix timestamp* — до какого момента времени
- **field** *string* — ключ из *default_data*; значения должны быть числовыми (т.е. можно получить статистику по bonus, но нельзя по status из примеров выше)
- **step** *number* — с каким шагом, в минутах
- **aggMethod** *string* — один из `sum, avg, max, min, count` — метод агрегации, когда несколько карточек попадают в интервал 

Пример: запросим *сумму* *bonus*'ов всех карточек *123*-го шаблона с *1 ноября* по *1 марта* с шагом в *неделю*:

```
GET /api/template/123/stat?field=bonus&timeFrom=1509483600&timeTo=1519851600&step=10080&aggMethod=sum
```  

Ответ:

``` js
{
  "template_id": 123,
  "field": "bonus",
  "stepPoints": [83,131,181,198,154,271,261,339,413,419,528,567,625,479,435,615,645,651]
}
```

Для визуализации статистики — по любым полям — есть удобные средства внутри кабинета, проще использовать их, чем API.
