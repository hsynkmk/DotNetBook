# Outbound Authentication

## Authenticating your calls to other services

When your app calls another API, it usually must authenticate — most commonly with a **bearer token** (JWT/OAuth access token), sometimes with a client certificate or API key. This file covers the **client side**: acquiring tokens, attaching them to requests, refreshing them, and the OAuth client-credentials flow for service-to-service auth.

> The **server side** (validating tokens, issuing them, ASP.NET Core Identity, OAuth/OIDC) is **[Chapter 10 — Identity & Security](../10-Identity/README.md)**. Here we're the *caller*.

```csharp
// Attach a bearer token to a single request
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);

// Or centralize it in a DelegatingHandler so EVERY request carries it
public class BearerTokenHandler(ITokenProvider tokens) : DelegatingHandler {
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct) {
        request.Headers.Authorization = new("Bearer", await tokens.GetTokenAsync(ct));
        return await base.SendAsync(request, ct);
    }
}
builder.Services.AddHttpClient<ApiClient>().AddHttpMessageHandler<BearerTokenHandler>();
```

A `DelegatingHandler` ([03-DelegatingHandlers.md](03-DelegatingHandlers.md)) is the clean place to attach auth — every outbound call gets the token without per-call code.

---

## OAuth client-credentials flow (service-to-service)

For **service-to-service** auth (no user involved — your service calling another), the **client-credentials** OAuth flow is standard: your service authenticates with its **client id + secret** to a token endpoint, gets an **access token**, and uses it on requests:

```
Your service → POST /token (client_id, client_secret, scope)  → Identity provider
            ← access_token (a JWT, valid for ~1 hour)
Your service → GET /api/resource  Authorization: Bearer <access_token>  → target API
```

```csharp
// Acquire a token via client credentials (conceptual; libraries handle caching/refresh)
public class ClientCredentialsTokenProvider(IHttpClientFactory factory, IMemoryCache cache) : ITokenProvider {
    public async Task<string> GetTokenAsync(CancellationToken ct) =>
        await cache.GetOrCreateAsync("access_token", async entry => {
            var client = factory.CreateClient("identity");
            var response = await client.PostAsync("/connect/token", new FormUrlEncodedContent(new Dictionary<string, string> {
                ["grant_type"] = "client_credentials",
                ["client_id"] = _clientId,
                ["client_secret"] = _clientSecret,
                ["scope"] = "api.read"
            }), ct);
            var token = await response.Content.ReadFromJsonAsync<TokenResponse>(ct);
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(token!.ExpiresIn - 60);  // refresh before expiry
            return token.AccessToken;
        }) ?? throw new InvalidOperationException("Token acquisition failed");
}
```

The key behaviors: **cache the token** (don't fetch one per request — they're valid for the lifetime the IdP returns) and **refresh before expiry** (cache with a TTL slightly less than `expires_in`). Libraries automate this (below).

---

## Use a library, don't hand-roll OAuth

Hand-coding token acquisition, caching, refresh, and the various OAuth flows is error-prone. Use a maintained library:

- **`Microsoft.Extensions.ServiceDiscovery` / `IHttpClientFactory` + a token handler** — wire a token provider into the handler pipeline.
- **IdentityModel** (`Duende.IdentityModel`) — helpers for token endpoints, client credentials, token caching.
- **`Microsoft.Identity.Web` / MSAL** — for Microsoft Entra ID (Azure AD): acquires and caches tokens, handles refresh, supports managed identity. The standard for Azure-hosted services calling Azure-protected APIs.
- **Managed identity** (`DefaultAzureCredential`) — for Azure-to-Azure calls, **no secrets at all** ([Ch20 Azure](../20-AzureIntegration/README.md)).

```csharp
// Microsoft.Identity.Web — acquire a token for a downstream API (handles caching/refresh)
var token = await _tokenAcquisition.GetAccessTokenForAppAsync("api://target-api/.default");
```

These handle token caching, refresh, and flow details correctly — don't reinvent them.

---

## Refresh on 401 (token expiry mid-flight)

Even with proactive refresh, a token can expire between fetch and use. A handler can detect a **401 Unauthorized**, refresh the token, and retry once:

```csharp
public class RefreshOnUnauthorizedHandler(ITokenProvider tokens) : DelegatingHandler {
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct) {
        request.Headers.Authorization = new("Bearer", await tokens.GetTokenAsync(ct));
        var response = await base.SendAsync(request, ct);
        if (response.StatusCode == HttpStatusCode.Unauthorized) {
            await tokens.InvalidateAsync(ct);                                       // force refresh
            request.Headers.Authorization = new("Bearer", await tokens.GetTokenAsync(ct));
            response = await base.SendAsync(await CloneAsync(request), ct);          // retry once (clone — requests are single-use)
        }
        return response;
    }
}
```

Refresh-and-retry-once handles expiry races. Note `HttpRequestMessage` is single-use, so a retry needs a **cloned** request. Don't loop indefinitely on 401 (a persistent 401 means bad credentials, not expiry). Many libraries do this for you.

---

## Other auth schemes

```csharp
// API key (header) — for simple APIs
request.Headers.Add("X-API-Key", apiKey);

// Basic auth (rare; only over HTTPS)
var creds = Convert.ToBase64String(Encoding.UTF8.GetBytes($"{user}:{pass}"));
request.Headers.Authorization = new("Basic", creds);

// Client certificate (mutual TLS) — configure on the handler
var handler = new SocketsHttpHandler();
handler.SslOptions.ClientCertificates = [cert];
```

- **API key** — simplest; a shared secret in a header. Adequate for low-security/internal APIs; keep the key out of source (config/secret store).
- **Basic auth** — `user:pass` base64-encoded; only over HTTPS, and largely superseded by tokens.
- **Client certificates (mTLS)** — both sides present certificates; strong, used for high-security service-to-service.

Bearer tokens (OAuth) are the modern default for most APIs; the others fit specific scenarios.

---

## Secrets management

Whatever the scheme, the **credentials are secrets** — handle them properly ([Ch03 §07](../03-HostingAndDI/07-Configuration.md), [Ch10](../10-Identity/README.md)):
- **Never** hardcode client secrets / API keys / passwords in source or `appsettings.json`.
- Local dev → User Secrets; production → environment variables or a **secret store** (Azure Key Vault, AWS Secrets Manager).
- **Best**: use **managed identity** (`DefaultAzureCredential`) for Azure-to-Azure calls — no secret to store, rotate, or leak at all.

---

## Common gotchas

### Fetching a token per request

Tokens are valid for their lifetime (often ~1 hour) — fetching one per request hammers the IdP and adds latency. **Cache** the token; refresh before expiry.

### Hardcoded secrets

Client secrets/API keys in source or committed config leak. Use User Secrets (dev), env vars / Key Vault (prod), or managed identity (no secret).

### Hand-rolling OAuth flows

Error-prone (caching, refresh, flow details). Use MSAL/`Microsoft.Identity.Web`/IdentityModel, or managed identity.

### Not refreshing on 401

A token expiring mid-flight causes spurious 401s. Refresh-and-retry-once (with a cloned request); don't loop forever (persistent 401 = bad creds, not expiry).

### Reusing a single-use `HttpRequestMessage` on retry

`HttpRequestMessage` can't be resent. Clone it for the retry, or rebuild it.

### Putting auth logic in every call

Scattering token attachment per call is repetitive and error-prone. Centralize in a `DelegatingHandler` so every request carries auth.

---

## Summary

- For outbound auth, attach a **bearer token** (`Authorization: Bearer`) — centralize it in a **`DelegatingHandler`** so every request carries it.
- **Service-to-service** auth uses the **OAuth client-credentials** flow (client id + secret → access token); **cache the token** and **refresh before expiry** (don't fetch per request).
- **Use a library** (MSAL/`Microsoft.Identity.Web`, IdentityModel) or **managed identity** (`DefaultAzureCredential`, no secrets) — don't hand-roll OAuth flows/caching/refresh.
- Handle **token expiry** with refresh-and-retry-once on 401 (clone the single-use request); other schemes (API key, mTLS) fit specific needs.
- **Secrets** (client secrets/API keys) belong in User Secrets (dev) / env vars or a vault (prod), never in source — or eliminate them with managed identity. Server-side validation/issuance: [Ch10](../10-Identity/README.md).

→ Next: [06-Sockets.md](06-Sockets.md)
