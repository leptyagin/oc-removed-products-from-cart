# Opencart 3: Список удаленных товаров на странице товара / List of deleted products on cart page

### English version down below

### Цель
Задача от клиента звучит так: 
`Нужно на странице корзины вывести товары, которые удалили из корзины. То есть на странице корзины есть список добавленных в нее позиций. Если какой-то товар удаляем из списка, то товар переходит чуть ниже в список с кнопкой вернуть в корзину. Блоки должны работать ajax, без перезагрузки страницы.`  

Достаточно интересная задача в контексте опенкарт, в дефолтной версии такого функционала нет и на просторах интернета я не нашел какого-то решения, поэтому хочется поделиться своим. 

Оформил решение задачи в отдельный файл `solution-russian.md`, в котором постараюсь поэтапно с подробностями распишу какой код, куда и зачем, возможно кому-то пригодиться. Чуть позже планирую оформить в ocmod модификатор.

Буду рад любой обратной связи. Мой [телеграм](https://t.me/leptyagin)

## English version

### Description

The task:
`It's necessary to display the products that were deleted on the cart page. That is, on the cart page there is a list of items added to it. If user delete a certain item from the list, then the product goes to the list below with the return to cart button. The blocks should work ajax, without reloading.`

Quite an interesting task in the context of opencart, there is no such functionality in the default version and I have not found any solution on the Internet, so I want to share my own solution.

I've designed the solution to the problem in a separate file `solution-english.md`, in which I'll try to write in stages with details what code, where and why, it may be useful to someone. Later wanna make ocmod modification.

I'll be glad of any feedback. You can contact me in [telegram](https://t.me/leptyagin)
