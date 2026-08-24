# 05 — API Testing

## Why API testing is central

Most functionality today is delivered as an **API**. The common shapes:

- **REST** — resource-oriented HTTP.
- **SOAP** — older, XML/envelope-based.
- **gRPC** — built on **Google Protocol Buffers** (binary, contract-first).

If most of what your software does is exposed through APIs, then testing those APIs is a large part of testing the software.

## Tools

- **SoapUI** — API testing (SOAP and REST).
- **Postman** — the ubiquitous manual API client.
- **Swagger UI** — .NET (and others) generate this from the API; you can exercise endpoints from the browser.

> **Practitioner observation (a real gap):** In their experience, **Postman scripts are rarely run in automation** — they're mostly used *manually*. Postman *can* run collections in CI, but many teams never wire that up. Treat "we have Postman collections" as **not** the same as "our API is tested in the pipeline."

## Nobody exposes raw REST to the outside world

A crucial real-world point:

> **Nobody exposes raw REST APIs to the outside world directly.** Most consumers use an **SDK strategy** — the provider ships a client library, and consumers call *that*, not the naked HTTP endpoint.

This gives **three distinct things you might be testing**, and conflating them causes confusion:

1. **Web API testing** — hit the **endpoint directly** (Postman / SoapUI, or xUnit against a hosted endpoint).
2. **SDK testing** — test the **generated client SDK** / call-level interface — e.g. a generated `.dll` (.NET) or `.jar` (Java) per module.
3. **Hybrid** — you get a **call-level interface** for the client, but under the hood it **actually hits REST**. This lets you change the URI (and other transport details) without changing the caller.
   - *Example:* **Informatica MDM Hub** generates a **new `.jar` when the data model changes**; you call the hub through that jar, and the jar talks REST underneath.

| Approach | What you call | What you're really testing |
|----------|---------------|----------------------------|
| Web API | The HTTP endpoint directly | Transport + endpoint behavior |
| SDK | A generated client library | The client-facing contract |
| Hybrid | A call-level interface that wraps REST | The interface *and* (indirectly) the transport |

## Testing an API with xUnit

You can test an API the same way you test any code — against known inputs and expected outputs, **including expected failures**:

```csharp
public class MathApiTests
{
    // Positive case: a Fact checking the SUT against a hard-coded value.
    [Fact]
    public void Add_TwoAndTwo_ReturnsFour()
        => Assert.Equal(4, new MathApi().Add(2, 2));

    // Negative case: integer overflow should THROW; the test passes
    // precisely because the expected exception is thrown.
    [Fact]
    public void Add_Overflow_Throws()
        => Assert.Throws<OverflowException>(() => new MathApi().AddChecked(int.MaxValue, 1));
}
```

Note the negative test: **the test passes *because* the expected exception was thrown.** Asserting on failure modes is as much a part of API testing as asserting on success.

## Historical: why ASP.NET MVC exists (the TDD-friendliness story)

A genuinely instructive piece of history:

- Classic **ASP.NET Web Forms** (and the older classic ASP.NET pipeline) mapped a URL more or less **directly to a file resource** (`Page.aspx`). This pipeline was **not TDD-friendly** — there was no clean seam to substitute for a test.
- **ASP.NET MVC** was created — with Microsoft effectively "eating" the older ASP.NET — partly to fix this. MVC maps a **URI → controller** (an indirection) instead of URI → `.aspx` file. That indirection is exactly what makes **test mapping** possible: the classic move became **"swap the controller / write a test controller."**

This introduces a useful design idea for testable APIs:

> **The Mediator pattern** — place a mediator *between* the controller and the service. It keeps things **modular and testable**: the controller talks to the mediator, the mediator routes to services, and you can substitute either side in a test.

## The API vs SPI divide (why API testing *really* matters in product companies)

This is the deep point of the module.

There is a **problem domain** and a **solution domain**. The API the **consumer** needs and the API the **service provider** needs are *different*:

- **API — Application Programming Interface** — what the **consumer** calls.
- **SPI — Service Provider Interface** — what the **provider** implements/plugs into.

Historical example: **Microsoft OLE DB** had an **OLE DB Consumer** and an **OLE DB Provider** — two sides of the same divide. The universal consumer side later became **ADO.NET**.

In real life, APIs must be somewhat **"intelligent"**: **one API call may map to many implementations** (many providers behind one consumer-facing call). That's exactly where testing gets hard and important:

> **Extensive API testing matters most in OEM / product companies**, where the API–SPI divide is *real* — one published API, many provider implementations behind it, all of which must honor the contract.

## Modularity: the precondition for reuse

To reuse classes / components / modules / services, the service must be written **modularly** — for instance, called via a **Mediator** or a **plugin**. If it *isn't* modular, consumers are forced to depend on a client SDK to get a clean seam.

> **One engineer's opinion (contested):** *Directly consuming REST services is not good practice* — better options exist (SDKs, mediators, plugins). **But** direct consumption **works fine when provider and consumer sit "in the same shop"** and there is **no API–SPI divide** — i.e., you control both ends and no third-party provider will ever plug in behind the API. Context decides.

## "Use the source, Luke" — a culture note

- In **OEM / product companies**, "**the source code IS the documentation**" ("use the source, Luke") is an acceptable stance — the consumers are sophisticated and the source is the contract of record.
- In a **services company** (Wipro/UST-style), saying that **gets you fired.** The client paid for documentation and predictable contracts, not a code-reading expedition.

This is another instance of the guide's recurring theme: **the "right" practice depends on the kind of company you're in.**

## Check your understanding

1. Name the three API shapes and one distinguishing feature of each.
2. Why is "we have Postman collections" not the same as "our API is tested"?
3. Distinguish web API testing, SDK testing, and hybrid testing. Give the Informatica MDM Hub as an example of one of them.
4. Write (in words) a negative API test. Why does it "pass" when something goes wrong?
5. Explain why ASP.NET MVC is more TDD-friendly than Web Forms. What role does the URI→controller indirection play?
6. Define API and SPI. Why does the API–SPI divide make extensive API testing matter most in OEM/product companies?
7. When is directly consuming a REST service acceptable, and when is it not?
