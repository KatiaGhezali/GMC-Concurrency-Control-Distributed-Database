# GMC – Concurrency Control and Distributed Database Design

## Introduction

This project explores concurrency control and distributed database design through a banking system scenario.

The objective is to understand how a database can safely manage simultaneous transactions and how data can be organized and distributed across multiple branches.

The assignment is divided into two parts:

- **Part 1:** Transaction Management
- **Part 2:** Distributed Database Planning

---

# Part 1 – Transaction Management

## Step 1: Identify the Concurrency Issue

In the banking system, two users may attempt to transfer money from the same account at the same time.

One possible problem is a **Lost Update**.

A lost update occurs when two transactions read the same initial value and then update it independently. The second transaction may overwrite the result of the first transaction.

### Example

Suppose an account has a balance of $1,000.

Two transactions happen simultaneously:

- Transaction T1 transfers $200 from the account.
- Transaction T2 transfers $300 from the same account.

Without concurrency control, the following could happen:

| Operation | Transaction T1                  | Transaction T2                  |
| --------- | ------------------------------- | ------------------------------- |
| 1         | Reads balance: $1,000           |                                 |
| 2         |                                 | Reads balance: $1,000           |
| 3         | Calculates $1,000 - $200 = $800 |                                 |
| 4         |                                 | Calculates $1,000 - $300 = $700 |
| 5         | Writes balance: $800            |                                 |
| 6         |                                 | Writes balance: $700            |

The final balance becomes $700 instead of the correct balance of $500.

T1's update has been overwritten by T2. This is a lost update problem.

---

## Step 2: Propose a Locking Mechanism

To prevent this problem, the banking system should use database locks.

There are two main types of locks:

**1. Shared Lock (S-lock)**

- Used when a transaction only needs to read data.
- Multiple transactions can hold a shared lock on the same data simultaneously.
- Suitable for viewing account balances.

**2. Exclusive Lock (X-lock)**

- Used when a transaction needs to modify data.
- Only one transaction can hold an exclusive lock on a particular record at a time.
- Other transactions must wait until the lock is released.
- Suitable for money transfers.

### Proposed solution

For this banking system:

- Use shared locks when users view account balances.
- Use exclusive locks when users transfer money.

This allows multiple users to view balances simultaneously while ensuring that only one transaction modifies a particular account at a time.

---

## Step 3: Choose a Concurrency Control Strategy

The recommended strategy is **Pessimistic Locking**.

Pessimistic locking assumes that conflicts may occur, so it locks the data before modifying it.

When a user initiates a transfer:

1. The system requests an exclusive lock on the account.
2. If another transaction already holds the lock, the new transaction must wait.
3. Once the lock is available, the transaction reads the current balance.
4. It performs the transfer and updates the balance.
5. It commits the transaction and releases the lock.

### Why choose pessimistic locking?

Pessimistic locking is appropriate for banking operations because incorrect balances and lost updates must be prevented.

It ensures that transactions modifying the same account are executed safely and sequentially.

Optimistic locking is another possible approach. It allows transactions to proceed without initially locking records, then checks for conflicts before committing. However, if conflicts occur, a transaction may need to be retried.

For this scenario, pessimistic locking is preferable because protecting financial data is a priority.

---

## Step 4: Demonstrate a Safe Transaction Schedule

Consider the same account with an initial balance of $1,000.

- T1 transfers $200.
- T2 transfers $300.

With exclusive locking, the transactions are executed in a safe order.

| Step | Transaction T1                  | Transaction T2                  |
| ---- | ------------------------------- | ------------------------------- |
| 1    | Requests an exclusive lock      |                                 |
| 2    | Acquires the lock               |                                 |
| 3    |                                 | Requests an exclusive lock      |
| 4    |                                 | Waits because T1 holds the lock |
| 5    | Reads balance: $1,000           |                                 |
| 6    | Deducts $200; new balance: $800 |                                 |
| 7    | Commits the transaction         |                                 |
| 8    | Releases the lock               |                                 |
| 9    |                                 | Acquires the lock               |
| 10   |                                 | Reads updated balance: $800     |
| 11   |                                 | Deducts $300; new balance: $500 |
| 12   |                                 | Commits and releases the lock   |

### Is this schedule safe?

**Yes.**

T2 cannot modify the account while T1 holds the exclusive lock. After T1 commits, T2 reads the updated balance of $800 and deducts $300.

The final balance is therefore $500, which is correct.

This approach follows the principle of **Two-Phase Locking (2PL)**. In particular, strict two-phase locking keeps exclusive locks until the transaction commits or rolls back.

In a real banking system, a transfer should also update both the source and destination accounts as part of one atomic transaction. Both account updates must succeed together or be rolled back together.

---

# Part 2 – Distributed Database Planning

The banking system operates across three branches:

- Tunis
- Sousse
- Sfax

The goal is to organize the data so that each branch can access the information it needs while maintaining consistency across the system.

## Step 1: Propose a Horizontal Fragmentation Strategy

Horizontal fragmentation divides a table into smaller tables based on rows.

For the `Customers` table, customers can be divided according to their branch.

For example, the original table could contain:

| customer_id | name  | age | branch |
| ----------- | ----- | --- | ------ |
| 1           | Ahmed | 25  | Tunis  |
| 2           | Sara  | 30  | Sousse |
| 3           | Amine | 28  | Sfax   |
| 4           | Nadia | 22  | Tunis  |
| 5           | Karim | 35  | Sousse |

We can create three horizontal fragments:

**Tunis Customers**

Contains customers assigned to the Tunis branch.

```sql
SELECT *
FROM Customers
WHERE branch = 'Tunis';
```

**Sousse Customers**

Contains customers assigned to the Sousse branch.

```sql
SELECT *
FROM Customers
WHERE branch = 'Sousse';
```

**Sfax Customers**

Contains customers assigned to the Sfax branch.

```sql
SELECT *
FROM Customers
WHERE branch = 'Sfax';
```

### Why use horizontal fragmentation?

- Each branch can access its own customers more efficiently.
- It reduces the amount of data each branch needs to manage.
- It improves local query performance.
- It allows the system to organize customer records by branch.

The fragments should satisfy three important properties:

- **Completeness:** Every customer must belong to a fragment.
- **Disjointness:** A customer should not appear in multiple fragments.
- **Reconstruction:** The original Customers table can be reconstructed by combining the fragments.

---

## Step 2: Propose a Vertical Fragmentation Strategy

Vertical fragmentation divides a table into smaller tables based on columns.

For example, the original `Customers` table might contain:

| customer_id | name  | age | email             | password_hash |
| ----------- | ----- | --- | ----------------- | ------------- |
| 1           | Ahmed | 25  | ahmed@example.com | hash_123      |
| 2           | Sara  | 30  | sara@example.com  | hash_456      |

We can separate the authentication information into another table.

**Customers table**

Contains general customer information:

| customer_id | name  | age |
| ----------- | ----- | --- |
| 1           | Ahmed | 25  |
| 2           | Sara  | 30  |

**CustomerCredentials table**

Contains authentication information:

| customer_id | email             | password_hash |
| ----------- | ----------------- | ------------- |
| 1           | ahmed@example.com | hash_123      |
| 2           | sara@example.com  | hash_456      |

Both tables share `customer_id`.

It acts as the primary key in both tables, and it is also a foreign key in `CustomerCredentials` referencing `Customers`.

### Why use vertical fragmentation?

- It separates sensitive authentication information from general customer data.
- It improves security by limiting access to credentials.
- It allows the system to retrieve general customer information without accessing sensitive login details.
- It makes access control easier to manage.

---

## Step 3: Decide Which Data Should Be Replicated Across Branches

Replication means keeping copies of data at multiple locations so that different branches can access it.

The banking system manages three main types of data.

### 1. Customer Information

**Proposed strategy: Replicate essential customer information across branches.**

Each branch should be able to access basic customer details, even if the customer originally registered at another branch.

For example, a customer registered in Tunis should still be identifiable when visiting the Sousse branch.

Sensitive information, such as password hashes, should have stricter access controls.

### 2. Account Balances

**Proposed strategy: Maintain one authoritative balance for each account.**

All branches should be able to access the current account balance.

However, they should not independently update separate copies of the same balance because this could lead to inconsistent financial information.

Account updates should be coordinated through the authoritative database record, with any replicated or cached values synchronized appropriately.

### 3. Transaction History

**Proposed strategy: Store the authoritative transaction history at the originating branch and make it accessible to other branches.**

For example:

- Transactions originating in Tunis are stored at Tunis.
- Transactions originating in Sousse are stored at Sousse.
- Transactions originating in Sfax are stored at Sfax.

Other branches can retrieve this information when necessary through replication or distributed queries.

### Summary

| Data Type            | Replication Strategy                                                                      | Reason                                                                |
| -------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Customer Information | Replicate essential information across branches                                           | Allows branches to access customer details                            |
| Account Balances     | Maintain one authoritative record per account                                             | Prevents conflicting balance updates                                  |
| Transaction History  | Keep the authoritative history at the originating branch and make it accessible elsewhere | Maintains a clear source of truth and supports access across branches |

---

## Step 4: Choose an Allocation Strategy for Transaction History

The recommended strategy is **Static Allocation**.

Static allocation means assigning data to a specific location according to a predefined rule.

For this banking system, each transaction is stored at the branch where it originates.

For example:

- Transactions originating in Tunis are allocated to the Tunis database.
- Transactions originating in Sousse are allocated to the Sousse database.
- Transactions originating in Sfax are allocated to the Sfax database.

### Why choose static allocation?

- Each branch has a clear responsibility for its transactions.
- Data is organized according to the branch where the transaction occurred.
- It reduces unnecessary data movement.
- It provides a clear location for storing and managing transaction records.

Other branches can still access the transaction history when needed through distributed queries or replication.

Dynamic allocation could be useful if data placement needed to change frequently based on workload or access patterns. However, static allocation is simpler and more appropriate for this scenario.

---

# Final Summary

| Requirement                      | Proposed Solution                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------------- |
| Concurrency issue                | Lost Update                                                                                       |
| Locking mechanism                | Shared locks for reading and exclusive locks for transfers                                        |
| Concurrency control strategy     | Pessimistic Locking                                                                               |
| Transaction safety               | Strict Two-Phase Locking                                                                          |
| Horizontal fragmentation         | Divide customers according to their branch                                                        |
| Vertical fragmentation           | Separate customer credentials from general customer information                                   |
| Customer information replication | Replicate essential customer information                                                          |
| Account balance replication      | Maintain one authoritative balance per account                                                    |
| Transaction history replication  | Keep the authoritative history at the originating branch and make it accessible to other branches |
| Transaction history allocation   | Static Allocation                                                                                 |

## Conclusion

This scenario demonstrates how concurrency control and distributed database design can help a banking system maintain data accuracy, security, and availability.

Pessimistic locking and exclusive locks prevent conflicting updates when multiple users transfer money from the same account. Horizontal and vertical fragmentation organize customer information efficiently, while appropriate replication and static allocation strategies allow the three branches to access the information they need without compromising consistency.
