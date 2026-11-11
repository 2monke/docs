---
status: shipped
collections: [media, pages, products]
features: []
---

# Admin form layout & density — field rows

Documentation forms in the admin used to stack every field full width. A five-field
product (title, type, price, description, images + files) therefore filled a screen
of mostly-empty lines: a price input ran the entire width of the column, and fields
that are read together (name and type) sat far apart.

This is the convention for putting related fields on one line, plus the mechanics
that make it work in this codebase.

## The mechanism

Payload's `type: 'row'` renders its children side by side and sizes each child from
its own `admin.width` via the `--field-width` custom property. Two behaviours come
for free and are relied on here:

- **Rows stack on narrow screens.** Below Payload's `mid-break` (1024px) every row
  becomes a block: full-width, one field per line, exactly as before this pass.
- **Widths are shares of the row**, i.e. of the form's *main column* — not of the
  viewport. `third` is a third of the column, which is already narrower than the
  screen; treat the numbers as ratios, not as screen fractions.

## The vocabulary

`src/lib/utils/admin-field-layout.ts` owns the words and the one non-obvious rule:

```ts
FIELD_WIDTH = { quarter, third, half, twoThirds, threeQuarters, full }
```

Use these constants instead of raw percentages so every screen sizes fields from the
same short list instead of inventing `'37%'`.

### The custom-component rule

Payload's *native* field components apply `admin.width` themselves
(`mergeFieldStyles`). A field with a custom `components.Field` bypasses that
machinery, so inside a row it collapses to its content width unless the component
puts `--field-width` on its own root:

```tsx
import { fieldWidthStyle } from '@/lib/utils/admin-field-layout'

return <div className="ap-field" style={fieldWidthStyle(field)}>…</div>
```

`fieldWidthStyle` mirrors Payload's own logic (width → `--field-width`, otherwise
`flex: 1 1 auto`, with an explicit `style.flex` winning). **A field with a custom
component inside a row must declare a width** — `tests/int/admin-form-layout.int.spec.ts`
fails otherwise, because the failure mode is silent (the row just looks wrong).

Also: a custom control's own `maxWidth` cap (if it has one) has to go once the field
lives in a row — the row is what governs the width now (`ProductTypeSelectField`
dropped its `27.1rem` cap for exactly this reason).

### The native-furniture rule (for controls that replace a built-in field)

A custom *control* inside a document form still has to look like the fields around it.
Two parts:

- **Wear Payload's markup.** Use `FieldLabel`, `FieldError`, `FieldDescription` and a
  `field-type` root (as `ProductTypeSelectField` does). That gives the field Payload's
  label typography, its error line, its description (which the help-tooltip enhancer
  then collapses into the ℹ beside the label), and Payload's field rhythm —
  `.render-fields > .field-type { margin-bottom: var(--base) }`.
- **Wear Payload's control recipe.** A bare `<select>` renders in the OS font with the
  OS arrow and a native bezel; in this admin that read as *completely different* from
  the title input next to it. `.ap-select` therefore strips `appearance`, draws its own
  chevron, and borrows the input's typography (body font, weight 400, `formInput()`'s
  padding rhythm) on top of the shared control frame.

Measured parity (Chromium, enabled state, title input vs product-type select):
height, border (colour + width), radius, background, text colour, font family, size and
weight all identical; the only intended difference is the select's larger right padding
for its chevron. Text sits within 0.5px of the same vertical centre in both.
The parity is asserted end-to-end (`tests/e2e/admin-form-density.e2e.spec.ts`) on
*computed* styles, so a future reset cannot silently bring the browser chrome back —
jsdom cannot see it, only a real engine can.

## Products — the first screen (2026-10-08)

| Line | Fields | Widths |
| --- | --- | --- |
| 1 | title \| product type | `twoThirds` \| `third` |
| 2 | price | `third`, alone on the line |
| 3 | description | full width (rich text, its own line) |
| 4 | images \| digital files | images: no width (fills), digital files: `half` |

Deliberate details:

- **price is 1/3, alone** — a short number does not need the column; keeping it in a
  one-child row is what makes the width apply at all (outside a row, `admin.width`
  has no consumer).
- **images carries no width** so that when `digitalFiles` is hidden (bundles) the
  artwork field fills the line instead of leaving half of it empty. On a normal
  product the two are ~half and half.
- **the reading order is unchanged** — title, type, price, description, images,
  files — because the visible order of a form is muscle memory. Widths may not
  reorder a form; the spec pins the flattened order.
- **conditions kept** — `bundleProducts` (bundles only) stays full width between
  type and price; `digitalFiles` still hides for bundles.

## Rolling it out to another screen

1. List the fields and pair the ones that are read together (identity, numbers,
   media). Keep full width for anything long-form (rich text, arrays, blocks).
2. Wrap pairs in `type: 'row'` with `FIELD_WIDTH` values; leave the form's order
   alone.
3. Give every custom component in a row a width **and** `fieldWidthStyle` on its root.
4. Add a case to `tests/int/admin-form-layout.int.spec.ts` for the screen: the rows,
   their widths, their conditions, and the flattened order.
5. Check the live form at 1440px (one line, sane ratios) and at ≤1024px (stacked).

Natural next candidates, in rough order of payoff: Pages (title \| slug), Commission
Settings, Theme editor (already grouped, but its top fields are one-per-line), and
the Globals.

## Verification

- `tests/int/admin-form-layout.int.spec.ts` — the width contract, the Products rows,
  reading order, conditions, the custom-component rule, and the native-furniture rule
  (14 cases).
- The sanitized config of the running app was inspected directly (Payload's own
  `payload.collections.products.config`): three rows, `title @66%`, `productType @33%`,
  `price @33%`, `digitalFiles @50%`.
- Live measurement of the rendered form (Chromium, seeded artist): three rows, each on
  one line at 1440px (`title`/`productType` ≈ 2:1, `price` a third, media pair ≈ 1:1);
  all rows stack at 900px; the product-type select is exactly its 199px cell.
- Field parity measured live as described above; pinned by the e2e spec's computed
  style comparison.
