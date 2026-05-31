# Layouts

## Arranging controls on screen

A **layout** is a container that positions and sizes its child controls. MAUI gives you several layout types, each with a different arrangement strategy; choosing the right one (and nesting them sensibly) is most of building a responsive UI that adapts across phone, tablet, and desktop. The main ones: **Grid** (rows/columns), **StackLayout** (a line), **FlexLayout** (wrapping/flexible), and **AbsoluteLayout** (explicit coordinates).

```xml
<Grid RowDefinitions="Auto,*,Auto" ColumnDefinitions="*,2*">
    <Label Grid.Row="0" Grid.ColumnSpan="2" Text="Header" />
    <CollectionView Grid.Row="1" Grid.Column="0" />
    <BoxView Grid.Row="1" Grid.Column="1" />
    <Button Grid.Row="2" Grid.ColumnSpan="2" Text="Footer" />
</Grid>
```

---

## Grid — the workhorse

**Grid** arranges children in rows and columns — the most flexible and most-used layout. You define `RowDefinitions`/`ColumnDefinitions` with three sizing modes:

| Size | Meaning |
|---|---|
| `Auto` | size to the content |
| `*` (star) | take remaining space proportionally (`2*` = twice the share of `*`) |
| fixed (e.g. `100`) | exact device-independent units |

Children declare their cell via the **attached properties** `Grid.Row`/`Grid.Column` (and span with `Grid.RowSpan`/`Grid.ColumnSpan`) ([02-XAML.md](02-XAML.md)):

```xml
<Grid RowDefinitions="Auto,*" ColumnDefinitions="100,*,Auto" RowSpacing="8" ColumnSpacing="12">
    ...
</Grid>
```

Grid is ideal for structured, two-dimensional layouts (forms, dashboards, master/detail). The `*`/`Auto`/fixed mix is what makes a Grid responsive — star columns absorb extra space as the screen grows.

---

## StackLayout — a single line

**`VerticalStackLayout`** and **`HorizontalStackLayout`** stack children in one direction. They're simple and fast (the dedicated single-orientation versions are more efficient than the older general `StackLayout`):

```xml
<VerticalStackLayout Spacing="10" Padding="16">
    <Label Text="Name" />
    <Entry Placeholder="Enter name" />
    <Button Text="Submit" />
</VerticalStackLayout>
```

Use stacks for simple linear arrangements (a form's fields, a toolbar of buttons). **Gotcha**: a stack sizes to its content and doesn't constrain children — nesting a scrollable/large control inside a stack (especially a `CollectionView` inside a `VerticalStackLayout`) can break scrolling or measure infinitely. Put large/scrolling content in a Grid cell (`*`) instead.

---

## FlexLayout — wrapping and flexible

**FlexLayout** implements CSS Flexbox-style arrangement: children flow in a direction and can **wrap** to new lines, with control over justification, alignment, and growth:

```xml
<FlexLayout Wrap="Wrap" JustifyContent="SpaceEvenly" AlignItems="Center">
    <Button Text="Tag1" />
    <Button Text="Tag2" />
    <Button Text="Tag3" />   <!-- wraps to next line when out of room -->
</FlexLayout>
```

FlexLayout shines for collections of items that should wrap responsively (tag clouds, button bars, card grids that reflow by screen width) — where a Grid's fixed rows/columns would be too rigid.

---

## AbsoluteLayout — explicit positioning

**AbsoluteLayout** positions children at explicit coordinates/sizes (proportional or absolute), via the `AbsoluteLayout.LayoutBounds` and `LayoutFlags` attached properties:

```xml
<AbsoluteLayout>
    <Image AbsoluteLayout.LayoutBounds="0,0,1,1"
           AbsoluteLayout.LayoutFlags="All" />              <!-- full-screen background -->
    <Button AbsoluteLayout.LayoutBounds="0.5,0.9,100,40"
            AbsoluteLayout.LayoutFlags="PositionProportional" Text="Overlay" />
</AbsoluteLayout>
```

Use it for overlays, layered UI, or pixel-precise placement (a floating action button, a watermark). It's the least responsive layout — coordinates don't adapt automatically — so reserve it for genuinely positional UI, not general structure.

---

## ScrollView and content sizing

When content can exceed the screen, wrap it in a **`ScrollView`** (scrolls in one direction) so it scrolls instead of clipping. But **don't** wrap a `CollectionView`/`ListView` in a `ScrollView` — those scroll themselves and virtualize; nesting them defeats virtualization and breaks scrolling. Give scrolling list controls a bounded space (a Grid `*` cell) directly.

```xml
<ScrollView>
    <VerticalStackLayout>...long form...</VerticalStackLayout>
</ScrollView>
```

---

## Choosing a layout

```
Two-dimensional structure (rows AND columns)?   → Grid
Simple one-direction line of a few controls?     → VerticalStackLayout / HorizontalStackLayout
Items that should wrap/reflow by width?          → FlexLayout
Overlays / precise coordinates?                  → AbsoluteLayout
A long scrollable list of data items?            → CollectionView (not a stack in a ScrollView)
```

Real screens nest these: a Grid for overall structure, stacks inside cells, a CollectionView in the main content cell. Favor **Grid** for structure and the **single-orientation stacks** for simple lines; reach for FlexLayout/AbsoluteLayout for their specific cases.

---

## Common gotchas

### CollectionView/ListView inside a StackLayout or ScrollView

Stacks and ScrollViews give infinite space, defeating the list's virtualization and breaking scroll. Put scrolling list controls in a bounded Grid `*` cell, not inside a stack/ScrollView.

### Overusing nested StackLayouts

Deeply nested stacks for 2D layouts are inefficient and hard to maintain. Use a single Grid for two-dimensional structure instead of stacks-in-stacks.

### Forgetting `*` for responsive sizing

All-`Auto`/fixed rows don't grow with the screen. Use a `*` row/column for the content that should absorb extra space — that's what makes the layout adapt.

### AbsoluteLayout for general structure

Absolute coordinates don't adapt across screen sizes/orientations. Reserve AbsoluteLayout for overlays/positional UI; use Grid/Flex for adaptive structure.

### Confusing `LayoutFlags` proportional vs absolute

`AbsoluteLayout` bounds can be proportional (0–1) or absolute units depending on `LayoutFlags`. Mixing them up places things in the wrong spot — set flags deliberately.

---

## Summary

- **Layouts** arrange children: **Grid** (rows/columns — the responsive workhorse, with `Auto`/`*`/fixed sizing and `Grid.Row`/`Column` attached properties), **VerticalStackLayout/HorizontalStackLayout** (simple lines), **FlexLayout** (wrapping/flexible, Flexbox-style), **AbsoluteLayout** (explicit coordinates for overlays).
- Use **`*`** (star) sizing in Grid to absorb extra space — the key to responsive layouts; nest a Grid for structure with stacks inside cells.
- **Never** wrap a virtualizing list (`CollectionView`/`ListView`) in a `ScrollView` or stack — it breaks virtualization/scroll; give it a bounded Grid `*` cell.
- Wrap overflowing **static** content in a **`ScrollView`**; reserve **AbsoluteLayout** for overlays/precise placement (it doesn't adapt).
- Pick by shape: 2D → Grid, line → stack, wrap → Flex, overlay → Absolute, data list → CollectionView.

→ Next: [04-Controls.md](04-Controls.md)
