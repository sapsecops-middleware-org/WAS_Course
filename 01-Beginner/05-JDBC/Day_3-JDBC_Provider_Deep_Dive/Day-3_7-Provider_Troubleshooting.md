# Lesson 10 — Troubleshooting JDBC Provider Problems (WAS ND)

> **Golden Rule:** `Save → Sync → Verify → Restart`

Today: **3 classic production problems**. Learn these well — they WILL happen to you one day.

---

## 🔴 Problem 1 — Typo in the Class Name

### The Story
The app crashes. You check the log (`SystemOut.log`) and see:

```text
java.lang.ClassNotFoundException:
oracle.jdbc.pool.OracleConnectionPooldataSource
                                  ↑
                              lowercase 'd' — typo!
```

### The Problem
Java is **case-sensitive**:

```text
OraclePool D ataSource   ✅ correct
OracleConnectionPool d ataSource   ❌ — small "d"
```

`DataSource` and `dataSource` are two different words to Java.

###-Life Example
It's like calling your bank "HBSC" instead of "HSBC". One letter wrong → nobody knows who you mean.

### The Fix
1. Open **Admin Console**
2. Go to: **Resources → JDBC → JDBC Providers**
3. Click your provider (e.g., *Oracle JDBC Driver - DigiBank*)
4. Fix the class name:

```text
oracle.jdbc.pool.OracleConnectionPoolDataSource
                                  ↑
                              uppercase 'D'
```

5. **Save → Sync → Restart** the server

### ✅ Lesson
- Copy-paste class names. **Never type them by hand.**
- If you see `ClassNotFoundException` → suspect a spelling/case error first.

---

## 🔴 Problem 2 — Wrong Scope (Provider to Some Servers)

### The Story
- You created the JDBC Provider at **Node01** scope.
- AppServer01 (on Node01) → works fine ✅
- AppServer02 (on Node02) → "DataSource not found" ❌

### Why?
Think of **scope** like a house:

- **Node scope** = the JDBC Provider lives inside Node01's house only.
- Node02's house has no key to it. So AppServer02 can't see it.

### Real-Life Example
You install a printer in one office room. Only that room can use it. Other rooms can't — even though they're in the same building.

### The Fix

| Option | Action | Verdict |
|---|---|---|
| **Option 1** ✅ | Delete and recreate at **Cell scope** | Best choice — whole building can see it |
| **Option 2** ❌ | Create a copy at Node02 scope | Works, but a **maintenance nightmare** |

 ✅ Lesson
> **In a cluster environment, always create the JDBC Provider at Cell scope.**

---

## 🔴 Problem 3 — Nodes Not Synchronized

### The Story
- You created the JDBC Provider in the Admin Console.
- You clicked **Save**. You felt happy. You went for coffee. ☕
- But you **forgot to sync** the nodes.

### What Happens

```text
Node01 (VM2) → has new config     ✅
Node02 (VM3) → still old config   ❌

AppServer02 restarts → still missing JDBC Provider
```

### Why?
In WAS ND:
- Admin Console saves changes to the **Cell repository** (the master copy).
- Each node has its **own local copy** of the config.
- **Sync** = copying the master copy to each node.

No sync = nodes run the old config.

### Real-Life Example
You update the head-office noticeboard but forget to email the branch offices. Branch staff keep following the old rules.

### The Fix

**Option A — wsadmin command:**

```javascript
AdminNodeManagement.syncActiveNodes()
```

**Option B — Admin Console:**

1. **System Administration → Nodes**
2. Select all nodes
3. Click **Full Resynchronize**

### ✅ Lesson (Prevention)
After **every** configuration change:

```text
Save → Sync → Verify sync status → Restart if needed
```

---

## 📋 Quick Summary Card

| # | Problem | Symptom | Fix | Golden Rule |
|---|---|---|---|---|
| 1 | Typo in class name | `ClassNotFoundException` | Correct the case (`DataSource`) | Copy-paste, never type |
| 2 | Wrong scope | One server sees it, another doesn't | Use **Cell scope** in clusters | Cell scope = safe choice |
| 3 | No sync | Console looks fine, server still broken | `syncActiveNodes()` or console sync | Save → Sync → Verify → Restart |

---

## 📝 Homework (Mental Check)

1. Why does Java reject `dataSource` but accept `DataSource`?
2. Which scope should you use for a cluster — Node or Cell? Why?
3. What are the 4 steps after every config change?
