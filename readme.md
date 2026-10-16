# Astro Fakestore

We would like to apologize if you were kept awake by the fact that we created a Fake Store site
(recap exercise) which heavily depended on client-side javascript.

Since then we've learned a lot, and we would like to make it up to you: the same Fake Store,
but this time with Astro. No client-side javascript needed.

## What's in this folder

This is a real Astro project — it installs and builds as-is. The boring parts are done for you;
the parts that teach you something are marked with a `TODO`.

```text
src/
├── data/products.json          ✅ given — the product data (from fakestoreapi.com)
├── styles/reset.css            ✅ given
├── styles/global.css           ✅ given — custom properties, body styling
├── layouts/BaseLayout.astro    🔨 head + font are given, the wiring is yours
├── components/
│   ├── Header.astro            🔨 CSS given, markup is yours
│   ├── ProductsList.astro      🔨 CSS given, script + markup are yours
│   ├── ProductCard.astro       🔨 CSS given, data binding is yours
│   └── PrevNext.astro          ✅ given — finished, you only pass it props
└── pages/index.astro           🔨 yours
```

Every `.astro` component keeps its own CSS in a `<style>` tag. Those styles are **scoped**:
the `h1` rule in `Header.astro` cannot leak into `ProductCard.astro`. Only `global.css` and
`reset.css` are site-wide, and they are imported once — in the layout.

## Getting started

1. Copy this folder to wherever you keep your projects, and rename it to `fakestore`.
2. `git init`, then commit. Create a repo on GitHub and push.
3. `npm install`, then `npm run dev`.

You should see a mostly empty page. Good. That's the starting point.

### The PR ritual

Every section below is one feature branch. Each time you finish a section:

```sh
git switch -c feature/<name>   # at the start of the section
# ... work, commit as you go ...
git push -u origin feature/<name>
# open a PR on GitHub, merge it
git switch main && git pull
```

Practice makes perfect!

## 1. Layout — `feature/layout`

Open `src/layouts/BaseLayout.astro`. The HTML shell and the Google Font are already there.
Your job:

- Import `reset.css` and `global.css`. Importing them here means every page that uses this
  layout gets them — that is the whole reason a layout exists.
- Make the page title settable from outside, via [props](https://docs.astro.build/en/basics/astro-components/#component-props).
  Something like `Fake Store - {pageTitle}`.
- Put a `<slot />` inside the `<main>`. That is where a page's content lands.

Then use the layout in `src/pages/index.astro` and pass it a `pageTitle`.

## 2. Header — `feature/header`

`src/components/Header.astro` has its CSS but no markup. Add a `<h1>` containing a link to `/`
with the text *Fake Astro Store*, and render the component in the layout — above the `<main>`.

## 3. The products collection — `feature/products`

Let us assume the product data changes rarely, which makes it a perfect
[content collection](https://docs.astro.build/en/guides/content-collections/). The JSON is
already in `src/data/products.json`.

Create `src/content.config.js` and define a `products` collection. Because the source is a JSON
file, use the [`file()` loader](https://docs.astro.build/en/guides/content-collections/#built-in-loaders).

Give it a [Zod schema](https://docs.astro.build/en/guides/content-collections/#defining-datatypes-with-zod)
so a typo in the data fails the build instead of the page. Implement this — it's not that hard:

```js
import { defineCollection } from "astro:content";
import { file } from "astro/loaders";
import { z } from "astro/zod"; // not from "astro:content": deprecated since Astro 6, removed in 8
```

```js
schema: z.object({
  id: z.number(),
  title: z.string(),
  price: z.number(),
  description: z.string(),
  category: z.string(),
  image: z.string(),
  rating: z.object({
    rate: z.number(),
    count: z.number(),
  }),
}),
```

Now use it. In `src/components/ProductsList.astro`, fetch the entries with
[`getCollection`](https://docs.astro.build/en/guides/content-collections/#querying-collections)
and loop over them in the template. **For now, just render the title.**

> Watch out: entries come back [wrapped](https://docs.astro.build/en/guides/content-collections/#using-content-in-astro-templates)
> in an object with a `data` property. The title is `product.data.title`, not `product.title`.

Render `ProductsList` on the index page.

## 4. Product card — `feature/product-card`

We could build the card inside the list, but a separate component is reusable — and we will
reuse it on the detail page in a minute.

`src/components/ProductCard.astro` has its CSS and a markup skeleton full of placeholders.

- Destructure `product` from `Astro.props`.
- Replace the placeholders with real data.
- Add the class `popular` to the `<article>`, but **only** when `product.rating.rate > 4.5`.
  The CSS for the corner ribbon is already there, waiting for that class.
  Use [`class:list`](https://docs.astro.build/en/reference/directives-reference/#classlist), not
  `class={rate > 4.5 && "popular"}`: the second one writes `class="false"` into the HTML of every
  product that is not popular.

Then swap the plain `<li>` in `ProductsList` for the card:

```jsx
<ul>
  {
    products.map((product) => (
      <li>
        <ProductCard product={product.data} />
      </li>
    ))
  }
</ul>
```

Note `product.data` — the card wants the product, not the collection entry around it.

Et voilà, the index page is done.

## 5. Detail page — `feature/product-detail`

One page per product, generated at build time. No server, no fetch, no spinner.

- Create `src/pages/products/[id].astro`.
- Use [dynamic routes in SSG mode](https://docs.astro.build/en/guides/routing/#static-ssg-mode):
  inside `getStaticPaths`, get all products and return one object per product.

  ```js
  return products.map((product) => ({
    params: { id: product.data.id.toString() },
    props: { product: product.data },
  }));
  ```

  Whatever you put in `params` becomes the URL. Whatever you put in `props` is handed to the
  page — so the page never has to look the product up again.

- Read the product from `Astro.props`, wrap the page in `BaseLayout` and render the
  `ProductCard`. Don't worry about how it looks yet.
- Finally, in `ProductsList`, wrap each card in a link so the index actually reaches the detail:

  ```jsx
  <a href={`/products/${product.data.id}`}>
    <ProductCard product={product.data} />
  </a>
  ```

## 6. Paging — `feature/paging`

`src/components/PrevNext.astro` is finished. It expects three props: `prev`, `next` and `path`.
It takes a `path` rather than hardcoding `/products/` so you can reuse it for any collection.

The work is in `getStaticPaths`: alongside `product`, hand the page its neighbours. You will
need the index of each product, so map with `(product, i)`.

```js
next: i < products.length - 1 ? products[i + 1].data : null,
prev: i > 0 ? products[i - 1].data : null,
```

Add `PrevNext` to the detail page with `path="products"`, and check that you can walk forwards
and backwards through the whole catalogue — and back to the index.

## Done?

Run `npm run build`. It should exit without errors and produce one `index.html` plus one page
per product in `dist/`. Open `dist/products/1/index.html` in a text editor: the product data is
sitting there as plain HTML. That is static site generation.

If you want to compare, `../fakestore-final` holds a worked-out version.
