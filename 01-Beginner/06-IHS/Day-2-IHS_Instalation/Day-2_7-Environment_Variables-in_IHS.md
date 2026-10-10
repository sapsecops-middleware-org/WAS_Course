# IBM HTTP Server (IHS) — Environment Variables Explained

A beginner-friendly guide to how IBM HTTP Server (IHS) uses environment variables at startup, the role of the `envvars` file, and why `ServerRoot` / `HTTPD_ROOT` anchor the entire configuration.

---

## 1. What Is an Environment Variable?

An **environment variable** is a value that a program reads when it starts, telling it how to behave.

- Think of it as a **note stuck on the wall** that the program reads before beginning work.
- Examples:
  - "Your home folder is here."
  - "Use this language."
  - "Look for libraries in this directory."

> [!NOTE]
> Environment variables are read **at startup**. If they are wrong, the program starts confused — or not at all.

---

## 2. Where Does IHS Keep These Variables?

IHS stores its startup environment variables in a single file:

```text
/opt/IBM/HTTPServer/conf/envvars
```

| Attribute | Detail |
|---|---|
| Location | `conf` directory — same place as `httpd.conf` |
| Purpose | Sets variables that IHS worker processes read at startup |
| Editing | Rarely edited — but you **must know it exists** |
| Key symptom | `"library not found"` errors at startup → check this file first |

> [!TIP]
> Think of `envvars` as IHS's **morning checklist**. If a checklist item is wrong, IHS starts the day confused.

---

## 3. Reading the Sample `envvars` File — Line by Line

### 3.1 Language Setting

```bash
HTTPD_LANG=en
```

- Sets the language/locale to English (`en`).
- Used mainly by **Java modules** inside IHS that need to know the locale.
- Only change this if you need a different language or region.

### 3.2 Java Home

```bash
JAVA_HOME=/opt/IBM/HTTPServer/java/jre
```

- Tells the system **where Java lives**.
- Some IHS modules that perform Java work require Java to run.
- Without `JAVA_HOME`, those modules cannot find Java → startup errors.

> [!TIP]
> Analogy: If a worker needs a toolbox, `JAVA_HOME` tells them exactly which shelf the toolbox is on.

### 3.3 Library Path

```bash
LD_LIBRARY_PATH=/opt/IBM/HTTPServer/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH
```

Two pieces here:

| Piece | Meaning |
|---|---|
| `LD_LIBRARY_PATH` | A list of folders where Linux searches for shared libraries (`.so` files — "helper code" files) |
| `/opt/IBM/HTTPServer/lib` | IHS's own library folder, added to the search list |
| `:$LD_LIBRARY_PATH` | The colon means "keep whatever was already there, and add mine on top" |
| `export` | Makes the variable visible to **child processes** (the worker processes IHS spawns). Without `export`, it stays private and is useless. |

> [!TIP]
> Analogy: Imagine a list of warehouses where a worker can pick up parts. `LD_LIBRARY_PATH` is that list — we just added warehouse to it.

---

## 4. The Big One: `HTTPD_ROOT`

When IHS starts, it internally sets:

```text
HTTPD_ROOT=/opt/IBM/HTTPServer
`` does this matter so much?

Because `httpd.conf` often uses **short, relative paths**, such as:

```apache
Logs/access_log
```

Access log — **where**? Here is the answer:

- IHS takes the relative path and prepends `HTTPD_ROOT` to it.
- So `logs/access_log` actually means:

```text
/opt/IBM/HTTPServer/logs/access_log
```

### Why `ServerRoot` Must Be the FIRST Line in `httpd.conf`

| Rule | Reason |
|---|---|
| `ServerRoot` is first | It is where `HTTPD_ROOT` comes from — the **anchor point** |
| All relative paths resolve from it | Every short path in the config is resolved against this anchor |
| If it is missing or wrong | IHS does not know where "home" is — everything after it falls apart |

> [!TIP]
> Analogy: `ServerRoot` is your **home address**. Directions like "two streets left, then right" (relative paths) only work if everyone knows the starting house.

---

## 5. Quick Cheat Sheet

| Item | What It Is | Why It Matters |
|---|---|---|
| `envvars` file | `/opt/IBM/HTTPServer/conf/envvars` | Holds startup variables for IHS |
| `HTTPD_LANG` | Language setting | For Java modules |
| `JAVA_HOME` | Path to Java | Java-dependent modules need it |
| `LD_LIBRARY_PATH` | Where to find libraries | Fixes "library not found" errors |
| `export` | Makes a variable public | Child processes can see it |
| `HTTP IHS's "home" folder | Resolves all relativeServerRoot` | First line of `httpd.conf` | Must be set before everything else |

---

## 6. Real-World Troubleshooting Tips

| Symptom | Check | What to Verify |
|---|---|---|
| `Error loading shared libraries` on startup | `LD_LIBRARY_PATH` in `envvars` | Is `/opt/IBM/HTTPServer/lib` listed? |
| Java module fails to load | `JAVA_HOME` | Does that folder actually exist? |
| Logs/files appearing in weird places | `ServerRoot` | A wrong `ServerRoot` makes relative paths land in the wrong folder |

> [!NOTE]
> **Golden rule:** You almost never edit `envvars` — but knowing it exists saves you hours when startup fails.

---

## 7. One-Sentence Summary

`envvars` sets IHS's startup "notes" (language, Java, libraries), and `HTTPD_ROOT` (derived from `ServerRoot`) is IHS's home address that every relative path in `httpd.conf` depends on.
