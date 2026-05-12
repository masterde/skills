---
name: rtl-arabic-nuxt
description: Add RTL (right-to-left) layout, Arabic typography, logical CSS properties, and bidirectional text handling to a Nuxt UI / Tailwind CSS project. Mandatory for any XYZ surface targeting the GCC region. Use when adding RTL support, Arabic UI, bidi, internationalization for Arabic, or saying "make this rtl", "add arabic", "support gcc users", "rtl nuxt", "arabic typography".
---

# RTL + Arabic for Nuxt UI / Tailwind

Retrofitting RTL onto an LTR-first design is painful. Doing it on day one is mostly mechanical. This skill encodes both paths so XYZ ventures targeting GCC users (which is most of them) get first-class Arabic support.

## What "RTL support" actually means

It's three separate concerns, frequently conflated:

1. **Direction** — `dir="rtl"` on `<html>` so the browser mirrors layout, scrollbars, and form controls.
2. **Logical CSS** — using `margin-inline-start` / `padding-inline-end` / `start`/`end` instead of `left`/`right`, so layouts auto-mirror.
3. **Typography** — loading an Arabic typeface, setting font-family fallbacks, and tuning line-height for Arabic's vertical metrics.

Get any one wrong and the UI feels off, even if the others are right.

## Process

### 1. Decide scope

Ask the user:

- **Bilingual or Arabic-only?** Most XYZ products are bilingual (English + Arabic). RuleModel may be English-only; confirm per product.
- **User-chooseable or auto-detect?** Prefer user-chooseable via a language switcher; auto-detect from `Accept-Language` only as a default.
- **Where does the language preference persist?** Local storage for unauth users; user profile column once signed in.

### 2. Install i18n

```bash
pnpm add @nuxtjs/i18n
```

`nuxt.config.ts`:
```ts
export default defineNuxtConfig({
  modules: ["@nuxt/ui", "@nuxtjs/i18n"],
  i18n: {
    defaultLocale: "en",
    locales: [
      { code: "en", language: "en-US", dir: "ltr", name: "English", file: "en.json" },
      { code: "ar", language: "ar-BH", dir: "rtl", name: "العربية", file: "ar.json" },
    ],
    strategy: "prefix_except_default",
    detectBrowserLanguage: {
      useCookie: true,
      cookieKey: "i18n_redirected",
      redirectOn: "root",
    },
  },
});
```

`ar-BH` (Arabic — Bahrain) is the right BCP-47 tag for XYZ's primary market. For KSA-targeted products via `xyz.sa`, use `ar-SA` instead.

### 3. Set `dir` and `lang` on `<html>`

In `app.vue` (or a layout):

```vue
<script setup lang="ts">
const { locale, locales } = useI18n();
const current = computed(() =>
  (locales.value as any[]).find((l) => l.code === locale.value),
);
useHead({
  htmlAttrs: {
    lang: () => current.value?.language ?? "en",
    dir: () => current.value?.dir ?? "ltr",
  },
});
</script>
```

This is what flips the browser into RTL mode. Most layout mirroring happens automatically once `dir="rtl"` is set.

### 4. Use logical CSS in Tailwind

**The golden rule:** never write `ml-4` / `pr-2` / `left-0` / `text-left` again. Always use the logical equivalents:

| Physical (avoid) | Logical (use) |
|---|---|
| `ml-4` | `ms-4` (margin-inline-start) |
| `mr-4` | `me-4` (margin-inline-end) |
| `pl-2` | `ps-2` (padding-inline-start) |
| `pr-2` | `pe-2` (padding-inline-end) |
| `left-0` | `start-0` |
| `right-0` | `end-0` |
| `text-left` | `text-start` |
| `text-right` | `text-end` |
| `border-l` | `border-s` |
| `rounded-tl-lg` | `rounded-ss-lg` (start-start) |

Tailwind v3.3+ supports all of these natively. Nuxt UI v4 components already use logical properties internally, so they mirror automatically.

To enforce this convention, add an ESLint rule (or a simple grep in CI):
```bash
grep -rn "\\b\\(ml-\\|mr-\\|pl-\\|pr-\\|left-\\|right-\\|text-left\\|text-right\\)" app/ pages/ components/ && exit 1
```

### 5. Arabic typography

Don't ship `font-family: sans-serif` for Arabic. Pick one:

| Font | When to use |
|---|---|
| **IBM Plex Sans Arabic** | Best general-purpose; pairs well with IBM Plex Sans for bilingual interfaces |
| **Noto Sans Arabic** | Free, comprehensive Unicode coverage; the safe default |
| **Cairo** | Modern Latin-influenced shapes; good for marketing surfaces |
| **Tajawal** | Friendly, casual products (e.g., Orderly-style consumer apps) |

Load via `@nuxt/fonts` (preferred) or Google Fonts. Example:

```ts
// nuxt.config.ts
modules: ["@nuxt/fonts"],
fonts: {
  families: [
    { name: "IBM Plex Sans", provider: "google" },
    { name: "IBM Plex Sans Arabic", provider: "google" },
  ],
},
```

Then in CSS:
```css
:root {
  --font-sans: "IBM Plex Sans", system-ui, sans-serif;
  --font-sans-arabic: "IBM Plex Sans Arabic", system-ui, sans-serif;
}

html[lang^="ar"] body {
  font-family: var(--font-sans-arabic);
  line-height: 1.7; /* Arabic generally wants more vertical breathing room */
}
```

### 6. Bidi text handling

For mixed Arabic + English content (extremely common — product names, code, URLs), wrap explicit-direction spans:

```vue
<p :dir="locale === 'ar' ? 'rtl' : 'ltr'">
  <span dir="ltr">Dealmatter.co</span> {{ $t("welcome_message") }}
</p>
```

For user-generated content, set `dir="auto"` on the container — the browser auto-detects per paragraph.

### 7. Icons + chevrons

Directional icons (back arrows, chevrons, ordered-list bullets) need to flip in RTL. Most icon libraries don't do this automatically.

```vue
<UIcon :name="locale === 'ar' ? 'i-lucide-chevron-left' : 'i-lucide-chevron-right'" />
```

Or with a CSS class:
```css
html[dir="rtl"] .flip-rtl {
  transform: scaleX(-1);
}
```

Don't flip non-directional icons (search, settings, user) — that makes them look broken.

### 8. Numbers and dates

- **Numbers:** Arabic-Indic digits (`٠١٢٣٤٥٦٧٨٩`) are rare in the GCC; most users prefer Western Arabic numerals (`0-9`). Stick with the default unless the product specifically targets a market that prefers Arabic-Indic.
- **Dates:** Use `Intl.DateTimeFormat(locale)`. Bahrain uses the Gregorian calendar in business contexts; show Hijri only when the product is religious/cultural.
- **Currency:** `Intl.NumberFormat("ar-BH", { style: "currency", currency: "BHD" })` for BHD; `currency: "SAR"` for KSA, `"AED"` for UAE.

### 9. Test the result

- Toggle the language switcher → entire layout mirrors, fonts swap, content reads correctly
- Form inputs accept Arabic text; placeholders read right-to-left
- A mixed-direction paragraph (English brand name + Arabic copy) renders cleanly
- Chevrons / back arrows point the correct way per language
- No `ml-`/`mr-`/`left-`/`right-` Tailwind classes remain (CI grep is clean)

## Common mistakes

- **Setting `dir="rtl"` only on `<body>`** — scrollbars and form-control mirroring need it on `<html>`.
- **Mirroring icons that shouldn't mirror** — only directional icons flip; logos, search, settings, user, calendar do not.
- **Using `transform: scaleX(-1)` on text** — this mirrors the glyphs themselves, making text unreadable. Use it only on directional icons.
- **Hard-coding Latin font for headings** — Arabic users hate seeing their language rendered in a font that doesn't have proper Arabic glyphs; the browser falls back ugly.
- **Right-aligning all text** — `text-align: right` for Arabic users is correct *only when* `dir="rtl"` is set. Use `text-start` / `text-end`, not `text-left` / `text-right`.

## When NOT to use this skill

- The product is English-only and will never expand (rare for XYZ — confirm before assuming).
- Adding a second non-Arabic LTR language (French, German) — i18n setup applies but RTL/typography sections don't.
