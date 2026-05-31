# Static Files

## Serving files from disk

Static file middleware serves files (HTML, CSS, JS, images, fonts) directly from a folder — by default `wwwroot` — without routing through endpoints. It's how SPAs, Blazor WebAssembly, and any app with client assets ship their static content.

```csharp
var app = builder.Build();
app.UseStaticFiles();          // serve files from wwwroot/
app.Run();
// wwwroot/css/site.css  →  GET /css/site.css
// wwwroot/index.html    →  GET /index.html  (and / if default files enabled)
```

`UseStaticFiles` matches the request path against files under the web-root and streams the file with appropriate headers (content type, caching). If no file matches, it calls the next middleware (so it doesn't interfere with your API routes).

---

## Pipeline placement

```csharp
app.UseHttpsRedirection();
app.UseStaticFiles();          // BEFORE routing — short-circuits for matched files (no auth/routing overhead)
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

Place `UseStaticFiles` **early** (before routing/auth): static assets are usually public and serving them shouldn't pay routing/auth cost. A matched file short-circuits the pipeline. (If certain files need authorization, see below.)

---

## Default files & directory structure

```csharp
app.UseDefaultFiles();    // serves index.html for "/" (must come BEFORE UseStaticFiles)
app.UseStaticFiles();
```

`UseDefaultFiles` rewrites a request for a directory (`/`) to its default document (`index.html`, `default.html`). It must run *before* `UseStaticFiles` (it only rewrites the path; static files then serves it). For a combined setup, `app.UseFileServer()` enables default files + static files (+ optional directory browsing) in one call.

---

## Content types & caching headers

Static file middleware sets the `Content-Type` from the file extension (via a `FileExtensionContentTypeProvider`) and can add caching headers:

```csharp
app.UseStaticFiles(new StaticFileOptions {
    OnPrepareResponse = ctx => {
        // cache static assets aggressively (they're versioned/fingerprinted)
        ctx.Context.Response.Headers.CacheControl = "public,max-age=31536000,immutable";
    }
});

// Add a custom content type
var provider = new FileExtensionContentTypeProvider();
provider.Mappings[".webmanifest"] = "application/manifest+json";
app.UseStaticFiles(new StaticFileOptions { ContentTypeProvider = provider });
```

For long-lived caching, **fingerprint** asset filenames (`site.abc123.css`) so a content change produces a new URL — then you can cache `immutable` for a year. Unknown extensions aren't served by default (you must add a mapping) — a safety feature.

---

## Serving from a different / additional folder

```csharp
app.UseStaticFiles(new StaticFileOptions {
    FileProvider = new PhysicalFileProvider(Path.Combine(builder.Environment.ContentRootPath, "assets")),
    RequestPath = "/assets"     // GET /assets/x.png → ContentRoot/assets/x.png
});
```

You can serve from folders outside `wwwroot` (or embedded resources via `ManifestEmbeddedFileProvider`) by supplying a `FileProvider` and a `RequestPath`. Useful for shared assets, plugin content, or files outside the web root.

---

## SPA fallback (single-page apps)

A SPA (React/Angular/Vue, or Blazor WebAssembly) handles routing client-side, so any non-file, non-API path should return `index.html` (the app's shell), letting the client router take over:

```csharp
app.UseStaticFiles();
app.MapControllers();              // or MapGet API routes
app.MapFallbackToFile("index.html");   // any unmatched path → index.html (client routing)
```

`MapFallbackToFile` serves the SPA shell for unmatched routes — so `/products/42` (a client route, not a server file/endpoint) returns the app, which then renders the right view. Place it **last** so it doesn't shadow real endpoints/files. (Blazor: [Ch14](../14-Blazor/README.md).)

---

## Security considerations

Static file serving touches the filesystem, so:
- **The web root is public.** Anything in `wwwroot` is served to anyone — never put secrets, config, or source there.
- **Path traversal is prevented** — the middleware blocks `../` escapes and only serves under the configured provider's root. Don't bypass this with custom file-serving from user input.
- **Authorization on static files** — by default static files skip auth (they short-circuit before it). To protect specific files, serve them via an **endpoint** (with `[Authorize]`) that streams the file, or place `UseStaticFiles` after auth for a protected subtree. Don't rely on obscurity.
- **Unknown extensions aren't served** by default — a guard against serving unexpected file types.

---

## Static asset delivery in modern .NET

.NET 9 added **`MapStaticAssets`** (and build-time asset optimization for Blazor/Razor) — it fingerprints, compresses (gzip/brotli at build), and sets optimal caching headers automatically:

```csharp
app.MapStaticAssets();   // optimized static asset delivery (fingerprinting, compression, caching)
```

For Razor/Blazor apps this replaces manual `UseStaticFiles` tuning with build-time-optimized delivery (precompressed, fingerprinted, immutable-cached). Prefer it where supported; `UseStaticFiles` remains for general/custom file serving.

---

## Common gotchas

### `UseStaticFiles` placed after routing/auth

Serving public assets then pays routing/auth cost unnecessarily (and may be blocked by auth). Place it early — unless you intend to protect the files.

### `UseDefaultFiles` after `UseStaticFiles`

It rewrites the path, so it must come **before** `UseStaticFiles`, or the rewrite happens too late. Or use `UseFileServer`.

### Fallback shadowing endpoints

`MapFallbackToFile` placed before your API routes catches everything → APIs return the SPA shell. Map it **last**.

### Secrets in `wwwroot`

Everything there is publicly served. Keep config/secrets/source out of the web root.

### Custom content type missing

Files with extensions not in the default map aren't served (404). Add the mapping via `FileExtensionContentTypeProvider`.

### No caching headers → slow repeat loads

Without cache headers, clients re-fetch assets every time. Fingerprint filenames and set long `Cache-Control` (or use `MapStaticAssets`).

---

## Summary

- **`UseStaticFiles`** serves files from `wwwroot` (the public web root); place it **early** in the pipeline so public assets skip routing/auth.
- **`UseDefaultFiles`** (before static files) serves `index.html` for `/`; `UseFileServer` combines them.
- Set **content types** (custom mappings) and **caching headers** (fingerprint filenames + long `Cache-Control`/`immutable`); serve from extra folders via a `FileProvider` + `RequestPath`.
- **`MapFallbackToFile("index.html")`** (placed last) enables SPA client-side routing.
- The web root is **public** — no secrets there; auth is skipped for static files by default (protect via an endpoint if needed); path traversal is blocked.
- Prefer **`MapStaticAssets`** (.NET 9+) for build-optimized, fingerprinted, compressed asset delivery in Razor/Blazor apps.

→ Next: [12-ProblemDetails.md](12-ProblemDetails.md)
