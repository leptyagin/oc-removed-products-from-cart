__Обратите внимание, что всё ниже описанное справедливо для opencart 3.0.3.8 Русская сборка (от 28 августа 2021) [ссылка](https://opencart-russia.ru/). Так же советую делать бекап редактируемых файлов__

## Идея

по дефолту при добавлении товара в корзину, опенкарт добавляет новую строку в таблицу `'db_prefix'_cart` (возможно у вас какой-то кастомный префикс к таблицам, у меня дефолтный - 'oc'), в которой сохранятеся id сессии юзера и id товара, по которому потом получает данные о товаре, а при удалении эта строка удаляется из таблицы.

Идея залючается в том чтобы добавить в таблицу столбец `is_deleted` (boolean), который будет хранить информацию удален ли товар (0 или 1), и вместо того чтобы удалять строку, в этот столбец будет заноситься информация о состоянии, и вся дальнейшая обработка будет происходить по этому значению в этом столбце, в laravel есть подобный функционал soft delete, поэтому далее для удобства я буду называть функционал, который мы реализуем мягким удалением.
## Этапы

### База данных

Необходимо в базе данных найти таблицу `oc_cart` и добавить новый столбец с названием `is_deleted`, тип boolean (tinyInt), nullable, с значением по умолчанию null. Вы можете это сделать через веб-интерфейс (phpMyAdmin) или через sql запрос, как вам удобно, думаю не должно составить труда. 

### Запросы в бд 

(финальный файл в репозитории `/ready/CartClass.php`)

Открываем файл `system/library/cart/cart.php`. Ищем метод `getProducts()`, собственно он и отвечает за вывод товаров, которые находятся в корзине. Массив, который возвращает этот метод, работает для всего движка, там где есть данные о добавленных товарах в корзину. 

1. Ищем переменную `$cart_query`, в которой sql запрос. Добавляем в запрос условие `is_deleted = 0`. В итоге запрос должен выглядеть так:

```sh
$cart_query = $this->db->query("SELECT * FROM " . DB_PREFIX . "cart WHERE is_deleted = 0 AND api_id = '" . (isset($this->session->data['api_id']) ? (int)$this->session->data['api_id'] : 0) . "' AND customer_id = '" . (int)$this->customer->getId() . "' AND session_id = '" . $this->db->escape($this->session->getId()) . "'");
```

<!-- 2. Далее видим в коде, что идет перебор данных, которые получили в ответ на sql запрос, в этом переборе ищем `$product_data[] = array(` и ниже добавляем строку:

```sh
'is_deleted'    => $cart['is_deleted'],
``` -->

3. В самом конце класса добавим новый метод, который будет отвечать за это самое мягкое удаление товара: 

```sh
public function removeProduct($cart_id)
{
    $this->db->query("UPDATE `" . DB_PREFIX . "cart` SET is_deleted = 1 WHERE cart_id = '" . (int)$cart_id . "'");
}
```
Как видим, метод принимает аргумент `$cart_id`, который является auto increment значением, которое просто идентифицирует нужный товар. В данном методе проверять сессию пользователя и сам product_id нет необходимости. Сам этот код отвечает за удаление товара из корзины, а именно `is_deleted` становиться `true`

4. Сразу же после добавляем еще один метод
```sh
public function restoreProduct($cart_id)
{
    $this->db->query("UPDATE `" . DB_PREFIX . "cart` SET is_deleted = 0 WHERE cart_id = '" . (int)$cart_id . "'");
}
```
Этот код отвечает за восстановление товара в корзину, то есть `is_delete` переходит состояние `false`.

5. Далее интресный момент, получам список удаленных товаров.

Выше озанкомились с методом `getProducts()`, который возвращает массив товаров, которые находятся в корзине, в котором обрабатываются такие данные как количество, цена, цена по скидке, выбранные опции и в данном случае в таблице, где выводятся удаленные товары, нужно сохранить все эти данные о товаре. Мы просто скопируем код из метода `getProducts()`, но изменим условие `is_deleted = 1` в запросе, думаю тут не должно возникнуть вопросов.

6. Далее предвидем ошибку и посмотрим на метод `hasProducts()`. Как понятно из имени метод проверяет есть ли товары в корзине, а именно возвращает количество (integer) товаров в корзине, далее в контроллере будет выполнена проверка, есть это число больше 0, то идет дальнейшая обработка и данные уходят в файл темплейта с таблицей и всем остальным, а если не больше 0, то выводится темлплейт, в котором просто выводится сообщение, что в корзине отсутсвуют товары, это не совсем то что нужно, поэтому нужно чтобы удаленные товары тоже подсчитывались и поэтому в итоге метод выглядит так:
```sh
public function hasProducts()
{
	return (int)(count($this->getProducts()) + count($this->getRemovedProducts()));
}
```

## Работа с контроллером корзины 

(финальный файл в репозитории `/ready/CartController.php`)

Переходим в класс контроллер, который отвечает на страницу корзины по директории `catalog/controller/checkout/cart.php`

В методе `index()` ищем строку `$products = $this->cart->getProducts();`
, ниже видим перебор массива с обработкой данных, после этого перебора начнем писать код для обработки удаленных товаров

1. Объявляем массив, который передается в `$data`

```sh
$data['removed_products'] = array();
```

2. В новую переменную прокидывыаем данные, которые вернул метод `getRemovedProducts()` в php классе Cart

```sh
$removed_products = $this->cart->getRemovedProducts();
```

3. Следующим шагом будет обработка данных, которые вернул метод `getRemovedProducts()` и на самом деле это код почти идентичен коду перебора `$products = $this->cart->getProducts();`, поэтому можете скопировать его за исключением того что `$data['products'][] = array(` заменить на `$data['removed_products'][] = array(`


4. После редактирования метода `index()` отредактируем метод `remove()`.

Строку `$this->cart->remove($this->request->post['key']);` заменим на `$this->cart->removeProduct($key);`

Уберем все unset'ы и заметим, что в объект `$json` добавляются ключи `total` и `success`. К ключу `success` вопросов нет, он передает строку с текстом об уведомлении, что товар удален, которую берет из языкового файла, а вот ключ `total` не совсем подходит, вернее его реализация, поэтому поменяем его на следующий код, который более корректно отрабатывает в момент, когда все товары из корзины удалены и остались только удаленные товары

```sh
$data['products'] = array();
$products = $this->cart->getProducts();
foreach ($products as $product) {
	if ($this->customer->isLogged() || !$this->config->get('config_customer_price')) {
		$unit_price = $this->tax->calculate($product['price'], $product['tax_class_id'], $this->config->get('config_tax'));
		$price = $this->currency->format($unit_price, $this->session->data['currency']);
		$total = $this->currency->format($unit_price * $product['quantity'], $this->session->data['currency']);
	} else {
		$price = false;
		$total = false;
	}
}
if($products) {
	$json['total'] = sprintf($this->language->get('text_items'), $this->cart->countProducts() + (isset($this->session->data['vouchers']) ? count($this->session->data['vouchers']) : 0), $this->currency->format($total, $this->session->data['currency']));
} else {
	$json['total'] = sprintf($this->language->get('text_items'), 0, $this->currency->format(0, $this->session->data['currency']));
}
```
и чтобы этот код сработал нужно подключил модель коризны, добавим `$this->load->model('checkout/cart');` рядом с `$this->load->language('checkout/cart');`

5. Добавим метод `restore()` для восстановления товара, тело метода идентично методу `remove()`, за исключением того что нужно `$this->cart->removeProduct($key);` заменить на `$this->cart->restoreProduct($key);`


## Работа с view темплейтом корзины 
На самом деле, тут мало чего интересного, поэтому просто посмотреть финальный файл в репозитории `/ready/cart.twig`. Обратите внимание что таблице, в которой выводятся товары, которые добавлены в корзину, я присвоил `id="products-in-cart"`, а таблице, которая выводит удаленные товары, `id="deleted-products"`.


## ajax запросы

(финальный файл в репозитории `/ready/common.js`)

Открываем файл по `catalog/view/javascript/common.js` и ищем функционал, который отвечает за работу с корзиной `var cart = {`. Этот объект хранит в ключах значения функции.

1. Выше этого кода добавим функцию, которая будет отправлять запрос в контроллер в метод `index` и обновляет данные таблиц и мини-корзины.

```sh
function updateCart() {
	$.ajax({
		url: 'index.php?route=checkout/cart',
		dataType: 'html',
		success: function (html) {
			$('#products-in-cart').html($(html).find('#products-in-cart').html());
			$('#deleted-products').html($(html).find('#deleted-products').html());
			$('#cart > ul').load('index.php?route=common/cart/info ul li');
		}
	});
}
```


2. Рассмотрим `remove`, который можно найти `'remove': function(key) {`: 
 Видим стандартный ajax запрос, который отправляет запрос к `index.php?route=checkout/cart/remove`, где `remove` метод в контроллере, который мы выше редактировали. Обратим внимание на `success`, идет обновление информации о количестве товара и цене в мини-корзине в шапке сайта, далее идет проверка по на какой странице находится юзер, по итогам которой перекидывает на страницу корзины, нас это не совсем устраивает. Переделаем функцию вот таким образом:

```sh
success: function(json) {
	$('.alert-dismissible, .text-danger').remove();

	if (json['redirect']) {
		location = json['redirect'];
	}

	if (json['success']) {
		$('#content').parent().before('<div class="alert alert-success alert-dismissible"><i class="fa fa-check-circle"></i> ' + json['success'] + ' <button type="button" class="close" data-dismiss="alert">&times;</button></div>');

		setTimeout(function () {
			$('#cart > button').html('<span id="cart-total"><i class="fa fa-shopping-cart"></i> ' + json['total'] + '</span>');
		}, 100);

		updateCart();
	}
},
```

3. Далее создадим новую функцию в объекте `cart` по ключу `restore`, которая, на самом деле, почти идентична функции `remove`, за исключением того что запрос отправляется в `index.php?route=checkout/cart/remove`


## Итог
Теперь для достоверности советую обновить кэш модификаторов и проследить чтобы применился актульный js