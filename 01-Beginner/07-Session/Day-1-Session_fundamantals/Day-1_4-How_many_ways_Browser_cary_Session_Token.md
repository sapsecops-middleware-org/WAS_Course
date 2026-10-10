# Part 3: Two Ways the Browser Carries the Token

The server creates the session. But the browser must show the token on every request. There are exactly two ways:

## Method 1 — Cookies (The Standard Way) 🍪

- After login, the server sends a small cookie to your browser:

```http
Cookie: JSESSIONID=A3F9X8B2:0001
```

- The browser stores it automatically.
- On every next request to that site, the browser automatically attaches it.
- You do nothing. It's invisible.

> [!TIP]
> **Analogy:** A wristband at a concert. Once put on, security sees it every time you enter. No effort from you.

This is how all banks do it — HDFC, ICICI, Citi, everyone.

## Method 2 — URL Rewriting (The Fallback) 🔗

If cookies are disabled in the browser, the server has a problem: *"Where do I stick the session ID?"*

**Answer:** directly into the URL:

```text
https://netbanking.citibank.co.in/dashboard;jsessionid=A3F9X8B2:0001
```

Notice the `;jsessionid=...` glued to the end. Every link on the page now carries this.

> [!TIP]
> **Analogy:** Instead of a wristband, your token number is written on your shirt. It works — but everyone can see it.

---

# Part 4: Why URL Rewriting is a Security Risk (Very Important) 🚨

> [!IMPORTANT]
> This is where I stop you as your senior. **URL rewriting in banking = fired.** Here's why:

## Risk 1 — Browser History

- The full URL (with session ID) is saved in history.
- Anyone opening that browser later sees your session ID.

## Risk 2 — Server Logs

- Every URL hit is logged on servers, proxies, firewalls.
- Your session ID is now sitting in plain text log files forever.
- An attacker with log access can steal it and hijack your session — **no password needed**.

## Risk 3 — Referer Header

- If the page has an external link (ad, help page), the browser sends the full URL as the `Referer` to that external site.
- Your session ID just leaked to a third party.

## Risk 4 — Shoulder Surfing / Screen Sharing

- The ID is visible in the address bar.
- Copy-paste it into a chat, and you've handed over your session.

## What the Regulations Say

> [!NOTE]
> **PCI-DSS** (card industry security standard) and **RBI guidelines** require:

- Session tracking must use **cookies, not URLs**.
- Cookies must have:
  - `Secure` flag → cookie only travels over HTTPS (never plain HTTP).
  - `HttpOnly` flag → JavaScript cannot read the cookie (protects from XSS attacks stealing it).

> We'll configure those flags in detail on **Day 28**.

**The rule:** cookies + `Secure` + `HttpOnly`. **Never URL rewriting.**

> [!TIP]
> **Practical tip:** In WAS, you can actually disable URL rewriting in the admin console so it can never be used. Good banks do exactly that.

---