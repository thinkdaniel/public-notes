# BASE Properties in Databases

## Introduction

BASE is a set of properties that describe how many modern distributed databases handle data — particularly NoSQL systems designed for high availability and scale. It emerged as an alternative to ACID for systems where strict consistency can be relaxed in exchange for better performance and availability.

**BASE stands for:**

- **B**asically **A**vailable
- **S**oft state
- **E**ventually consistent

BASE is often framed as the opposite of ACID, but it's more accurate to say it sits at a different point on the **CAP theorem** trade-off: it favours availability over strict consistency.

---

## Practical Context: CRM Application

We'll continue using the same **Customer Relationship Management (CRM)** application, but now consider features where strict ACID guarantees aren't required — or would hurt performance and scalability.

Examples of CRM features that suit BASE:

- **Activity feeds**: Recent interactions with a customer
- **Notifications**: Alerts for sales reps
- **Lead scores**: Calculated priority scores for prospects
- **Analytics dashboards**: Aggregated sales metrics
- **Customer tags and labels**: Flexible metadata on customer records

We'll model these using a document-style store (e.g. MongoDB) alongside our relational core.

> **Architectural note**: This guide assumes **polyglot persistence** — the CRM uses two databases: a relational database (PostgreSQL/MySQL) for ACID-sensitive features like orders and payments, and a document store (MongoDB) for BASE-suited features like feeds and scores. This is a common real-world pattern, but not the only option. Some databases like CockroachDB or Cassandra allow tunable consistency per query, letting a single database handle both patterns. MongoDB itself now supports multi-document ACID transactions, so a single document store can cover both use cases too. The two-database approach here is used to illustrate the contrast clearly.

### Sample Schema (Document Store)

```json
// Customer document with embedded activity feed
{
  "customer_id": "cust_250",
  "name": "Acme Corp",
  "email": "contact@acme.com",
  "status": "active",
  "lead_score": 87,
  "tags": ["enterprise", "renewal-due", "high-value"],
  "last_activity": "2024-03-15T14:30:00Z",
  "activity_feed": [
    {
      "type": "call",
      "rep_id": "rep_01",
      "note": "Discussed renewal",
      "timestamp": "2024-03-15T14:30:00Z"
    },
    {
      "type": "email",
      "rep_id": "rep_02",
      "note": "Sent proposal",
      "timestamp": "2024-03-14T10:00:00Z"
    }
  ],
  "notifications_unread": 3
}
```

---

## 1. Basically Available

### Definition

**Basically Available** means the system guarantees a response to every request, even during partial failures. The response might not be the most up-to-date data, but the system stays operational. Availability is prioritised over consistency.

### CRM Example: Activity Feed During Node Failure

In a distributed CRM deployment, customer data is replicated across multiple nodes. If one node goes down, the system still serves requests from the remaining nodes rather than refusing to respond.

```javascript
// Node 1 (primary) goes offline
// System automatically routes to Node 2 or Node 3

async function getCustomerActivityFeed(customerId) {
  try {
    // Attempt to read from primary node
    return await primaryNode.find({ customer_id: customerId });
  } catch (NodeUnavailableError) {
    // Fall back to replica - data may be slightly stale
    return await replicaNode.find({ customer_id: customerId });
  }
}
```

### What "Basically" Means

The system doesn't guarantee:

- The data is perfectly up to date
- All nodes return the same value at the same moment

But it does guarantee:

- ✅ A response is always returned
- ✅ The system doesn't go down because one node failed
- ✅ Sales reps can still view customer history during partial outages

### Why This Is Acceptable Here

An activity feed being a few seconds or minutes out of date has no meaningful business impact. Compare this to an account balance, where returning stale data could cause real financial harm — that would still use ACID.

---

## 2. Soft State

### Definition

**Soft state** means the state of the system may change over time, even without new input. This is because replicas are still synchronising, background processes are updating computed values, or caches are being refreshed. The data is in flux — it's not guaranteed to be stable at any given moment.

This contrasts with ACID, where the database is always in a definite, stable state.

### CRM Example: Lead Scores

Lead scores in a CRM are calculated from many signals: recent activity, email opens, deal stage, company size, and so on. Rather than recalculating synchronously on every event, the score is updated asynchronously by a background job.

```javascript
// Lead score recalculation runs asynchronously
async function recalculateLeadScore(customerId) {
  const signals = await gatherSignals(customerId);
  // This takes time - the score is "soft" in the meantime
  const newScore = scoreModel.calculate(signals);

  await db.customers.updateOne(
    { customer_id: customerId },
    { $set: { lead_score: newScore, score_updated_at: new Date() } },
  );
}

// Event triggers background job, doesn't wait for result
eventBus.on("customer.email_opened", (event) => {
  recalculateLeadScore(event.customerId); // fire and forget
});
```

### State Is In Flux

At any given moment, a customer's lead score might be:

- Reflecting yesterday's activity (not yet updated)
- Being updated on Node A but not yet replicated to Node B
- Temporarily at an intermediate value mid-calculation

```javascript
// Two reps querying at the same moment may see different scores
repA.getLeadScore("cust_250"); // Returns: 87 (Node A, just updated)
repB.getLeadScore("cust_250"); // Returns: 82 (Node B, not yet synced)
```

### Why This Is Acceptable Here

Lead scores are a guide, not a financial record. A sales rep acting on a score that is a few minutes stale is not a meaningful problem. The score will stabilise once all nodes sync — which leads us to eventual consistency.

---

## 3. Eventually Consistent

### Definition

**Eventually consistent** means that if no new updates are made to a piece of data, all replicas will converge to the same value — eventually. There is no guarantee of _when_, but the system will reach consistency given enough time and the absence of new writes.

### CRM Example: Notification Counts

When a sales activity is logged — a call, an email, a meeting — a notification is sent to relevant team members. In a distributed CRM, this notification count is replicated across nodes.

```javascript
// Rep logs a new activity on Node A
async function logActivity(customerId, repId, activityData) {
  await db.customers.updateOne(
    { customer_id: customerId },
    {
      $push: { activity_feed: activityData },
      $inc: { notifications_unread: 1 },
      $set: { last_activity: new Date() },
    },
  );
  // Node A is updated immediately
  // Nodes B and C will sync shortly after
}
```

**Timeline of consistency:**

```
T+0ms:   Rep logs activity on Node A
         Node A: notifications_unread = 4  ✅
         Node B: notifications_unread = 3  ❌ (not yet synced)
         Node C: notifications_unread = 3  ❌ (not yet synced)

T+50ms:  Replication propagates to Node B
         Node A: notifications_unread = 4  ✅
         Node B: notifications_unread = 4  ✅
         Node C: notifications_unread = 3  ❌ (not yet synced)

T+120ms: Replication propagates to Node C
         Node A: notifications_unread = 4  ✅
         Node B: notifications_unread = 4  ✅
         Node C: notifications_unread = 4  ✅  ← Eventually consistent
```

### Conflict Resolution

When two nodes receive conflicting writes before syncing, the system needs a strategy to resolve them.

```javascript
// Rep A and Rep B both update tags simultaneously on different nodes
// Node A write: tags = ["enterprise", "renewal-due"]
// Node B write: tags = ["enterprise", "high-value"]

// Common resolution strategies:
// 1. Last Write Wins (LWW) - use the most recent timestamp
// 2. Merge - combine both sets of changes
// 3. Application-level resolution - let the app decide

// Example: Merge strategy for tags (sensible for additive changes)
function mergeTags(nodeAState, nodeBState) {
  return {
    tags: [...new Set([...nodeAState.tags, ...nodeBState.tags])],
    // Result: ["enterprise", "renewal-due", "high-value"]
  };
}
```

### CRM Example: Analytics Dashboard

Sales dashboards aggregate data across many customers and reps. Recalculating these in real time for every change would be expensive. Instead, they update periodically:

```javascript
// Dashboard metrics updated every 5 minutes via batch job
async function refreshDashboardMetrics() {
  const metrics = await db.orders.aggregate([
    { $match: { order_date: { $gte: startOfDay() } } },
    {
      $group: {
        _id: "$rep_id",
        total_sales: { $sum: "$total_amount" },
        order_count: { $sum: 1 },
      },
    },
  ]);

  await db.dashboard.replaceOne(
    { type: "daily_sales" },
    { metrics, refreshed_at: new Date() },
    { upsert: true },
  );
}
```

During those 5 minutes, the dashboard is temporarily inconsistent — but it will converge to the correct state when the next refresh runs.

---

## BASE in Action: CRM Notification System

Here's how all three BASE properties work together in a real CRM feature:

```javascript
// A customer opens a marketing email
// This should: update the activity feed, recalculate lead score, notify the assigned rep

async function handleEmailOpenEvent(event) {
  const { customerId, repId, emailId, timestamp } = event;

  // BASICALLY AVAILABLE: write succeeds even if some nodes are degraded
  // The write goes to whichever nodes are healthy
  await db.customers.updateOne(
    { customer_id: customerId },
    {
      $push: {
        activity_feed: {
          type: "email_open",
          email_id: emailId,
          timestamp,
        },
      },
      $set: { last_activity: timestamp },
    },
    { writeConcern: { w: 1 } }, // Acknowledge from 1 node, don't wait for all replicas
  );

  // SOFT STATE: lead score update is deferred - state is temporarily stale
  jobQueue.enqueue("recalculate_lead_score", { customerId });

  // EVENTUALLY CONSISTENT: notification will reach the rep's dashboard
  // when replication catches up - typically within milliseconds to seconds
  jobQueue.enqueue("send_rep_notification", {
    repId,
    message: `Customer ${customerId} opened your email`,
    timestamp,
  });
}
```

**What BASE allows here:**

- The write completes quickly without waiting for all replicas to acknowledge
- The rep's notification might appear with a slight delay across different devices
- The lead score update happens in the background, not blocking the response
- If one node is temporarily down, the event is still processed successfully

---

## Real-World Implications

### Where BASE Fits in a CRM

| Feature             | ACID or BASE | Reason                          |
| ------------------- | ------------ | ------------------------------- |
| Process payment     | **ACID**     | Financial accuracy is critical  |
| Update order status | **ACID**     | Must be precise and consistent  |
| Activity feed       | **BASE**     | Slight delay is acceptable      |
| Lead score          | **BASE**     | Approximate value is fine       |
| Notification count  | **BASE**     | Eventual delivery is acceptable |
| Analytics dashboard | **BASE**     | Periodic refresh is fine        |
| Inventory deduction | **ACID**     | Can't oversell                  |
| Customer tags       | **BASE**     | Additive, low-stakes            |

### BASE Trade-offs

**Benefits:**

- ✅ Higher availability — system keeps running during partial failures
- ✅ Better write performance — no waiting for all replicas to confirm
- ✅ Scales horizontally — add more nodes without coordination overhead
- ✅ Tolerates network partitions between nodes

**Costs:**

- ⚠️ Stale reads are possible — users may briefly see outdated data
- ⚠️ Conflict resolution logic is needed for concurrent writes
- ⚠️ Harder to reason about data state at any given moment
- ⚠️ Not suitable where precision is required

---

## BASE in Modern Applications

Like ACID, you won't implement BASE mechanics from scratch. The distributed database or framework handles replication, conflict resolution, and convergence. Your job is to make the right architectural choices.

As a data engineer, you **don't manually implement BASE**. Instead:

- Use MongoDB with appropriate write concerns — replication is handled automatically
- Use Cassandra's tunable consistency levels — choose between speed and strictness per query
- Use Redis for caching — understand that cached values are soft state
- Understand BASE concepts to decide **which features can tolerate eventual consistency**

---

## Summary

| Property                  | Purpose                                           | CRM Example                                               |
| ------------------------- | ------------------------------------------------- | --------------------------------------------------------- |
| **Basically Available**   | System responds even during partial failures      | Activity feed served from replica when primary is down    |
| **Soft State**            | Data may be in flux between updates               | Lead score is stale until background job recalculates     |
| **Eventually Consistent** | All replicas converge to the same value over time | Notification counts sync across nodes within milliseconds |

### Key Takeaways

1. BASE trades strict consistency for **availability and performance**
2. Suitable for features where **approximate or slightly stale data is acceptable**
3. The same CRM application will use **both ACID and BASE** — the choice is per-feature
4. Eventual consistency doesn't mean "wrong forever" — it means "correct soon"
5. Common BASE databases include **MongoDB, Cassandra, DynamoDB, and Redis**

---

## ACID vs BASE: Choosing in a CRM Context

| Situation                      | Choose |
| ------------------------------ | ------ |
| Money is moving                | ACID   |
| Inventory is changing          | ACID   |
| User-facing counters and feeds | BASE   |
| Audit logs                     | ACID   |
| Recommendations and scores     | BASE   |
| Order records                  | ACID   |
| Analytics aggregates           | BASE   |

A mature CRM system uses both — a relational ACID database for transactional core data, and a distributed BASE store for high-volume, low-stakes operational data.

---

## Further Reading

- [CAP Theorem Explained](https://www.ibm.com/topics/cap-theorem)
- [MongoDB Replication](https://www.mongodb.com/docs/manual/replication/)
- [Cassandra Consistency Levels](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)
- [Eventual Consistency — Werner Vogels (Amazon)](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
