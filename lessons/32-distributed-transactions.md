# Урок 32: Distributed Transactions — 2PC, Saga


## Введение

Distributed Transaction — это транзакция, которая затрагивает несколько независимых систем (баз данных, сервисов), но должна выполниться атомарно: либо все операции успешны, либо все откатываются.

**Проблема:**

```javascript
// Перевод денег между банками
async function transferMoney(fromBank, toBank, amount) {
  await fromBank.debit(amount);   // Списать
  await toBank.credit(amount);    // Зачислить

  // Что если toBank.credit упал?
  // Деньги списаны, но не зачислены! ❌
}
```

**ACID в распределённых системах:**

- **Atomicity**: все или ничего
- **Consistency**: данные остаются консистентными
- **Isolation**: транзакции не влияют друг на друга
- **Durability**: результат сохраняется

В распределённой системе ACID дорого обходится. Часто используют более слабые гарантии.

## Two-Phase Commit (2PC)

2PC — классический протокол для distributed transactions с ACID гарантиями.

### Архитектура

```
Coordinator (Transaction Manager)
     ↓
Participants (Databases, Services)
```

### Две фазы

**Phase 1: Prepare (голосование)**

```
Coordinator → PREPARE → Participant 1
                     → Participant 2
                     → Participant 3

Participant:
  - Выполнить транзакцию локально
  - Записать в WAL (Write-Ahead Log)
  - Заблокировать ресурсы
  - Ответить: YES или NO
```

**Phase 2: Commit или Abort**

```
Если все ответили YES:
  Coordinator → COMMIT → All participants
  Participants commit локально

Если хотя бы один NO:
  Coordinator → ABORT → All participants
  Participants rollback локально
```

### Реализация 2PC

```javascript
class TwoPhaseCommitCoordinator {
  constructor() {
    this.transactions = new Map();
  }

  async beginTransaction(participants) {
    const txId = this.generateTxId();

    this.transactions.set(txId, {
      participants,
      state: 'INIT',
      votes: new Map(),
    });

    return txId;
  }

  async prepare(txId) {
    const tx = this.transactions.get(txId);

    if (!tx) {
      throw new Error('Transaction not found');
    }

    console.log(`[Coordinator] PREPARE phase for ${txId}`);

    tx.state = 'PREPARING';

    // Phase 1: отправить PREPARE всем participants
    const preparePromises = tx.participants.map(async (participant) => {
      try {
        const vote = await participant.prepare(txId);
        tx.votes.set(participant.id, vote);
        return vote;
      } catch (error) {
        console.error(`[Coordinator] Participant ${participant.id} failed:`, error);
        tx.votes.set(participant.id, 'NO');
        return 'NO';
      }
    });

    const votes = await Promise.all(preparePromises);

    // Проверить голоса
    const allYes = votes.every(vote => vote === 'YES');

    if (allYes) {
      return await this.commit(txId);
    } else {
      return await this.abort(txId);
    }
  }

  async commit(txId) {
    const tx = this.transactions.get(txId);

    console.log(`[Coordinator] COMMIT phase for ${txId}`);

    tx.state = 'COMMITTING';

    // Phase 2: отправить COMMIT всем participants
    const commitPromises = tx.participants.map(async (participant) => {
      try {
        await participant.commit(txId);
        console.log(`[Coordinator] Participant ${participant.id} committed`);
      } catch (error) {
        console.error(`[Coordinator] Participant ${participant.id} commit failed:`, error);
        // В 2PC нельзя откатить после COMMIT решения!
        // Нужен recovery механизм
      }
    });

    await Promise.all(commitPromises);

    tx.state = 'COMMITTED';
    this.transactions.delete(txId);

    return 'COMMITTED';
  }

  async abort(txId) {
    const tx = this.transactions.get(txId);

    console.log(`[Coordinator] ABORT phase for ${txId}`);

    tx.state = 'ABORTING';

    // Phase 2: отправить ABORT всем participants
    const abortPromises = tx.participants.map(async (participant) => {
      try {
        await participant.abort(txId);
        console.log(`[Coordinator] Participant ${participant.id} aborted`);
      } catch (error) {
        console.error(`[Coordinator] Participant ${participant.id} abort failed:`, error);
      }
    });

    await Promise.all(abortPromises);

    tx.state = 'ABORTED';
    this.transactions.delete(txId);

    return 'ABORTED';
  }

  generateTxId() {
    return `tx_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Participant
class TwoPhaseCommitParticipant {
  constructor(id, db) {
    this.id = id;
    this.db = db;
    this.preparedTransactions = new Map();
  }

  async prepare(txId) {
    console.log(`[Participant ${this.id}] Preparing ${txId}`);

    try {
      // Начать локальную транзакцию
      await this.db.beginTransaction(txId);

      // Выполнить операции (переданные ранее)
      const operations = this.preparedTransactions.get(txId);

      if (!operations) {
        return 'NO';
      }

      for (const op of operations) {
        await this.db.execute(op);
      }

      // Записать в WAL
      await this.db.writeAheadLog(txId, operations);

      // Заблокировать ресурсы
      await this.db.lockResources(txId);

      console.log(`[Participant ${this.id}] Vote: YES`);
      return 'YES';

    } catch (error) {
      console.error(`[Participant ${this.id}] Prepare failed:`, error);
      await this.db.rollback(txId);
      return 'NO';
    }
  }

  async commit(txId) {
    console.log(`[Participant ${this.id}] Committing ${txId}`);

    await this.db.commit(txId);
    await this.db.unlockResources(txId);

    this.preparedTransactions.delete(txId);
  }

  async abort(txId) {
    console.log(`[Participant ${this.id}] Aborting ${txId}`);

    await this.db.rollback(txId);
    await this.db.unlockResources(txId);

    this.preparedTransactions.delete(txId);
  }

  addOperations(txId, operations) {
    this.preparedTransactions.set(txId, operations);
  }
}
```

### Пример использования

```javascript
// Создать coordinator и participants
const coordinator = new TwoPhaseCommitCoordinator();

const participant1 = new TwoPhaseCommitParticipant('DB1', db1);
const participant2 = new TwoPhaseCommitParticipant('DB2', db2);

// Transfer money
async function transferMoney(fromAccount, toAccount, amount) {
  const txId = await coordinator.beginTransaction([participant1, participant2]);

  // Подготовить операции для каждого participant
  participant1.addOperations(txId, [
    { type: 'UPDATE', table: 'accounts', set: { balance: -amount }, where: { id: fromAccount } }
  ]);

  participant2.addOperations(txId, [
    { type: 'UPDATE', table: 'accounts', set: { balance: +amount }, where: { id: toAccount } }
  ]);

  // Выполнить 2PC
  const result = await coordinator.prepare(txId);

  console.log(`Transaction ${txId}: ${result}`);
}

await transferMoney('account1', 'account2', 100);
```

### Проблемы 2PC

**1. Blocking Protocol**

```
Phase 1: Participant проголосовал YES
         Ресурсы заблокированы
         Ждёт решения Coordinator

Если Coordinator упал:
  Participant продолжает держать блокировки
  Другие транзакции заблокированы ❌
```

**2. Single Point of Failure**

```
Coordinator упал после Phase 1:
  Participants не знают, commit или abort
  Ресурсы заблокированы до восстановления Coordinator
```

**3. Performance**

```
2 network round-trips:
  1. PREPARE → votes
  2. COMMIT/ABORT → acks

Latency = 2 × network RTT
```

**4. Availability**

```
Если один participant недоступен:
  Вся транзакция ABORT
  Нарушение availability
```

### Three-Phase Commit (3PC)

Решает проблему blocking в 2PC, добавляя промежуточную фазу.

**Фазы:**
1. **CanCommit**: можно ли commit?
2. **PreCommit**: подготовиться к commit
3. **DoCommit**: commit

При падении coordinator, participants могут сделать timeout и принять решение самостоятельно.

**Проблема:** Сложнее, медленнее, редко используется.

## Saga Pattern

Saga — альтернатива 2PC для distributed transactions с eventual consistency.

**Идея:** Разбить транзакцию на последовательность локальных транзакций. При ошибке — compensating transactions (откат).

### Choreography-based Saga

Каждый сервис слушает события и выполняет свою часть.

```javascript
// Order Saga: Order → Payment → Inventory → Shipping

// Order Service
async function createOrder(orderData) {
  const order = await db.orders.create({
    ...orderData,
    status: 'PENDING',
  });

  await eventBus.publish('OrderCreated', {
    orderId: order.id,
    userId: orderData.userId,
    total: orderData.total,
    items: orderData.items,
  });

  return order;
}

eventBus.subscribe('PaymentCompleted', async (event) => {
  await db.orders.update(event.orderId, { status: 'PAID' });

  await eventBus.publish('OrderPaid', {
    orderId: event.orderId,
    items: event.items,
  });
});

eventBus.subscribe('PaymentFailed', async (event) => {
  await db.orders.update(event.orderId, { status: 'CANCELLED' });

  await eventBus.publish('OrderCancelled', {
    orderId: event.orderId,
    reason: 'payment_failed',
  });
});

// Payment Service
eventBus.subscribe('OrderCreated', async (event) => {
  try {
    const payment = await processPayment(event.userId, event.total);

    await eventBus.publish('PaymentCompleted', {
      orderId: event.orderId,
      paymentId: payment.id,
      items: event.items,
    });

  } catch (error) {
    await eventBus.publish('PaymentFailed', {
      orderId: event.orderId,
      error: error.message,
    });
  }
});

// Compensating transaction
eventBus.subscribe('OrderCancelled', async (event) => {
  // Refund payment если был успешный платёж
  const payment = await db.payments.findOne({ orderId: event.orderId });

  if (payment && payment.status === 'COMPLETED') {
    await refundPayment(payment.id);

    await eventBus.publish('PaymentRefunded', {
      orderId: event.orderId,
      paymentId: payment.id,
    });
  }
});

// Inventory Service
eventBus.subscribe('OrderPaid', async (event) => {
  try {
    await reserveInventory(event.items);

    await eventBus.publish('InventoryReserved', {
      orderId: event.orderId,
    });

  } catch (error) {
    // Недостаточно товара — compensate
    await eventBus.publish('InventoryReservationFailed', {
      orderId: event.orderId,
      error: error.message,
    });
  }
});

eventBus.subscribe('InventoryReservationFailed', async (event) => {
  // Trigger refund
  await eventBus.publish('OrderCancelled', {
    orderId: event.orderId,
    reason: 'inventory_unavailable',
  });
});

eventBus.subscribe('OrderCancelled', async (event) => {
  // Release reserved inventory
  await releaseInventory(event.orderId);
});
```

**Преимущества:**
- Loose coupling
- Каждый сервис автономен
- Нет single point of failure

**Недостатки:**
- Сложно отследить flow
- Нет centralized orchestration
- Сложнее debugging

### Orchestration-based Saga

Центральный orchestrator управляет saga.

```javascript
class OrderSagaOrchestrator {
  constructor() {
    this.sagas = new Map();
  }

  async execute(orderData) {
    const sagaId = this.generateSagaId();

    const saga = {
      sagaId,
      orderId: null,
      status: 'STARTED',
      steps: [],
      compensations: [],
    };

    this.sagas.set(sagaId, saga);

    try {
      // Step 1: Create Order
      console.log(`[Saga ${sagaId}] Step 1: Create Order`);

      const order = await orderService.createOrder(orderData);
      saga.orderId = order.id;

      saga.steps.push({
        name: 'CreateOrder',
        status: 'COMPLETED',
        data: { orderId: order.id },
      });

      saga.compensations.push(() => orderService.cancelOrder(order.id));

      // Step 2: Process Payment
      console.log(`[Saga ${sagaId}] Step 2: Process Payment`);

      const payment = await paymentService.charge({
        orderId: order.id,
        userId: orderData.userId,
        amount: order.total,
      });

      saga.steps.push({
        name: 'ProcessPayment',
        status: 'COMPLETED',
        data: { paymentId: payment.id },
      });

      saga.compensations.push(() => paymentService.refund(payment.id));

      // Step 3: Reserve Inventory
      console.log(`[Saga ${sagaId}] Step 3: Reserve Inventory`);

      await inventoryService.reserve({
        orderId: order.id,
        items: orderData.items,
      });

      saga.steps.push({
        name: 'ReserveInventory',
        status: 'COMPLETED',
      });

      saga.compensations.push(() => inventoryService.release(order.id));

      // Step 4: Schedule Shipping
      console.log(`[Saga ${sagaId}] Step 4: Schedule Shipping`);

      const shipping = await shippingService.schedule({
        orderId: order.id,
        address: orderData.shippingAddress,
      });

      saga.steps.push({
        name: 'ScheduleShipping',
        status: 'COMPLETED',
        data: { trackingNumber: shipping.trackingNumber },
      });

      // Success
      saga.status = 'COMPLETED';
      console.log(`[Saga ${sagaId}] COMPLETED successfully`);

      return {
        success: true,
        orderId: order.id,
        trackingNumber: shipping.trackingNumber,
      };

    } catch (error) {
      console.error(`[Saga ${sagaId}] ERROR:`, error.message);

      // Compensate в обратном порядке
      await this.compensate(sagaId);

      saga.status = 'COMPENSATED';

      throw new Error(`Order failed: ${error.message}`);
    }
  }

  async compensate(sagaId) {
    const saga = this.sagas.get(sagaId);

    if (!saga) return;

    console.log(`[Saga ${sagaId}] Starting compensation`);

    // Выполнить compensations в обратном порядке
    for (let i = saga.compensations.length - 1; i >= 0; i--) {
      const compensate = saga.compensations[i];
      const step = saga.steps[i];

      try {
        console.log(`[Saga ${sagaId}] Compensating: ${step.name}`);
        await compensate();
        step.status = 'COMPENSATED';

      } catch (error) {
        console.error(`[Saga ${sagaId}] Compensation failed for ${step.name}:`, error);
        step.status = 'COMPENSATION_FAILED';
        // Continue compensating other steps
      }
    }

    console.log(`[Saga ${sagaId}] Compensation completed`);
  }

  generateSagaId() {
    return `saga_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Использование
const orchestrator = new OrderSagaOrchestrator();

app.post('/api/orders', async (req, res) => {
  try {
    const result = await orchestrator.execute(req.body);
    res.json(result);

  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

**Преимущества:**
- Централизованная логика
- Легче отследить flow
- Проще debugging

**Недостатки:**
- Orchestrator — potential bottleneck
- Tight coupling с orchestrator
- Single point of failure

### Saga State Machine

```javascript
class SagaStateMachine {
  constructor(sagaDefinition) {
    this.definition = sagaDefinition;
    this.currentStep = 0;
  }

  async execute(context) {
    const { steps } = this.definition;

    for (let i = 0; i < steps.length; i++) {
      this.currentStep = i;
      const step = steps[i];

      try {
        console.log(`Executing step: ${step.name}`);

        const result = await step.action(context);

        context[step.name] = result;

      } catch (error) {
        console.error(`Step ${step.name} failed:`, error);

        // Compensate все предыдущие steps
        await this.compensate(context, i);

        throw error;
      }
    }

    return context;
  }

  async compensate(context, failedStepIndex) {
    const { steps } = this.definition;

    // Compensate в обратном порядке
    for (let i = failedStepIndex - 1; i >= 0; i--) {
      const step = steps[i];

      if (step.compensate) {
        try {
          console.log(`Compensating step: ${step.name}`);
          await step.compensate(context);

        } catch (error) {
          console.error(`Compensation failed for ${step.name}:`, error);
          // Log но продолжить compensating
        }
      }
    }
  }
}

// Определение Order Saga
const orderSagaDefinition = {
  name: 'OrderSaga',
  steps: [
    {
      name: 'CreateOrder',
      action: async (ctx) => {
        const order = await orderService.createOrder(ctx.orderData);
        return { orderId: order.id };
      },
      compensate: async (ctx) => {
        await orderService.cancelOrder(ctx.CreateOrder.orderId);
      },
    },
    {
      name: 'ProcessPayment',
      action: async (ctx) => {
        const payment = await paymentService.charge({
          orderId: ctx.CreateOrder.orderId,
          amount: ctx.orderData.total,
        });
        return { paymentId: payment.id };
      },
      compensate: async (ctx) => {
        await paymentService.refund(ctx.ProcessPayment.paymentId);
      },
    },
    {
      name: 'ReserveInventory',
      action: async (ctx) => {
        await inventoryService.reserve({
          orderId: ctx.CreateOrder.orderId,
          items: ctx.orderData.items,
        });
      },
      compensate: async (ctx) => {
        await inventoryService.release(ctx.CreateOrder.orderId);
      },
    },
  ],
};

// Использование
const saga = new SagaStateMachine(orderSagaDefinition);

const result = await saga.execute({
  orderData: {
    userId: 'user123',
    total: 99.99,
    items: [{ productId: 'prod1', quantity: 2 }],
  },
});
```

## Idempotency в Distributed Transactions

Критически важно для retry механизмов.

```javascript
class IdempotentTransactionHandler {
  constructor(redis) {
    this.redis = redis;
  }

  async executeIdempotently(txId, action) {
    const key = `tx:${txId}:status`;

    // Проверить, не выполнена ли уже
    const status = await this.redis.get(key);

    if (status === 'COMPLETED') {
      console.log(`Transaction ${txId} already completed`);
      const result = await this.redis.get(`tx:${txId}:result`);
      return JSON.parse(result);
    }

    if (status === 'IN_PROGRESS') {
      console.log(`Transaction ${txId} already in progress`);
      throw new Error('Transaction already in progress');
    }

    // Пометить как in progress
    await this.redis.set(key, 'IN_PROGRESS', 'EX', 3600);

    try {
      const result = await action();

      // Сохранить результат
      await this.redis.set(`tx:${txId}:result`, JSON.stringify(result), 'EX', 86400);
      await this.redis.set(key, 'COMPLETED', 'EX', 86400);

      return result;

    } catch (error) {
      await this.redis.del(key);
      throw error;
    }
  }
}

// Использование
const handler = new IdempotentTransactionHandler(redis);

app.post('/api/transfer', async (req, res) => {
  const { txId, fromAccount, toAccount, amount } = req.body;

  try {
    const result = await handler.executeIdempotently(txId, async () => {
      // Transfer logic
      await debit(fromAccount, amount);
      await credit(toAccount, amount);

      return { success: true, txId };
    });

    res.json(result);

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

## Outbox Pattern

Гарантия атомарности между DB update и event publish.

```javascript
// Проблема: DB commit успешен, но event не отправлен
async function createOrder(orderData) {
  await db.orders.create(orderData);  // ✅ Успешно

  await eventBus.publish('OrderCreated', orderData);  // ❌ Упал
}

// Решение: Outbox Pattern

// 1. Записать в outbox в той же транзакции
async function createOrder(orderData) {
  await db.transaction(async (tx) => {
    const order = await tx.orders.create(orderData);

    // Записать event в outbox table
    await tx.outbox.create({
      eventType: 'OrderCreated',
      payload: JSON.stringify({ orderId: order.id, ...orderData }),
      status: 'PENDING',
    });
  });
}

// 2. Background worker публикует события из outbox
class OutboxPublisher {
  constructor(db, eventBus) {
    this.db = db;
    this.eventBus = eventBus;
  }

  async start() {
    setInterval(() => this.publishPendingEvents(), 1000);
  }

  async publishPendingEvents() {
    const events = await this.db.outbox.findAll({
      where: { status: 'PENDING' },
      limit: 100,
    });

    for (const event of events) {
      try {
        await this.eventBus.publish(
          event.eventType,
          JSON.parse(event.payload)
        );

        // Пометить как опубликованное
        await this.db.outbox.update(event.id, { status: 'PUBLISHED' });

      } catch (error) {
        console.error('Failed to publish event:', error);
        // Retry later
      }
    }
  }
}

const publisher = new OutboxPublisher(db, eventBus);
publisher.start();
```

## Мониторинг Distributed Transactions

```javascript
const promClient = require('prom-client');

const distributedTxDuration = new promClient.Histogram({
  name: 'distributed_tx_duration_seconds',
  help: 'Distributed transaction duration',
  labelNames: ['saga', 'status'],
  buckets: [0.1, 0.5, 1, 2, 5, 10, 30],
});

const distributedTxTotal = new promClient.Counter({
  name: 'distributed_tx_total',
  help: 'Total distributed transactions',
  labelNames: ['saga', 'status'],
});

const compensationTotal = new promClient.Counter({
  name: 'distributed_tx_compensations_total',
  help: 'Total compensations executed',
  labelNames: ['saga', 'step'],
});

class MonitoredSagaOrchestrator {
  async execute(sagaName, orderData) {
    const start = Date.now();

    try {
      const result = await this.executeSaga(orderData);

      const duration = (Date.now() - start) / 1000;

      distributedTxDuration.observe({ saga: sagaName, status: 'success' }, duration);
      distributedTxTotal.inc({ saga: sagaName, status: 'success' });

      return result;

    } catch (error) {
      const duration = (Date.now() - start) / 1000;

      distributedTxDuration.observe({ saga: sagaName, status: 'failure' }, duration);
      distributedTxTotal.inc({ saga: sagaName, status: 'failure' });

      throw error;
    }
  }

  async compensate(sagaName, stepName) {
    compensationTotal.inc({ saga: sagaName, step: stepName });
    // ... compensation logic
  }
}
```

## 2PC vs Saga

| Критерий | 2PC | Saga |
|----------|-----|------|
| **Consistency** | Strong (ACID) | Eventual |
| **Isolation** | Да (блокировки) | Нет |
| **Availability** | Низкая (blocking) | Высокая |
| **Performance** | Медленно | Быстрее |
| **Complexity** | Простая логика | Compensating logic |
| **Use Case** | Financial, critical | Большинство случаев |

## Best Practices

### 1. Используйте Saga вместо 2PC

```
2PC: редко нужен
Saga: подходит для большинства случаев
```

### 2. Делайте операции идемпотентными

```javascript
// ❌ Bad: не идемпотентно
async function debit(account, amount) {
  const balance = await getBalance(account);
  await setBalance(account, balance - amount);
}

// ✅ Good: идемпотентно
async function debit(txId, account, amount) {
  await db.query(`
    UPDATE accounts
    SET balance = balance - $1
    WHERE id = $2 AND NOT EXISTS (
      SELECT 1 FROM processed_transactions WHERE tx_id = $3
    )
  `, [amount, account, txId]);

  await db.query(`
    INSERT INTO processed_transactions (tx_id, account, amount)
    VALUES ($1, $2, $3)
    ON CONFLICT DO NOTHING
  `, [txId, account, amount]);
}
```

### 3. Timeout для компенсаций

```javascript
async function compensateWithTimeout(compensationFn, timeout = 30000) {
  return Promise.race([
    compensationFn(),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Compensation timeout')), timeout)
    ),
  ]);
}
```

### 4. Dead Letter Queue для failed compensations

```javascript
async function compensate(sagaId) {
  for (const step of saga.compensations) {
    try {
      await step.compensate();
    } catch (error) {
      // Отправить в DLQ для manual intervention
      await sendToDLQ({
        sagaId,
        step: step.name,
        error: error.message,
      });
    }
  }
}
```

## Выводы

Distributed Transactions — сложная проблема с trade-offs:

**Two-Phase Commit (2PC):**
- ACID гарантии
- Blocking, низкая availability
- Редко используется в современных системах

**Saga:**
- Eventual consistency
- Non-blocking, высокая availability
- Требует compensating transactions
- **Choreography**: loose coupling, сложнее tracking
- **Orchestration**: centralized, проще debugging

**Ключевые паттерны:**
- Idempotency для retry
- Outbox для атомарности DB + events
- Monitoring для visibility

В большинстве случаев Saga предпочтительнее 2PC.

## Что читать дальше?

- [Урок 33: Репликация данных](33-data-replication.md)
- [Урок 34: Eventual Consistency](34-eventual-consistency.md)
- [Урок 27: Event-Driven Architecture](27-event-driven-architecture.md)

## Проверь себя

1. В чём разница между 2PC и Saga?
2. Как работает Two-Phase Commit?
3. Почему 2PC — blocking protocol?
4. Что такое compensating transaction?
5. В чём разница между Choreography и Orchestration Saga?
6. Как обеспечить idempotency в distributed transactions?
7. Что такое Outbox Pattern и зачем он нужен?
8. Какие проблемы у 2PC?
9. Когда использовать Saga вместо 2PC?
10. Как мониторить distributed transactions?

---
**Предыдущий урок**: [Урок 31: Consensus алгоритмы — Paxos, Raft](31-consensus-algorithms.md)
**Следующий урок**: [Урок 33: Репликация данных](33-data-replication.md)
