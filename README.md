# Geo-Intelligent Auto Catalog System — Full Implementation

## Core Design Principle

> **Generated top products are kept internally, but the visible/output list is the catalogue-matched subset only.**

This means:

1. Build nearby-shop frequency products from real registered shops
2. Generate category-wise candidates using Gemini AI (category-aware: fresh vs packaged)
3. Keep that full generated result internally
4. **Match generated items against the master catalogue (Postgres `product` table)**
5. **Return only matched catalogue products to frontend**

Because of this, **frontend does not need changes** — backend always returns valid catalogue products.

---

## Rollout Plan

This implementation is split into **2 phases**.

### Phase 1: Testing Only For Single Pincode `560100`

Purpose:

- prove the aggregation logic works
- validate category grouping
- validate top-product generation from nearby shops
- validate catalogue matching (only master-catalogue products in output)
- validate Mongo insert into `geo_catalogs`
- validate `POST /geo/test/pincode`

In this phase:

- only run catalog build for `560100`
- use `retailercatalog` as primary source
- use fallback mock category data if real data is too low
- **do not depend on Gemini** (use static fallback in `ai.service.js`)
- **match all generated products against master catalogue before returning**

Main entrypoint for this phase:

- `buildPincodeCatalog('560100')`
- API:
  - `POST http://localhost:2210/cms/apis/v1/geo/test/pincode`

Expected result:

- one `geo_catalogs` document with `level = PINCODE`
- `pincode = 560100`
- multiple categories
- multiple products inside each category
- **every returned product name exists in the master catalogue**
- internal `_rawGeneratedProducts` kept for debugging but NOT sent to frontend

### Phase 2: Full Production-Style Implementation

Purpose:

- expand from single pincode to all pincodes
- add hierarchy fallback:
  - `PINCODE -> CITY -> STATE -> COUNTRY`
- add nightly cron at `2:00 AM` (**one pincode at a time, sequential**)
- apply geo catalog to shops using `/geo/apply`
- generate top products using a hybrid of:
  - common nearby-shop products from DB (frequency counts)
  - Gemini AI suggested products (**category-aware**)
- **match ALL generated products against master catalogue before storing/returning**

Category-aware Gemini generation:

- **Fresh/local categories** (dairy, vegetables, fruits): common generic names like `tomato`, `bhindi`, `okra`, `paneer`
- **Packaged categories** (snacks, beverages, personal care): brand/company-specific names like `Lays Classic Salted`, `Bikaji Bhujia`, `Colgate MaxFresh`

In this phase:

- cron runs every day at `2 AM`
- **processes one pincode at a time** (sequential, not parallel) to avoid overloading Gemini API
- all shop pincodes are processed
- city/state/country catalogs are rebuilt after pincode generation
- Gemini is not standalone source of truth
- final top-product list is a merged list of:
  - nearby real shop frequency products
  - Gemini-suggested high-likelihood category products
- **Before output, every product is matched against master catalogue**
- **Only catalogue-matched products appear in API response and stored document**

---

## What To Test First

Test order should be:

1. local DB connections (Postgres + MongoDB)
2. `POST /geo/test/pincode` for `560100`
3. Verify Mongo `geo_catalogs` insert
4. Verify returned products exist in master catalogue
5. `POST /geo/test/shop` for a known shopId
6. `POST /geo/apply`
7. Mongo `customcatalog` insert verification
8. Manual cron trigger via `POST /geo/cron/trigger`
9. Nightly cron execution
10. Gemini integration (Phase 2)

---

## File-By-File Implementation

### File 1: `backend/catalogue_mgmt_service/src/apis/utils/normalizeProduct.js`

**Action: EDIT existing file (replace contents)**

What this code does:

- keeps retailer product names exactly as stored in source data
- removes only leading/trailing whitespace
- does not change casing, units, punctuation, or numbers

```js
const normalize = (value = '') => {
  return String(value || '')
    .toLowerCase()
    .replace(/[,()\-_\[\]]+/g, ' ') // remove symbols like , ( ) - _ [ ]
    .replace(/\s+/g, ' ')           // collapse multiple spaces
    .trim();                        // remove leading/trailing spaces
};

module.exports = normalize;
```

---

### File 2: `backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/geoCatalogSchema.js`

**Action: EDIT existing file (replace contents)**

What this code does:

- Mongo collection `geo_catalogs`
- stores multiple categories in one document
- stores both `matchedProducts` (catalogue-verified) and raw `products`
- supports hierarchy levels: `PINCODE`, `CITY`, `STATE`, `COUNTRY`

```js
const mongoose = require('mongoose');

const geoCatalogProductSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true,
    },
    count: {
      type: Number,
      default: 0,
      min: 0,
    },
    catalogueProductId: {
      type: String,
      default: null,
    },
    source: {
      type: String,
      enum: ['DB', 'AI', 'FALLBACK'],
      default: 'DB',
    },
  },
  { _id: false },
);

const geoCatalogCategorySchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true,
    },
    products: {
      type: [geoCatalogProductSchema],
      default: [],
    },
  },
  { _id: false },
);

const geoCatalogSchema = new mongoose.Schema(
  {
    level: {
      type: String,
      required: true,
      enum: ['PINCODE', 'CITY', 'STATE', 'COUNTRY'],
    },
    pincode: {
      type: String,
      default: null,
      index: true,
    },
    city: {
      type: String,
      default: null,
    },
    state: {
      type: String,
      default: null,
    },
    country: {
      type: String,
      default: null,
    },
    categories: {
      type: [geoCatalogCategorySchema],
      default: [],
    },
    lastBuildAt: {
      type: Date,
      default: null,
    },
    buildStatus: {
      type: String,
      enum: ['SUCCESS', 'PARTIAL', 'FALLBACK', 'FAILED'],
      default: 'SUCCESS',
    },
  },
  { timestamps: true },
);

geoCatalogSchema.index({ level: 1, pincode: 1 }, { sparse: true });
geoCatalogSchema.index({ level: 1, city: 1 }, { sparse: true });
geoCatalogSchema.index({ level: 1, state: 1 }, { sparse: true });
geoCatalogSchema.index({ level: 1, country: 1 }, { sparse: true });

const GeoCatalog = mongoose.model('geo_catalogs', geoCatalogSchema);

module.exports = GeoCatalog;
```

---

### File 3: `backend/catalogue_mgmt_service/src/apis/services/v1/catalogueMatcher.service.js`

**Action: CREATE new file**

What this code does:

- **This is the critical new piece** — matches generated product names against the master catalogue
- Queries the Postgres `product` table via Objection.js model
- Uses case-insensitive `ILIKE` matching
- Returns only products that exist in the catalogue
- Attaches the matched catalogue product ID to each result

Why this exists:

- Frontend should never show raw generated names directly
- Only products existing in our catalogue should be returned
- This is the filter between "generated internally" and "shown to user"

```js
const { Logger: log } = require('sarvm-utility');
const MongoProduct = require('../../models/mongoCatalog/productSchema');

/**
 * Normalize product name → same logic as dumK
 */
const normalizeKey = (name = '') =>
  String(name)
    .trim()
    .toLowerCase()
    .replace(/\s+/g, '_');

/**
 * Match product names against MongoDB `product` collection.
 * Uses dumK (dummyKey) for fast exact matching.
 *
 * @param {string[]} productNames
 * @returns {Promise<Map<string, { id: string, name: string, dummyKey: string, image: string }>>}
 */
const matchAgainstCatalogue = async (productNames = []) => {
  if (!productNames.length) return new Map();

  const matchedMap = new Map();

  try {
    // Normalize + dedupe
    const uniqueNames = [
      ...new Set(
        productNames.map((n) => normalizeKey(n)).filter(Boolean)
      ),
    ];

    const CHUNK_SIZE = 200;

    for (let i = 0; i < uniqueNames.length; i += CHUNK_SIZE) {
      const chunk = uniqueNames.slice(i, i + CHUNK_SIZE);

      try {
        // 🔥 Match using dumK (fast indexed lookup)
        const results = await MongoProduct.find({
          dumK: { $in: chunk },
          status: 'PUBLISHED',
        })
          .select('_id prdNm dumK media.img1')
          .lean();

        results.forEach((product) => {
          const key = product.dumK;

          if (!matchedMap.has(key)) {
            matchedMap.set(key, {
              id: String(product._id),
              name: product.prdNm,
              dummyKey: product.dumK,
              image: product?.media?.img1 || null,
            });
          }
        });
      } catch (chunkError) {
        log.warn({
          warn: 'CatalogueMatcher(Mongo): chunk query failed',
          error: chunkError.message,
          chunkSize: chunk.length,
        });
      }
    }

    log.info({
      info: 'CatalogueMatcher(Mongo): matching complete',
      inputCount: uniqueNames.length,
      matchedCount: matchedMap.size,
    });
  } catch (error) {
    log.error({
      error: 'CatalogueMatcher(Mongo): matching failed',
      details: error.message,
    });
  }

  return matchedMap;
};

/**
 * Filter products to only catalogue-matched ones
 */
const filterByCatalogue = (products = [], catalogueMap) => {
  return products
    .map((product) => {
      const key = normalizeKey(product.name);
      const matched = catalogueMap.get(key);

      if (!matched) return null;

      return {
        ...product,
        name: matched.name, // use canonical name
        catalogueProductId: matched.id,
        dummyKey: matched.dummyKey,
        image: matched.image,
      };
    })
    .filter(Boolean);
};

module.exports = {
  matchAgainstCatalogue,
  filterByCatalogue,
};
```

---

### File 4: `backend/catalogue_mgmt_service/src/apis/services/v1/ai.service.js`

**Action: EDIT existing file (replace contents)**

What this code does:

- Phase 1: provides static fallback product lists (mock, no API call)
- Phase 2: calls Gemini API for **category-aware** product suggestions
  - Fresh/local categories → common/generic names (tomato, paneer, curd)
  - Packaged categories → brand-specific names (Lays Classic Salted, Bisleri Water)
- All generated names go through catalogue matching before being shown

Phase 2 Gemini environment variables needed in `.lcl.env`:
```
GEMINI_API_KEY=your_key_here
GEMINI_MODEL=gemini-2.0-flash
```

```js
const { Logger: log } = require('sarvm-utility');

/* ---------------- CATEGORY CONFIG ---------------- */

const FRESH_CATEGORIES = new Set([
  'dairy',
  'vegetables',
  'fruits',
  'meat',
  'fish',
  'flowers',
  'restaurant',
  'home_food',
]);

const normalizeCategory = (str = '') =>
  String(str)
    .toLowerCase()
    .replace(/\s+/g, '_')
    .trim();

const isFreshCategory = (categoryName) => {
  return FRESH_CATEGORIES.has(normalizeCategory(categoryName));
};

/* ---------------- FALLBACK ---------------- */

const fallbackCatalog = {
  dairy: [
    'Amul Gold Milk 500ml',
    'Amul Taaza Milk 1L',
    'Mother Dairy Curd 400g',
    'Amul Butter 100g',
    'Amul Paneer 200g',
  ],
  snacks: [
    'Lays Classic Salted 52g',
    'Kurkure Masala Munch 90g',
    'Bingo Mad Angles 72g',
    'Parle-G Biscuits 250g',
  ],
  beverages: [
    'Bisleri Water 1L',
    'Coca Cola 750ml',
    'Pepsi 750ml',
    'Tropicana Juice 1L',
  ],
  vegetables: ['Tomato', 'Onion', 'Potato', 'Carrot'],
  fruits: ['Banana', 'Apple', 'Orange', 'Mango'],
};

const getFallbackProducts = (categories = []) => {
  const result = {};

  categories.forEach((cat) => {
    const key = normalizeCategory(cat);
    result[cat] = fallbackCatalog[key] || [];
  });

  return result;
};

/* ---------------- GEMINI SETUP ---------------- */

let geminiAvailable = false;
let GoogleGenerativeAI = null;

try {
  const geminiModule = require('@google/generative-ai');
  GoogleGenerativeAI = geminiModule.GoogleGenerativeAI;
  geminiAvailable = true;
} catch (e) {
  geminiAvailable = false;
}

/* ---------------- PROMPT BUILDER ---------------- */

const buildPrompt = (categoryName, existingProducts = []) => {
  const isFresh = isFreshCategory(categoryName);
  const existing = existingProducts.slice(0, 10).join(', ');

  if (isFresh) {
    return `
Generate 15 common ${categoryName} products.

Rules:
- Use generic names (no brands)
- Example: Tomato, Onion, Milk, Paneer
${existing ? `Avoid: ${existing}` : ''}

Return JSON array only.
`;
  }

  return `
Generate 15 realistic ${categoryName} products used in Indian stores.

Rules:
- Include brand + size
- Example:
  - Amul Gold Milk 500ml
  - Lays Classic Salted 52g
  - Bisleri Water 1L
${existing ? `Avoid: ${existing}` : ''}

Return JSON array only.
`;
};

/* ---------------- GEMINI FUNCTION ---------------- */

const getGeminiCategoryProducts = async (categoryName, existingProducts = []) => {
  const normalized = normalizeCategory(categoryName);

  /* -------- CHECK GEMINI AVAILABILITY -------- */

  if (!geminiAvailable || !process.env.GEMINI_API_KEY) {
    log.warn({
      warn: 'Gemini not available, using fallback',
      category: categoryName,
    });

    const fallback = getFallbackProducts([normalized]);
    return fallback[normalized] || [];
  }

  try {
    log.info({
      info: 'Gemini API call started',
      category: categoryName,
      existingCount: existingProducts.length,
    });

    const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

    const model = genAI.getGenerativeModel({
      model: process.env.GEMINI_MODEL || 'gemini-2.0-flash',
    });

    const prompt = buildPrompt(categoryName, existingProducts);

    const result = await model.generateContent(prompt);
    const responseText = result.response.text();

    log.info({
      info: 'Gemini raw response received',
      category: categoryName,
      response: responseText,
    });

    /* -------- PARSE RESPONSE -------- */

    const match = responseText.match(/\[[\s\S]*?\]/);

    if (!match) {
      throw new Error('Invalid JSON from Gemini');
    }

    const parsed = JSON.parse(match[0]);

    if (!Array.isArray(parsed)) {
      throw new Error('Gemini output not array');
    }

    const cleaned = parsed
      .map((name) => String(name).trim())
      .filter(Boolean);

    log.info({
      info: 'Gemini parsed successfully',
      category: categoryName,
      productCount: cleaned.length,
    });

    return cleaned;
  } catch (error) {
    log.error({
      error: 'Gemini API failed',
      category: categoryName,
      message: error.message,
      stack: error.stack,
    });

    log.warn({
      warn: 'Using fallback products',
      category: categoryName,
    });

    const fallback = getFallbackProducts([normalized]);
    return fallback[normalized] || [];
  }
};

/* ---------------- EXPORTS ---------------- */

module.exports = {
  getGeminiCategoryProducts,
  getFallbackProducts,
  isFreshCategory,
};
```

---

### File 5: `backend/catalogue_mgmt_service/src/apis/services/v1/shopGeo.service.js`

**Action: CREATE new file**

What this code does:

- resolves shop location metadata (pincode/city/state/country) for grouping
- fetches from RMS APIs, falls back to local Mongo collections

```js
const axios = require('axios');
const { Logger: log } = require('sarvm-utility');

const RetailerCatalog = require('../../models/mongoCatalog/retailerSchema');

/* ---------------- CACHE ---------------- */

/**
 * Cache: shopId → pincode
 */
const pincodeCache = new Map();

/* ---------------- HELPERS ---------------- */

const normalizeShop = (doc) => {
  if (!doc?.shopId) return null;

  return {
    shopId: Number(doc.shopId),
    url: doc.url || null,
  };
};

/**
 * 🔥 ROBUST PINCODE EXTRACTOR (FIXED)
 */
const extractPincode = (data = {}) => {
  return (
    data?.shop?.location?.pincode ||
    data?.location?.pincode ||
    data?.pincode ||
    data?.vendor?.pincode ||
    data?.address?.pincode ||
    data?.shop?.pincode ||
    null
  );
};

/**
 * Fetch pincode from URL (ONLY ONCE)
 */
const fetchPincodeFromURL = async (shopId, url) => {
  if (!url) return null;

  // ✅ Check cache
  if (pincodeCache.has(shopId)) {
    return pincodeCache.get(shopId);
  }

  try {
    console.log('👉 Fetching profile for shop:', shopId);
    console.log('👉 URL:', url);

    const response = await axios.get(url, {
      timeout: 5000,
    });

    const data = response?.data || {};

    // 🔥 DEBUG (VERY IMPORTANT)
    console.log('👉 PROFILE RESPONSE:', JSON.stringify(data, null, 2));

    const pincode = extractPincode(data);

    console.log('👉 Extracted pincode:', pincode);

    const finalPincode = pincode ? String(pincode) : null;

    // ✅ Only cache VALID pincodes
    if (finalPincode) {
      pincodeCache.set(shopId, finalPincode);
    }

    return finalPincode;
  } catch (error) {
    log.warn({
      warn: 'Failed to fetch shop profile',
      shopId,
      url,
      error: error.message,
    });

    return null;
  }
};

/* ---------------- CORE FUNCTIONS ---------------- */

/**
 * Get all unique shops (shopId + url)
 */
const getAllShops = async () => {
  const docs = await RetailerCatalog.find(
    {
      shopId: { $ne: null },
      url: { $ne: null },
    },
    { shopId: 1, url: 1 }
  ).lean();

  console.log('👉 Total retailer docs:', docs.length);

  const map = new Map();

  docs.forEach((doc) => {
    const shop = normalizeShop(doc);
    if (!shop) return;

    if (!map.has(shop.shopId)) {
      map.set(shop.shopId, shop);
    }
  });

  const shops = Array.from(map.values());

  console.log('👉 Unique shops:', shops.length);

  return shops;
};

/**
 * Get all shop locations (shopId → pincode)
 */
const getAllShopLocations = async () => {
  const shops = await getAllShops();

  const results = [];

  for (const shop of shops) {
    let pincode;

    if (pincodeCache.has(shop.shopId)) {
      pincode = pincodeCache.get(shop.shopId);
    } else {
      pincode = await fetchPincodeFromURL(
        shop.shopId,
        shop.url
      );
    }

    console.log('👉 Shop:', shop.shopId, 'Pincode:', pincode);

    if (pincode) {
      results.push({
        shopId: shop.shopId,
        pincode,
      });
    }
  }

  console.log('👉 Final shop locations:', results.length);

  return results;
};

/**
 * Get shops by pincode
 */
const getShopsByPincode = async (targetPincode) => {
  const normalized = String(targetPincode);

  const allShops = await getAllShopLocations();

  return allShops.filter(
    (shop) => String(shop.pincode) === normalized
  );
};

/**
 * Get single shop location
 */
const getShopLocation = async (shopId) => {
  const numericShopId = Number(shopId);

  const shops = await getAllShops();

  const shop = shops.find(
    (s) => s.shopId === numericShopId
  );

  if (!shop) return null;

  let pincode;

  if (pincodeCache.has(shop.shopId)) {
    pincode = pincodeCache.get(shop.shopId);
  } else {
    pincode = await fetchPincodeFromURL(
      shop.shopId,
      shop.url
    );
  }

  return {
    shopId: shop.shopId,
    pincode,
  };
};

/* ---------------- EXPORTS ---------------- */

module.exports = {
  getAllShops,
  getAllShopLocations,
  getShopsByPincode,
  getShopLocation,
};
```

---

### File 6: `backend/catalogue_mgmt_service/src/apis/services/v1/pincodeCatalogBuilder.service.js`

**Action: CREATE new file**

What this code does:

- **This is the core builder** for Phase 1 and Phase 2
- Takes a pincode → finds shops → finds retailer products → groups by category → counts frequency
- Merges Gemini/fallback suggestions
- **Matches ALL products against master catalogue before storing/returning**
- Only catalogue-matched products appear in the final output

Key flow:

1. Shop lookup by pincode
2. Retailer product fetch from MongoDB
3. Extract product names and categories, count frequency
4. For each category: get Gemini/fallback suggestions
5. Merge DB products + AI products
6. **Match entire merged list against master catalogue (MongoDB)**
7. **Keep only matched products**
8. Store result in `geo_catalogs`
9. Return catalogue-matched products

```js
const { Logger: log } = require('sarvm-utility');

const RetailerCatalog = require('../../models/mongoCatalog/retailerSchema');
const GeoCatalog = require('../../models/mongoCatalog/geoCatalogSchema');
const exactProductName = require('../../utils/normalizeProduct');

const aiService = require('./ai.service');
const shopGeoService = require('./shopGeo.service');

/* ---------------- CONFIG ---------------- */

const DB_LIMIT = 10;   // max DB products per category
const AI_LIMIT = 10;   // max AI products per category

/* ---------------- HELPERS ---------------- */

const normalizeCategory = (str = '') =>
  String(str)
    .toLowerCase()
    .replace(/\s+/g, '_')
    .trim();

const normalizeName = (str = '') =>
  String(str)
    .toLowerCase()
    .replace(/[^a-z0-9]/g, '');

/**
 * Extract category from retailer doc
 */
const getCategoryName = (doc) => {
  const raw =
    doc?.category ||
    doc?.catalog?.catPnm ||
    '';

  const first = String(raw).split('/')[0];

  return normalizeCategory(first);
};

/* ---------------- STEP 1: BUILD DB PRODUCTS ---------------- */

const buildRetailerCategories = (retailerDocs = []) => {
  const categoryMap = {};

  retailerDocs.forEach((doc) => {
    const name = exactProductName(doc?.catalog?.prdNm || '');
    const category = getCategoryName(doc);

    if (!name || !category) return;

    if (!categoryMap[category]) {
      categoryMap[category] = {};
    }

    categoryMap[category][name] =
      (categoryMap[category][name] || 0) + 1;
  });

  return Object.entries(categoryMap).map(([name, products]) => ({
    name,
    products: Object.entries(products)
      .map(([n, c]) => ({
        name: n,
        count: c,
        source: 'DB',
      }))
      .sort((a, b) => b.count - a.count),
  }));
};

/* ---------------- STEP 2: MERGE DB + AI ---------------- */

const buildTopProducts = async (categories = []) => {
  const finalCategories = [];

  for (const category of categories) {
    const dbProducts = category.products || [];

    const finalList = [];
    const usedNames = new Set();

    /* -------- 1. ADD DB PRODUCTS -------- */

    for (const p of dbProducts) {
      const clean = exactProductName(p.name);
      const norm = normalizeName(clean);

      if (!usedNames.has(norm)) {
        finalList.push({
          name: clean,
          count: p.count,
          source: 'DB',
        });
        usedNames.add(norm);
      }

      if (finalList.length >= DB_LIMIT) break;
    }

    /* -------- 2. ADD AI PRODUCTS (CATEGORY-WISE) -------- */

    let aiProducts = [];

    try {
      aiProducts = await aiService.getGeminiCategoryProducts(
        category.name,
        dbProducts.map((p) => p.name)
      );
    } catch (err) {
      log.error({
        error: 'AI generation failed',
        category: category.name,
        details: err.message,
      });

      const fallback = aiService.getFallbackProducts([category.name]);
      aiProducts = fallback[category.name] || [];
    }

    let aiAdded = 0;

    for (const name of aiProducts) {
      const clean = exactProductName(name);
      const norm = normalizeName(clean);

      // skip duplicates
      if (usedNames.has(norm)) continue;

      finalList.push({
        name: clean,
        count: 0,
        source: 'AI',
      });

      usedNames.add(norm);
      aiAdded++;

      if (aiAdded >= AI_LIMIT) break;
    }

    /* -------- FINAL CATEGORY -------- */

    if (finalList.length) {
      finalCategories.push({
        name: category.name,
        products: finalList,
      });
    }
  }

  return finalCategories;
};

/* ---------------- MAIN BUILDER ---------------- */

const buildPincodeCatalog = async (pincode) => {
  const normalizedPincode = String(pincode);

  log.info({
    info: `Building geo catalog for pincode: ${normalizedPincode}`,
  });

  /* -------- STEP 1: GET SHOPS -------- */

  const shops = await shopGeoService.getShopsByPincode(normalizedPincode);
  const shopIds = shops.map((s) => Number(s.shopId)).filter(Boolean);

  let categories = [];

  /* -------- STEP 2: FETCH RETAILER PRODUCTS -------- */

  if (shopIds.length) {
    const retailerDocs = await RetailerCatalog.find({
      shopId: { $in: shopIds },
      catalog: { $ne: null },
    }).lean();

    /* -------- STEP 3: BUILD DB PRODUCTS -------- */

    const retailerCategories = buildRetailerCategories(retailerDocs);

    /* -------- STEP 4: MERGE DB + AI -------- */

    categories = await buildTopProducts(retailerCategories);
  } else {
    log.warn({
      warn: `No shops found for pincode: ${normalizedPincode}`,
    });
  }

  /* -------- STEP 5: SAVE RAW DATA (NO FILTERING) -------- */

  const document = await GeoCatalog.findOneAndUpdate(
    { level: 'PINCODE', pincode: normalizedPincode },
    {
      $set: {
        level: 'PINCODE',
        pincode: normalizedPincode,
        categories, // 🔥 RAW DATA (IMPORTANT)
        lastBuildAt: new Date(),
        buildStatus: categories.length ? 'SUCCESS' : 'FALLBACK',
      },
    },
    { upsert: true, new: true }
  );

  log.info({
    info: `Geo catalog built successfully`,
    pincode: normalizedPincode,
    categoryCount: document.categories.length,
  });

  return {
    success: true,
    categories: document.categories,
  };
};

module.exports = {
  buildPincodeCatalog,
};
```

---

### File 7: `backend/catalogue_mgmt_service/src/apis/services/v1/geoHierarchy.service.js`

**Action: CREATE new file**

What this code does:

- Phase 2 service for hierarchy rebuild and apply
- Rebuilds `CITY`, `STATE`, `COUNTRY` level catalogs by aggregating `PINCODE` documents
- Provides fallback resolution: `PINCODE → CITY → STATE → COUNTRY`
- `/geo/apply` — applies geo catalog products to a shop's `customcatalog`

```js
const { Logger: log } = require('sarvm-utility');

const GeoCatalog = require('../../models/mongoCatalog/geoCatalogSchema');
const RetailerCatalog = require('../../models/mongoCatalog/retailerSchema');

const shopGeoService = require('./shopGeo.service');
const { getMasterCatalogSet } = require('./masterCatalog.service');

/* ---------------- HELPERS ---------------- */

const normalize = (str = '') =>
  String(str)
    .toLowerCase()
    .replace(/[^a-z0-9]/g, '')
    .trim();

const normalizeCategory = (str = '') =>
  String(str)
    .toLowerCase()
    .replace(/\s+/g, '_')
    .trim();

/**
 * Build shop category map
 */
const buildShopCategoryMap = (retailerDocs = []) => {
  const map = new Map();

  retailerDocs.forEach((doc) => {
    const categoryRaw =
      doc?.category ||
      doc?.catalog?.catPnm ||
      '';

    const category = normalizeCategory(
      String(categoryRaw).split('/')[0]
    );

    const name = String(doc?.catalog?.prdNm || '').trim();

    if (!category || !name) return;

    if (!map.has(category)) {
      map.set(category, []);
    }

    map.get(category).push({
      name,
      source: 'SHOP',
    });
  });

  return map;
};

/**
 * 🔥 FILTER GEO PRODUCTS USING MASTER CATALOG
 */
const filterGeoProducts = (geoProducts = [], masterSet) => {
  return geoProducts.filter((p) =>
    masterSet.has(normalize(p.name))
  );
};

/**
 * 🔥 CORE MERGE LOGIC
 *
 * 1. COMMON
 * 2. RETAILER
 * 3. GEO
 */
const mergeCategoryProducts = (shopProducts = [], geoProducts = []) => {
  const geoMap = new Map();
  const used = new Set();

  geoProducts.forEach((p) => {
    geoMap.set(normalize(p.name), p);
  });

  const common = [];
  const shopOnly = [];
  const geoOnly = [];

  /* -------- STEP 1: COMMON + SHOP -------- */

  for (const shopProduct of shopProducts) {
    const norm = normalize(shopProduct.name);

    if (geoMap.has(norm)) {
      const geoMatch = geoMap.get(norm);

      common.push({
        ...geoMatch,
        source: 'COMMON',
      });

      used.add(norm);
    } else {
      shopOnly.push({
        name: shopProduct.name,
        count: 0,
        source: 'SHOP',
      });
    }
  }

  /* -------- STEP 2: GEO ONLY -------- */

  for (const geoProduct of geoProducts) {
    const norm = normalize(geoProduct.name);

    if (!used.has(norm)) {
      geoOnly.push({
        ...geoProduct,
        source: 'GEO',
      });
    }
  }

  return [...common, ...shopOnly, ...geoOnly];
};

/* ---------------- MAIN FUNCTION ---------------- */

const getGeoCatalogWithFallback = async ({
  shopId,
  pincode,
  city,
  state,
  country,
}) => {
  try {
    /* -------- STEP 1: FETCH GEO CATALOG -------- */

    let geoCatalog = null;

    if (pincode) {
      geoCatalog = await GeoCatalog.findOne({
        level: 'PINCODE',
        pincode: String(pincode),
      })
        .sort({ updatedAt: -1 })
        .lean();
    }

    if (!geoCatalog && city) {
      geoCatalog = await GeoCatalog.findOne({
        level: 'CITY',
        city,
      })
        .sort({ updatedAt: -1 })
        .lean();
    }

    if (!geoCatalog && state) {
      geoCatalog = await GeoCatalog.findOne({
        level: 'STATE',
        state,
      })
        .sort({ updatedAt: -1 })
        .lean();
    }

    if (!geoCatalog && country) {
      geoCatalog = await GeoCatalog.findOne({
        level: 'COUNTRY',
        country,
      })
        .sort({ updatedAt: -1 })
        .lean();
    }

    if (!geoCatalog) return null;

    /* -------- STEP 2: IF NO SHOP → RETURN RAW -------- */

    if (!shopId) return geoCatalog;

    /* -------- STEP 3: GET MASTER CATALOG -------- */

    const masterSet = await getMasterCatalogSet();

    /* -------- STEP 4: FETCH SHOP PRODUCTS -------- */

    const retailerDocs = await RetailerCatalog.find({
      shopId: Number(shopId),
      catalog: { $ne: null },
    }).lean();

    if (!retailerDocs.length) {
      return geoCatalog;
    }

    const shopCategoryMap = buildShopCategoryMap(retailerDocs);

    /* -------- STEP 5: CATEGORY-WISE PROCESS -------- */

    const finalCategories = [];

    geoCatalog.categories.forEach((geoCategory) => {
      const categoryName = normalizeCategory(geoCategory.name);

      // ❗ Only include categories present in shop
      if (!shopCategoryMap.has(categoryName)) return;

      const shopProducts = shopCategoryMap.get(categoryName);
      const geoProducts = geoCategory.products || [];

      /* -------- 🔥 FILTER GEO PRODUCTS -------- */

      const filteredGeoProducts = filterGeoProducts(
        geoProducts,
        masterSet
      );

      /* -------- 🔥 MERGE -------- */

      const mergedProducts = mergeCategoryProducts(
        shopProducts,
        filteredGeoProducts
      );

      finalCategories.push({
        name: geoCategory.name,
        products: mergedProducts,
      });
    });

    return {
      ...geoCatalog,
      categories: finalCategories,
    };
  } catch (error) {
    log.error({
      error: 'Error in getGeoCatalogWithFallback',
      details: error.message,
    });
    throw error;
  }
};

module.exports = {
  getGeoCatalogWithFallback,
};
```

---

### File 8: `backend/catalogue_mgmt_service/src/apis/controllers/v1/geo.js`

**Action: CREATE new file**

What this code does:

- Exposes all geo service methods as HTTP endpoints
- Includes error handling that returns proper JSON error responses
- Includes manual cron trigger endpoint for testing

```js
const { Logger: log } = require('sarvm-utility');

const { buildPincodeCatalog } = require('../../services/v1/pincodeCatalogBuilder.service');
const {
  applyGeoCatalogToShop,
  getGeoCatalogWithFallback,
} = require('../../services/v1/geoHierarchy.service');

const shopGeoService = require('../../services/v1/shopGeo.service');
const { runGeoCatalogJobOnce } = require('../../../jobs/geoCatalog.job');

/* ---------------- TEST PINCODE ---------------- */

const testPincodeCatalog = async (req, res) => {
  try {
    const { pincode } = req.body;

    if (!pincode) {
      return res.status(400).json({
        success: false,
        message: 'pincode is required',
      });
    }

    const result = await buildPincodeCatalog(pincode);

    return res.status(200).json({
      success: true,
      pincode: String(pincode),
      categoryCount: result.categories.length,
      totalProducts: result.categories.reduce(
        (c, cat) => c + (cat.products || []).length,
        0
      ),
      categories: result.categories,
    });
  } catch (error) {
    log.error({
      error: 'Error in testPincodeCatalog',
      details: error.message,
    });

    return res.status(500).json({
      success: false,
      message: 'Internal server error',
    });
  }
};

/* ---------------- TEST SHOP ---------------- */

const testShopCatalog = async (req, res) => {
  try {
    const { shopId } = req.body;

    if (!shopId) {
      return res.status(400).json({
        success: false,
        message: 'shopId is required',
      });
    }

    const location = await shopGeoService.getShopLocation(Number(shopId));

    if (!location?.pincode) {
      return res.status(404).json({
        success: false,
        message: 'Pincode not found for shop',
      });
    }

    const result = await buildPincodeCatalog(location.pincode);

    return res.status(200).json({
      success: true,
      shopId: Number(shopId),
      pincode: location.pincode,
      categoryCount: result.categories.length,
      totalProducts: result.categories.reduce(
        (c, cat) => c + (cat.products || []).length,
        0
      ),
      categories: result.categories,
    });
  } catch (error) {
    log.error({
      error: 'Error in testShopCatalog',
      details: error.message,
    });

    return res.status(500).json({
      success: false,
      message: 'Internal server error',
    });
  }
};

/* ---------------- APPLY GEO CATALOG ---------------- */

const applyGeoCatalog = async (req, res) => {
  try {
    const { shopId, pincode } = req.body;

    if (!shopId) {
      return res.status(400).json({
        success: false,
        message: 'shopId is required',
      });
    }

    const result = await applyGeoCatalogToShop({
      shopId,
      pincode,
    });

    return res.status(200).json(result);
  } catch (error) {
    log.error({
      error: 'Error in applyGeoCatalog',
      details: error.message,
    });

    return res.status(500).json({
      success: false,
      message: 'Internal server error',
    });
  }
};

/* ---------------- GET GEO CATALOG ---------------- */

const getResolvedGeoCatalog = async (req, res) => {
  try {
    const { shopId, pincode } = req.query;

    const location = shopId
      ? await shopGeoService.getShopLocation(Number(shopId))
      : null;

    const catalog = await getGeoCatalogWithFallback({
      shopId: shopId ? Number(shopId) : null,
      pincode: pincode || location?.pincode,
      city: location?.city,
      state: location?.state,
      country: location?.country,
    });

    if (!catalog) {
      return res.status(404).json({
        success: false,
        message: 'No geo catalog found',
      });
    }

    return res.status(200).json({
      success: true,
      catalog,
    });
  } catch (error) {
    log.error({
      error: 'Error in getResolvedGeoCatalog',
      details: error.message,
    });

    return res.status(500).json({
      success: false,
      message: 'Internal server error',
    });
  }
};

/* ---------------- 🔥 MANUAL CRON TRIGGER ---------------- */

const triggerCronManually = async (req, res) => {
  try {
    log.info({ info: 'Manual geo cron trigger started' });

    // Run async (non-blocking)
    runGeoCatalogJobOnce();

    return res.status(202).json({
      success: true,
      message: 'Geo catalog cron job triggered successfully',
    });
  } catch (error) {
    log.error({
      error: 'Error triggering cron manually',
      details: error.message,
    });

    return res.status(500).json({
      success: false,
      message: 'Failed to trigger cron job',
    });
  }
};

/* ---------------- EXPORTS ---------------- */

module.exports = {
  testPincodeCatalog,
  testShopCatalog,
  applyGeoCatalog,
  getResolvedGeoCatalog,
  triggerCronManually, // 🔥 IMPORTANT
};
```

---

### File 9: `backend/catalogue_mgmt_service/src/apis/routes/v1/geo.js`

**Action: CREATE new file**

What this code does:

- Mounts all geo endpoints under `/geo`
- All routes use proper logging

Final URLs (base: `http://localhost:2210/cms/apis/v1`):

| Method | URL | Purpose | Phase |
|--------|-----|---------|-------|
| POST | `/geo/test/pincode` | Build catalog for one pincode | 1 |
| POST | `/geo/test/shop` | Build catalog by shopId | 1 |
| POST | `/geo/apply` | Apply geo catalog to shop | 2 |
| GET | `/geo/catalog` | Get catalog with fallback | 2 |
| POST | `/geo/cron/trigger` | Manually trigger cron job | 2 |

```js
const express = require('express');
const { Logger: log } = require('sarvm-utility');

const GeoController = require('../../controllers/v1/geo');

const router = express.Router();

// Phase 1 — test endpoints
router.post('/test/pincode', async (req, res, next) => {
  log.info({ info: 'Geo route :: test pincode catalog' });
  return GeoController.testPincodeCatalog(req, res, next);
});

router.post('/test/shop', async (req, res, next) => {
  log.info({ info: 'Geo route :: test shop catalog' });
  return GeoController.testShopCatalog(req, res, next);
});

// Phase 2 — production endpoints
router.post('/apply', async (req, res, next) => {
  log.info({ info: 'Geo route :: apply geo catalog' });
  return GeoController.applyGeoCatalog(req, res, next);
});

router.get('/catalog', async (req, res, next) => {
  log.info({ info: 'Geo route :: get resolved geo catalog' });
  return GeoController.getResolvedGeoCatalog(req, res, next);
});

// Phase 2 — manual cron trigger for testing
router.post('/cron/trigger', async (req, res, next) => {
  log.info({ info: 'Geo route :: manual cron trigger' });
  return GeoController.triggerCronManually(req, res, next);
});

module.exports = router;
```

---

### File 10: `backend/catalogue_mgmt_service/src/jobs/geoCatalog.job.js`

**Action: CREATE new file in new `jobs` directory**

What this code does:

- Runs full geo-catalog generation nightly at `2:00 AM IST`
- **Processes ONE pincode at a time** (sequential) to avoid overloading Gemini API rate limits
- Adds proper error handling per pincode — one failure does not stop the whole job
- Rebuilds hierarchy after all pincodes
- Includes delay between pincodes (configurable)

Cron schedule: `0 2 * * *` → minute 0, hour 2, every day

```js
const cron = require('node-cron');
const { Logger: log } = require('sarvm-utility');

const shopGeoService = require('../apis/services/v1/shopGeo.service');
const { buildPincodeCatalog } = require('../apis/services/v1/pincodeCatalogBuilder.service');
const { rebuildHierarchyCatalogs } = require('../apis/services/v1/geoHierarchy.service');

let isStarted = false;
let isRunning = false;

// Delay helper to avoid overloading APIs
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

// Configurable delay between pincode builds (ms)
const PINCODE_DELAY_MS = parseInt(process.env.GEO_CRON_PINCODE_DELAY_MS || '2000', 10);

/**
 * Run the full geo catalog job once.
 * Processes ONE pincode at a time (sequential, not parallel).
 * Each pincode may call Gemini API, so we space them out.
 */
const runGeoCatalogJobOnce = async () => {
  if (isRunning) {
    log.warn({ warn: 'Geo catalog job already running, skipping this invocation' });
    return;
  }

  isRunning = true;
  const startTime = Date.now();

  try {
    log.info({ info: 'Geo catalog job started' });

    const shopLocations = await shopGeoService.getAllShopLocations();

    const pincodes = [
      ...new Set(
        shopLocations.map((shop) => shop?.pincode).filter(Boolean)
      ),
    ];

    log.info({
      info: `Geo catalog job: processing ${pincodes.length} pincodes sequentially`,
    });

    let successCount = 0;
    let failCount = 0;

    // Process ONE pincode at a time
    for (let i = 0; i < pincodes.length; i++) {
      const pincode = pincodes[i];

      try {
        log.info({
          info: `Geo catalog job: processing pincode ${pincode} (${i + 1}/${pincodes.length})`,
        });

        await buildPincodeCatalog(pincode);
        successCount++;
      } catch (error) {
        failCount++;

        log.error({
          error: `Geo catalog job: failed for pincode ${pincode}`,
          details: error.message,
        });

        // Continue to next pincode — don't stop the whole job
      }

      // Delay between pincodes to avoid API rate limits
      if (i < pincodes.length - 1) {
        await delay(PINCODE_DELAY_MS);
      }
    }

    // Rebuild hierarchy after all pincodes are processed
    try {
      await rebuildHierarchyCatalogs();
      log.info({ info: 'Geo catalog job: hierarchy rebuild complete' });
    } catch (error) {
      log.error({
        error: 'Geo catalog job: hierarchy rebuild failed',
        details: error.message,
      });
    }

    const durationSeconds = Math.round((Date.now() - startTime) / 1000);

    log.info({
      info: 'Geo catalog job completed',
      pincodesProcessed: pincodes.length,
      successCount,
      failCount,
      durationSeconds,
    });
  } catch (error) {
    log.error({
      error: 'Geo catalog job fatal error',
      details: error.message,
      stack: error.stack,
    });
  } finally {
    isRunning = false;
  }
};

/**
 * Schedule the nightly cron job.
 * Runs at 2:00 AM IST every day.
 */
const startGeoCatalogJob = () => {
  if (isStarted) return;

  cron.schedule(
    '0 2 * * *',
    async () => {
      try {
        await runGeoCatalogJobOnce();
      } catch (error) {
        log.error({
          error: 'Geo catalog cron failed',
          details: error.message,
        });
      }
    },
    {
      scheduled: true,
      timezone: 'Asia/Kolkata',
    }
  );

  isStarted = true;

  log.info({ info: 'Geo catalog cron job scheduled at 2:00 AM IST' });
};

module.exports = {
  startGeoCatalogJob,
  runGeoCatalogJobOnce,
};
```

---

### File 11: `backend/catalogue_mgmt_service/src/apis/routes/v1/index.js`

**Action: EDIT existing file**

What to change: Add the geo router import and mount it.

Add this import near the top (after existing imports):

```js
const geoRouter = require('./geo');
```

Add this line in the router.use section:

```js
router.use('/geo', geoRouter);
```

Full updated file:

```js
const express = require('express');
const router = express.Router();

const categoryRouter = require('./category');
const productRouter = require('./product');
const publishRouter = require('./publish');
const catalogRoutere = require('./catalog');
const metaDataRoutere = require('./metaData/index');
const DataTree = require('./DataTree');
const requestMasterCatalog = require('./requestMasterCatalog');
const customCatalog = require('./customCatalog');
const geoRouter = require('./geo');

const retailerCatalogRouter = require('./retailerCatalog/index');
const bulkUpdateCatalog = require('./BulkUpdateProduct');

router.use('/customCatalog', customCatalog);
router.use('/geo', geoRouter);
router.use('/newProductReq', requestMasterCatalog);
router.use('/catalog', catalogRoutere);
router.use('/category', categoryRouter);
router.use('/product', productRouter);
router.use('/publish', publishRouter);
router.use('/metadata', metaDataRoutere);
router.use('/retailercatalog', retailerCatalogRouter);
router.use('/bulkupdate', bulkUpdateCatalog);
router.use('/dataTree', DataTree);

module.exports = router;
```

---

### File 12: `backend/catalogue_mgmt_service/src/InitApp/index.js`

**Action: EDIT existing file**

What to change: Add cron job startup after DB connections.

Add this import near the top:

```js
const { startGeoCatalogJob } = require('../jobs/geoCatalog.job');
```

Add this line after `Mongo.connect()`:

```js
startGeoCatalogJob();
```

Full updated file:

```js
const {
  Logger: log,
  ErrorHandler: { BaseError, INTERNAL_SERVER_ERROR, PAGE_NOT_FOUND_ERROR },
  ReqLogger,
  AuthManager,
} = require('sarvm-utility');
const express = require('express');
const cors = require('cors');
const IP = require('ip');
const cuid = require('cuid');
const _ = require('lodash');
const createNamespace = require('cls-hooked').createNamespace;
const config = require('../config');
const sessionName = config.session_name;
const session = createNamespace(sessionName);
const SqlDb = require('@db/SQL');
const { Mongo } = require('@db/index');
const consumer = require('../common/aws/index');
const { startGeoCatalogJob } = require('../jobs/geoCatalog.job');

const init = async (app) => {
  app.use(AuthManager.decodeAuthToken);
  app.use((req, res, next) => {
    session.run(() => {
      res.locals.sessionId = _.isUndefined(req.headers.sessionid) ? cuid() : req.headers.sessionid;

      try {
        res.locals.clientIp = _.isUndefined(req.headers.clientip)
          ? _.get(req, 'headers.x-forwarded-for', _.get(req, 'headers.X-Forwarded-For', IP.address()))
          : req.headers.clientip;
      } catch (err) {
        console.log(err);
      }

      session.set('sessionId', res.locals.sessionId);
      session.set('clientIp', res.locals.clientIp);
      next();
    });
  });

  app.use(express.json({ limit: '1mb' }));
  app.use(
    express.urlencoded({
      limit: '1mb',
      extended: true,
    }),
  );
  app.use(cors());

  if (!config.isTest) {
    app.use(ReqLogger);
  }

  const dbConnection = new SqlDb();
  dbConnection.connect();
  Mongo.connect((err) => {
    if (err) return console.error(err);
  });

  // Start nightly geo catalog cron job
  startGeoCatalogJob();

  // consumer.start()
};

module.exports = init;
```

---

### File 13: `backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/retailerSchema.js`

**Action: VERIFY existing file has `shopId` index**

The existing file should have `retailer_Catalog.index({ shopId: 1 })`. If not, add it. This makes pincode product queries fast.

---

### File 14: `backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/customCatalogSchema.js`

**Action: VERIFY existing file has `UNVERIFIED` enum**

Ensure `productNameStatus` includes `'UNVERIFIED'` in its enum. This is needed for `/geo/apply`.

---

### File 15: `.lcl.env` additions for Phase 2

**Action: EDIT existing `.lcl.env`**

Add these lines at the bottom:

```env
# Gemini AI (Phase 2)
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-2.0-flash

# Geo Cron Configuration
GEO_CRON_PINCODE_DELAY_MS=2000
```

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    BUILD PINCODE CATALOG                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. getShopsByPincode(pincode)                                 │
│     └─> shopIds[]                                              │
│                                                                 │
│  2. RetailerCatalog.find({ shopId: { $in: shopIds } })         │
│     └─> retailerProducts[]                                     │
│                                                                 │
│  3. Group by category, count product name frequency             │
│     └─> { dairy: { "Amul Gold Milk": 120, ... }, ... }         │
│                                                                 │
│  4. For each category: getGeminiCategoryProducts()              │
│     ├─ Fresh category → generic names (Tomato, Paneer)         │
│     └─ Packaged category → brand names (Lays, Parle-G)        │
│                                                                 │
│  5. Merge DB products + AI products (hybrid ranked list)        │
│     └─> DB products score by real frequency                     │
│     └─> AI products score by synthetic declining score          │
│                                                                 │
│  ★ 6. matchAgainstCatalogue(allProductNames)                   │
│     └─> Query Postgres `product` table                         │
│     └─> Only keep products that exist in catalogue             │
│                                                                 │
│  7. Store in geo_catalogs (MongoDB)                             │
│                                                                 │
│  8. Return catalogue-matched products only                      │
│     └─> Frontend shows ONLY valid catalogue products           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Postman Testing Guide

### Prerequisites

1. MongoDB running locally on port `27017`
2. PostgreSQL running locally on port `5432` with database `cms`
3. Backend started: `cd backend/catalogue_mgmt_service && npm run lcl`

Expected base URL: `http://localhost:2210/cms/apis/v1`

### Authorization Header

All requests need the system token:

```
Authorization: Bearer <your_system_token_from_env>
```

---

### TEST 1: Health Check

**When to test:** First, to verify backend is running.

```
GET http://localhost:2210/cms/apis/healthcheck
```

Expected: `200 OK` with `ts`, `buildNumber`, `serviceName`.

---

### TEST 2: Build Catalog by Pincode (Phase 1 — Primary Test)

**When to test:** After health check passes and both DBs are connected.

```
POST http://localhost:2210/cms/apis/v1/geo/test/pincode
Content-Type: application/json

{
  "pincode": "560100"
}
```

**What to verify in response:**

| Field | Expected |
|-------|----------|
| `success` | `true` |
| `pincode` | `"560100"` |
| `buildStatus` | `"SUCCESS"`, `"PARTIAL"`, or `"FALLBACK"` |
| `categoryCount` | `>= 1` |
| `totalProducts` | `>= 1` |
| `categories[].products[].name` | Each name must exist in your Postgres `product` table |
| `categories[].products[].catalogueProductId` | Non-null if catalogue matched |
| `categories[].products[].source` | `"DB"`, `"AI"`, or `"FALLBACK"` |

**How to verify catalogue matching actually worked:**

Take any product name from the response, e.g., `"Amul Gold Milk 500ml"`.
Run this SQL in your Postgres `cms` database:

```sql
SELECT id, name FROM product WHERE LOWER(name) = LOWER('Amul Gold Milk 500ml') AND status = 'ACTIVE';
```

The product **must exist**. If a generated product does NOT exist in the master catalogue, it will NOT appear in the API response.

---

### TEST 3: Build Catalog by ShopId (Phase 1 — Alternate Test)

**When to test:** If you know a valid shopId in your system.

```
POST http://localhost:2210/cms/apis/v1/geo/test/shop
Content-Type: application/json

{
  "shopId": 1234
}
```

**What to verify:** Same as TEST 2, plus:

| Field | Expected |
|-------|----------|
| `shopId` | `1234` |
| `pincode` | Auto-resolved from shop metadata |
| `city` | Auto-resolved |
| `state` | Auto-resolved |

---

### TEST 4: Verify Mongo Document (After TEST 2 or 3)

**When to test:** After a successful build.

Open MongoDB shell or Compass:

```js
db.geo_catalogs.find({ level: "PINCODE", pincode: "560100" }).pretty()
```

**Verify:**

- `categories` array exists and is non-empty
- Each category has `products` array
- Each product has `name`, `count`, `source`, `catalogueProductId`
- `buildStatus` is `"SUCCESS"` or `"PARTIAL"`
- `lastBuildAt` is a recent timestamp

---

### TEST 5: Get Catalog with Fallback (Phase 2)

**When to test:** After pincode document exists.

```
GET http://localhost:2210/cms/apis/v1/geo/catalog?pincode=560100
```

Or by shopId:

```
GET http://localhost:2210/cms/apis/v1/geo/catalog?shopId=1234
```

**Verify:**

- Returns the matching document
- Falls back through PINCODE → CITY → STATE → COUNTRY

---

### TEST 6: Apply Geo Catalog to Shop (Phase 2)

**When to test:** After TEST 4 confirms a geo_catalogs document exists.

```
POST http://localhost:2210/cms/apis/v1/geo/apply
Content-Type: application/json

{
  "shopId": 1234,
  "pincode": "560100"
}
```

**Expected response:**

```json
{
  "success": true,
  "level": "PINCODE",
  "categories": 5,
  "insertedProducts": 32
}
```

**Verify in MongoDB:**

```js
db.customcatalogs.find({ shopId: 1234 }).pretty()
```

Check:
- Products inserted
- `productNameStatus` = `"UNVERIFIED"`
- `category` and `subCategory` filled

---

### TEST 7: Manual Cron Trigger (Phase 2)

**When to test:** After all above tests pass, to test the full cron flow.

```
POST http://localhost:2210/cms/apis/v1/geo/cron/trigger
Content-Type: application/json
```

**Expected response:**

```json
{
  "success": true,
  "message": "Geo catalog cron job triggered. Check server logs for progress."
}
```

**Important:** This returns `202 Accepted` immediately. The job runs in the background. Check server logs for:

```
Geo catalog job: processing pincode 560100 (1/N)
Geo catalog job: processing pincode 560101 (2/N)
...
Geo catalog job completed - pincodesProcessed: N, successCount: X, failCount: Y
Geo catalog job: hierarchy rebuild complete
```

**Verify hierarchy after job completes:**

```js
db.geo_catalogs.find({ level: "PINCODE" }).count()
db.geo_catalogs.find({ level: "CITY" }).count()
db.geo_catalogs.find({ level: "STATE" }).count()
db.geo_catalogs.find({ level: "COUNTRY" }).count()
```

---

## API Test Sequence Summary

Follow this exact order:

| Step | API | Purpose | Phase |
|------|-----|---------|-------|
| 1 | `GET /healthcheck` | Verify backend running | 1 |
| 2 | `POST /v1/geo/test/pincode` | Build catalog for `560100` | 1 |
| 3 | MongoDB check | Verify `geo_catalogs` document | 1 |
| 4 | SQL check | Verify returned products exist in catalogue | 1 |
| 5 | `POST /v1/geo/test/shop` | Build by shopId | 1 |
| 6 | `GET /v1/geo/catalog` | Fetch with fallback | 2 |
| 7 | `POST /v1/geo/apply` | Apply to shop | 2 |
| 8 | MongoDB check | Verify `customcatalogs` | 2 |
| 9 | `POST /v1/geo/cron/trigger` | Run full cron | 2 |
| 10 | MongoDB check | Verify hierarchy docs | 2 |

---

## Phase 2 Changes After Phase 1 Implementation

After Phase 1 is working and tested, make these changes for Phase 2:

### 1. Install Gemini SDK

```bash
cd backend/catalogue_mgmt_service
npm install @google/generative-ai
```

### 2. Add Gemini API Key to `.lcl.env`

```env
GEMINI_API_KEY=your_actual_gemini_api_key
GEMINI_MODEL=gemini-2.0-flash
```

### 3. Verify `ai.service.js` auto-switches

The `ai.service.js` file already has Phase 2 Gemini code. It checks:

```js
if (!geminiAvailable || !process.env.GEMINI_API_KEY) {
  // Use fallback (Phase 1 behavior)
}
```

Once the SDK is installed and the API key is set, it automatically switches to real Gemini calls. **No code changes needed.**

### 4. Enable the cron job

The cron is already initialized in `InitApp/index.js`. It runs at 2 AM IST. To test manually:

```
POST http://localhost:2210/cms/apis/v1/geo/cron/trigger
```

### 5. Configure delay between pincodes

In `.lcl.env`:

```env
GEO_CRON_PINCODE_DELAY_MS=2000
```

This adds 2-second delay between processing each pincode, preventing Gemini API rate limit issues.

---

## Frontend

**No frontend changes are needed.**

Because the backend already returns only catalogue-matched products, the frontend receives valid product data. The existing `geo-catalog-test` page and any future integration will work as-is.

If you still want the test page for manual validation:

- Route: `/geo-catalog-test`
- Already configured in `app-routing.module.ts`:

```ts
{
  path: 'geo-catalog-test',
  loadChildren: () => import('./pages/geo-catalog-test/geo-catalog-test.module').then(m => m.GeoCatalogTestPageModule)
}
```

---

## New Files Summary

| File | Action | Phase |
|------|--------|-------|
| `src/apis/utils/normalizeProduct.js` | Edit | 1 |
| `src/apis/models/mongoCatalog/geoCatalogSchema.js` | Edit | 1 |
| `src/apis/services/v1/catalogueMatcher.service.js` | **CREATE** | 1 |
| `src/apis/services/v1/ai.service.js` | Edit | 1+2 |
| `src/apis/services/v1/shopGeo.service.js` | Create | 1 |
| `src/apis/services/v1/pincodeCatalogBuilder.service.js` | Create | 1 |
| `src/apis/services/v1/geoHierarchy.service.js` | Create | 2 |
| `src/apis/controllers/v1/geo.js` | Create | 1+2 |
| `src/apis/routes/v1/geo.js` | Create | 1+2 |
| `src/apis/routes/v1/index.js` | Edit (add geo route) | 1 |
| `src/jobs/geoCatalog.job.js` | **CREATE** | 2 |
| `src/InitApp/index.js` | Edit (add cron startup) | 2 |
| `.lcl.env` | Edit (add Gemini + cron vars) | 2 |

---

## Error Handling Summary

| Layer | Error | Handling |
|-------|-------|----------|
| Shop lookup | Remote API timeout | Falls back to local MongoDB collections |
| Shop lookup | No shops found for pincode | Uses fallback categories |
| Retailer query | MongoDB query error | Logs error, uses fallback categories |
| Catalogue matching | Postgres query fails for chunk | Skips that chunk, continues with others |
| Catalogue matching | No matches found | Returns empty categories (valid response) |
| Gemini API | SDK not installed | Auto-falls back to static list |
| Gemini API | API key missing | Auto-falls back to static list |
| Gemini API | Rate limit / timeout | Falls back to static list for that category |
| Gemini API | Invalid JSON response | Falls back to static list for that category |
| Cron job | One pincode fails | Logs error, continues to next pincode |
| Cron job | Already running | Skips second invocation (guard flag) |
| Cron job | Hierarchy rebuild fails | Logs error, pincode data still saved |
| Controller | Any unhandled error | Returns `500` with JSON error message |
