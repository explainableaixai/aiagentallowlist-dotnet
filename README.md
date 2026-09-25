# AlphaQuantum.AIAgentAllowlist

[AI agent allow list checks for .NET agents](https://www.aiagentallowlist.com). When a model in your application decides to open a web page, this client asks whether that page is safe for an unattended agent to act on. The answer names the kind of page (login, checkout, upload, account settings and similar) and gives a verdict your code can enforce.

```bash
dotnet add package AlphaQuantum.AIAgentAllowlist
```

Requires .NET 8. No dependencies beyond the base class library.

## The idea in one example

```csharp
using AlphaQuantum.AIAgentAllowlist;

var guard = new AIAgentAllowlistClient(Environment.GetEnvironmentVariable("AQ_API_KEY")!);
var r = await guard.CheckAsync("https://portal.example.com/account/billing");

Console.WriteLine(r["verdict"].GetString());          // e.g. "deny"
Console.WriteLine(r["matched"].GetProperty("id"));    // the page type that triggered it
```

Reading a pricing page is harmless. Submitting a payment form is not. Both can live on the same domain, so the check works on the URL, not just the site.

## Two kinds of answer

**Full URL in.** The response carries `verdict` and `matched`. `matched.layer` tells you which part of the service decided: `rules` for built-in path patterns, `page_type_db` for pages known from the catalogue. `matched.id` names the page type.

**Bare domain in.** The response describes the site: `found`, `language`, and `page_types`, an object mapping each page type to its URL on that site.

Both come back as `Dictionary<string, JsonElement>`. Use `GetProperty`, `TryGetProperty` and `EnumerateObject` to walk nested values.

## Guarding a browsing tool

Most .NET agent frameworks, including Semantic Kernel and AutoGen for .NET, let you expose C# methods as tools. Put the check inside the tool, so every path to the web goes through it:

```csharp
public sealed class BrowserTool(AIAgentAllowlistClient guard, IPageFetcher fetcher)
{
    [Description("Open a web page and return its text.")]
    public async Task<string> OpenAsync(string url, CancellationToken ct)
    {
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(4));

        Dictionary<string, JsonElement> r;
        try { r = await guard.CheckAsync(url, cts.Token); }
        catch (Exception) { return "This page could not be cleared by policy. Ask the user to open it."; }

        if (r.TryGetValue("verdict", out var v) && v.GetString() == "allow")
            return await fetcher.GetTextAsync(url, ct);

        var kind = r.TryGetValue("matched", out var m) && m.TryGetProperty("id", out var id)
            ? id.GetString() : "restricted";
        return $"Policy stops agents on {kind} pages. Ask the user to complete this step.";
    }
}
```

Why the tool returns text instead of throwing: the model reads the message and adjusts its plan. An unhandled exception usually ends the conversation, and the user never learns why.

Why only `allow` passes: new verdict values may appear. A guard that looks for `deny` would let unknown ones through.

Why a four-second limit: a stuck check should stop the step, not stall the agent.

## Showing people what the agent will skip

Before a long task, look up the domains involved and list their sensitive pages. People approve automation more readily when they can see its limits:

```csharp
var site = await guard.CheckAsync("salesforce.com");
if (site["found"].GetBoolean())
    foreach (var p in site["page_types"].EnumerateObject())
        Console.WriteLine($"{p.Name,-12} {p.Value.GetString()}");
```

A domain that is not in the catalogue returns `found: false`. URL checks still apply the path rules there, so common login and checkout paths are caught anyway.

## Configuration

```csharp
new AIAgentAllowlistClient(
    apiKey: key,
    httpClient: sharedHttpClient,                       // optional
    baseUrl: "https://www.aiagentallowlist.com/api");   // default
```

When you pass no `HttpClient`, the client creates one with a 30 second timeout. In ASP.NET Core, register it as a typed client through `IHttpClientFactory` and set a shorter timeout there.

## Exceptions

- `ArgumentException` for an empty key (constructor) or empty URL (call).
- `ApiException` for non-success HTTP responses, carrying `StatusCode` and `Body`. A 429 means back off. A 401 or 403 means check the key and quota.
- `TaskCanceledException` for timeouts and cancelled tokens.

## Observe first, then enforce

Run the guard in logging mode for a week: check every URL and record the verdict, but let the agent continue. The log shows which workflows reach sensitive pages and how often. Switch to enforcement starting with agents that can spend money, send messages or change records. That order avoids surprise failures in production.

## Keeping an audit trail

Record the agent run, the URL, the verdict and `matched.id` for every check. `ILogger` with structured properties is enough:

```csharp
logger.LogInformation("Agent {Run} {Verdict} {PageType} {Url}", runId, verdict, kind, url);
```

When someone asks what an agent tried to do, this is the answer.

## Background

Guidance on LLM security, such as the OWASP Top 10 for LLM Applications entry on excessive agency, recommends that agents act only within narrow limits and hand risky steps to people. Page-type checks are a concrete way to draw those limits on the open web.

## Related controls

Agents talk to more than web pages. Checking which services keep [data isolation in conversational AI tools](https://www.aitoolsblocklist.com/does-ai-train-on-your-data.php) stops agents from passing data to unapproved ones. [Shadow AI discovery](https://www.shadowaitools.com/how-it-works.php) in your logs reveals AI assistants already active on the network. Policies based on subject matter can run a [website category checker](https://www.websitecategorizationapi.com/website-url-category-check.php) on pages agents read.

Also packaged as [a Go module for agent executors](https://pkg.go.dev/github.com/explainableaixai/aiagentallowlist-go) and [a TypeScript-friendly npm package](https://www.npmjs.com/package/aiagentallowlist).

## License

MIT
