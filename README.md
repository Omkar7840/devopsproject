# Geo-Intelligent Auto Catalog System V2 — Full Implementation

## Core Updates in V2
1. **Segmented Schema (`trending` vs `popular`)**: Categories now have two distinct lists. `trending` contains products aggregated from nearby real shops. `popular` contains AI-generated or fallback products.
2. **Fuzzy Catalogue Matching**: If the generated or aggregated product matches "a little" with the master catalogue (substring/partial match), the system will successfully map it to the common master catalogue product.
3. **Frontend Integration**: 
   - **Registered Shops**: Intersection logic. Shows products from the retailer's catalog that are geographically popular *first*, followed by the rest of the retailer's catalog.
   - **Google API Shops**: Fallback logic. Uses the shop's Google-provided pincode, city, or state to fetch and display the geo-catalog automatically.

---

## Part 1: Backend Implementation

### 1. Update Schema
**File:** `backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/geoCatalogSchema.js`

Split the `products` array into `trending` and `popular`.

```js
const mongoose = require('mongoose');

const geoCatalogProductSchema = new mongoose.Schema(
  {
    name: { type: String, required: true, trim: true },
    count: { type: Number, default: 0, min: 0 },
    catalogueProductId: { type: String, default: null },
    source: { type: String, enum: ['DB', 'AI', 'FALLBACK'], default: 'DB' },
  },
  { _id: false },
);

const geoCatalogCategorySchema = new mongoose.Schema(
  {
    name: { type: String, required: true, trim: true },
    trending: { type: [geoCatalogProductSchema], default: [] }, // Nearby Shops
    popular: { type: [geoCatalogProductSchema], default: [] },  // AI Generated
  },
  { _id: false },
);

const geoCatalogSchema = new mongoose.Schema(
  {
    level: { type: String, required: true, enum: ['PINCODE', 'CITY', 'STATE', 'COUNTRY'] },
    pincode: { type: String, default: null, index: true },
    city: { type: String, default: null },
    state: { type: String, default: null },
    country: { type: String, default: null },
    categories: { type: [geoCatalogCategorySchema], default: [] },
    lastBuildAt: { type: Date, default: null },
    buildStatus: { type: String, enum: ['SUCCESS', 'PARTIAL', 'FALLBACK', 'FAILED'], default: 'SUCCESS' },
  },
  { timestamps: true },
);

geoCatalogSchema.index({ level: 1, pincode: 1 }, { sparse: true });
geoCatalogSchema.index({ level: 1, city: 1 }, { sparse: true });

module.exports = mongoose.model('geo_catalogs', geoCatalogSchema);
```

### 2. Update Partial/Fuzzy Matcher
**File:** `backend/catalogue_mgmt_service/src/apis/services/v1/catalogueMatcher.service.js`

Matches products "a little" using Postgres `ILIKE '%term%'`.

```js
const { Logger: log } = require('sarvm-utility');
const Product = require('../../models/product');

const matchAgainstCatalogue = async (productNames = []) => {
  if (!productNames.length) return new Map();

  const matchedMap = new Map();
  const uniqueNames = [...new Set(productNames.map((n) => String(n).trim()).filter(Boolean))];
  const allMatched = [];

  // Match chunks using ILIKE %name% for fuzzy matching
  for (let i = 0; i < uniqueNames.length; i += 100) {
    const chunk = uniqueNames.slice(i, i + 100);
    try {
      const results = await Product.query()
        .select('id', 'name', 'dummyKey', 'image', 'status')
        .where('status', '=', 'ACTIVE')
        .where((builder) => {
          chunk.forEach((productName) => {
            const safeName = productName.replace(/[%_]/g, '\\$&'); // Escape SQL wildcards
            builder.orWhere('name', 'ilike', `%${safeName}%`);
          });
        });
      allMatched.push(...results);
    } catch (error) {
      log.warn({ warn: 'CatalogueMatcher chunk failed', details: error.message });
    }
  }

  // Map generated names back to their best fuzzy-matched DB product
  uniqueNames.forEach((searchName) => {
    const searchKey = searchName.toLowerCase();
    
    // Find the first DB product that partially matches (either contains search word, or search word contains it)
    const match = allMatched.find(
      (dbProd) => dbProd.name.toLowerCase().includes(searchKey) || searchKey.includes(dbProd.name.toLowerCase())
    );

    if (match && !matchedMap.has(searchKey)) {
      matchedMap.set(searchKey, { id: match.id, name: match.name });
    }
  });

  return matchedMap;
};

const filterByCatalogue = (products = [], catalogueMap) => {
  return products.map((product) => {
    const key = String(product.name).trim().toLowerCase();
    const matched = catalogueMap.get(key);
    if (!matched) return null;

    return {
      ...product,
      name: matched.name, // Use common/standardized catalogue name
      catalogueProductId: matched.id,
    };
  }).filter(Boolean);
};

module.exports = { matchAgainstCatalogue, filterByCatalogue };
```

### 3. Update Pincode Builder for Trending vs Popular
**File:** `backend/catalogue_mgmt_service/src/apis/services/v1/pincodeCatalogBuilder.service.js`

Update the `applyCatalogueMatching` function to separate the arrays.

```js
// Inside pincodeCatalogBuilder.service.js
const applyCatalogueMatching = async (categories = []) => {
  const allProductNames = [];
  categories.forEach((cat) => {
    (cat.products || []).forEach((p) => allProductNames.push(p.name));
  });

  const catalogueMap = await matchAgainstCatalogue(allProductNames);

  // Split into trending and popular
  return categories.map((category) => {
    const validProducts = filterByCatalogue(category.products, catalogueMap);
    
    const trending = validProducts.filter(p => p.source === 'DB').slice(0, TOP_PRODUCTS_LIMIT);
    const popular = validProducts.filter(p => p.source !== 'DB').slice(0, TOP_PRODUCTS_LIMIT);

    return {
      name: category.name,
      trending,
      popular
    };
  });
};
```
*(Ensure all code accessing `category.products` in this file is updated to `category.trending` and `category.popular` where applicable)*

### 4. Update Geo Hierarchy Aggregation
**File:** `backend/catalogue_mgmt_service/src/apis/services/v1/geoHierarchy.service.js`

```js
const aggregateCategories = (documents = []) => {
  const categoryMap = new Map();

  documents.forEach((document) => {
    (document?.categories || []).forEach((category) => {
      const categoryName = String(category?.name || '').trim().toLowerCase();
      if (!categoryName) return;

      if (!categoryMap.has(categoryName)) {
        categoryMap.set(categoryName, { trending: new Map(), popular: new Map() });
      }

      const mapObj = categoryMap.get(categoryName);

      const addProduct = (product, mapRef) => {
        const pName = String(product?.name || '').trim().toLowerCase();
        if (!pName) return;
        mapRef.set(pName, {
          count: (mapRef.get(pName)?.count || 0) + Number(product?.count || 0),
          catalogueProductId: product.catalogueProductId,
          source: product.source,
        });
      };

      (category.trending || []).forEach(p => addProduct(p, mapObj.trending));
      (category.popular || []).forEach(p => addProduct(p, mapObj.popular));
    });
  });

  return Array.from(categoryMap.entries()).map(([name, maps]) => {
    const mapToArray = (map) => Array.from(map.entries())
      .map(([pName, data]) => ({ name: pName, ...data }))
      .sort((l, r) => r.count - l.count || l.name.localeCompare(r.name))
      .slice(0, 10);

    return {
      name,
      trending: mapToArray(maps.trending),
      popular: mapToArray(maps.popular),
    };
  }).sort((l, r) => l.name.localeCompare(r.name));
};
```

---

## Part 2: Frontend Implementation (`hha_web`)

### 1. Angular Service
**File:** `frontend/hha_web/src/app/services/geo-catalog.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { environment } from 'src/environments/environment';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class GeoCatalogService {
  constructor(private http: HttpClient) {}

  getResolvedGeoCatalog(params: { shopId?: number; pincode?: string; city?: string; state?: string }): Observable<any> {
    let httpParams = new HttpParams();
    if (params.shopId) httpParams = httpParams.set('shopId', params.shopId.toString());
    if (params.pincode) httpParams = httpParams.set('pincode', params.pincode);
    if (params.city) httpParams = httpParams.set('city', params.city);
    if (params.state) httpParams = httpParams.set('state', params.state);

    return this.http.get(`${environment.apiBaseUrl}/cms/apis/v1/geo/catalog`, { params: httpParams });
  }
}
```

### 2. Component Logic (Shop Catalog Page)
**File:** `frontend/hha_web/src/app/pages/shop-details/shop-details.component.ts`

```typescript
import { Component, OnInit } from '@angular/core';
import { GeoCatalogService } from 'src/app/services/geo-catalog.service';
import { ShopService } from 'src/app/services/shop.service'; // Assuming existing service

@Component({
  selector: 'app-shop-details',
  templateUrl: './shop-details.component.html',
  styleUrls: ['./shop-details.component.scss']
})
export class ShopDetailsComponent implements OnInit {
  shopData: any; // Passed from router or fetched
  displayCatalog: any[] = [];
  isLoading = true;

  constructor(
    private geoCatalogService: GeoCatalogService,
    private shopService: ShopService
  ) {}

  ngOnInit() {
    this.loadShopCatalog();
  }

  loadShopCatalog() {
    this.isLoading = true;

    if (this.shopData.isRegistered) {
      // 1. Registered Shop Flow
      this.shopService.getRetailerCatalog(this.shopData.shopId).subscribe(retailerCatalog => {
        this.geoCatalogService.getResolvedGeoCatalog({ shopId: this.shopData.shopId }).subscribe(
          (geoRes) => {
            this.processRegisteredShop(retailerCatalog, geoRes.catalog);
            this.isLoading = false;
          },
          (error) => {
             // Fallback if no geo catalog exists
             this.displayCatalog = retailerCatalog;
             this.isLoading = false;
          }
        );
      });
    } else {
      // 2. Google API (Unregistered) Shop Flow
      // Pass whatever location data Google gave us
      this.geoCatalogService.getResolvedGeoCatalog({ 
        pincode: this.shopData.pincode,
        city: this.shopData.city,
        state: this.shopData.state
      }).subscribe(
        (geoRes) => {
          this.processUnregisteredShop(geoRes.catalog);
          this.isLoading = false;
        },
        (error) => {
          this.displayCatalog = []; // No data available
          this.isLoading = false;
        }
      );
    }
  }

  // Merges Retailer Catalog with Geo Insights
  processRegisteredShop(retailerCatalog: any[], geoCatalog: any) {
    const geoProductNames = new Set<string>();
    
    // Aggregate all geo names (trending + popular)
    (geoCatalog?.categories || []).forEach(cat => {
      (cat.trending || []).forEach(p => geoProductNames.add(p.name.toLowerCase()));
      (cat.popular || []).forEach(p => geoProductNames.add(p.name.toLowerCase()));
    });

    const matchedGeoItems = [];
    const remainingRetailerItems = [];

    // Split retailer catalog into geographically relevant vs remaining
    retailerCatalog.forEach(item => {
      const isGeoRelevant = geoProductNames.has(item.productName.toLowerCase());
      if (isGeoRelevant) {
        matchedGeoItems.push({ ...item, isRecommended: true });
      } else {
        remainingRetailerItems.push(item);
      }
    });

    // Display Matched/Relevant first, then the rest
    this.displayCatalog = [...matchedGeoItems, ...remainingRetailerItems];
  }

  // Prepares Geo Catalog for Unregistered Shops
  processUnregisteredShop(geoCatalog: any) {
    this.displayCatalog = [];
    
    (geoCatalog?.categories || []).forEach(cat => {
      // Show Trending (Local shops) first, then Popular (AI generated)
      (cat.trending || []).forEach(p => {
        this.displayCatalog.push({ productName: p.name, category: cat.name, isTrending: true });
      });
      (cat.popular || []).forEach(p => {
        this.displayCatalog.push({ productName: p.name, category: cat.name, isPopular: true });
      });
    });
  }
}
```

### 3. Frontend HTML Template
**File:** `frontend/hha_web/src/app/pages/shop-details/shop-details.component.html`

```html
<div class="catalog-container" *ngIf="!isLoading">
  
  <div *ngIf="displayCatalog.length === 0" class="no-data">
    No products found for this shop.
  </div>

  <div class="product-item" *ngFor="let item of displayCatalog">
    <!-- UI Badges for Recommended/Trending -->
    <div class="badges">
       <span class="badge recommended" *ngIf="item.isRecommended">🔥 Local Favorite</span>
       <span class="badge trending" *ngIf="item.isTrending">📈 Nearby Trending</span>
       <span class="badge popular" *ngIf="item.isPopular">⭐ Highly Popular</span>
    </div>

    <div class="product-info">
      <h3>{{ item.productName }}</h3>
      <p>{{ item.category }}</p>
    </div>
  </div>

</div>
```

---

## Part 3: Postman Testing Method

### Testing the Backend Hierarchy & Fuzzy Matching
1. **Trigger Pincode Build**
   - Method: `POST`
   - URL: `http://localhost:2210/cms/apis/v1/geo/test/pincode`
   - Body: `{ "pincode": "560100" }`
   - **Verification**: Check if the response contains `trending` and `popular` arrays inside the categories. Verify that product names returned have been "fuzzily matched" (e.g., if AI generated "Amul Milk", the response should standardize it to "Amul Gold Milk 500ml" if that exists in Postgres).

2. **Test Fallback Endpoint (Google API Shop Simulation)**
   - Method: `GET`
   - URL: `http://localhost:2210/cms/apis/v1/geo/catalog?city=Bangalore&state=Karnataka` (Simulating missing pincode)
   - **Verification**: The API should cascade from Pincode -> City -> State -> Country and return the City or State catalog.

### Testing Frontend Display Modes
1. **Mock a Registered Shop**
   - Ensure the mocked `retailerCatalog` contains some products that exist in the geo-catalog and some that don't.
   - **Verification**: In your UI, the matched items should appear at the top of the list with the "🔥 Local Favorite" badge, and the remaining products should appear directly beneath them.
   
2. **Mock an Unregistered Google Shop**
   - Pass `isRegistered: false` and `pincode: "560100"`.
   - **Verification**: The UI should display the catalog purely based on the Geo-Catalog. Items from `trending` will show the "📈 Nearby Trending" badge, and items from `popular` will show the "⭐ Highly Popular" badge.
