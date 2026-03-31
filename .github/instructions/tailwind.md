# Tailwind CSS 4 — Usage Conventions

## Core Principle

Utility-first. Write Tailwind classes in templates. Avoid custom CSS.
Dark theme is the default in this project.

## Class Ordering

Follow this order in class attributes for consistency:

1. Layout (`flex`, `grid`, `block`, `hidden`)
2. Positioning (`relative`, `absolute`, `sticky`)
3. Sizing (`w-`, `h-`, `min-`, `max-`)
4. Spacing (`p-`, `m-`, `gap-`)
5. Typography (`text-`, `font-`, `leading-`)
6. Visual (`bg-`, `border-`, `rounded-`, `shadow-`)
7. Interactive (`hover:`, `focus:`, `active:`)
8. Responsive (`sm:`, `md:`, `lg:`, `xl:`)
9. Dark mode (`dark:`)

```html
<!-- Ordered example -->
<div class="flex items-center w-full p-4 text-sm text-gray-700 bg-white rounded-lg shadow-sm hover:shadow-md dark:bg-gray-800 dark:text-gray-200">
```

## Responsive Design

Mobile-first approach. Base styles are mobile, layer up:

```html
<!-- Mobile: stack, Tablet: side-by-side, Desktop: wider gap -->
<div class="flex flex-col gap-2 md:flex-row md:gap-4 lg:gap-6">
```

## Dark Mode

Use `class` strategy (configured in `tailwind.config.js`):

```html
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-100">
```

Always provide dark mode variants for:
- Background colors
- Text colors
- Border colors
- Shadow colors (use `dark:shadow-none` or dark shadow variants)

## Design Tokens

All project-specific values go in `tailwind.config.js`:

```javascript
// tailwind.config.js
export default {
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#f5f3ff',
          500: '#8b5cf6',
          900: '#4c1d95',
        },
      },
    },
  },
}
```

Never hardcode hex colors in templates. Use config tokens.

## When @apply Is Acceptable

Only in component `<style scoped>` blocks for:
- Complex pseudo-element styling
- Third-party component overrides
- Animations that need keyframes

```vue
<style scoped>
.custom-scrollbar::-webkit-scrollbar-thumb {
  @apply bg-gray-300 rounded-full hover:bg-gray-400;
}
</style>
```

## Rules

- No custom CSS files unless wrapping third-party styles
- No inline `style` attributes
- No hardcoded color values — always reference config tokens
- Use `@tailwindcss/typography` for prose/markdown content
- Use `@tailwindcss/forms` for form element resets
- Max class string length: if a class list exceeds ~120 chars, extract to a computed or use a wrapper component
