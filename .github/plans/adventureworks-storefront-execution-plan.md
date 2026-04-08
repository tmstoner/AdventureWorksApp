# AdventureWorks Storefront Execution Plan

## Summary

This plan evolves the existing .NET 8 Aspire starter solution into a storefront application backed by the full AdventureWorks database. The backend remains the single database-facing service and exposes REST APIs for product catalog, product images, categories, and shopping cart operations. The frontend is embedded into the existing `AdventureWorksApp.Web` project as a React application served by ASP.NET Core in production and integrated with a Vite development workflow locally. All products are treated as purchasable, product images are served from binary data stored in the database, and cart totals always reflect the current database price.

## Architecture Targets

1. Keep `AdventureWorksApp.AppHost` as the local orchestration entry point.
2. Preserve the Aspire service names `apiservice` and `webfrontend`.
3. Keep cross-service health, resilience, and telemetry defaults in `AdventureWorksApp.ServiceDefaults`.
4. Turn `AdventureWorksApp.ApiService` into the REST backend for catalog, image, category, and cart features.
5. Replace the current Blazor UI in `AdventureWorksApp.Web` with an embedded React frontend.
6. Keep all database access in the API service only.

## Data And Domain Decisions

### Product Catalog

1. Use the full AdventureWorks product catalog.
2. Show all products as purchasable.
3. Use the current AdventureWorks `ListPrice` as the authoritative storefront price.
4. Do not block products based on inventory, sell dates, or discontinued-style flags in v1.

### Product Images

1. Serve images directly from the database through API endpoints.
2. Use one deterministic primary image per product.
3. Select the primary image as the associated photo with the lowest `ProductPhotoID` for that product.
4. Use the primary image's thumbnail binary column for catalog thumbnails.
5. Use the primary image's full-size binary column for product detail pages.
6. If a product has no associated photo, return `null` image URLs in catalog responses and let the frontend render a placeholder.

### Shopping Cart

1. Add app-owned tables for carts and cart items under a dedicated schema such as `app`.
2. Support anonymous carts first, with future-ready columns for authenticated ownership.
3. Store product identity and quantity in the cart, but do not store a frozen item price in v1.
4. Recalculate line totals and cart totals from current product prices on every cart read and mutation.
5. Defer checkout, tax, shipping, and promotions to a later phase.

## Backend Execution Steps

1. Inspect the AdventureWorks schema through the MSSQL MCP service and confirm the exact joins for:
   - product core data
   - category and subcategory names
   - product descriptions
   - product photos and their binary columns
2. Replace the weather sample endpoint in `AdventureWorksApp.ApiService` with route groups under:
   - `/api/products`
   - `/api/categories`
   - `/api/cart`
3. Add a data-access layer in the API service that isolates SQL queries and maps them into storefront DTOs.
4. Create product list and product detail DTOs that expose only storefront-relevant data.
5. Add image endpoints:
   - `GET /api/products/{productId}/thumbnail`
   - `GET /api/products/{productId}/image`
6. Ensure image endpoints return raw bytes with the correct content type if it can be derived from the data source; otherwise use a consistent fallback content type based on the stored format.
7. Add category endpoints that support storefront navigation and filtering.
8. Add cart endpoints:
   - `POST /api/cart`
   - `GET /api/cart/{cartId}`
   - `POST /api/cart/{cartId}/items`
   - `PATCH /api/cart/{cartId}/items/{itemId}`
   - `DELETE /api/cart/{cartId}/items/{itemId}`
   - `DELETE /api/cart/{cartId}`
9. Create app-owned database objects for cart persistence.
10. Add request validation, problem-details responses, and OpenAPI generation for the REST surface.
11. Add a database health check to the API service while preserving the existing Aspire health endpoint pattern.

## Frontend Execution Steps

1. Keep `AdventureWorksApp.Web` as the single hosted frontend project in the solution.
2. Remove the current Blazor-specific application structure and replace it with a React app embedded inside the web project.
3. Place the React source in a dedicated subfolder inside `AdventureWorksApp.Web`, managed by Vite.
4. Configure the ASP.NET Core host to:
   - proxy to the Vite dev server in development
   - serve built static assets in production
   - preserve same-origin API calls to `/api`
5. Build the React app in TypeScript.
6. Create frontend routes for:
   - catalog page
   - product detail page
   - cart page
7. Build catalog UI features for:
   - search
   - category and subcategory filtering
   - sorting
   - paging
   - loading, empty, and error states
8. Build product detail UI features for:
   - full-size image display
   - product metadata
   - current price display
   - quantity selection
   - add-to-cart action
9. Build cart UI features for:
   - line item list
   - quantity updates
   - remove item
   - empty cart state
   - totals that always refresh from the API response
10. Render placeholder imagery when a product does not have a primary photo.

## API Contract Outline

### Catalog

1. `GET /api/products`
   - Supports `search`, `categoryId`, `subcategoryId`, `page`, `pageSize`, `sort`, `minPrice`, and `maxPrice` query parameters.
2. `GET /api/products/{productId}`
   - Returns the detailed product model and image URLs.
3. `GET /api/categories`
   - Returns categories with optional subcategory data for filtering.

### Images

1. `GET /api/products/{productId}/thumbnail`
   - Returns the primary image thumbnail bytes.
2. `GET /api/products/{productId}/image`
   - Returns the primary full-size image bytes.

### Cart

1. `POST /api/cart`
   - Creates a cart and returns the cart identifier.
2. `GET /api/cart/{cartId}`
   - Returns cart contents with live-priced totals.
3. `POST /api/cart/{cartId}/items`
   - Adds a product or increments quantity if it already exists.
4. `PATCH /api/cart/{cartId}/items/{itemId}`
   - Updates quantity and returns recalculated totals.
5. `DELETE /api/cart/{cartId}/items/{itemId}`
   - Removes an item and returns the updated cart.
6. `DELETE /api/cart/{cartId}`
   - Clears or deletes the cart.

## Database Plan

1. Keep AdventureWorks product, category, description, and photo data as the source of truth for the catalog.
2. Add a dedicated app schema for write-owned storefront data.
3. Create a minimal cart persistence model:
   - `app.Carts`
   - `app.CartItems`
4. Include future-ready fields for authenticated ownership, such as an optional user identifier on the cart.
5. Keep current pricing live by joining cart items back to AdventureWorks product data instead of storing item price snapshots.

## Testing Plan

1. Preserve the existing Aspire integration-test model in `AdventureWorksApp.Tests`.
2. Replace the current root-page smoke test coverage with broader distributed tests for:
   - web root responds successfully
   - product list endpoint returns expected shape
   - product detail endpoint returns expected shape
   - thumbnail endpoint returns image content
   - full-image endpoint returns image content
   - cart creation works
   - add-to-cart works
   - quantity update recalculates totals
   - item removal recalculates totals
3. Add tests for products without photos to verify placeholder-friendly API behavior.
4. Add tests that verify cart totals always reflect the current database price.

## Delivery Phases

1. Phase 1: Schema discovery and API contract confirmation.
2. Phase 2: Catalog, category, and image endpoints.
3. Phase 3: Cart schema and cart endpoints with live price calculation.
4. Phase 4: Embedded React shell and catalog UI.
5. Phase 5: Product detail and cart UI.
6. Phase 6: Integration tests, cleanup, and documentation.

## Edge Cases To Handle

1. Products with no photo association.
2. Products with multiple associated photos, resolved by the lowest `ProductPhotoID` rule.
3. Large full-size image payloads that should not be used on listing views.
4. Products with missing descriptions or sparse AdventureWorks metadata.
5. Cart requests where a referenced product no longer exists.
6. Cart totals changing between requests because pricing is always live.
7. SPA fallback routing accidentally swallowing `/api` or health-check routes.

## Deferred Work

1. Authentication and cart ownership by signed-in user.
2. Checkout and order submission.
3. Price snapshots at checkout time.
4. Tax, shipping, discounts, and promotions.
5. Inventory-aware availability rules.