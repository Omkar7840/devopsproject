# Quick Review Popup — Implementation Guide

## Feature Summary

When a retailer opens the **Pending** tab in My Catalog and there are pending products, a full-screen modal automatically appears asking **"Are You Selling This?"** for each product. The retailer taps **Yes** (move to Unavailable) or **No** (remove from catalog), and the next product loads instantly. **No API call is made until the user closes the popup or finishes all products**, at which point a single batched call is made. This saves significant time compared to the current one-by-one flow.

---

## Files Changed

| File | Action |
|------|--------|
| `frontend/retailer_app/src/app/pages/catelog/my-catalog/my-catalog.component.ts` | **MODIFY** — Add Quick Review logic |
| `frontend/retailer_app/src/app/pages/catelog/my-catalog/my-catalog.component.html` | **MODIFY** — Add Quick Review modal HTML |
| `frontend/retailer_app/src/app/pages/catelog/my-catalog/my-catalog.component.scss` | **MODIFY** — Add Quick Review modal styles |

> [!NOTE]
> **No new files need to be created.** All changes are additions to existing files.  
> **No backend changes are required.** The existing `insertCatalogData` and `updateCatalogData` APIs already support batched products and removed products.

---

## Step 1 — TypeScript Changes

**File:** `frontend/retailer_app/src/app/pages/catelog/my-catalog/my-catalog.component.ts`

### 1A. Add new properties

Find the line (around line 55):

```typescript
pendingOrUnpublishedProducts: Set<number> = new Set();
```

**Add the following lines BELOW it:**

```typescript
  // ── Quick Review Popup Properties ──
  quickReviewOpen: boolean = false;
  quickReviewProducts: any[] = [];
  quickReviewIndex: number = 0;
  quickReviewCurrentProduct: any = null;
  quickReviewProductsToUnpublish: any[] = [];
  quickReviewProductsToRemove: any[] = [];
  quickReviewProcessing: boolean = false;
  quickReviewJustClosed: boolean = false;
```

### 1B. Add auto-open trigger inside `segmentChanged()`

Find the `segmentChanged()` method (around line 548). At the **very end** of the method, just **before** the line `loading.dismiss();` (around line 569), add:

```typescript
      // ── Auto-open Quick Review when Pending tab is selected ──
      if (this.productStatus === 'pending' && this.filteredProducts.length > 0 && !this.quickReviewJustClosed) {
        this.openQuickReview();
      }
      this.quickReviewJustClosed = false;
```

So the end of `segmentChanged()` should look like:

```typescript
    } else {
      this.filteredProducts = [];
    }
    // ── Auto-open Quick Review when Pending tab is selected ──
    if (this.productStatus === 'pending' && this.filteredProducts.length > 0 && !this.quickReviewJustClosed) {
      this.openQuickReview();
    }
    this.quickReviewJustClosed = false;
    loading.dismiss();
  }
```

### 1C. Add all Quick Review methods

Add the following methods **at the end of the class**, just **before** the closing `}` of the class (before line 1065):

```typescript
  // ═══════════════════════════════════════════════════════
  // QUICK REVIEW POPUP — Batch review pending products
  // ═══════════════════════════════════════════════════════

  /**
   * Opens the Quick Review popup.
   * Collects all currently filtered pending products and starts the review cycle.
   */
  openQuickReview(): void {
    // Deep-clone pending products so local mutations don't affect the list until commit
    this.quickReviewProducts = this.filteredProducts
      .filter((p: any) => p.status === 'pending')
      .map((p: any) => JSON.parse(JSON.stringify(p)));

    if (this.quickReviewProducts.length === 0) {
      return;
    }

    this.quickReviewIndex = 0;
    this.quickReviewCurrentProduct = this.quickReviewProducts[this.quickReviewIndex];
    this.quickReviewProductsToUnpublish = [];
    this.quickReviewProductsToRemove = [];
    this.quickReviewProcessing = false;
    this.quickReviewOpen = true;
  }

  /**
   * Returns the display counter string, e.g. "3/25"
   */
  getQuickReviewCounter(): string {
    return `${this.quickReviewIndex + 1}/${this.quickReviewProducts.length}`;
  }

  /**
   * YES — the retailer sells this product.
   * Mark it for status change to 'unpublished' (Unavailable) and move to next.
   */
  onQuickReviewYes(): void {
    if (!this.quickReviewCurrentProduct) return;

    // Clone the product and set status to 'unpublished'
    const product = { ...this.quickReviewCurrentProduct, status: 'unpublished' };
    this.quickReviewProductsToUnpublish.push(product);

    this.advanceQuickReview();
  }

  /**
   * NO — the retailer does NOT sell this product.
   * Mark it for removal from catalog and move to next.
   */
  onQuickReviewNo(): void {
    if (!this.quickReviewCurrentProduct) return;

    this.quickReviewProductsToRemove.push(this.quickReviewCurrentProduct);

    this.advanceQuickReview();
  }

  /**
   * Advances to the next product, or commits if all products are reviewed.
   */
  private advanceQuickReview(): void {
    if (this.quickReviewIndex >= this.quickReviewProducts.length - 1) {
      // All products reviewed — commit the batch
      this.commitQuickReview();
    } else {
      this.quickReviewIndex++;
      this.quickReviewCurrentProduct = this.quickReviewProducts[this.quickReviewIndex];
    }
  }

  /**
   * CLOSE (✕) — the retailer exits early.
   * Commit whatever decisions have been made so far.
   */
  closeQuickReview(): void {
    this.commitQuickReview();
  }

  /**
   * Commits all accumulated Yes/No decisions in a single batched API call.
   * - Products marked YES → updateCatalogData (status changed to 'unpublished')
   * - Products marked NO  → insertCatalogData  (removedProducts)
   * Both calls are made in parallel. After completion, the catalog is refreshed.
   */
  private async commitQuickReview(): Promise<void> {
    this.quickReviewJustClosed = true;
    this.quickReviewOpen = false;
    this.quickReviewCurrentProduct = null;

    const hasUnpublish = this.quickReviewProductsToUnpublish.length > 0;
    const hasRemove = this.quickReviewProductsToRemove.length > 0;

    if (!hasUnpublish && !hasRemove) {
      // Nothing was changed — just close
      return;
    }

    const loader = await this.commonService.presentLoader();
    loader.present();
    this.quickReviewProcessing = true;

    try {
      const promises: Promise<any>[] = [];

      // ── Batch update: move products to 'unpublished' ──
      if (hasUnpublish) {
        promises.push(
          lastValueFrom(
            this.productsService.updateCatalogData({
              products: this.quickReviewProductsToUnpublish
            })
          )
        );
      }

      // ── Batch remove: remove products from catalog ──
      if (hasRemove) {
        promises.push(
          lastValueFrom(
            this.productsService.insertCatalogData({
              products: [],
              removedProducts: this.quickReviewProductsToRemove
            })
          )
        );
      }

      await Promise.all(promises);

      // ── Update local state ──
      const removedIds = new Set(this.quickReviewProductsToRemove.map((p: any) => p.id));
      const unpublishedIds = new Set(this.quickReviewProductsToUnpublish.map((p: any) => p.id));

      // Remove products from local data structures
      this.getProductDataFromBackend.forEach((catalog: any) => {
        catalog.categories.forEach((category: any) => {
          category.products = category.products.filter((p: any) => !removedIds.has(p.id));
          // Update status of unpublished products
          category.products.forEach((p: any) => {
            if (unpublishedIds.has(p.id)) {
              p.status = 'unpublished';
            }
          });
        });
      });

      this.productList = this.productList.filter((p: any) => !removedIds.has(p.id));
      removedIds.forEach(id => this.pendingOrUnpublishedProducts.delete(id));

      const totalProcessed = this.quickReviewProductsToUnpublish.length + this.quickReviewProductsToRemove.length;
      this.commonService.success(
        `${totalProcessed} product${totalProcessed > 1 ? 's' : ''} updated successfully`
      );

      // Refresh the current view
      this.segmentChanged();

    } catch (error: any) {
      console.error('Error in Quick Review batch commit:', error);
      this.commonService.danger(
        error?.error?.error?.message || 'An error occurred while updating products'
      );
      // Refresh from backend on error to get accurate state
      this.getCatalogDataFromBackend();
    } finally {
      this.quickReviewProcessing = false;
      loader.dismiss();

      // Clear accumulated arrays
      this.quickReviewProductsToUnpublish = [];
      this.quickReviewProductsToRemove = [];
      this.quickReviewProducts = [];
    }
  }
```

---

## Step 2 — HTML Changes

**File:** `frontend/retailer_app/src/app/pages/catelog/my-catalog/my-catalog.component.html`

Add the Quick Review modal **at the very end of the file**, after the last `</ion-modal>` tag (after line 254):

```html
<!-- ═══════════════════════════════════════════════════════ -->
<!-- QUICK REVIEW POPUP — Batch verify pending products     -->
<!-- ═══════════════════════════════════════════════════════ -->
<ion-modal [isOpen]="quickReviewOpen" class="quick-review-modal" [backdropDismiss]="false"
  mode="ios" [breakpoints]="[0, 1]" initialBreakpoint="1"
  [leaveAnimation]="leaveAnimation" [canDismiss]="canDismiss">
  <ng-template>
    <div class="quick-review-wrapper">

      <!-- Header with close button -->
      <ion-header mode="ios" class="ion-no-border">
        <ion-toolbar>
          <ion-title class="quick-review-title">
            {{'areyousellingthis' | language : 'Are You Selling This ?'}}
          </ion-title>
          <ion-buttons slot="end">
            <ion-button class="quick-review-close" (click)="closeQuickReview()">
              <ion-icon name="close-outline"></ion-icon>
            </ion-button>
          </ion-buttons>
        </ion-toolbar>
      </ion-header>

      <!-- Product display -->
      <ion-content class="ion-padding" *ngIf="quickReviewCurrentProduct">
        <div class="quick-review-content">

          <!-- Product image area -->
          <div class="quick-review-image-container">
            <div class="quick-review-badge-left">
              <ion-img src="assets/images/Insert_photo.svg"></ion-img>
            </div>
            <div class="quick-review-product-image">
              <ion-img
                [src]="quickReviewCurrentProduct?.media?.imgTh
                       ? quickReviewCurrentProduct.media.imgTh
                       : quickReviewCurrentProduct?.media?.img1
                         ? quickReviewCurrentProduct.media.img1
                         : 'assets/images/defaultImage.png'">
              </ion-img>
            </div>
            <div class="quick-review-badge-right">
              <ion-img
                [src]="quickReviewCurrentProduct?.grd
                       ? '/assets/award/award-' + quickReviewCurrentProduct.grd + '.svg'
                       : '/assets/award/award0.png'">
              </ion-img>
            </div>
          </div>

          <!-- Counter -->
          <div class="quick-review-counter">
            <span>{{ getQuickReviewCounter() }}</span>
          </div>

          <!-- Product name -->
          <div class="quick-review-product-name">
            <p>{{ quickReviewCurrentProduct.id | language : quickReviewCurrentProduct.prdNm }}</p>
          </div>

          <!-- Yes / No buttons -->
          <div class="quick-review-actions">
            <ion-button expand="block" class="quick-review-yes-btn" shape="round" mode="ios"
              (click)="onQuickReviewYes()">
              {{'yes' | language : 'Yes'}}
            </ion-button>
            <ion-button expand="block" class="quick-review-no-btn" shape="round" mode="ios"
              (click)="onQuickReviewNo()">
              {{'no' | language : 'No'}}
            </ion-button>
          </div>

        </div>
      </ion-content>

    </div>
  </ng-template>
</ion-modal>
```

---

## Step 3 — SCSS Changes

**File:** `frontend/retailer_app/src/app/pages/catelog/my-catalog/my-catalog.component.scss`

Add the following styles **at the very end of the file** (after line 316):

```scss
// ═══════════════════════════════════════════════════════
// QUICK REVIEW POPUP STYLES
// ═══════════════════════════════════════════════════════

.quick-review-modal {
  --height: auto;
  --max-height: 90vh;

  .quick-review-wrapper {
    background: #fff;
    min-height: 500px;
    display: flex;
    flex-direction: column;
  }

  .quick-review-title {
    font-size: 18px;
    font-weight: 700;
    text-align: center;
    color: #333;
  }

  .quick-review-close {
    font-size: 28px;
    color: #666;
  }

  .quick-review-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0 16px;
  }

  // ── Image container with badge overlays ──
  .quick-review-image-container {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 100%;
    padding: 20px 0;

    .quick-review-product-image {
      width: 180px;
      height: 180px;
      display: flex;
      align-items: center;
      justify-content: center;

      ion-img {
        max-width: 100%;
        max-height: 100%;
        object-fit: contain;
      }
    }

    .quick-review-badge-left {
      position: absolute;
      left: 20px;
      top: 20px;

      ion-img {
        width: 40px;
        height: 40px;
        opacity: 0.7;
      }
    }

    .quick-review-badge-right {
      position: absolute;
      right: 20px;
      top: 20px;

      ion-img {
        width: 40px;
        height: 40px;
      }
    }
  }

  // ── Counter badge ──
  .quick-review-counter {
    display: flex;
    justify-content: flex-end;
    width: 100%;
    padding-right: 10px;
    margin-bottom: 8px;

    span {
      background: #f0f0f0;
      color: #666;
      font-size: 13px;
      font-weight: 500;
      padding: 4px 12px;
      border-radius: 12px;
    }
  }

  // ── Product name pill ──
  .quick-review-product-name {
    width: 100%;
    text-align: center;
    margin-bottom: 24px;

    p {
      display: inline-block;
      background: #f5f5f5;
      border: 1px solid #e0e0e0;
      border-radius: 20px;
      padding: 10px 30px;
      font-size: 16px;
      font-weight: 500;
      color: #333;
      margin: 0;
      min-width: 200px;
    }
  }

  // ── Action buttons ──
  .quick-review-actions {
    width: 100%;
    padding: 0 16px;

    .quick-review-yes-btn {
      --background: var(--ion-color-primary, #10BAB2);
      --color: #fff;
      --border-radius: 25px;
      margin-bottom: 12px;
      height: 48px;
      font-size: 16px;
      font-weight: 600;
      text-transform: capitalize;
    }

    .quick-review-no-btn {
      --background: var(--ion-color-primary, #10BAB2);
      --color: #fff;
      --border-radius: 25px;
      height: 48px;
      font-size: 16px;
      font-weight: 600;
      text-transform: capitalize;
    }
  }
}
```

---

## How It Works — End to End

```
1. User navigates to My Catalog → Pending tab
2. segmentChanged() fires → filters pending products → auto-opens Quick Review
3. Modal appears with first pending product, counter shows "1/N"
4. User taps YES:
   → Product clone added to quickReviewProductsToUnpublish[] (status → 'unpublished')
   → Index advances, next product shown, counter updates to "2/N"
5. User taps NO:
   → Product added to quickReviewProductsToRemove[]
   → Index advances, next product shown
6. When user taps ✕ OR last product is reviewed:
   → commitQuickReview() fires
   → Loader shown
   → Two API calls in parallel:
     a) updateCatalogData({ products: [...unpublished] })
     b) insertCatalogData({ products: [], removedProducts: [...removed] })
   → Local state updated (products removed / status changed)
   → Success toast shown
   → segmentChanged() called to refresh the product list
   → Loader dismissed
```

---

## API Calls Used (Existing — No Changes Needed)

| Method | API | Purpose |
|--------|-----|---------|
| `updateCatalogData()` | `PUT /rms/apis/v2/catalog/update/{shopId}` | Changes product status (pending → unpublished) |
| `insertCatalogData()` | `PUT /rms/apis/v2/catalog/{shopId}` | Removes products from catalog via `removedProducts` array |

Both methods are already defined in `products.service.ts` and work with arrays of products.

---

## Important: Loop Guard

Since `commitQuickReview()` calls `this.segmentChanged()` to refresh the list, and `segmentChanged()` auto-opens the Quick Review when pending products exist, a `quickReviewJustClosed` boolean flag is used to prevent the popup from immediately reopening after a commit. The flag is set to `true` before committing and reset to `false` at the end of `segmentChanged()`.

---

## Testing Checklist

- [ ] Open My Catalog → Pending tab with pending products → Quick Review popup auto-opens
- [ ] Counter shows correct "1/N" format
- [ ] Product image, name, and badge display correctly
- [ ] Tap **Yes** → product counter advances, next product shows
- [ ] Tap **No** → product counter advances, next product shows
- [ ] Tap **✕ (close)** midway → loader shows → products processed in batch → success toast → pending list updates
- [ ] Review all products (reach end) → auto-commits → same behavior as closing
- [ ] After commit: products marked Yes appear in **Unavailable** tab
- [ ] After commit: products marked No are removed entirely from all tabs
- [ ] Open Pending tab with **no** pending products → popup does NOT open
- [ ] Existing product-card click behavior (cross button, card click) still works unchanged
- [ ] Pull-to-refresh still works
- [ ] Switching between category tabs (All, Grocery, etc.) while on Pending recalculates correctly
- [ ] If API call fails → error toast shown, catalog refreshes from backend
- [ ] Popup does NOT reopen after committing (loop guard works)
