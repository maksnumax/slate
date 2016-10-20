#Общие положения (Generall items)

* Во всех запросах и ответах следует использовать кодировку UTF8

* Для работы вам потребуется получить номер мерчанта (`merchantId`), доступный в клиентском интерфейсе, а так же сгенерировать и задать секретный ключ (`sharedsec`), что так же можно сделать из клиентского интерфейса. Обратите внимание, что **секретный ключ ни в каком виде не должен покидать вашего сервера**. В случае если он передаётся как часть HTML или каким-либо образом используется в JavaScript, это является **проблемой безопасности** и даёт злоумышленнику возможность делать вызовы к платёжной системе от вашего имени.

* Для корректной обработки транзакций вам необходимо реализовать обработчики двух URL - один для возвращения пользователя на ваш сайт, другой для приёма уведомлений (`webhook`/`callback`) о состоянии транзакций, их можно задать на вкладке "Интеграция" клиентского интерфейса.

* Данные тестовых карт для проверки работоспособности в sandbox-режиме

|Параметр|Значение|Комментарий|
|----|-------|-------|
|Имя держателя карты|CARD HOLDER|||
|CVV|123|||
|Номер карты|4929509947106878|Для успешных|
|Номер карты|4485913187374384|Для неуспешных|

* Примеры приложения для работы с API на PHP находятся на [GitHub](https://github.com/MandarinPay):

|Название|Описание|Cсылка|
|----|-------|-------|
|Приложение платежного шлюза|Приложение позволяет ознакомиться с основными принципами и методами работы платежного API|[GitHub](https://github.com/MandarinPay/php-example)|
|`Callback` модуль платежного шлюза|Данный модуль может перехватывать `Callback` вызовы, складывать из в БД, а так же отправлять email потранзакционно/реестром о совершенных операциях. Имеет API для запроса реестров из БД|[GitHub](https://github.com/MandarinPay/MandarinCallback_module)|
|Приложение информационного шлюза|Приложение позволяет ознакомиться с основными принципами и методами работы информационного шлюза, в т.ч. упрощенной идентификации|[GitHub](https://github.com/MandarinPay/MandarinInfo_module)|


# Оплата через API платежной страницы

```html
<form action="https://secure.mandarinpay.com/Pay" method="POST"> 
<input type="hidden" name="merchantId" value="812" />  
<input type="hidden" name="price" value="12.34" /> 
<input type="hidden" name="orderId" value="325GVD" />
<input type="hidden" name="customer_email" value="test@test.com" /> 
<input type="hidden" name="customValue1" value="client value 1" /> 
<input type="hidden" name="customName1" value="client name 1" /> 
<input type="hidden" name="customValue2" value="client value 2" />  
<input type="hidden" name="sign" value="d41d8cd98f00b204e9800998ecf8427e" /> 
<input type="submit" value="Оплатить" /> 
</form>
```

`POST` запрос на адрес `https://secure.mandarinpay.com/Pay`

Перечень и описание возможных полей приведены в таблице.


|Поле|Обяз-ть|Описание|
|----|----|-------|
|merchantId|Да|Id мерчанта|
|price  |Да |Сумма платежа. Обязательна к передаче в текущей версии системы.|
|orderId|Да |Уникальный номер заказа в системе Продавца. |
|customer_email|Да |Email пользователя|
|sign|Да |Контрольная подпись данных о платеже. Методику подсчёта см. ниже|
|customValue1|Нет|Дополнительные параметры, которые могут использоваться для прикрепления дополнительной информации к данным Платежа и Плательщика. При использовании совместно с customName данные будут отображаться на платежной странице. Не передаются в `Callback`|
|customValue2|Нет|См. выше|
|customValue3|Нет|См. выше|
|customName1|Нет|Используется совместно с customValue для отображения передаваемой информации Плательщику на платежной странице.|
|customName2|Нет|См. выше|
|customName3|Нет|См. выше|
|extra_*|Нет|Данные поля используются в случае, если необходимо передать дополнительную информацию в `Callback` о результате платежа и не видны Плательщику. Должны иметь вид `extra_[A-Za-z0-9_]+`|
|customer_phone|Нет|Телефон пользователя|


# Подсчёт поля sign

> Пример генерации поля sign и формирования формы на PHP

```php
<?php
function calc_sign($secret, $fields)
{
        ksort($fields);
        $secret_t = '';
        foreach($fields as $key => $val)
        {
                $secret_t = $secret_t . '-' . $val;
        }
        $secret_t = substr($secret_t, 1) . '-' . $secret;
        return hash("sha256", $secret_t);
}

function generate_form($secret, $fields)
{
        $sign = calc_sign($secret, $fields);
        $form = "";
        foreach($fields as $key => $val)
        {
                $form = $form . '<input type="hidden" name="'.$key.'" value="' . htmlspecialchars($val) . '"/>'."\n";
        }
        $form = $form . '<input type="hidden" name="sign" value="'.$sign.'"/>';
        return $form;
}

$f = generate_form("123", $values = array(
   "email" => "user@example.com",
   "merchantId" => "1",
   "orderId" => "123",
   "price" => "123.00",
));
echo $f;
?>
```
> Пример генерации поля sign на языке C\#

```cs

public static string Calculate(string secret, IDictionary<string, string> values)
{
    using (var sha256 = System.Security.Cryptography.SHA256.Create())
        return BitConverter.ToString(sha256.ComputeHash(Encoding.UTF8.GetBytes(string.Join("-", values.OrderBy(x => x.Key, StringComparer.Ordinal).Select(x => x.Value)) + "-" + secret))).ToLower().Replace("-", "");
}


Calculator.Calculate("123", new Dictionary<string, string>
{
   ["email"] = "user@example.com",
   ["merchantId"] = "1",
   ["orderId"] = "123",
   ["price"] = "123.00",
});
```

Контрольная сумма вычисляется **от всех полей, присутствующих в запросе, перечисленных в алфавитном порядке** и sharedsec (уникальный кода, который знает только платёжная система и Продавец) и разделённых символом “-”.

Например: 
`sign = sha256(email + "-" + merchantId + "-" + orderId + "-" +  price + "-" + sharedsec)`

<aside class="warning">
ВСЕ ПОЛЯ ЗАПРОСА, ПЕРЕЧИСЛЕННЫЕ В АЛФАВИТНОМ ПОРЯДКЕ
</aside>

# Виджет (pop-up widget)

Виджет оплаты представляет из себя pop-up открывающийся в `iframe` на стороне Продавца. Последующее открытие страницы с 3DS также происходит в `iframe`

## Оплата через виджет (widget pay)

Для использования виджета необходимо подключить скрипт

```html
<script src="https://secure.mandarinpay.com/api/widget.js" type="text/javascript"></script>
```

После чего использовать ```mandarin.payForm("#id_формы");``` для показа виджета с оплатой. Форма генерируется тем же способом, что и при работе с API платежной страницы.

Так же можно использовать ```onclick="return mandarin.payForm(this);"``` в качестве обработчика ссылок и кнопок внутри формы оплаты.


```html
<form onsubmit="return false;"> 
<input type="hidden" name="merchantId" value="812" />  
<input type="hidden" name="price" value="12.34" /> 
<input type="hidden" name="orderId" value="325GVD" /> 
<input type="hidden" name="customValue1" value="client value 1" /> 
<input type="hidden" name="customValue2" value="client value 2" />  
<input type="hidden" name="customValue3" value="client value 3" /> 
<input type="hidden" name="sign" value="d41d8cd98f00b204e9800998ecf8427e" />

<a href="#" onclick="return mandarin.payForm(this);">Оплатить</a>

</form>
```

## Привязка карт через виджет форм (widget card binding)

```html
<form onsubmit="return false;">
<input type="hidden" name="merchantId" value="812" />
<input type="hidden" name="orderId" value="325GVD" />
<input name='customer_phone' type="hidden" value='+71234567890' />
<input name='customer_email' type="hidden" value='user@example.com'/>
<input type="hidden" name="sign" value="d41d8cd98f00b204e9800998ecf8427e" />

<a href="#" onclick="return mandarin.bindCardForm(this);">Привязать</a>

</form>
```

Для привязки карты через виджет вам необходимо на своём сайте сгенерировать форму.

Назначение параметров соответствует параметрам из API платежной страницы. Так же можно использовать `mandarin.bindCardForm("#id");`

Поля `customer_phone` и `customer_email` являются обязательными для привязки карты


# Встроенная форма оплаты (emded API)

> Скрипт

```html
<script src="https://secure.mandarinpay.com/api/embed.js" type="text/javascript"></script>
```

Данная форма доступна для использования только по предварительному согласованию. Обратитесь к вашему менеджеру.

Для использования API внедрённых форм необходимо подключить скрипт, после чего из JS доступны для вызова функции `mandarinpay.embed.pay(selector, onsuccess, onerror)` и `mandarinpay.embed.bindCard(selector, onsuccess, onerror)`

`selector` - jQuery-совместимый селектор, указывающий на форму или HTML-элемент внутри формы либо сама форма (`this` из вызова `onclick` на элементе подойдёт)

`onsuccess` и `onerror` - функции обратного вызова

> onsuccess

```js
{
	transaction:
	{
		id: "c8a42608-ac02-449d-aae4-265778df5e27"
	}
}
```

> onerror

```js
{
	error: "Текст ошибки"
}
```

Форма состоит из двух частей:

* Сгенерированные у вас на сервере поля с `type="hidden"` с информацией о транзакции или привязке карты (подробности генерации см. в документации по API платежной страницы). 

<aside class="notice">
Напоминаем, что поле sign обязательно должно генерироваться на стороне сервера, а ключ мерчанта не должен попадать в JavaScript-часть.
</aside>

* Видимые пользователю или доступные для изменения другим образом поля с карточными данными. **Обратите внимание, что в полях отсутствует атрибут** `name`, что позволяет не сохранять карточные данные на вашем сервере. Вместо этого они идентифицируются атрибутом `data-pay-name`. 

<aside class="warning">
В случае обнаружения атрибут NAME при работе по данной схеме Продацец будет незамедлительно отключен от системы
</aside>

> Форма для совешения платежа

```html
<form id="form-pay">
	<!--Генерируется на сервере-->
    <input name="price" type="hidden" value="12.34" />
    <input name="orderId" type="hidden" value="123321" />
    <input name="merchantId" value="1" type="hidden" />
    <input name="sign" value="testing" type="hidden" />
    <input name='customer_phone' type="hidden" value='+71234567890' />
    <input name='customer_email' type="hidden" value='user@example.com' />

    <!--Заполняется пользователем-->
    <input type="text" data-pay-name="CardNumber" /><br />
    <input type="text" data-pay-name="CardHolder" value="CARD HOLDER" /><br />
    <input type="text" data-pay-name="CardExpireYear" value="20" /><br />
    <input type="text" data-pay-name="CardExpireMonth" value="12" /><br />
    <input type="text" data-pay-name="Cvv" value="123" /><br />

	   <!--Обработчик оплаты и ответа на неё-->
    <a href="#" onclick="return mandarinpay.embed.pay(this, function (data) { alert('Success. Transaction id is ' + data.transaction.id); }, function (error) { alert('error: ' + error.error); });" class="btn btn-default">
        Pay
    </a>
</form>
```

В случае с привязкой карты вместо `mandarinpay.embed.pay` следует вызывать `mandarinpay.embed.bindCard`.

Следует обратить внимание, что форма является одноразовой, после попытки совершить платёж в ней будет сохранён id операции и повторные попытки вызова `mandarinpay.embed.pay` не будут приводить к созданию новой. Вы можете безопасно повторять вызовы в рамках одной операции.


# REST API

>Пример генерации requestId на языке PHP

```php
function gen_auth($merchantId, $secret)
{
        $reqid = time() ."_". microtime(true) ."_". rand();
        $hash = hash("sha256", $merchantId ."-". $reqid ."-". $secret);
        return $merchantId ."-".$hash ."-". $reqid;
}
```

>Пример генерации requestId на языке C\#

```cs
public static string GenerateXAuth(string secret)
{
  var requestId = Guid.NewGuid().ToString("N");
    string hash;
    using (var sha256 = System.Security.Cryptography.SHA256.Create())
        hash = BitConverter.ToString(sha256.ComputeHash(Encoding.UTF8.GetBytes($"{merchantId}-{requestId}-{secret}"))).ToLower().Replace("-", "");
    return $"{merchantId}-{hash}-{requestId}";
}
```

Все запросы должны быть с заголовком `Content-Type: application/json`
Аутентикация запросов происходит посредством заголовка `X-Auth` следующего содержания:
`merchantId-SHA256(merchantId-requestId-secret)-requestId`

`requestId` - уникальный номер запроса, представляющий из себя текущий таймстамп в миллисекундах, либо набор байт, сгенерированный криптографически-надёжным генератором случайных числел.

## Создание транзакции без привязки карты (pay/payout)

`POST` `https://secure.mandarinpay.com/api/transactions`

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "pay/payout",
      "price": "10"
  },
  "customerInfo":
  {
    "email": "user@example.com",
    "phone": "+79001234567"
  },
  "customValues":
  [
	 {"name": "custom field 0", "value": "123321"},
	 {"name": "custom field 1", "value": "123321"}
  ]
}
```

>Standard HTTP 200 JSON response:

```json
{
  "id": "43913ddc000c4d3990fddbd3980c1725",
  "userWebLink": "https://secure.mandarinpay.com/Pay?transaction=0eb51e74-e704-4c36-b5cb-8f0227621518"
}
```
>Standard HTTP 400 Bad request:

```json
{
  "error": "Invalid request"
  
}
```

В поле `action` допустимы значения:

|Значение|Описание|
|----|-------------|
|pay|списание денег с карты|
|payout|зачисление денег на карту|

Для совершения транзакции необходимо перенаправить пользователя на url, полученный в поле `userWebLink`. 

Переход пользователя обратно на ваш сайт не означает окончательного выполнения транзакции, ожидайте получение callback-вызова.


## Создание привязки карты (card-binding)

`POST` `https://secure.mandarinpay.com/api/card-bindings`

```json
{
    "customerInfo":
    {
        "email": "user@example.com",
        "phone": "+79001234567"
    }
}
```

>Standard HTTP 200 JSON response:

```json
{
    "id": "0eb51e74-e704-4c36-b5cb-8f0227621518",
    "userWebLink": "https://secure.mandarinpay.com/CardBindings/New?id=0eb51e74-e704-4c36-b5cb-8f0227621518"
}
```

Полученный `id` следует сохранить, а пользователя перенаправить по ссылке, полученной в поле `userWebLink`. 

Привязанную карту можно будет использовать после получения callback-уведомления об её успешной привязке.

## Создание транзакции по привязанной карте (transaction-with-card-binding)

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "pay/payout",
      "price": "10"
  },
  "target":
  {
      "card": "0eb51e74-e704-4c36-b5cb-8f0227621518"
  }
}
```

>Standard HTTP 200 JSON response:

```json
{
  "id": "43913ddc000c4d3990fddbd3980c1725"
}
```

>Standard HTTP 400 Bad request:

```json
{
  "error": "Invalid request"
}
```

`POST` `https://secure.mandarinpay.com/api/transactions`

В поле `action` допустимы значения:

|Значение|Описание|
|----|-------------|
|pay|списание денег с карты|
|payout|зачисление денег на карту|

Транзакция будет совершена в асинхронном режиме, ожидайте получение callback-вызова

## Создание односторонней привязки карты по известному номеру (oneside-card-binding-with-known-card)

```json
{
    "customerInfo":
    {
        "email": "user@example.com",
        "phone": "+79001234567"
    },
	"target":
    {
      "knownCardNumber": "4012888888881881"
    }
}
```

>Standard HTTP 200 JSON response:

```json
{
    "id": "0eb51e74-e704-4c36-b5cb-8f0227621518"
}
```

`POST` `https://secure.mandarinpay.com/api/card-bindings`

Данная ривязка доступна для использования только на с использованием `action=payout`.

## Отмена транзакции (refund)

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "reversal"
  },
  "target":
  {
      "transaction": "43913ddc000c4d3990fddbd3980c1725"
  }
}
```

>Standard HTTP 200 JSON response:

```json
{
  "id": "43913ddc000c4d3990fddbd3980c1725"
}
```

`POST` `https://secure.mandarinpay.com/api/transactions`

Отмена будет совершена в асинхронном режиме, ожидайте получение callback-вызова

## Создание транзакции на повторное списание (rebill)

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "pay",
      "price": "10"
  },
  "target":
  {
      "rebill": "43913ddc000c4d3990fddbd3980c1725"
  }
}
```

>Standard HTTP 200 JSON response:

```json
{
  "id": "43913ddc000c4d3990fddbd3980c1725"
}
```

>Standard HTTP 400 Bad request:

```json
{
  "error": "Invalid request"
}
```

`POST` `https://secure.mandarinpay.com/api/transactions`

В поле `rebill` необходимо указать Id ранее проведённой успешной транзакции на списание средств с карты.

Транзакция будет совершена в асинхронном режиме, ожидайте получение callback-вызова

## Создание транзакции по известному номеру карты (create-transaction-with-known-card)

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "payout",
      "price": "10"
  },
  "customerInfo":
  {
    "email": "user@example.com",
    "phone": "+79001234567"
  },
  "target":
  {
      "knownCardNumber": "4012888888881881"
  }
}
```

>Standard HTTP 200 JSON response:

```json
{
  "id": "43913ddc000c4d3990fddbd3980c1725"
}
```

>Standard HTTP 400 Bad request:

```json
{
  "error": "Invalid request"
}
```

`POST` `https://secure.mandarinpay.com/api/transactions`

Транзакция будет совершена в асинхронном режиме, ожидайте получение callback-вызова

## Создание выплаты на мобильный телефон {#create-transaction-to-mobile-phone}

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "payout",
      "price": "10"
  },
  "customerInfo":
  {
    "email": "user@example.com",
    "phone": "+79001234567"
  },
  "target": "phone"
}
```

>Standard HTTP 200 JSON response:

```json
{
  "id": "43913ddc000c4d3990fddbd3980c1725"
}
```

>Standard HTTP 400 Bad request:

```json
{
  "error": "Invalid request"
}
```

`POST` `https://secure.mandarinpay.com/api/transactions`

Транзакция будет совершена в асинхронном режиме, ожидайте получение callback-вызова

## Блокировка (предавторизация) средств с последующим списанием (preauth)

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "pay",
      "price": "10"
  },
  "target":
  {
      "preauth": "43913ddc000c4d3990fddbd3980c1725"
  }
}
```

>Standard HTTP 200 JSON response:

```json
{
  "id": "43913ddc000c4d3990fddbd3980c1725"
}
```

>Standard HTTP 400 Bad request:

```json
{
  "error": "Invalid request"
}
```

> Отказ от блокировки средств

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "reversal"
  },
  "target":
  {
      "preauth": "43913ddc000c4d3990fddbd3980c1725"
  }
}
```

`POST` `https://secure.mandarinpay.com/api/transactions`

Предавторизация производится аналогично обычной покупке как через API платежной страницы, так и через REST API. Необходимо в поле `action` передать значение `preauth`. 

После того как придёт уведомление об успехе предавторизации полученый вв нём Id транзакции можно использовать для полного или частичного списания через REST API, используя этот номер в качесве значения для `target`->`preauth`:

Транзакция будет совершена в асинхронном режиме, ожидайте получение callback-вызова

## Переводы средств через систему БЭСТ/Юнистрим (BEST / Unistream)

```json
{
  "payment":
  {
      "orderId": "123321",
      "action": "payout",
      "price": "10"
  },
  "customerInfo":
  {
    "email": "user@example.com",
    "phone": "+79001234567",
    "firstName": "Василий",
    "middleName": "Иванович",
    "lastName": "Пупкин",
    "document":
    {
        "rfnspDocId": 21,
        "docSeria": "7709",
        "docNum": "543987",
        "docIssuer":  "УПРАВЛЕНИЕ ФЕДЕРАЛЬНОЙ МИГРАЦИОННОЙ СЛУЖБЫ РОССИИ ПО ГОР.МОСКВЕ",
        "docIssuerCode": "770-001",
        "docIssuingDate": "2011-01-01",
    },
    "birthDate": "1980-01-01",
    "birthPlace": "г. Москва",
    "countryCode": "643",
    "taxInfo":
    {
          "isRussianFederationTaxResident": true
    }
  },
  "target": "bestmt"
}
```

`POST` `https://secure.mandarinpay.com/api/transactions`

Транзакция будет совершена в асинхронном режиме, ожидайте получение callback-вызова

#Уведомления о статусе транзакций и привязок карт (notifications)

>Пример проверки `sign` на языке PHP {#sign-check-php}

```php
function check_sign($secret, $req)
{
        $sign = $req['sign'];
        unset($req['sign']);
        $to_hash = '';
        if (!is_null($req) && is_array($req)) {
                ksort($req);
                $to_hash = implode('-', $req);
        }

        $to_hash = $to_hash .'-'. $secret;
        $calculated_sign = hash('sha256', $to_hash);
        return $calculated_sign == $sign;
}

check_sign("123", $_POST);

```

>Пример проверки `sign` на языке C\# {#sign-check-csharp}

```cs
public static string Calculate(string secret, IDictionary<string, string> values)
{
    using (var sha256 = System.Security.Cryptography.SHA256.Create())
        return BitConverter.ToString(sha256.ComputeHash(Encoding.UTF8.GetBytes(string.Join("-", values.OrderBy(x => x.Key, StringComparer.Ordinal).Select(x => x.Value)) + "-" + secret))).ToLower().Replace("-", "");
}

public static bool CheckSign(string secret, HttpRequest request)
{
    var sign = request.Form["sign"];
    if(string.IsNullOrWhiteSpace(sign))
      return false;
    return sign ==
        Calculate(secret, request.Form.Keys.Where(k => k != "sign").ToDictionary(k => k, k => request.Form[k]));
}

```

Уведомления высылаются на указанный в клиентском интерфейсе `CallbackUrl` в теле POST-запроса с `Content-Type: application/x-www-urlencoded`. 

В качестве ответа обработчик должен вернуть HTTP-код `200` и тело ответа `OK`.

Для подтверждения того, что уведомление пришло именно от системы MandarinPay необходимо проверять значение поля `sign`, являющегося SHA256-суммой от значений всех параметров запроса, отсортированных алфавитно по именам и `SharedSec`, разделённых символом `-`.

В зависимости от значения `object_type` уведомления бывают следующих видов:

|Тип|Описание|
|----|---------|
|`card_binding`|привязка карты|
|`transaction`|транзакция|

## Уведомления о привязках карт (notification-card-binding)


|Поле|Пример|Описание|
|----|---------|-----|
|`card_binding`|`38467cb0-da73-4755-84da-2acb7f638a59`|Идентификационный номер привязки|
|`card_holder`|`VASILY PUPKIN`|Держатель карты|
|`card_number`|`123456XXXXXX7890`|Номер карты|
|`card_expiration_year`|`18`|Год срока действия карты|
|`card_expiration_month`|`9`|Месяц срока действия карты|
|`status`|`success`|Статус привязки


## Уведомления о транзакциях (notification-transaction)

|Поле|Пример|Описание|
|----|---------|-----|
|`transaction`|`05991602ed7c47d996ca7ab119b07f66`|ID транзакции в системе|
|`status`|`success`, `failed`|Статус операции|
|`action`|`pay`, `payout`|Действие (покупка или выплата на карту)|
|`merchantId`|`42`|Id мерчанта|
|`price`|`12.34`|Сумма платежа. Обязательна к передаче в текущей версии системы.|
|`orderId`|`1234OR`|Уникальный номер заказа в системе Продавца.|
|`customer_email`|`user@example.com`|Email пользователя|
|`customer_phone`|`+71234567890`|Телефон пользователя|


# Информационный протокол XML API MandarinInfo (упрощенная идентификация)

Документация расположена по [адресу](https://mandarinpay.com/sites/default/files/XML_API_MandarinInfo.pdf)