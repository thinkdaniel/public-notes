# ACID Properties in Databases

## Introduction

ACID is a set of properties that guarantee reliable processing of database transactions. These properties are essential for maintaining data integrity in relational databases, especially in systems where multiple users access and modify data concurrently.

**ACID stands for:**

- **A**tomicity
- **C**onsistency
- **I**solation
- **D**urability

---

## Practical Context: CRM Application

Let's explore ACID properties using a **Customer Relationship Management (CRM)** application as our example. This system manages:

- **Customers**: Customer information and contact details
- **Accounts**: Financial accounts for each customer
- **Orders**: Purchase orders placed by customers
- **Inventory**: Product stock levels
- **Activities**: Sales activities and interactions

### Sample Database Schema

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    status VARCHAR(20)
);

CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    customer_id INT,
    balance DECIMAL(10, 2),
    credit_limit DECIMAL(10, 2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP,
    total_amount DECIMAL(10, 2),
    status VARCHAR(20),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE inventory (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    quantity INT,
    price DECIMAL(10, 2)
);

CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10, 2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES inventory(product_id)
);
```

---

## 1. Atomicity

### Definition

**Atomicity** ensures that a transaction is treated as a single, indivisible unit of work. Either **all** operations in the transaction succeed, or **none** of them do. There's no partial completion.

### CRM Example: Processing an Order

When a customer places an order, multiple operations must occur:

```sql
BEGIN TRANSACTION;

-- Step 1: Create the order
INSERT INTO orders (order_id, customer_id, order_date, total_amount, status)
VALUES (1001, 250, NOW(), 299.99, 'pending');

-- Step 2: Add order items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, price)
VALUES (5001, 1001, 42, 2, 149.99);

-- Step 3: Deduct from inventory
UPDATE inventory
SET quantity = quantity - 2
WHERE product_id = 42;

-- Step 4: Update customer account balance
UPDATE accounts
SET balance = balance - 299.99
WHERE customer_id = 250;

COMMIT;
```

### What Happens Without Atomicity?

Imagine if **Step 3 fails** (e.g., insufficient inventory), but Steps 1, 2, and 4 succeed:

- ✅ Order is created
- ✅ Order items recorded
- ❌ Inventory update fails
- ✅ Customer charged

**Result**: Customer is charged, but the product isn't actually available. Data is inconsistent!

### With Atomicity

If **any step fails**, the entire transaction is rolled back:

- All changes are undone
- Database returns to its state before the transaction
- Customer is not charged, order is not created

```sql
BEGIN TRANSACTION;

-- All steps...

-- If any step fails:
ROLLBACK;  -- Everything is undone

-- If all steps succeed:
COMMIT;  -- All changes are permanent
```

---

## 2. Consistency

### Definition

**Consistency** ensures that a transaction brings the database from one valid state to another valid state, maintaining all defined rules, constraints, and triggers.

### CRM Example: Account Balance Rules

Business rules in our CRM:

1. Customer balance cannot exceed credit limit
2. Inventory quantity cannot be negative
3. Order total must match sum of order items
4. Customer status must be 'active' to place orders

```sql
-- Add constraints to enforce consistency
ALTER TABLE accounts ADD CONSTRAINT chk_credit_limit
CHECK (balance <= credit_limit);

ALTER TABLE inventory ADD CONSTRAINT chk_quantity
CHECK (quantity >= 0);

ALTER TABLE customers ADD CONSTRAINT chk_status
CHECK (status IN ('active', 'inactive', 'suspended'));
```

### Consistency Violation Example

```sql
BEGIN TRANSACTION;

-- Attempt to create an order that violates credit limit
UPDATE accounts
SET balance = balance - 5000.00  -- This would exceed credit limit
WHERE customer_id = 250 AND credit_limit = 1000.00;

-- Transaction fails, rolls back
-- Database constraint prevents consistency violation
```

### With Consistency

```sql
BEGIN TRANSACTION;

-- Check if customer can afford the purchase
IF (SELECT balance + 299.99 <= credit_limit FROM accounts WHERE customer_id = 250) THEN
    -- Process order
    UPDATE accounts SET balance = balance - 299.99 WHERE customer_id = 250;
    -- ... other steps
    COMMIT;
ELSE
    ROLLBACK;  -- Preserve consistency
END IF;
```

The database remains in a consistent state with all rules enforced.

---

## 3. Isolation

### Definition

**Isolation** ensures that concurrent transactions execute independently without interfering with each other. Changes made by one transaction are not visible to others until the transaction is committed.

### Isolation Levels

1. **READ UNCOMMITTED**: Lowest isolation, allows dirty reads
2. **READ COMMITTED**: Prevents dirty reads
3. **REPEATABLE READ**: Prevents dirty and non-repeatable reads
4. **SERIALIZABLE**: Highest isolation, full isolation

### CRM Example: Concurrent Order Processing

**Scenario**: Two sales representatives (Rep A and Rep B) try to sell the last 5 units of a product simultaneously.

#### Without Proper Isolation (Race Condition)

```sql
-- Rep A checks inventory
SELECT quantity FROM inventory WHERE product_id = 42;
-- Returns: 5 units available

-- Rep B checks inventory (simultaneously)
SELECT quantity FROM inventory WHERE product_id = 42;
-- Returns: 5 units available

-- Rep A sells 3 units
UPDATE inventory SET quantity = quantity - 3 WHERE product_id = 42;
-- Now: 2 units

-- Rep B sells 5 units (believes 5 are available!)
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 42;
-- Now: -3 units (IMPOSSIBLE!)
```

**Result**: Overselling! Negative inventory!

#### With Proper Isolation

```sql
-- Rep A's transaction
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT quantity FROM inventory WHERE product_id = 42 FOR UPDATE;
-- Locks the row, prevents concurrent access

UPDATE inventory SET quantity = quantity - 3 WHERE product_id = 42;
-- Updates to 2 units

COMMIT;  -- Releases lock

-- Rep B's transaction (starts after Rep A commits)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT quantity FROM inventory WHERE product_id = 42 FOR UPDATE;
-- Returns: 2 units (correct current value)

-- Attempts to update
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 42;
-- This would result in -3, violates CHECK constraint

ROLLBACK;  -- Transaction fails safely
```

### Common Isolation Problems

1. **Dirty Read**: Reading uncommitted changes from another transaction
2. **Non-Repeatable Read**: Getting different values when reading the same row twice
3. **Phantom Read**: Getting different sets of rows when executing the same query twice

---

## 4. Durability

### Definition

**Durability** guarantees that once a transaction is committed, its changes are permanent, even in the event of system failures (power loss, crashes, etc.).

### CRM Example: Order Confirmation

```sql
BEGIN TRANSACTION;

-- Customer places order
INSERT INTO orders (order_id, customer_id, order_date, total_amount, status)
VALUES (1001, 250, NOW(), 299.99, 'confirmed');

-- Update inventory
UPDATE inventory SET quantity = quantity - 2 WHERE product_id = 42;

-- Charge customer
UPDATE accounts SET balance = balance - 299.99 WHERE customer_id = 250;

COMMIT;  -- Order confirmed, customer notified
```

### What Durability Guarantees

**After COMMIT succeeds:**

- ✅ Customer receives order confirmation email
- ✅ **System crashes immediately** → Order is still saved when system restarts
- ✅ **Power outage occurs** → Order persists in the database
- ✅ **Database server reboots** → Order remains confirmed

### How Durability is Achieved

1. **Write-Ahead Logging (WAL)**
   - All changes are written to a log file before being applied to database
   - Log is stored on persistent storage (disk)

2. **Transaction Log**
   - Records all committed transactions
   - Used for recovery after failures

3. **Checkpoints**
   - Periodically flush dirty pages from memory to disk
   - Ensure data is persisted

### Without Durability

If durability weren't guaranteed:

- Customer receives confirmation, but order disappears after system restart
- Payment is processed, but order record is lost
- Inventory is deducted, but no record of the sale exists

---

## ACID in Action: Complete CRM Transaction

Let's see all ACID properties working together in a complete order processing scenario:

```sql
-- Complete order processing with ACID properties
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;  -- Isolation

-- Check customer status (Consistency)
DECLARE @customer_status VARCHAR(20);
SELECT @customer_status = status FROM customers WHERE customer_id = 250;

IF @customer_status != 'active' THEN
    ROLLBACK;  -- Maintain consistency
    RAISE ERROR 'Customer account is not active';
END IF;

-- Check credit limit (Consistency)
DECLARE @current_balance DECIMAL(10, 2);
DECLARE @credit_limit DECIMAL(10, 2);
SELECT @current_balance = balance, @credit_limit = credit_limit
FROM accounts WHERE customer_id = 250;

IF (@current_balance + 299.99) > @credit_limit THEN
    ROLLBACK;  -- Maintain consistency
    RAISE ERROR 'Credit limit exceeded';
END IF;

-- Check inventory availability (Isolation with locking)
DECLARE @available_quantity INT;
SELECT @available_quantity = quantity
FROM inventory
WHERE product_id = 42
FOR UPDATE;  -- Lock to prevent concurrent issues

IF @available_quantity < 2 THEN
    ROLLBACK;  -- Atomicity - all or nothing
    RAISE ERROR 'Insufficient inventory';
END IF;

-- All checks passed, proceed with order (Atomicity)
-- Step 1: Create order
INSERT INTO orders (order_id, customer_id, order_date, total_amount, status)
VALUES (1001, 250, NOW(), 299.99, 'confirmed');

-- Step 2: Add order items
INSERT INTO order_items (order_item_id, order_id, product_id, quantity, price)
VALUES (5001, 1001, 42, 2, 149.99);

-- Step 3: Update inventory
UPDATE inventory SET quantity = quantity - 2 WHERE product_id = 42;

-- Step 4: Update account
UPDATE accounts SET balance = balance - 299.99 WHERE customer_id = 250;

-- All steps successful
COMMIT;  -- Durability - changes are now permanent
-- Even if system crashes here, transaction is saved

-- Result: Order processed with all ACID guarantees
```

---

## Real-World Implications

### Why ACID Matters in CRM

1. **Financial Accuracy**: Customer accounts must be precise
2. **Inventory Management**: Prevent overselling
3. **Customer Trust**: Orders and payments must be reliable
4. **Compliance**: Meet regulatory requirements for data integrity
5. **Concurrent Users**: Multiple sales reps working simultaneously

### ACID Trade-offs

**Benefits:**

- ✅ Data integrity guaranteed
- ✅ Reliable transactions
- ✅ Easier to reason about data state

**Costs:**

- ⚠️ Performance overhead (locking, logging)
- ⚠️ Reduced concurrency
- ⚠️ May not scale as well for some use cases

---

## When to Relax ACID (BASE Alternative)

Some modern systems use **BASE** (Basically Available, Soft state, Eventually consistent) instead:

- **NoSQL databases** (MongoDB, Cassandra)
- **Distributed systems** with high availability requirements
- **Systems where eventual consistency is acceptable**

Example: Social media "likes" don't need ACID - it's okay if count is slightly off temporarily.

But for CRM financial transactions: **ACID is essential!**

---

## Summary

| Property        | Purpose                                     | CRM Example                                              |
| --------------- | ------------------------------------------- | -------------------------------------------------------- |
| **Atomicity**   | All or nothing execution                    | Order processing must complete fully or not at all       |
| **Consistency** | Maintain valid database state               | Credit limits and inventory constraints must be enforced |
| **Isolation**   | Prevent concurrent transaction interference | Multiple reps can't oversell the same inventory          |
| **Durability**  | Persist committed changes                   | Confirmed orders survive system crashes                  |

### Key Takeaways

1. ACID properties work together to ensure **data integrity**
2. Essential for systems handling **financial data** or **critical transactions**
3. Implemented through **transactions**, **constraints**, **locks**, and **logging**
4. Some performance trade-off, but worth it for data reliability
5. Modern databases (PostgreSQL, MySQL, SQL Server, Oracle) provide full ACID compliance

---

## ACID in Modern Applications

**Good news**: You don't need to write all that SQL code manually!

In modern development, **ACID properties are handled by the platform**:

- **Databases** (PostgreSQL, MySQL, SQL Server) automatically handle transactions
- **ORMs and frameworks** (SQLAlchemy, Django, Spring Boot) manage BEGIN/COMMIT/ROLLBACK for you
- **Data tools** (pandas with SQLAlchemy, Apache Spark) provide transaction support

### What This Means for Data Engineers

As a data engineer, you **don't manually implement ACID**. Instead:

- Use `pandas.to_sql()` with appropriate parameters - the platform handles atomicity
- Use SQLAlchemy sessions - transactions are managed automatically
- Focus on your **data pipeline logic** - the framework ensures ACID compliance
- Understand ACID concepts to make the right design choices

**Your job**: Write reliable data pipelines and understand data integrity

**Platform's job**: Handle transactions, commits, rollbacks, and durability

Understanding ACID helps you make informed decisions, but you won't be writing `BEGIN TRANSACTION` and `COMMIT` statements in your daily work.

---

## Further Reading

- [PostgreSQL Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [MySQL InnoDB and ACID](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html)
- [Understanding Database Locking](https://use-the-index-luke.com/sql/table-of-contents)
