# Entity-Scoped Quota Management: Simplified Design

This document explains what it takes to support quota limits for specific entities (like **Users**, **Products**, or **Sandboxes**) using our existing centralized quota system.

---

## 1. The Core Concept (Why do we need this?)

Currently, our quota system only understands two levels:
1. **Organization Level** (e.g., Max 20 sandboxes per Org)
2. **Project Level** (e.g., Max 5 sandboxes per Project)

To support entity-level limits (e.g., **Max 10 snapshots per Sandbox**, or **Max 5 API keys per User**), the system must track limits and usage for specific individual instances. 

To keep the centralized quota engine clean, we should not add service-specific columns (like `sandbox_id` or `user_id`) to the central database. Instead, we use generic columns.

---

## 2. The 3 Steps to Implement This

### Step 1: Database Changes (Centralized DB)
We need to tell the quota tables which specific entity instance we are tracking usage for.

1. **Add `ENTITY` to the scope list:** Allow quotas to be configured with the scope `'ENTITY'` (in addition to `'ORG'` and `'PROJECT'`).
2. **Add Generic Columns to the Usage Table:**
   * `parent_entity_type` (e.g., `"sandbox"`, `"user"`, `"product"`)
   * `parent_entity_id` (e.g., `"01a00066-acc1..."`)

This way, the central database stays generic and doesn't need to know anything about sandbox or user tables.

---

### Step 2: API Changes
When a service (like `aiagent-service`) requests to reserve a slot, it will pass the entity type and entity ID to the centralized `tenant-service`.

**Example Request Payload:**
```json
{
  "requested_count": 1,
  "parent_entity_type": "sandbox",
  "parent_entity_id": "01a00066-acc1-71b2-8eda-74db3a5e4aee"
}
```

---

### Step 3: Enforcement Logic
The central quota engine will:
1. Find the quota configuration for the resource (e.g., `snapshots`, limit = 10).
2. Lock the specific row for that `parent_entity_id`.
3. Check: `current_usage + 1 <= 10`.
4. If allowed, increment the count and return `allowed: true`.

---

## 3. Trade-offs (Simple Comparison)

* **Advantages:**
  * **Unified System:** One central place to see and configure limits for everything (sandboxes, users, products, etc.).
  * **Custom Limits:** We can easily increase limits for specific customers (e.g., give an Enterprise customer 50 snapshots per sandbox instead of the default 10).

* **Disadvantages:**
  * **Database Size:** Every active entity instance (every user, every sandbox) will create a row in the central usage table, causing it to grow quickly.
  * **Cleanup Work:** When a sandbox or user is deleted, we must make sure to delete its usage record in the central database as well.
