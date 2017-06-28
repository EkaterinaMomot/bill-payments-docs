# QIWI Bill Payments REST API {#pull_rest_api}

###### Последнее обновление: 2017-05-31 | [Редактировать на GitHub](https://github.com/QIWI-API/bill-payments-rest-api-docs/blob/master/_payments_invoice_api_ru.html.md)

## Последовательность операций

<img src="images/pullrest_1.png" />

* Пользователь формирует заказ на сайте провайдера.

* Провайдер выполняет запрос [Cоздать счет](#invoice_rest) с параметрами авторизации.

* Провайдер перенаправляет пользователя на [Платежную форму](#checkout) Visa QIWI Wallet для оплаты счета. Если перенаправление по какой-либо причине не произошло, пользователю потребуется оплатить счет через любой другой интерфейс Visa QIWI Wallet (сайт qiwi.com, платежный терминал QIWI, мобильное приложение Android или iOS).

* После успешной оплаты счета провайдер получает [уведомление](#notification) (предварительно необходимо настроить отправку уведомлений в [Личном кабинете](https://kassa.qiwi.com)). Уведомления об оплате счета содержат параметры авторизации, которые необходимо проверять на сервере провайдера. Уведомления могут высылаться как при оплате счета, так и в других случаях (если пользователь отклонил счет или при оплате возникла ошибка).

* При необходимости, провайдер:
  * [запрашивает текущий статус оплаты счета](#invoice-status),
  * [отменяет счет](#cancel) (при условии, что он еще не был оплачен).

* После подтверждения оплаты счета провайдер исполняет заказ пользователя.

## Авторизация {#auth}

Запросы мерчанта авторизуются посредством HTTP basic-авторизации. Для авторизации используется [SECRET_KEY](#auth_param).<br>

~~~shell
user@server:~$ curl "адрес сервера"
  --header "Authorization: Bearer MjMyNDQxMjM6NDUzRmRnZDQ0Mw=="
~~~

<aside class="notice">
Заголовок представляет собой параметр Authorization, значение которого представлено как: "Bearer <a href="#auth_param">SECRET_KEY</a>"
</aside>



## Выставление счета за покупку {#invoice_rest}


Запрос выставляет новый счет на указанный номер телефона (номер кошелька QIWI Wallet). Тип запроса - HTTP PUT.

~~~shell
user@server:~$ curl "https://api.qiwi.com/api/v3/prv/bills/BILL-1"
  -X PUT --header "Accept: text/json"
  --header "Authorization: Bearer ***"
  -d 'user=tel%3A%2B79161111111&amount=10.00&ccy=RUB&comment=test&lifetime=2016-09-25T15:00:00'


HTTP/1.1 200 OK
Content-Type: text/json
{
  "response": {
     "result_code": 0,
     "bill": {
        "bill_id": "BILL-1",
        "amount": "10.00",
        "ccy": "RUB",
        "status": "waiting",
        "error": 0,
        "user": "tel:+79161111111",
        "comment": "test"
     }
  }
}
~~~

~~~http
PUT /api/v3/prv/bills/BILL-1 HTTP/1.1
Accept: text/json
Authorization: Bearer ***
Host: api.qiwi.com
Content-Type: application/x-www-form-urlencoded; charset=utf-8

user=tel%3A%2B79031234567%26amount=10.0%26ccy=RUB%26comment=test%26lifetime=2016-11-25T09%3A00%3A00


HTTP/1.1 200 OK
Content-Type: text/json
{
  "response": {
     "result_code": 0,
     "bill": {
        "bill_id": "BILL-1",
        "amount": "10.00",
        "ccy": "RUB",
        "status": "waiting",
        "error": 0,
        "user": "tel:+79031234567",
        "comment": "test"
     }
  }
}
~~~

<h3 class="request method">Запрос → PUT</h3>

<ul class="nestedList url">
    <li><h3>URL <span>https://api.qiwi.com/api/v3/prv/bills/<a>bill_id</a></span></h3>
        <ul>
        <strong>В pathname PUT-запроса используется параметр счета:</strong>
             <li><strong>bill_id</strong> - уникальный идентификатор счета в системе провайдера.</li>
        </ul>
    </li>
</ul>

<ul class="nestedList header">
    <li><h3>HEADERS</h3>
        <ul>
             <li>Accept: text/json</li>
             <li>Content-type: application/x-www-form-urlencoded; charset=utf-8</li>
             <li>Authorization: Bearer <a href="#auth_param">SECRET_KEY</a></li>
        </ul>
    </li>
</ul>

<ul class="nestedList params">
    <li><h3>Parameters</h3><span>Параметры передаются в теле запроса как formdata</span>
    </li>
</ul>

Параметр|Описание|Тип|Обяз.
---------|--------|---|------
user | Идентификатор номера QIWI Wallet, на который выставляется счет (в международном формате), с префиксом "tel:" | String(20)|+
amount | Сумма, на которую выставляется счет. Способ округления зависит от валюты | Number(6.3)|+
ccy | Идентификатор валюты (Alpha-3 ISO 4217 код). Может использоваться любая валюта, предусмотренная договором с КИВИ | String(3)|+
comment | Комментарий к счету | String(255)|+
lifetime | Дата, до которой счет будет доступен для оплаты. Если счет не будет оплачен до этой даты, ему присваивается финальный статус и последующая оплата станет невозможна.<br> **Внимание! По истечении 28 суток от даты выставления счет автоматически будет переведен в финальный статус.**|dateTime|+
pay_source |<br>"mobile" - оплата счета будет производиться с баланса мобильного телефона пользователя, <br>"qw" – любым способом через интерфейс Visa QIWI Wallet.<br> По умолчанию "qw" |String|-



<h3 class="request">Ответ ←</h3>



[Параметры ответа](#response_bill)

[Ответ в случае ошибки](#response_error)

## Проверка статуса оплаты счета {#invoice-status}

Проверка статуса позволяет получить текущий статус оплаты счета клиентом.

<h3 class="request method">Запрос → GET</h3>

~~~http
GET /api/v3/prv/bills/BILL-1 HTTP/1.1
Accept: text/json
Authorization: Bearer ***
Host: api.qiwi.com
Content-Type: application/x-www-form-urlencoded; charset=utf-8

HTTP/1.1 200 OK
Content-Type: text/json
{
  "response": {
     "result_code": 0,
     "bill": {
        "bill_id": "BILL-1",
        "amount": "10.00",
        "ccy": "RUB",
        "status": "waiting",
        "error": 0,
        "user": "tel:+79031234567",
        "comment": "test"
     }
  }
}
~~~

~~~shell
user@server:~$ curl "https://api.qiwi.com/api/v3/prv/bills/BILL-1"
  --header "Authorization: Bearer ***"
  --header "Accept: text/json"

HTTP/1.1 200 OK
Content-Type: text/json
{
  "response": {
     "result_code": 0,
     "bill": {
        "bill_id": "BILL-1",
        "amount": "10.00",
        "ccy": "RUB",
        "status": "waiting",
        "error": 0,
        "user": "tel:+79031234567",
        "comment": "test"
     }
  }
}
~~~

<ul class="nestedList url">
    <li><h3>URL <span>https://api.qiwi.com/api/v3/prv/bills/<a>bill_id</a></span></h3>
        <ul>
        <strong>Параметр передается в pathname.</strong>
             <li><strong>bill_id</strong> - уникальный идентификатор счета в системе провайдера.</li>
        </ul>
    </li>
</ul>

<ul class="nestedList header">
    <li><h3>HEADERS</h3>
        <ul>
             <li>Authorization: Bearer <a href="#auth_param">SECRET_KEY</a></li>
             <li>Accept: text/json</li>
        </ul>
    </li>
</ul>

<h3 class="request">Ответ ←</h3>

[Параметры ответа](#response_bill)

[Ответ в случае ошибки](#response_error)



## Отмена неоплаченного счета {#cancel}
позволяет отменить неоплаченный клиентом счет.

<h3 class="request method">Запрос → PATCH</h3>

~~~http
PATCH /api/v2/prv/bills/BILL-2 HTTP/1.1
Accept: text/json
Authorization: Basic ***
Host: api.qiwi.com
Content-Type: application/x-www-form-urlencoded; charset=utf-8

status=rejected

HTTP/1.1 200 OK
Content-Type: text/json
{
   "response": {
      "result_code": 0,
      "bill": {
         "bill_id": "BILL-2",
         "amount": "10.00",
         "ccy": "RUB",
         "status": "rejected",
         "error": 0,
         "user": "tel:+79031234567",
         "comment": "test"
      }
   }
}
~~~

~~~shell
user@server:~$ curl "https://api.qiwi.com/api/v3/prv/bills/BILL-2"
  -X PATCH
  --header "Authorization: Bearer ***"
  --header "Accept: text/json"
  -d 'status=rejected'


HTTP/1.1 200 OK
Content-Type: text/json
{
   "response": {
      "result_code": 0,
      "bill": {
         "bill_id": "BILL-2",
         "amount": "10.00",
         "ccy": "RUB",
         "status": "rejected",
         "error": 0,
         "user": "tel:+79031234567",
         "comment": "test"
      }
   }
}
~~~

<ul class="nestedList url">
    <li><h3>URL <span>https://api.qiwi.com/api/v3/prv/bills/<a>bill_id</a></span></h3>
        <ul>
        <strong>Параметры:</strong>
             <li><strong>bill_id</strong> - уникальный идентификатор счета в системе провайдера, передается в pathname.</li>
             <li><strong>status</strong> - строка "rejected" (статус для отмены), передается в body запроса.</li>
        </ul>
    </li>
</ul>

<ul class="nestedList header">
    <li><h3>HEADERS</h3>
        <ul>
             <li>Authorization: Basic <a href="#auth_param">SECRET_KEY</a></li>
             <li>Accept: text/json</li>
        </ul>
    </li>
</ul>

<h3 class="request">Ответ ←</h3>

[Параметры ответа](#response_bill)

[Ответ в случае ошибки](#response_error)



## Возврат оплаченного счета {#refund}

С помощью данного запроса можно произвести полный или частичный возврат средств по счету, оплаченному клиентом, на его учетную запись Visa QIWI Wallet. При этом создается платеж, обратный платежу на оплату счета. Валюта платежа совпадает с валютой исходного счета.

По одному и тому же счету можно выполнять несколько операций возврата, при условии что:

* сумма всех операций возврата не превышает суммы исходного счета;
* для разных операций возврата одного счета используются разные идентификаторы.

<aside class="warning">
Если сумма, переданная в запросе, превышает сумму самого счета либо сумму счета, оставшуюся после предыдущих возвратов, в ответе будет возвращен код ошибки 242.
</aside>

### Последовательность операций возврата

<img src="images/pullrest_2.png" />

* Провайдер отправляет запрос на осуществление возврата.
* Чтобы убедиться, что возврат платежа проведен успешно, можно периодически опрашивать API о [текущем статусе возврата](#refund_status) до получения финального статуса.
* Данный сценарий можно повторять несколько раз до тех пор, пока счет не будет полностью отменен (возвращена вся сумма).

<h3 class="request method">Запрос → PUT</h3>

~~~shell
user@server:~$ curl "https://api.qiwi.com/api/v3/prv/bills/test234578/refund/122swbill"
  -v -w "%{http_code}"
  -X PUT
  --header "Accept: text/json"
  --header "Authorization: Bearer ***"
  -d 'amount=5.0'

HTTP/1.1 200 OK
Content-Type: text/json
{
   "response": {
      "result_code": 0,
      "refund": {
         "refund_id": "122swbill",
         "amount": "5.00",
         "status": "success",
         "error": 0
      }
   }
}
~~~

~~~http
PUT /api/v3/prv/bills/BILL-1/refund/122swbill HTTP/1.1
Accept: text/json
Authorization: Bearer ***
Host: api.qiwi.com
Content-Type: application/x-www-form-urlencoded; charset=utf-8

amount=5.0

HTTP/1.1 200 OK
Content-Type: text/json
{
   "response": {
      "result_code": 0,
      "refund": {
         "refund_id": "122swbill",
         "amount": "5.00",
         "status": "success",
         "error": 0
      }
   }
}
~~~

<ul class="nestedList url">
    <li><h3>URL <span>https://api.qiwi.com/api/v3/prv/bills/<a>bill_id</a>/refund/<a>refund_id</a></span></h3>
        <ul>
        <strong>Параметры передаются в pathname.</strong>
             <li><strong>bill_id</strong> - уникальный идентификатор счета в системе провайдера.</li>
             <li><strong>refund_id</strong> - идентификатор операции, уникальный в рамках операций возврата счета <br> `{bill_id}`. Формат идентификатора: строка от 1 до 9 символов, содержащая только прописные или строчные латинские буквы (a-z, A-Z) и цифры от 0 до 9.</li>
        </ul>
    </li>
</ul>

<ul class="nestedList header">
    <li><h3>HEADERS</h3>
        <ul>
             <li>Authorization: Bearer <a href="#auth_param">SECRET_KEY</a></li>
             <li>Accept: text/json</li>
             <li>Content-Type: application/x-www-form-urlencoded; charset=utf-8</li>
        </ul>
    </li>
</ul>

<ul class="nestedList params">
    <li><h3>Parameters</h3><span>Параметры передаются в теле запроса как formdata</span>
    </li>
</ul>

Параметр|Описание|Тип
---------|--------|---|------
amount | Сумма возврата. Должна быть меньше либо равна сумме счета `{bill_id}`, по которому производится возврат. Способ округления зависит от валюты счета | Number(6.3)


<h3 class="request">Ответ ←</h3>

[Параметры ответа](#response_refund)

[Ответ в случае ошибки](#response_error)

## Проверка статуса возврата {#refund_status}

Позволяет получить текущий статус операции возврата средств по счету.

<h3 class="request method">Запрос → GET</h3>

~~~shell
user@server:~$ curl "https://api.qiwi.com/api/v3/prv/bills/test234578/refund/122swbill"
  -v -w "%{http_code}"
  --header "Accept: text/json"
  --header "Authorization: Bearer ***"


HTTP/1.1 200 OK
Content-Type: text/json
{
   "response": {
      "result_code": 0,
      "refund": {
         "refund_id": "122swbill",
         "amount": "5.00",
         "status": "success",
         "error": 0
      }
   }
}
~~~

~~~http
GET /api/v3/prv/bills/BILL-1/refund/122swbill HTTP/1.1
Accept: text/json
Authorization: Bearer ***
Host: api.qiwi.com
Content-Type: application/x-www-form-urlencoded; charset=utf-8


HTTP/1.1 200 OK
Content-Type: text/json
{
   "response": {
      "result_code": 0,
      "refund": {
         "refund_id": "122swbill",
         "amount": "5.00",
         "status": "success",
         "error": 0
      }
   }
}
~~~

<ul class="nestedList url">
    <li><h3>URL <span>https://api.qiwi.com/api/v3/prv/bills/<a>bill_id</a>/refund/<a>refund_id</a></span></h3>
        <ul>
        <strong>Параметры передаются в pathname.</strong>
             <li><strong>bill_id</strong> - уникальный идентификатор счета в системе провайдера.</li>
             <li><strong>refund_id</strong> - идентификатор операции, уникальный в рамках операций возврата счета `bill_id`. Формат идентификатора: строка от 1 до 9 символов, содержащая только прописные или строчные латинские буквы (a-z, A-Z) и цифры от 0 до 9</li>
        </ul>
    </li>
</ul>

<ul class="nestedList header">
    <li><h3>HEADERS</h3>
        <ul>
             <li>Authorization: Bearer <a href="#auth_param">SECRET_KEY</a></li>
             <li>Accept: text/json</li>
        </ul>
    </li>
</ul>

<h3 class="request">Ответ ←</h3>

[Параметры ответа](#response_refund)

[Ответ в случае ошибки](#response_error)
