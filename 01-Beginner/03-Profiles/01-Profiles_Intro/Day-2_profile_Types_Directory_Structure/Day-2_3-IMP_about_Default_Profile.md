# WebSphere Application Server — Default Profile Explained

> [!NOTE]
> **Audience:** WAS administrators (beginner to intermediate)
> **Scope:** Understanding, checking, changing, and safely using the WebSphere *Default Profile*

---

## 1. The Big Picture

- One machine can host **many WAS profiles**.
- A **profile** = one WAS "instance" with its own servers, apps, and configuration.
- Example: A single banking server may host `AppSrv01`, `AppSrv02`, and `Dmgr01`.

> [!TIP]
> **Analogy:** One apartment building (the machine), many flats (the profiles). Each flat has its own kitchen, keys, and residents.

---

## 2. What Is a Default Profile?

When you type a WAS command **without specifying a profile**, WAS must choose one. The **Default Profile** is the profile WAS falls back to.

> **Rule:** No profile named? → WAS uses the default profile.

> [!TIP]
> **Analogy:** You say *"call a taxi"* without specifying a company. The hotel sends its **default** taxi company.

---

## 3. Check the Default Profile

Run:

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh -getDefaultName
```

Example output:

```text
AppSrv01
```

**Meaning:** *"If you don't tell me a profile, I'll use `AppSrv01`."*

---

## 4. Change the Default Profile

```bash
/opt/IBM/WebSphere/AppServer/bin/manageprofiles.sh \
  -setDefaultName \
  -profileName Dmgr01
```

Now `Dmgr01` is the default. Verify again with `-getDefaultName`.

---

## 5. Why This Matters — Real Production Story

Someone runs:

```bash
./startServer.sh PaymentsServer
```

What happens:

1. No profile is mentioned.
2. WAS looks in the **default profile** (`AppSrv01`).
3. But `PaymentsServer` actually lives in `AppSrv02`.
4. Result:

```text
Server not found
```

- The admin wastes **30–60 minutes** confused.
- In banking, that delay can mean **payment delays = money issues**.

> [!WARNING]
> On multi-profile machines, relying on the default profile is a common root cause of "Server not found" incidents.

---

## 6. The Golden Rule (Memorize This)

> [!IMPORTANT]
> In banking environments, **NEVER rely on the default profile. Always name it explicitly.**

```bash
./startServer.sh PaymentsServer -profileName AppSrv02
```

Why:

- ✅ No guessing
- ✅ No "Server not found" surprises
- ✅ Works the same on every machine, even if defaults differ
- ✅ Auditors love it — commands are explicit and traceable

---

## 7. Quick Summary Table

| Question | Answer |
|---|---|
| What is a default profile? | The profile WAS uses when you don't specify one |
| Check it | `manageprofiles.sh -getDefaultName` |
| Change it | `manageprofiles.sh -setDefaultName -profileName <name>` |
| Best practice | Always use `-profileName` explicitly |

---

## 8. Memory Trick

> *"Silence goes to the default. Speak the name, avoid the blame."*
