__please pay attention, all of the following code is true for opencart 3.0.3.8 russian version (August 28, 2021) [link](https://opencart-russia.ru/). Also advise you to backup the edited files__

## Idea

by default, when adding an item to the cart, opencart adds a new row to the table `db_prefix` _cart' (you may have custom prefix to the tables, my default is 'oc'), which stores the user's session id and the product id, which then receives product data, and when deleted this row is deleted from the table.

The idea is to add the `is_deleted` (boolean) column to the table, which will store information on whether the product has been deleted (0 or 1), and instead of deleting the row, status information will be entered in this column, and all further processing will take place according to this value in this column, in laravel there is a similar soft delete functionality, so for convenience, I will call the functionality that we implement soft delete.

## Stages

### Database

It is necessary to find the `oc_cart` table in the database and add a new column named `is_deleted`, type boolean (tinyInt), nullable, with the default value null. You can do this through the web interface (phpMyAdmin) or through an sql query, as it suits you, I think it should not be difficult. 

### Database queries 

(the final file in the repository `/ready/CartClass.php `)

Opening the file `system/library/cart/cart.php `. We are looking for the `GetProducts()` method, actually it is responsible for the output of goods that are in the cart. The array that this method returns works for the entire engine, where there is data about added items to the cart.

1. We are looking for the variable `$cart_query` in which the sql query is. Adding the condition `is_deleted = 0` to the query. As a result, the request should look like this:

```sh
$cart_query = $this->db->query("SELECT * FROM " . DB_PREFIX . "cart WHERE is_deleted = 0 AND api_id = '" . (isset($this->session->data['api_id']) ? (int)$this->session->data['api_id'] : 0) . "' AND customer_id = '" . (int)$this->customer->getId() . "' AND session_id = '" . $this->db->escape($this->session->getId()) . "'");
```

<!-- 2. Далее видим в коде, что идет перебор данных, которые получили в ответ на sql запрос, в этом переборе ищем `$product_data[] = array(` и ниже добавляем строку:

```sh
'is_deleted'    => $cart['is_deleted'],
``` -->

3. At the very end of the class, we will add a new method that will be responsible for this softest removal of the product:

```sh
public function removeProduct($cart_id)
{
    $this->db->query("UPDATE `" . DB_PREFIX . "cart` SET is_deleted = 1 WHERE cart_id = '" . (int)$cart_id . "'");
}
```

As you can see, the method takes the argument `$cart_id`, which is an auto increment value that simply identifies the desired product. In this method, there is no need to check the user's session and the product_id itself. This code itself is responsible for removing the product from the cart, namely `is_deleted` becomes `true'.


4. Immediately after adding another method

```sh
public function restoreProduct($cart_id)
{
    $this->db->query("UPDATE `" . DB_PREFIX . "cart` SET is_deleted = 0 WHERE cart_id = '" . (int)$cart_id . "'");
}
```
This code is responsible for restoring the product to the cart, that is, `is_delete` goes to the `false` state.

5. Next, an interesting moment, I will receive a list of deleted items.

Above, we discussed the `GetProducts()` method, which returns an array of products that are in the cart, which processes data such as quantity, price, discount price, selected options, and in this case, in the table where deleted products are displayed, you need to save all this product data. We'll just copy the code from the `GetProducts()` method, but change the condition `is_deleted = 1` in the request, I think there should be no questions.

6. Next, we anticipate the error and look at the `hasProducts()` method. As it is clear from the name, the method checks whether there are products in the cart, namely returns the quantity (integer) the goods in the cart, then a check will be performed in the controller, if this number is greater than 0, then further processing is underway and the data goes to the template file with the table and everything else, and if not more than 0, then a template is displayed, which simply displays a message that there are no goods in the cart, this is not exactly what it is necessary, therefore, that the deleted goods should also be counted, and therefore, in the end, the method looks like this:

```sh
public function hasProducts()
{
	return (int)(count($this->getProducts()) + count($this->getRemovedProducts()));
}
```

## Working with the cart controller 

(the final file in the repository `/ready/CartController.php `)

Let's go to the Controller class, which responds to the cart page in the directory `catalog/controller/checkout/cart'.php`

In the ' index() method`find the string `$products = $this->cart->GetProducts ();`
, the lower view of the array iteration with data processing, after this iteration we will start writing code for processing deleted goods

1. Declare the array that is passed to `$data`

```sh
$data['removed_products'] = array();
```

2. Scroll through the data returned by the `getRemovedProducts()` method in the php Cart class to the new variable

```sh
$removed_products = $this->cart->getRemovedProducts();
```

3. The next step will be processing the data returned by the `getRemovedProducts()` method and in fact this mail code is identical to the brute force code `$products = $this->cart->GetProducts();`, so you can copy it except that `$data['products']] [] = Array(`replace with `$Data['removed_products']) [] = Array(`

4. After editing the `index()' method`let's edit the `remove()` method.

The string `$this->cart->remove($this->request->post['key']);` replace with `$this->cart->remove Product($key);`

Let's remove all unsets and note that the `total` and `success` keys are added to the `$json` object. There are no questions about the `success` key, it passes a line with the text about the notification that the product has been deleted, which it takes from the language file, but the `total` key it doesn't quite fit, or rather its implementation, so we'll change it to the following code, which works more correctly at the moment when all the goods from the cart are deleted and only the deleted goods remain.

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
and for this code to work, you need to connect the cart model, add `$this->load->model('checkout/cart');`next to `$this->load->language('checkout/cart');`

5. Add the `restore()` method to restore the product, the body of the method is identical to the `remove()` method, except that `$this->cart->removeProduct($key);` replace with `$this->cart->restoreProduct($key);`


## Working with the cart view template 
Actually, there's not much interesting here, so just look at the final file in the repository `/ready/cart.twig`. Please note that the table that displays the products that are added to the cart, I assigned `id="products-in-cart"`, and the table that displays deleted products, `id="deleted-products"'.

## ajax requests

(the final file in the repository is '/ready/common.js`)

Opening the file by `catalog/view/javascript/common.js` and we are looking for a functional that is responsible for working with the cart `var cart = {`. This object stores the function values in the keys.

1. Above this code, we will add a function that will send a request to the controller in the `index` method and updates the data of tables and mini-cart.

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


2. Consider `remove`, which can be found by 'remove': function(key) {`: 
We see a standard ajax request that sends a request to `index.php?route=checkout/cart/remove`, where the `remove` method is in the controller that we edited above. Let's pay attention to the `success`, there is an update of information about the quantity of goods and the price in the mini-cart in the header of the site, then there is a check on which page the user is on, according to the results of which he throws to the cart page, this does not quite suit us. Let's redo the function in this way:

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

3. Next, we will create a new function in the `cart` object using the `restore` key, which, in fact, is almost identical to the `remove` function, except that the request is sent to `index.php?route=checkout/cart/remove`

## The result
Now, for reliability, I advise you to update the modifier cache and make sure that the current js is applied