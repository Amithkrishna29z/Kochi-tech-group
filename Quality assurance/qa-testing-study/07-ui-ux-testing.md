# 07 — UI / UX Testing

## Two flavors of UI testing

UI/UX testing comes in two distinct flavors — don't conflate them:

1. **Test in the UI what you couldn't test at the unit level** — behavior that only exists once things are rendered and interactive.
2. **Integration tests surfaced through the UI** — you drive the real screens to exercise how modules come together end-to-end.

The interesting part is *how* a program can reach into another program's interface and press its buttons. That mechanism is the heart of this module.

## Web: Selenium WebDriver

**Selenium WebDriver** is the classic web UI automation tool. It works because a **browser agent / extension injects into the DOM**; your **script and the agent communicate.** You tell the agent "find this element, click it," and the agent — living inside the browser — does it and reports back.

A key strength: because Selenium **abstracts the driver**, the *same script* can run against **Android, iOS, and web.** The script speaks to an abstraction; the driver underneath adapts to the platform.

```java
WebDriver driver = new ChromeDriver();
driver.get("https://example.com/login");
driver.findElement(By.id("username")).sendKeys("alice");   // "id" is a developer-provided hint
driver.findElement(By.id("password")).sendKeys("secret");
driver.findElement(By.cssSelector("button[type=submit]")).click();
assertTrue(driver.findElement(By.id("welcome")).isDisplayed());
```

## Desktop: Selenium and Ranorex

Two automation tools appear for desktop: **Selenium** and **Ranorex**.

Desktop UI is harder than web because rich toolkits — **WPF, Swing, JavaFX, Qt** — use **flexible UI where the toolkit owns the rendering.** There's no DOM to inspect. So how does an automation tool find a button?

- **Ranorex uses image processing** to locate elements — it literally looks at the pixels. This is **not fully foolproof** (themes, resolution, anti-aliasing, and overlapping windows all break it).
- To avoid relying on pixels, many teams include an **extra library that exposes an introspection / accessibility hierarchy**, so the automation tool can *query* for elements instead of *looking* for them.

### Automation-friendly vs "hostile-automatable"

This introduces a spectrum:

- **Windows / X-Windows** expose **introspection functions natively** — the OS itself lets you enumerate windows and controls. These are relatively **automation-friendly**, or as the roundtable put it, **"hostile-automatable"** (you can automate them from the *outside*, even without the app's cooperation).
- **Chromium-based UIs** have **no such rendering-introspection infrastructure** — the app draws itself and the OS can't see inside. So you must **inject a special library** (to expose a hierarchy) or fall back to **image processing.**

## The mechanism: agents, messages, and OS-mediated input

Underneath every UI automation tool is roughly the same choreography:

1. From a **host app** (your test runner) you **send a message** to the **application-under-test's window / port.**
2. An **agent installed on the test device** receives it, **resolves** the target element, and **taps the key / element.**
3. The actual key press or click is **delivered by the OS** — the operating system mediates input into the target application.

This is where a second spectrum matters — how *findable* the target element is:

- **Developer-aware automation** — you (the developer) **provide a hint / ID**: a `div` id, a CSS selector, an **accessibility label.** The automation tool asks for that specific thing and gets it. Reliable.
- **Externally / "hostile"-automatable** — no cooperation from the app; the tool must discover elements from the outside (OS introspection or image processing). Works, but more fragile.

> **Takeaway for developers:** the single most valuable thing you can do to make your UI testable is **provide stable IDs / accessibility labels.** It converts fragile "hostile" automation into reliable developer-aware automation — and, not coincidentally, it also makes your app usable by assistive technology.

## iOS: XCUITest

**XCUITest** is Apple's native UI automation framework.

- It **finds existing elements** and needs the **OS to mediate** the interaction (same OS-delivered-input idea as above).
- You **manually provide names / labels** for click targets — the developer-aware model.
- The introspection that makes this possible is the **accessibility infrastructure** — the *same* infrastructure used for visually-impaired users (e.g. **VoiceOver**). Accessibility labels are what the test framework reads to find elements. Good accessibility and good testability are the same investment.
- Because **Apple owns the test framework**, it grants tests some **sandboxing that would normally be forbidden in production** — the OS trusts its own automation framework to reach into the app.

## Mobile UI testing: cloud device farms and local devices

Mobile UI tests commonly run in the **cloud** (device farms — racks of real devices you rent), but they can also run **locally** with a device connected to your laptop via an **installed agent.**

A neat cross-platform trick: **Selenium can load an agent into the iOS *simulator*** and communicate with **XCUITest.** Effectively you're writing XCUITest, but *through Selenium* — so **one script runs on Android + iOS + web.**

- Caveat: for the one-script dream to hold, the **elements / keys must be kept roughly consistent across platforms** — and that consistency is a **manual guarantee** the team has to maintain. Nothing enforces it for you.

## Xamarin (left open)

**Xamarin** was mentioned as a wrinkle:

- `Xamarin.iOS` / `Xamarin.Android` — platform-specific bindings.
- **Xamarin.Forms** sits on top and behaves **WPF-like** (flexible, toolkit-owned rendering).
- **How exactly to test it was left open** in the discussion — flagged as an unresolved area, not a solved one.

## Check your understanding

1. What are the two flavors of UI testing? Give an example of each.
2. Explain how Selenium WebDriver actually clicks a button. What is the "agent," and where does it live?
3. Why is desktop UI (WPF/Qt/Swing) harder to automate than web? What two strategies do tools use to cope?
4. What does "hostile-automatable" mean, and why are Windows/X-Windows more hostile-automatable than Chromium UIs?
5. Contrast developer-aware automation with externally/hostile automation. Which is more reliable, and what does the developer do to enable it?
6. Why are accessibility labels (VoiceOver, etc.) central to XCUITest? What's the double payoff of adding them?
7. How can one Selenium script target Android, iOS, and web — and what manual guarantee does that depend on?
