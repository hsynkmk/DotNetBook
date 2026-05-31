# Forms and Validation

## `EditForm` — model-bound forms

Blazor's form story centers on **`EditForm`**, which binds to a model object, tracks edit state, and integrates validation. You bind inputs to model properties with Blazor's built-in input components (`InputText`, `InputNumber`, `InputSelect`, `InputCheckbox`, `InputDate`, `InputTextArea`), and `EditForm` raises `OnValidSubmit`/`OnInvalidSubmit` based on validation results.

```razor
<EditForm Model="model" OnValidSubmit="Save">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <InputText @bind-Value="model.Name" />
    <ValidationMessage For="() => model.Name" />

    <InputNumber @bind-Value="model.Age" />
    <button type="submit">Save</button>
</EditForm>

@code {
    private Person model = new();
    private void Save() { /* model is valid here */ }
}
```

---

## Validation with DataAnnotations

The most common validation approach: annotate the model with DataAnnotations attributes ([Ch03 §10](../03-HostingAndDI/10-Validation.md) uses the same attributes for options) and drop a `<DataAnnotationsValidator />` inside the form. Blazor validates on field change and on submit:

```csharp
public class Person {
    [Required, StringLength(50)] public string Name { get; set; } = "";
    [Range(0, 130)] public int Age { get; set; }
    [EmailAddress] public string? Email { get; set; }
}
```

- **`<DataAnnotationsValidator />`** runs the attribute validation against the bound model.
- **`<ValidationMessage For="() => model.Name" />`** shows the error for one field.
- **`<ValidationSummary />`** shows all errors.
- `OnValidSubmit` fires only when validation passes; `OnInvalidSubmit` when it doesn't.

This shares the same attribute vocabulary as server-side model validation ([Ch04 §08](../04-AspNetCore/08-Validation.md)) — and you can reuse the *same model class* on client and server, a Blazor advantage.

---

## `EditContext` — the engine underneath

`EditForm` is built on an **`EditContext`**, which tracks field state (modified/valid), raises field-change and validation events, and exposes `Validate()`. For advanced scenarios, create and pass your own:

```razor
<EditForm EditContext="editContext" OnSubmit="Submit">...</EditForm>

@code {
    private EditContext editContext = default!;
    private Person model = new();
    protected override void OnInitialized() {
        editContext = new EditContext(model);
        editContext.OnFieldChanged += (_, e) => { /* react to a specific field change */ };
    }
    void Submit() { if (editContext.Validate()) { /* save */ } }
}
```

`EditContext` lets you trigger validation programmatically, observe field changes, mark fields modified, and integrate non-DataAnnotations validation (FluentValidation, custom). `OnSubmit` (vs `OnValidSubmit`) gives you full control over when/whether to validate.

---

## Custom input components

The built-in inputs derive from **`InputBase<T>`**, which handles binding, validation wiring, and CSS state classes. To build a custom input (a date-range picker, a tag editor), derive from `InputBase<T>` and implement `TryParseValueFromString`:

```csharp
public class InputTags : InputBase<List<string>> {
    protected override bool TryParseValueFromString(
        string? value, out List<string> result, out string? error) {
        result = value?.Split(',').Select(s => s.Trim()).ToList() ?? [];
        error = null;
        return true;
    }
}
```

Deriving from `InputBase<T>` means your custom input participates in the form's validation and edit-state tracking automatically (it gets `valid`/`invalid`/`modified` CSS classes and `ValidationMessage` support) — far better than a raw `<input>` with manual binding.

---

## Server-side form handling (static SSR)

In static SSR (no interactivity — [02-RenderModes.md](02-RenderModes.md)), modern Blazor supports **standard HTTP form posts** with model binding via `[SupplyParameterFromForm]`, plus built-in antiforgery protection ([Ch10 §09](../10-Identity/09-Antiforgery.md)):

```razor
@page "/contact"
<EditForm Model="model" FormName="contact" OnValidSubmit="Submit">
    <AntiforgeryToken />
    <DataAnnotationsValidator />
    <InputText @bind-Value="model.Message" />
    <button type="submit">Send</button>
</EditForm>

@code {
    [SupplyParameterFromForm] private ContactModel model { get; set; } = new();
    void Submit() { /* handle posted form, no interactivity needed */ }
}
```

This lets forms work *without* an interactive render mode — the form posts over HTTP, Blazor binds and validates server-side, and antiforgery is enforced. A `FormName` disambiguates multiple forms on a page.

---

## Common gotchas

### Forgetting the validator component

Without `<DataAnnotationsValidator />` (or a FluentValidation validator) inside the `EditForm`, attributes are ignored and the form always "passes." Add the validator.

### Using a raw `<input>` instead of `Input*` components

A plain `<input>` doesn't integrate with `EditContext` validation or edit-state CSS. Use `InputText`/`InputNumber`/etc. (or derive from `InputBase<T>`).

### Mutating the model so binding breaks

Replacing the whole `Model` instance mid-edit (or binding to a different object than the `EditContext` tracks) desyncs validation state. Keep the bound model stable through the edit.

### Missing antiforgery token on SSR forms

A static-SSR form post without `<AntiforgeryToken />` is rejected (antiforgery is enforced). Include it (it's automatic inside `EditForm` with a `FormName` in many templates, but be explicit).

### Expecting client validation to be enough

Client-side validation is UX, not security. Always validate again on the server/in your domain — a malicious client can bypass the form. Reuse the model's attributes server-side.

---

## Summary

- **`EditForm`** binds a form to a model, tracks edit state, and raises `OnValidSubmit`/`OnInvalidSubmit`; use the **`Input*`** components (`InputText`, `InputNumber`, …) so fields integrate with validation and edit-state CSS.
- **DataAnnotations** validation via `<DataAnnotationsValidator />` + `<ValidationMessage>`/`<ValidationSummary>` — the same attributes you use for server model validation, reusable on the same model class.
- **`EditContext`** is the engine: programmatic `Validate()`, field-change events, and a hook for custom/FluentValidation; pass your own for advanced control (`OnSubmit`).
- Build **custom inputs** by deriving from **`InputBase<T>`** to inherit binding + validation participation.
- Static-SSR forms use standard **HTTP posts** (`[SupplyParameterFromForm]`, `FormName`) with **antiforgery** — forms work without an interactive render mode. Always re-validate server-side.

→ Next: [06-Routing.md](06-Routing.md)
