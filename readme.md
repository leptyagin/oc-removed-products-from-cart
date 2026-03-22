# OpenCart 3: Soft Delete for Cart Items

## Overview

This repository provides a custom solution for implementing a **"soft delete"** feature for products in the cart page in OpenCart 3.

### What problem does it solve?

By default, OpenCart removes products from the cart permanently when a user deletes them. This solution enhances the user experience by introducing a **temporary removal mechanism**:

- When a user removes a product from the cart, it is **not deleted permanently**
- Instead, it is moved to a separate "Removed Items" section below the cart
- Each removed item includes a **"Restore" button** to add it back to the cart
- All interactions work via **AJAX**, without page reloads

This behavior is similar to a "soft delete" pattern and is commonly used in modern e-commerce UX.

---

## Features

- Soft delete functionality for cart items
- Restore removed products with one click
- AJAX-based interactions (no page reloads)
- Clean separation of active and removed items
- Easy to integrate into existing OpenCart 3 projects

---

## Implementation

The step-by-step implementation is described in: [solution.md](https://github.com/leptyagin/oc-removed-products-from-cart/blob/main/solution.md)

In this file, you will find a detailed explanation of:
- What changes are required
- Where to apply them
- Why each step is necessary

---

## Notes

- This solution is designed specifically for **OpenCart 3**
- It does not rely on third-party extensions
- Can be later packaged as an **OCMOD modification** for easier reuse

---

## Feedback & Contact

If you have suggestions, improvements, or questions — feel free to reach out:

- LinkedIn: https://www.linkedin.com/in/leptyagin/
- Telegram: https://t.me/leptyagin

---

## License

This project is licensed under the MIT License.