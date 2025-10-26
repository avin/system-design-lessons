# Урок 51: E-commerce Platform

## Введение

E-commerce Platform — это комплексная система для онлайн-торговли, включающая каталог товаров, корзину, оформление заказа, платежи, управление запасами и доставку. Системы типа Amazon, eBay, Shopify обрабатывают миллионы транзакций ежедневно с высокими требованиями к доступности и консистентности.

В этом уроке мы спроектируем масштабируемую e-commerce платформу, способную обслуживать миллионы пользователей и товаров.

**Компоненты:**
- Product catalog
- Shopping cart
- Order management
- Inventory management
- Payment processing
- Shipping & tracking
- Search & recommendations

## Требования к системе

### Функциональные требования

1. **Product Catalog**:
   - Просмотр товаров с фильтрами
   - Поиск
   - Детальная страница товара

2. **Shopping Cart**:
   - Добавить/удалить товары
   - Обновить количество
   - Сохранение корзины

3. **Checkout**:
   - Оформление заказа
   - Выбор адреса доставки
   - Оплата (кредитные карты, PayPal)

4. **Order Management**:
   - Просмотр заказов
   - Отслеживание статуса
   - Отмена/возврат

5. **Inventory**:
   - Управление запасами
   - Резервирование товаров
   - Уведомления о наличии

6. **Seller Portal** (для marketplace):
   - Добавление товаров
   - Управление заказами
   - Analytics

### Нефункциональные требования

1. **Scalability**:
   - 100M users
   - 10M products
   - 1M orders/day ≈ 12 orders/sec

2. **Performance**:
   - Product page load: < 1 sec
   - Checkout: < 2 sec
   - Search: < 500ms

3. **Availability**: 99.99%

4. **Consistency**:
   - Strong consistency для inventory
   - Eventual consistency для product catalog

5. **Security**:
   - PCI DSS compliance для платежей
   - Защита от fraud

### Back-of-the-envelope

```
Products: 10M
Users: 100M (10M DAU)
Orders: 1M/day ≈ 12 orders/sec

Peak traffic (Black Friday):
- 10x normal load = 120 orders/sec
- 1000x page views/sec

Storage:
- Product data: 10M × 10KB = 100 GB
- Images: 10M × 5 images × 500KB = 25 TB
- Order data: 1M orders/day × 5KB = 5 GB/day × 365 = 1.8 TB/year

Inventory updates: 1M orders × 3 items/order = 3M updates/day
```

## Архитектура высокого уровня

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ↓
┌─────────────────┐
│      CDN        │
│  (Static Assets)│
└────┬────────────┘
     │
     ↓
┌─────────────────┐
│  API Gateway    │
└────┬────────────┘
     │
     ├──────┬──────┬──────┬──────┬──────┐
     ↓      ↓      ↓      ↓      ↓      ↓
┌────────┐┌────┐┌─────┐┌──────┐┌────┐┌────┐
│Product ││Cart││Order││Invent││Pay-││Ship│
│Service ││Svc ││ Svc ││ory   ││ment││ping│
└───┬────┘└─┬──┘└──┬──┘└───┬──┘└─┬──┘└─┬──┘
    │       │      │       │     │     │
    └───────┴──────┴───────┴─────┴─────┘
                   │
    ┌──────────────┼──────────────┐
    ↓              ↓              ↓
┌─────────┐  ┌──────────┐  ┌───────────┐
│Products │  │  Orders  │  │ Inventory │
│Database │  │ Database │  │  Database │
└─────────┘  └──────────┘  └───────────┘
```

## Ключевые компоненты

### 1. Product Service

```javascript
const express = require('express');

class ProductService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.elasticsearch = require('./elasticsearch');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Get product by ID
    app.get('/products/:productId', async (req, res) => {
      try {
        const { productId } = req.params;

        const product = await this.getProduct(productId);

        res.json(product);
      } catch (error) {
        res.status(404).json({ error: 'Product not found' });
      }
    });

    // Search products
    app.get('/products/search', async (req, res) => {
      try {
        const { query, category, minPrice, maxPrice, page = 1, limit = 20 } = req.query;

        const results = await this.searchProducts({
          query,
          category,
          minPrice: parseFloat(minPrice),
          maxPrice: parseFloat(maxPrice),
          page: parseInt(page),
          limit: parseInt(limit)
        });

        res.json(results);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get products by category
    app.get('/products/category/:categoryId', async (req, res) => {
      try {
        const { categoryId } = req.params;
        const { page = 1, limit = 20, sort = 'popular' } = req.query;

        const products = await this.getProductsByCategory(
          categoryId,
          parseInt(page),
          parseInt(limit),
          sort
        );

        res.json(products);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get product recommendations
    app.get('/products/:productId/recommendations', async (req, res) => {
      try {
        const { productId } = req.params;

        const recommendations = await this.getRecommendations(productId);

        res.json({ recommendations });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3001, () => {
      console.log('Product Service listening on port 3001');
    });
  }

  async getProduct(productId) {
    // Проверяем cache
    const cacheKey = `product:${productId}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Загружаем из БД
    const result = await this.db.query(
      `SELECT p.*, c.name as category_name, s.name as seller_name
       FROM products p
       JOIN categories c ON p.category_id = c.id
       LEFT JOIN sellers s ON p.seller_id = s.id
       WHERE p.id = $1 AND p.status = 'active'`,
      [productId]
    );

    if (result.rows.length === 0) {
      throw new Error('Product not found');
    }

    const product = this.formatProduct(result.rows[0]);

    // Загружаем варианты (размеры, цвета)
    product.variants = await this.getProductVariants(productId);

    // Загружаем изображения
    product.images = await this.getProductImages(productId);

    // Загружаем reviews
    product.reviews = await this.getProductReviews(productId, 5);

    // Кэшируем (1 hour)
    await this.redis.setex(cacheKey, 3600, JSON.stringify(product));

    return product;
  }

  async searchProducts({ query, category, minPrice, maxPrice, page, limit }) {
    // Используем Elasticsearch для поиска
    const must = [];

    if (query) {
      must.push({
        multi_match: {
          query,
          fields: ['name^3', 'description', 'tags'],
          fuzziness: 'AUTO'
        }
      });
    }

    const filter = [];

    if (category) {
      filter.push({ term: { category_id: category } });
    }

    if (minPrice || maxPrice) {
      const range = {};
      if (minPrice) range.gte = minPrice;
      if (maxPrice) range.lte = maxPrice;

      filter.push({ range: { price: range } });
    }

    const result = await this.elasticsearch.search({
      index: 'products',
      body: {
        query: {
          bool: { must, filter }
        },
        from: (page - 1) * limit,
        size: limit,
        sort: [
          { _score: 'desc' },
          { sales_count: 'desc' }
        ]
      }
    });

    return {
      products: result.hits.hits.map(hit => hit._source),
      total: result.hits.total.value,
      page,
      limit
    };
  }

  async getProductsByCategory(categoryId, page, limit, sort) {
    let orderBy = 'p.created_at DESC';

    switch (sort) {
      case 'price_asc':
        orderBy = 'p.price ASC';
        break;
      case 'price_desc':
        orderBy = 'p.price DESC';
        break;
      case 'popular':
        orderBy = 'p.sales_count DESC';
        break;
      case 'rating':
        orderBy = 'p.rating DESC';
        break;
    }

    const result = await this.db.query(
      `SELECT p.*
       FROM products p
       WHERE p.category_id = $1 AND p.status = 'active'
       ORDER BY ${orderBy}
       LIMIT $2 OFFSET $3`,
      [categoryId, limit, (page - 1) * limit]
    );

    return {
      products: result.rows.map(p => this.formatProduct(p)),
      page,
      limit
    };
  }

  async getRecommendations(productId) {
    // Стратегия: "Frequently bought together" + "Similar products"
    const product = await this.getProduct(productId);

    // 1. Frequently bought together
    const frequentlyBought = await this.db.query(
      `SELECT p2.id, p2.name, p2.price, COUNT(*) as frequency
       FROM order_items oi1
       JOIN order_items oi2 ON oi1.order_id = oi2.order_id
       JOIN products p2 ON oi2.product_id = p2.id
       WHERE oi1.product_id = $1 AND oi2.product_id != $1
       GROUP BY p2.id, p2.name, p2.price
       ORDER BY frequency DESC
       LIMIT 5`,
      [productId]
    );

    // 2. Similar products (same category, similar price)
    const similar = await this.db.query(
      `SELECT *
       FROM products
       WHERE category_id = $1
         AND id != $2
         AND price BETWEEN $3 * 0.8 AND $3 * 1.2
         AND status = 'active'
       ORDER BY sales_count DESC
       LIMIT 5`,
      [product.category_id, productId, product.price]
    );

    return {
      frequentlyBoughtTogether: frequentlyBought.rows,
      similar: similar.rows.map(p => this.formatProduct(p))
    };
  }

  async getProductVariants(productId) {
    const result = await this.db.query(
      'SELECT * FROM product_variants WHERE product_id = $1',
      [productId]
    );

    return result.rows;
  }

  async getProductImages(productId) {
    const result = await this.db.query(
      'SELECT url, is_primary FROM product_images WHERE product_id = $1 ORDER BY is_primary DESC, id ASC',
      [productId]
    );

    return result.rows.map(r => r.url);
  }

  async getProductReviews(productId, limit) {
    const result = await this.db.query(
      `SELECT r.*, u.username
       FROM reviews r
       JOIN users u ON r.user_id = u.id
       WHERE r.product_id = $1
       ORDER BY r.created_at DESC
       LIMIT $2`,
      [productId, limit]
    );

    return result.rows;
  }

  formatProduct(row) {
    return {
      id: row.id,
      name: row.name,
      description: row.description,
      price: parseFloat(row.price),
      discountPrice: row.discount_price ? parseFloat(row.discount_price) : null,
      categoryId: row.category_id,
      categoryName: row.category_name,
      sellerId: row.seller_id,
      sellerName: row.seller_name,
      rating: parseFloat(row.rating || 0),
      reviewsCount: parseInt(row.reviews_count || 0),
      salesCount: parseInt(row.sales_count || 0),
      inStock: row.stock_quantity > 0,
      stockQuantity: parseInt(row.stock_quantity)
    };
  }
}

const service = new ProductService();
```

### 2. Shopping Cart Service

```javascript
class CartService {
  constructor() {
    this.redis = require('./redis');
    this.db = require('./database');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Get cart
    app.get('/cart/:userId', async (req, res) => {
      try {
        const { userId } = req.params;

        const cart = await this.getCart(userId);

        res.json(cart);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Add to cart
    app.post('/cart/:userId/items', async (req, res) => {
      try {
        const { userId } = req.params;
        const { productId, variantId, quantity } = req.body;

        await this.addToCart(userId, productId, variantId, quantity);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Update cart item
    app.put('/cart/:userId/items/:productId', async (req, res) => {
      try {
        const { userId, productId } = req.params;
        const { quantity } = req.body;

        await this.updateCartItem(userId, productId, quantity);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Remove from cart
    app.delete('/cart/:userId/items/:productId', async (req, res) => {
      try {
        const { userId, productId } = req.params;

        await this.removeFromCart(userId, productId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Clear cart
    app.delete('/cart/:userId', async (req, res) => {
      try {
        const { userId } = req.params;

        await this.clearCart(userId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3002, () => {
      console.log('Cart Service listening on port 3002');
    });
  }

  async getCart(userId) {
    // Корзина хранится в Redis (быстрый доступ)
    const cartKey = `cart:${userId}`;
    const items = await this.redis.hgetall(cartKey);

    if (Object.keys(items).length === 0) {
      return { items: [], total: 0 };
    }

    // Загружаем детали продуктов
    const productIds = Object.keys(items);
    const products = await this.loadProducts(productIds);

    const cartItems = [];
    let total = 0;

    for (const productId of productIds) {
      const item = JSON.parse(items[productId]);
      const product = products.get(productId);

      if (!product) continue;

      const itemTotal = product.price * item.quantity;

      cartItems.push({
        productId,
        product,
        variantId: item.variantId,
        quantity: item.quantity,
        price: product.price,
        subtotal: itemTotal
      });

      total += itemTotal;
    }

    return {
      items: cartItems,
      total: total.toFixed(2),
      itemsCount: cartItems.reduce((sum, item) => sum + item.quantity, 0)
    };
  }

  async addToCart(userId, productId, variantId, quantity) {
    // Проверяем наличие товара
    const available = await this.checkInventory(productId, variantId, quantity);

    if (!available) {
      throw new Error('Product out of stock');
    }

    const cartKey = `cart:${userId}`;

    // Проверяем: товар уже в корзине?
    const existing = await this.redis.hget(cartKey, productId);

    if (existing) {
      const item = JSON.parse(existing);
      item.quantity += quantity;

      await this.redis.hset(cartKey, productId, JSON.stringify(item));
    } else {
      await this.redis.hset(cartKey, productId, JSON.stringify({
        variantId,
        quantity,
        addedAt: Date.now()
      }));
    }

    // TTL: 30 дней
    await this.redis.expire(cartKey, 30 * 24 * 3600);
  }

  async updateCartItem(userId, productId, quantity) {
    if (quantity <= 0) {
      return await this.removeFromCart(userId, productId);
    }

    // Проверяем наличие
    const available = await this.checkInventory(productId, null, quantity);

    if (!available) {
      throw new Error('Insufficient stock');
    }

    const cartKey = `cart:${userId}`;
    const existing = await this.redis.hget(cartKey, productId);

    if (!existing) {
      throw new Error('Item not in cart');
    }

    const item = JSON.parse(existing);
    item.quantity = quantity;

    await this.redis.hset(cartKey, productId, JSON.stringify(item));
  }

  async removeFromCart(userId, productId) {
    const cartKey = `cart:${userId}`;
    await this.redis.hdel(cartKey, productId);
  }

  async clearCart(userId) {
    const cartKey = `cart:${userId}`;
    await this.redis.del(cartKey);
  }

  async checkInventory(productId, variantId, quantity) {
    const result = await this.db.query(
      'SELECT stock_quantity FROM products WHERE id = $1',
      [productId]
    );

    if (result.rows.length === 0) return false;

    const stockQuantity = result.rows[0].stock_quantity;

    return stockQuantity >= quantity;
  }

  async loadProducts(productIds) {
    const result = await this.db.query(
      'SELECT * FROM products WHERE id = ANY($1)',
      [productIds]
    );

    const map = new Map();
    result.rows.forEach(product => {
      map.set(product.id, product);
    });

    return map;
  }
}

const service = new CartService();
```

### 3. Order Service

```javascript
class OrderService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
    this.kafka = require('./kafka');
    this.setupRoutes();
  }

  setupRoutes() {
    const app = express();
    app.use(express.json());

    // Create order (checkout)
    app.post('/orders', async (req, res) => {
      try {
        const {
          userId,
          items,
          shippingAddress,
          paymentMethod,
          promoCode
        } = req.body;

        const order = await this.createOrder({
          userId,
          items,
          shippingAddress,
          paymentMethod,
          promoCode
        });

        res.json(order);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Get order
    app.get('/orders/:orderId', async (req, res) => {
      try {
        const { orderId } = req.params;

        const order = await this.getOrder(orderId);

        res.json(order);
      } catch (error) {
        res.status(404).json({ error: 'Order not found' });
      }
    });

    // Get user orders
    app.get('/users/:userId/orders', async (req, res) => {
      try {
        const { userId } = req.params;
        const { page = 1, limit = 20 } = req.query;

        const orders = await this.getUserOrders(userId, page, limit);

        res.json(orders);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    // Cancel order
    app.post('/orders/:orderId/cancel', async (req, res) => {
      try {
        const { orderId } = req.params;
        const { userId } = req.body;

        await this.cancelOrder(orderId, userId);

        res.json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });

    app.listen(3003, () => {
      console.log('Order Service listening on port 3003');
    });
  }

  async createOrder({
    userId,
    items,
    shippingAddress,
    paymentMethod,
    promoCode
  }) {
    const orderId = this.generateOrderId();

    await this.db.query('BEGIN');

    try {
      // 1. Валидация и резервирование inventory
      const reservations = await this.reserveInventory(items);

      // 2. Вычисляем стоимость
      const pricing = await this.calculatePricing(items, promoCode);

      // 3. Создаём заказ
      await this.db.query(
        `INSERT INTO orders
         (id, user_id, subtotal, tax, shipping_cost, discount, total,
          shipping_address, payment_method, status, created_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, 'pending', NOW())`,
        [
          orderId,
          userId,
          pricing.subtotal,
          pricing.tax,
          pricing.shippingCost,
          pricing.discount,
          pricing.total,
          JSON.stringify(shippingAddress),
          paymentMethod
        ]
      );

      // 4. Создаём order items
      for (const item of items) {
        await this.db.query(
          `INSERT INTO order_items
           (order_id, product_id, variant_id, quantity, price, subtotal)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [
            orderId,
            item.productId,
            item.variantId,
            item.quantity,
            item.price,
            item.price * item.quantity
          ]
        );
      }

      await this.db.query('COMMIT');

      // 5. Обрабатываем платёж (асинхронно)
      await this.processPayment(orderId, pricing.total, paymentMethod);

      // 6. Очищаем корзину
      await this.clearCart(userId);

      // 7. Отправляем уведомление
      await this.notifyOrderCreated(orderId, userId);

      return {
        orderId,
        status: 'pending',
        total: pricing.total,
        estimatedDelivery: this.calculateDeliveryDate(shippingAddress)
      };
    } catch (error) {
      await this.db.query('ROLLBACK');
      throw error;
    }
  }

  async reserveInventory(items) {
    // Резервируем товары (пессимистичная блокировка)
    const reservations = [];

    for (const item of items) {
      const result = await this.db.query(
        `UPDATE products
         SET stock_quantity = stock_quantity - $1
         WHERE id = $2 AND stock_quantity >= $1
         RETURNING stock_quantity`,
        [item.quantity, item.productId]
      );

      if (result.rows.length === 0) {
        throw new Error(`Product ${item.productId} out of stock`);
      }

      reservations.push({
        productId: item.productId,
        quantity: item.quantity
      });
    }

    return reservations;
  }

  async calculatePricing(items, promoCode) {
    let subtotal = 0;

    for (const item of items) {
      subtotal += item.price * item.quantity;
    }

    // Скидка от promo code
    let discount = 0;

    if (promoCode) {
      const promo = await this.validatePromoCode(promoCode);

      if (promo) {
        discount = subtotal * (promo.discountPercent / 100);
      }
    }

    // Налог (примерно 10%)
    const tax = (subtotal - discount) * 0.1;

    // Доставка
    const shippingCost = subtotal > 50 ? 0 : 5.99;

    const total = subtotal - discount + tax + shippingCost;

    return {
      subtotal: subtotal.toFixed(2),
      discount: discount.toFixed(2),
      tax: tax.toFixed(2),
      shippingCost: shippingCost.toFixed(2),
      total: total.toFixed(2)
    };
  }

  async processPayment(orderId, amount, paymentMethod) {
    // Публикуем в Kafka для async обработки
    await this.kafka.send({
      topic: 'payments',
      messages: [{
        key: orderId,
        value: JSON.stringify({
          orderId,
          amount,
          paymentMethod,
          timestamp: Date.now()
        })
      }]
    });
  }

  async getOrder(orderId) {
    const result = await this.db.query(
      `SELECT o.*,
              json_agg(json_build_object(
                'productId', oi.product_id,
                'productName', p.name,
                'quantity', oi.quantity,
                'price', oi.price,
                'subtotal', oi.subtotal
              )) as items
       FROM orders o
       JOIN order_items oi ON o.id = oi.order_id
       JOIN products p ON oi.product_id = p.id
       WHERE o.id = $1
       GROUP BY o.id`,
      [orderId]
    );

    if (result.rows.length === 0) {
      throw new Error('Order not found');
    }

    return result.rows[0];
  }

  async getUserOrders(userId, page, limit) {
    const result = await this.db.query(
      `SELECT id, total, status, created_at
       FROM orders
       WHERE user_id = $1
       ORDER BY created_at DESC
       LIMIT $2 OFFSET $3`,
      [userId, limit, (page - 1) * limit]
    );

    return {
      orders: result.rows,
      page,
      limit
    };
  }

  async cancelOrder(orderId, userId) {
    // Проверяем: можно ли отменить
    const order = await this.getOrder(orderId);

    if (order.user_id !== userId) {
      throw new Error('Unauthorized');
    }

    if (!['pending', 'confirmed'].includes(order.status)) {
      throw new Error('Cannot cancel order in current status');
    }

    await this.db.query('BEGIN');

    try {
      // Обновляем статус
      await this.db.query(
        'UPDATE orders SET status = $1 WHERE id = $2',
        ['cancelled', orderId]
      );

      // Возвращаем товары в inventory
      await this.db.query(
        `UPDATE products p
         SET stock_quantity = stock_quantity + oi.quantity
         FROM order_items oi
         WHERE oi.order_id = $1 AND p.id = oi.product_id`,
        [orderId]
      );

      await this.db.query('COMMIT');

      // Возврат платежа (если был)
      await this.refundPayment(orderId);
    } catch (error) {
      await this.db.query('ROLLBACK');
      throw error;
    }
  }

  async notifyOrderCreated(orderId, userId) {
    await this.kafka.send({
      topic: 'notifications',
      messages: [{
        value: JSON.stringify({
          type: 'order_created',
          userId,
          orderId,
          timestamp: Date.now()
        })
      }]
    });
  }

  generateOrderId() {
    const Snowflake = require('./snowflake');
    return new Snowflake(1, 1).generate().toString();
  }
}

const service = new OrderService();
```

### 4. Inventory Service

```javascript
class InventoryService {
  constructor() {
    this.db = require('./database');
    this.redis = require('./redis');
  }

  async checkAvailability(productId, quantity) {
    // Проверяем в Redis (cache)
    const cacheKey = `inventory:${productId}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      const stock = parseInt(cached);
      return stock >= quantity;
    }

    // Загружаем из БД
    const result = await this.db.query(
      'SELECT stock_quantity FROM products WHERE id = $1',
      [productId]
    );

    if (result.rows.length === 0) return false;

    const stock = result.rows[0].stock_quantity;

    // Кэшируем (TTL 60 sec)
    await this.redis.setex(cacheKey, 60, stock.toString());

    return stock >= quantity;
  }

  async reserveStock(productId, quantity) {
    // Атомарная операция через database transaction
    const result = await this.db.query(
      `UPDATE products
       SET stock_quantity = stock_quantity - $1,
           reserved_quantity = reserved_quantity + $1
       WHERE id = $2 AND stock_quantity >= $1
       RETURNING stock_quantity`,
      [quantity, productId]
    );

    if (result.rows.length === 0) {
      throw new Error('Insufficient stock');
    }

    // Invalidate cache
    await this.redis.del(`inventory:${productId}`);

    return true;
  }

  async releaseReservation(productId, quantity) {
    await this.db.query(
      `UPDATE products
       SET reserved_quantity = reserved_quantity - $1
       WHERE id = $2`,
      [quantity, productId]
    );

    await this.redis.del(`inventory:${productId}`);
  }

  async confirmReservation(productId, quantity) {
    // Подтверждаем резервацию (при успешной оплате)
    await this.db.query(
      `UPDATE products
       SET reserved_quantity = reserved_quantity - $1,
           sales_count = sales_count + $1
       WHERE id = $2`,
      [quantity, productId]
    );

    await this.redis.del(`inventory:${productId}`);
  }

  async notifyLowStock(productId, threshold = 10) {
    const result = await this.db.query(
      'SELECT stock_quantity FROM products WHERE id = $1',
      [productId]
    );

    if (result.rows[0].stock_quantity < threshold) {
      // Отправляем notification
      console.log(`Low stock alert for product ${productId}`);
    }
  }
}

module.exports = InventoryService;
```

## Оптимизации

### 1. Database Sharding

```javascript
class ShardedOrderDatabase {
  constructor() {
    this.shards = [
      { id: 0, pool: new Pool({ host: 'db-shard-0' }) },
      { id: 1, pool: new Pool({ host: 'db-shard-1' }) },
      { id: 2, pool: new Pool({ host: 'db-shard-2' }) },
      { id: 3, pool: new Pool({ host: 'db-shard-3' }) }
    ];
  }

  getShard(orderId) {
    // Шардирование по order ID
    const hash = crypto.createHash('md5').update(orderId).digest();
    const shardId = hash.readUInt32BE(0) % this.shards.length;

    return this.shards[shardId].pool;
  }

  async query(orderId, sql, params) {
    const shard = this.getShard(orderId);
    return await shard.query(sql, params);
  }
}
```

### 2. Product Cache Warming

```javascript
class CacheWarmer {
  async warmPopularProducts() {
    // Загружаем топ-1000 популярных товаров в cache
    const result = await db.query(
      `SELECT id, name, price, description
       FROM products
       WHERE status = 'active'
       ORDER BY sales_count DESC, views_count DESC
       LIMIT 1000`
    );

    const pipeline = redis.pipeline();

    for (const product of result.rows) {
      const cacheKey = `product:${product.id}`;
      pipeline.setex(cacheKey, 3600, JSON.stringify(product));
    }

    await pipeline.exec();

    console.log('Cache warmed with 1000 popular products');
  }
}
```

### 3. Flash Sale Handling

```javascript
class FlashSaleService {
  async createFlashSale(productId, quantity, price, startTime, endTime) {
    const saleId = this.generateSaleId();

    // Используем Redis для inventory (atomic operations)
    const saleKey = `flash_sale:${saleId}`;

    await redis.hset(saleKey, {
      productId,
      quantity,
      price,
      sold: 0,
      startTime,
      endTime
    });

    return saleId;
  }

  async purchaseFlashSale(saleId, userId) {
    // Lua script для атомарности
    const luaScript = `
      local saleKey = KEYS[1]
      local quantity = tonumber(redis.call("HGET", saleKey, "quantity"))
      local sold = tonumber(redis.call("HGET", saleKey, "sold"))

      if sold >= quantity then
        return 0  -- sold out
      end

      redis.call("HINCRBY", saleKey, "sold", 1)
      return 1  -- success
    `;

    const result = await redis.eval(luaScript, 1, `flash_sale:${saleId}`);

    if (result === 0) {
      throw new Error('Flash sale sold out');
    }

    // Создаём заказ...
  }
}
```

## Мониторинг

```javascript
const prometheus = require('prom-client');

const ordersTotal = new prometheus.Counter({
  name: 'ecommerce_orders_total',
  help: 'Total orders created',
  labelNames: ['status']
});

const cartAdditions = new prometheus.Counter({
  name: 'ecommerce_cart_additions_total',
  help: 'Total items added to cart'
});

const checkoutDuration = new prometheus.Histogram({
  name: 'ecommerce_checkout_duration_seconds',
  help: 'Time to complete checkout',
  buckets: [1, 2, 5, 10, 30]
});

const inventoryLevel = new prometheus.Gauge({
  name: 'ecommerce_inventory_level',
  help: 'Current inventory levels',
  labelNames: ['product_id']
});
```

## Schema Design

```sql
-- Products
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  seller_id BIGINT,
  category_id BIGINT,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL,
  discount_price DECIMAL(10, 2),
  stock_quantity INT DEFAULT 0,
  reserved_quantity INT DEFAULT 0,
  sales_count INT DEFAULT 0,
  views_count INT DEFAULT 0,
  rating DECIMAL(3, 2),
  reviews_count INT DEFAULT 0,
  status VARCHAR(20) DEFAULT 'active',
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_category (category_id, status),
  INDEX idx_seller (seller_id),
  INDEX idx_search (name, status)
);

-- Orders
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  subtotal DECIMAL(10, 2),
  tax DECIMAL(10, 2),
  shipping_cost DECIMAL(10, 2),
  discount DECIMAL(10, 2),
  total DECIMAL(10, 2),
  shipping_address JSONB,
  payment_method VARCHAR(50),
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_user_date (user_id, created_at DESC),
  INDEX idx_status (status)
);

-- Order Items
CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  variant_id BIGINT,
  quantity INT,
  price DECIMAL(10, 2),
  subtotal DECIMAL(10, 2),
  INDEX idx_order (order_id),
  INDEX idx_product (product_id)
);
```

## Что читать дальше

- **Урок 48**: Ticketing System — обработка high concurrency
- **Урок 52**: Recommendation System — product recommendations
- **Урок 41**: Notification System — order notifications

## Проверь себя

1. Как предотвратить overselling при flash sale?
2. Спроектируйте систему для 100K orders/sec (Black Friday).
3. Реализуйте distributed lock для inventory management.
4. Как обрабатывать abandoned carts (reminder emails)?
5. Спроектируйте систему возвратов (returns & refunds).

---

[← Урок 50: Ride-sharing System](50-ride-sharing.md) | [Урок 52: Recommendation System →](52-recommendation-system.md)
