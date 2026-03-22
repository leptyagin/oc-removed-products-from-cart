# Soft Delete for Cart Items (OpenCart 3)

> ⚠️ This guide is written for **OpenCart 3.0.3.x**.  
> Always **backup your files and database** before making any changes.

---

## Idea

By default, OpenCart removes products from the cart by **deleting rows** from the `cart` table.

This solution introduces a **soft delete mechanism**:

- Instead of deleting a row, we mark it as deleted using a flag
- Deleted items are hidden from the main cart
- They are displayed separately as "Removed Items"
- Users can restore them back to the cart

This approach is similar to the **Soft Deletes pattern** used in frameworks like Laravel.

---

## ⚠️ Important Considerations (Fixes & Improvements)

Before implementation, note the following corrections and safety improvements:

- ❗ **Security issue (fixed below):**  
  Original queries updated rows by `cart_id` only — this is unsafe.  
  We must also check:
  - `session_id`
  - `customer_id`
  - `api_id`

- ❗ **Compatibility issue:**  
  `hasProducts()` should **NOT include deleted items**, otherwise checkout logic may break

- ❗ **Database best practice:**  
  Use `NOT NULL DEFAULT 0` instead of `NULL`

- ❗ **Avoid modifying core directly:**  
  Prefer **OCMOD/VQMOD** for production use

---

## 1. Database

Add a new column to the `cart` table:

```sql
ALTER TABLE `oc_cart`
ADD COLUMN `is_deleted` TINYINT(1) NOT NULL DEFAULT 0;
```

## 2. Modifying Cart Library

File: `system/library/cart/cart.php`

### 2.1 Filter Active Products

Find method:

```php
public function getProducts()
```

Update query:

```php
$cart_query = $this->db->query("
	SELECT * FROM " . DB_PREFIX . "cart
	WHERE is_deleted = 0
	AND api_id = '" . (int)($this->session->data['api_id'] ?? 0) . "'
	AND customer_id = '" . (int)$this->customer->getId() . "'
	AND session_id = '" . $this->db->escape($this->session->getId()) . "'
");
```

### 2.2 Add Soft Remove Method (Safe)

```php
public function removeProduct($cart_id)
{
	$this->db->query("
		UPDATE `" . DB_PREFIX . "cart`
		SET is_deleted = 1
		WHERE cart_id = '" . (int)$cart_id . "'
		AND api_id = '" . (int)($this->session->data['api_id'] ?? 0) . "'
		AND session_id = '" . $this->db->escape($this->session->getId()) . "'
		AND customer_id = '" . (int)$this->customer->getId() . "'
	");
}
```

### 2.3 Add Restore Method (Safe)

```php
public function restoreProduct($cart_id)
{
	$this->db->query("
		UPDATE `" . DB_PREFIX . "cart`
		SET is_deleted = 0
		WHERE cart_id = '" . (int)$cart_id . "'
		AND api_id = '" . (int)($this->session->data['api_id'] ?? 0) . "'
		AND session_id = '" . $this->db->escape($this->session->getId()) . "'
		AND customer_id = '" . (int)$this->customer->getId() . "'
	");
}
```

### 2.4 Add Method for Removed Products

Duplicate getProducts() logic and change condition:

```php
public function getRemovedProducts()
{
	$cart_query = $this->db->query("
		SELECT * FROM " . DB_PREFIX . "cart
		WHERE is_deleted = 1
		AND api_id = '" . (int)($this->session->data['api_id'] ?? 0) . "'
		AND customer_id = '" . (int)$this->customer->getId() . "'
		AND session_id = '" . $this->db->escape($this->session->getId()) . "'
	");

	// Copy full processing logic from getProducts()
}
```

## 3. Cart Controller

File: `catalog/controller/checkout/cart.php`

### 3.1 Load Removed Products

Inside `index()`:

```php
$data['removed_products'] = [];

$removed_products = $this->cart->getRemovedProducts();
```

Process them the same way as `$products`, but store in:

```php
$data['removed_products'][] = ...
```

### 3.2 Modify `remove()` Method

Replace:

```php
$this->cart->remove($this->request->post['key']);
```

With:

```php
$key = (int)$this->request->post['key'];
$this->cart->removeProduct($key);
```

### 3.3 Fix JSON Total Calculation

```php
$json['total'] = sprintf(
	$this->language->get('text_items'),
	$this->cart->countProducts() + (isset($this->session->data['vouchers']) ? count($this->session->data['vouchers']) : 0),
	$this->currency->format(
		$this->cart->getSubTotal(),
		$this->session->data['currency']
	)
);
```

### 3.4 Add Restore Method

```php
public function restore()
{
	$this->load->language('checkout/cart');

	$json = [];

	if (isset($this->request->post['key'])) {
		$key = (int)$this->request->post['key'];
		$this->cart->restoreProduct($key);

		$json['success'] = $this->language->get('text_success');

		$json['total'] = sprintf(
			$this->language->get('text_items'),
			$this->cart->countProducts(),
			$this->currency->format($this->cart->getSubTotal(), $this->session->data['currency'])
		);
	}

	$this->response->addHeader('Content-Type: application/json');
	$this->response->setOutput(json_encode($json));
}
```

## 4. View (Twig)

File: `catalog/view/theme/.../template/checkout/cart.twig`

Add two blocks:

- Active products

```twig
<div id="products-in-cart">
```

- Removed products

```twig
<div id="deleted-products">
```

Render `$removed_products` separately.

## 5. AJAX (JavaScript)

File: `catalog/view/javascript/common.js`

### 5.1 Update Cart Function

```javascript
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

### 5.2 Modify remove()

Replace success handler:

```javascript
success: function(json) {
	$('.alert-dismissible, .text-danger').remove();

	if (json['success']) {
		updateCart();

		setTimeout(function () {
			$('#cart-total').html(json['total']);
		}, 100);
	}
}
```

### 5.3 Add `restore()`

```javascript
restore: function(key) {
	$.ajax({
		url: 'index.php?route=checkout/cart/restore',
		type: 'post',
		data: 'key=' + key,
		dataType: 'json',
		success: function(json) {
			if (json['success']) {
				updateCart();

				setTimeout(function () {
					$('#cart-total').html(json['total']);
				}, 100);
			}
		}
	});
}
```

## Result

Now you have:

- Soft delete for cart items
- Separate "Removed Items" block
- Restore functionality
- AJAX updates without reload

## Final Notes

- Prefer packaging this as OCMOD
- Test with:
  - guest users
  - logged users
  - API carts
- Watch for:
  - session conflicts
  - totals calculation
  - third-party extensions compatibility
