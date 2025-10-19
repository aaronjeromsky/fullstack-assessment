# Stackline Full Stack Assignment

**Author**: Aaron Jeromsky

**Date**: Oct 19, 2025

## Bug Fixes

### 1. Unspecified `yarn.lock` Directory

**Issue:** The `outputFileTracingRoot` was not explicitly configured in `next.config.ts`, which could cause Next.js to trace dependencies from an incorrect directory during build. This might lead to missing or outdated packages being included in the production bundle.

**Fix:** Explicitly set `outputFileTracingRoot` in `next.config.ts` to ensure the project root directory is used:

```js
import path from 'path';

const nextConfig: NextConfig = {
  outputFileTracingRoot: path.join(__dirname),
};
```

**Why:** Ensures Next.js traces dependencies from the correct project root and prevents unexpected behavior in CI/CD or deployment environments.

### 2. Unconfigured Image Host Causing Broken Images

**Issue:** Next.js blocked images from `images-na.ssl-images-amazon.com` due to missing configuration, causing "Un-configured Host" errors and broken product images.

**Fix:** Added `images-na.ssl-images-amazon.com` hostname to `remotePatterns` in `next.config.ts`:

```js
images: {
  remotePatterns: [
    {
      protocol: 'https',
      hostname: 'images-na.ssl-images-amazon.com',
    }
  ]
}
```

**Why:** Follows Next.js security requirements for remote images. Without this, production builds would fail to load product images.

See: https://nextjs.org/docs/messages/next-image-unconfigured-host

### 3. Runtime Crashes from Missing `imageUrls`

**Issue:** `product.imageUrls` was sometimes `undefined`, causing fatal UI crashes when accessing `imageUrls[0]` during rendering. This was visible in the product listings page.

**Fix:** Normalized product data at fetch time to ensure safe array access:

```js
const normalizedProducts = data.products.map(product => ({
  ...product,
  imageUrls: product.imageUrls || []
}));
```

**Why:** This fix addresses the root cause at the data source rather than patching individual components to prevent a build up of technical debt and future errors stemming from `undefined` product data. This maintains data integrity by providing an empty array as a fallback (since no product image is available).

### 4. Category Selection State Not Resetting Properly

**Issue:** "Clear Filters" button failed to reset category states to empty strings, leaving stale `undefined`` values that prevented UI updates.

**Fix:** Explicitly reset all filter states to empty strings when clicking "Clear Filters" button:

```js
onClick={() => {
  setSearch("");
  setSelectedCategory("");
  setSelectedSubCategory("");
}}
```

**Why:** Using empty strings ensures consistent state management across components. This aligns with React best practices and prevents conditional rendering bugs.


## Improvements

### 1. Smart Subcategory Management

**enhancement:**
- Automatically hide empty subcategories from drop down list
- Reset subcategory when changing main category

**Implementation:**

```js
if (selectedCategory) {
  fetch(`/api/subcategories?category=${selectedCategory}`)
    .then((res) => res.json())
    .then((data) => setSubCategories(data.subCategories));
} else {
  setSubCategories([]);
  setSelectedSubCategory("");
}
```

**Why:**
- Prevent empty results by removing irrelevant category filters
- Implementation requires minimal client-side processing since it's handled at the API level

### 2. Contextual Category Navigation

**Enhancement:** Clicking category/subcategory badges now filters results for that (sub)category.

**Implementation:** Added click handlers to product card badges:

```js
<Badge
  variant="secondary"
  className="cursor-pointer hover:bg-secondary/80 transition-colors"
  onClick={() => handleCategoryClick(product.categoryName)}
>
  {product.categoryName}
</Badge>
```

**Why:** This improves discoverability by allowing users to navigate from the product card instead of only the filter drop down.

### 3. Dedicated Product Detail Button

**Enhancement:** Only "View Details" button navigates to the product page, previously the entire card was clickable.

**Implementation:** Removed click handler from product card wrapper, refactored "View Details" button:

```js
<Link
  href={{
    pathname: "/product",
    query: { product: JSON.stringify(product) },
  }}
  className="w-full"
>
  <Button variant="outline" className="cursor-pointer w-full">
    View Details
  </Button>
</Link>
```

**Why:** Prevents accidental navigation when users interact with other product card elements.

### 4. Visual Feedback for Interactive Elements

**Enhancement:** Added hover states and cursor indicators to interactive elements.

**Implementation:**

```js
<Badge
  variant="outline"
  className="cursor-pointer hover:bg-accent transition-colors"
  onClick={() => setSelectedSubCategory(product.subCategoryName)}
>
  {product.subCategoryName}
</Badge>
```

**Why:** Allows for predictable behavior by providing visual indicators for users.

### 5. Clean Product SKU URLs

**Enhancement:** Replaced product URL parameters from full JSON objects to single `sku` identifiers.

**Implementation:** Modified route definition and link generation to use SKU only:

```js
<Link
  href={{
    pathname: "/product",
    query: { sku: product.stacklineSku },
  }}
  className="w-full"
>
  <Button variant="outline" className="cursor-pointer w-full">
    View Details
  </Button>
</Link>
```

**Why:**
- Prevents URL parameter tampering by eliminating exposure of sensitive product data (e.g., pricing, inventory) in client-side URLs, improving security
- Creates human-readable, shareable links (e.g., `/product?sku=ABC123` vs. long encoded JSON strings), improving user experience
- Decouples URLs from product data structure, ensuring links remain valid when product fields change, allowing for easier maintainability

## Future Improvements

- **Consistent subcategory behavior:** Fix subcategory selection being hidden in some categories (e.g., Game Consoles & Accessories) or unnecessarily shown (e.g., when all products have the same subcategory)
- **Independent subcategory selection:** Allow subcategory selection without requiring a main category (requires changes to rendering logic on product listing page)