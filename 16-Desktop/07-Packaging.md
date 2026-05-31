# Packaging and Deployment

## Getting the app onto users' machines

Building the app is half the job; **getting it installed and updated** on users' machines is the other half. Desktop deployment on .NET has several options — **MSIX** (the modern Windows package), **ClickOnce** (simple click-to-install + auto-update), plain **self-contained / single-file** publish (an xcopy-able folder/exe), and the **Microsoft Store**. Each trades off install experience, update mechanism, and IT/enterprise friendliness. The right choice depends on your audience (consumers vs enterprise), update needs, and whether you require an installer at all.

---

## Self-contained vs framework-dependent

First decision (shared with all .NET deployment — [Ch19 Deployment](../19-Deployment/README.md)): does the target machine have the .NET runtime?

| Deployment | Runtime | Size | Use |
|---|---|---|---|
| **Framework-dependent** | requires .NET installed | small | controlled environments where .NET is present |
| **Self-contained** | bundles the runtime | larger | machines without .NET; version isolation |

```bash
dotnet publish -c Release -r win-x64 --self-contained        # bundles the runtime
dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true   # one .exe
```

**Self-contained** avoids "you must install .NET first" friction at the cost of size; **single-file** publish bundles everything into one executable for easy distribution. Most consumer desktop apps go self-contained for a frictionless install.

---

## MSIX — the modern Windows package

**MSIX** is Microsoft's modern packaging format for Windows apps. It provides:

- **Clean install/uninstall** — the package is contained; uninstalling removes everything (no registry/file residue).
- **Auto-update** — apps can update from a URL/store automatically.
- **App identity** — required for some modern Windows APIs (notifications, certain WinRT features), and used by WinUI 3 ([03-WinUI3.md](03-WinUI3.md)).
- **Signing** — packages are signed (a certificate is required), giving integrity/trust.

MSIX is the recommended modern format, especially for **WinUI 3** apps and Store distribution. The trade-off: it requires a **code-signing certificate** and some setup, and the containerization can complicate apps that expect free file-system/registry access. Use MSIX for modern, cleanly-installed, auto-updating Windows apps.

---

## ClickOnce — simple install + auto-update

**ClickOnce** is a long-standing, low-ceremony deployment for Windows desktop apps (WPF/WinForms): publish to a web/file share, users click to install, and the app **auto-updates** from that location on launch. It's especially popular for **internal/enterprise line-of-business** apps — minimal infrastructure, easy updates, per-user install (no admin rights needed). Modern .NET supports ClickOnce for WPF/WinForms.

It's less suited to consumer Store distribution or apps needing MSIX's containerization/identity, but for "deploy an internal tool that updates itself," ClickOnce is hard to beat for simplicity.

---

## Microsoft Store

Distributing via the **Microsoft Store** (using MSIX) gives discoverability, trusted install, automatic updates, and licensing/commerce handling. It's the path for consumer apps wanting Store reach. The cost is Store policies/certification and the MSIX packaging requirement.

---

## Plain xcopy / installer

The simplest deployment: publish a self-contained folder (or single file) and distribute it directly (a zip, a network share, or wrap it in a traditional installer like WiX/Inno Setup/MSI). No auto-update, no identity, no signing requirement (though signing is still recommended for trust). Good for simple internal tools or when you want full control and minimal ceremony — pair with a traditional installer (WiX → MSI) for enterprise deployment via group policy/SCCM.

---

## AOT and trimming for desktop

- **Trimming** ([Ch19](../19-Deployment/README.md)) removes unused IL to shrink self-contained apps — but UI frameworks rely heavily on **reflection** (XAML loading, data binding), so trimming desktop UI apps is **risky** and often only partially supported. Test trimmed builds thoroughly; expect to add trim annotations or disable trimming for reflection-heavy UI.
- **Native AOT** is **largely not supported for full XAML UI frameworks** (WPF/WinUI rely on reflection/dynamic features AOT disallows). AOT suits console/server apps, not typical desktop UI. (WinForms has some limited AOT exploration, but it's not the norm.) Don't plan on AOT for a XAML desktop app; rely on R2R/self-contained publish for startup/size instead.
- **ReadyToRun (R2R)** ([Ch19](../19-Deployment/README.md)) precompiles IL to native ahead of time (alongside IL), improving **startup** without AOT's restrictions — a safe optimization for desktop apps.

---

## Choosing a deployment

```
Modern Windows app, clean install, auto-update, WinUI 3/Store?  → MSIX
Internal WPF/WinForms tool that should auto-update simply?       → ClickOnce
Consumer app, Store discoverability?                            → Microsoft Store (MSIX)
Simple internal distribution / full control?                    → self-contained (single-file) + optional WiX/MSI
Need .NET not pre-installed?                                     → self-contained
```

---

## Common gotchas

### Forgetting users may not have .NET

A framework-dependent app fails on machines without the matching .NET runtime. Use **self-contained** publish for unmanaged/consumer machines.

### Expecting AOT/trimming to "just work" for XAML UIs

XAML frameworks rely on reflection — **Native AOT generally isn't supported**, and **trimming is risky** (can remove members used by binding/XAML loading). Use **R2R** for startup gains; test any trimming heavily.

### MSIX without a signing certificate

MSIX packages must be **signed** — no certificate, no install (beyond sideloading dev setups). Plan for a code-signing cert.

### Unsigned executables

Even with xcopy/installer deployment, unsigned apps trigger SmartScreen warnings and erode trust. Code-sign your binaries/installer.

### ClickOnce for the wrong scenario

ClickOnce is great for internal auto-updating apps but isn't the path for Store distribution or MSIX-identity-dependent features. Match the mechanism to the audience.

---

## Summary

- Decide **self-contained** (bundles the runtime — frictionless on machines without .NET; **single-file** for one exe) vs **framework-dependent** (smaller, needs .NET installed) first.
- **MSIX**: modern Windows packaging — clean install/uninstall, auto-update, app identity, **signing required** — recommended for **WinUI 3** and **Store** apps. **ClickOnce**: low-ceremony **auto-updating** install, ideal for **internal WPF/WinForms** tools. **Store** (MSIX) for consumer reach. **xcopy/WiX-MSI** for simple/controlled distribution.
- **AOT generally isn't supported** for XAML UI frameworks (reflection-heavy), and **trimming is risky** — use **ReadyToRun** for safe startup gains instead ([Ch19](../19-Deployment/README.md)).
- **Code-sign** binaries/packages (mandatory for MSIX; recommended everywhere) to avoid SmartScreen warnings and establish trust; match the deployment mechanism to your audience (consumer vs enterprise).

→ Next: [Questions.md](Questions.md)
