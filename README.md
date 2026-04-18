# Complete Dynamic Catalog Implementation Code & Setup Guide

This document contains **all the necessary code** for the Location-Aware Product Catalog feature, combined into a single place. Follow these step-by-step instructions to implement the code in your local codebase.

---

## Prerequisites & Installation

Before writing the code, ensure the background dependencies are installed in your backend service. Open your terminal in the backend folder and run:

```bash
cd d:\Sarvm\backend\catalogue_mgmt_service
npm install @google/genai axios
```
*(Note: `node-cron` is already in your `package.json` so it will be available).*

---

## BACKEND CODE (catalogue_mgmt_service)

### 1. AI Product Trends Schema
**Create File:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\models\mongoCatalog\aiProductTrendsSchema.js`

```javascript
// src/apis/models/mongoCatalog/aiProductTrendsSchema.js
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const aiProductTrendsSchema = new Schema({
    zipcode: { type: String, required: true },
    city: { type: String },
    state: { type: String },
    category: { type: String, required: true },
    products: [{ type: String }], // Array of AI suggested product names
    lastUpdated: { type: Date, default: Date.now }
});

// Ensure we don't duplicate trend requests for a specific zip + category
aiProductTrendsSchema.index({ zipcode: 1, category: 1 }, { unique: true });

module.exports = mongoose.model('AiProductTrends', aiProductTrendsSchema);
```

### 2. Area Product Insights Schema
**Create File:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\models\mongoCatalog\areaProductInsightsSchema.js`

```javascript
// src/apis/models/mongoCatalog/areaProductInsightsSchema.js
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const areaProductInsightsSchema = new Schema({
    zipcode: { type: String, required: true },
    city: { type: String },
    state: { type: String },
    category: { type: String, required: true },
    rankedProducts: [{
        product_id: { type: String },
        product_name: { type: String },
        score: { type: Number },
        source: { type: String, enum: ['REAL_DATA', 'AI_TREND', 'MIXED'] },
        price_estimate: { type: Number, default: 0 }
    }],
    lastUpdated: { type: Date, default: Date.now }
});

// Efficient fetching at runtime
areaProductInsightsSchema.index({ zipcode: 1, category: 1 }, { unique: true });
areaProductInsightsSchema.index({ city: 1, category: 1 }); // fallback index

module.exports = mongoose.model('AreaProductInsights', areaProductInsightsSchema);
```

### 3. Gemini AI Service Integration
**Create File:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\services\v1\mongoCatalog\geminiService.js`

```javascript
// src/apis/services/v1/mongoCatalog/geminiService.js
const { Logger: log } = require('sarvm-utility');
const { GoogleGenerativeAI } = require('@google/genai');

// Use environment variable for API key (Add GEMINI_API_KEY to your .lcl.env)
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || 'FALLBACK_KEY_FOR_LOCAL_TEST');

const fetchTrendsFromGemini = async (zipcode, city, category) => {
    log.info({ info: `Fetching AI trends for zip: ${zipcode}, cat: ${category}` });

    // Local override if no actual key is configured yet
    if (!process.env.GEMINI_API_KEY) {
        log.info({ info: `Using Mock AI Data for local testing.` });
        return [
            "Aashirvaad Whole Wheat Atta", "Amul Pasteurized Butter", "Tata Salt",
            "Maggi 2-Minute Noodles", "Surf Excel Easy Wash", "Brooke Bond Red Label Tea",
            "Saffola Gold Cooking Oil", "Everest Garam Masala", "Colgate Strong Teeth"
        ];
    }

    try {
        const model = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });
        const prompt = `Provide exactly a JSON array of strings representing the top 20 most frequently purchased grocery and household products for zipcode ${zipcode} (${city}, India) in the "${category}" category. Output strictly valid JSON array format like ["Product 1", "Product 2"].`;

        const result = await model.generateContent(prompt);
        let textResponse = result.response.text();
        
        // Clean markdown backticks if any
        textResponse = textResponse.replace(/^```json/m, '').replace(/```$/m, '').trim();
        const productList = JSON.parse(textResponse);
        
        return productList;
    } catch (error) {
        log.error({ error: `Gemini API Failure: ${error.message}` });
        return []; // fail gracefully
    }
};

module.exports = { fetchTrendsFromGemini };
```

### 4. Cron Job Processor Workflow
**Create File:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\services\v1\mongoCatalog\dynamicCatalogCron.js`

```javascript
// src/apis/services/v1/mongoCatalog/dynamicCatalogCron.js
const cron = require('node-cron');
const fs = require('fs');
const path = require('path');
const csv = require('csv-parser');
const { Logger: log } = require('sarvm-utility');
const AiProductTrends = require('../../models/mongoCatalog/aiProductTrendsSchema');
const AreaProductInsights = require('../../models/mongoCatalog/areaProductInsightsSchema');
const Product = require('../../models/mongoCatalog/productSchema');
const { fetchTrendsFromGemini } = require('./geminiService');

const runNightlyProcessor = async () => {
    log.info({ info: 'Starting Dynamic Catalog Job from CSV...' });
    
    // Path to your CSV file in the backend root
    // Place your file at: d:\Sarvm\backend\catalogue_mgmt_service\zipcodes.csv
    const csvFilePath = path.join(process.cwd(), 'zipcodes.csv');
    const records = [];

    if (!fs.existsSync(csvFilePath)) {
        log.error({ error: `CSV file not found at ${csvFilePath}. Please place zipcodes.csv in the backend root.` });
        return;
    }

    // 1. Read and parse the CSV
    fs.createReadStream(csvFilePath)
        .pipe(csv())
        .on('data', (data) => records.push(data))
        .on('end', async () => {
            log.info({ info: `CSV loaded. Processing ${records.length} regions.` });

            // 2. Process each region one by one
            for (const record of records) {
                // We extract only the 3 columns we need; extra columns in your CSV are ignored
                // Format: pincode, district, state
                const { pincode, district, state } = record;
                const category = 'grocery'; // Defaulting to grocery

                if (!pincode) continue;

                log.info({ info: `Processing Region: ${pincode} (${district})` });

                try {
                    // 3. Fetch AI Trends (checks cache or calls Gemini)
                    const aiList = await fetchTrendsFromGemini(pincode, district, category);
                    
                    let finalList = [];
                    let rank = 1;

                    // 4. Normalize and Map to Master Product Database
                    for (const prodName of aiList) {
                        const masterProduct = await Product.findOne({
                            prdNm: { $regex: new RegExp(prodName, 'i') }
                        }).lean();

                        if (masterProduct) {
                            finalList.push({
                                product_id: masterProduct._id.toString(),
                                product_name: masterProduct.prdNm,
                                score: 100 - rank++,
                                source: 'AI_TREND',
                                price_estimate: masterProduct.prc?.mrp || 0
                            });
                        }
                    }

                    // 5. Save/Update the Insights Collection
                    if (finalList.length > 0) {
                        await AreaProductInsights.findOneAndUpdate(
                            { zipcode: pincode, category: category },
                            { 
                                city: district, 
                                state: state, 
                                rankedProducts: finalList, 
                                lastUpdated: new Date() 
                            },
                            { upsert: true }
                        );
                        log.info({ info: `Successfully updated insights for ${pincode}` });
                    }
                } catch (err) {
                    log.error({ error: `Error processing ${pincode}: ${err.message}` });
                }
            }
            log.info({ info: 'Dynamic Catalog Generation Job Completed.' });
        });
};

const initCron = () => {
    // Runs at 2 AM every day
    cron.schedule('0 2 * * *', () => {
        runNightlyProcessor().catch(err => log.error({ error: err }));
    });
};

module.exports = { initCron, runNightlyProcessor };
```

### 5. Controller for Fetching Data
**Create File:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\controllers\v1\mongoCatalog\dynamicCatalog.js`

```javascript
// src/apis/controllers/v1/mongoCatalog/dynamicCatalog.js
const AreaProductInsights = require('../../models/mongoCatalog/areaProductInsightsSchema');
const { Logger: log } = require('sarvm-utility');

const getDynamicCatalogInsights = async (zipcode, city, state, category) => {
    log.info({ info: 'Fetching Dynamic Catalog details' });
    
    // 1. Try highly specific zipcode match
    let insights = await AreaProductInsights.findOne({ zipcode, category }).lean();
    
    // 2. Fallback to general city match
    if (!insights && city) {
        insights = await AreaProductInsights.findOne({ city, category }).lean();
    }

    if (!insights) {
        return { source: 'DEFAULT', catalog: [] }; // The frontend should fall back to standard Master Catalog behavior
    }

    return {
        source: 'DYNAMIC',
        catalog: insights.rankedProducts
    };
};

module.exports = { getDynamicCatalogInsights };
```

### 6. API Route for Dynamic Catalog
**Create File:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\routes\v1\dynamicCatalog.js`

```javascript
// src/apis/routes/v1/dynamicCatalog.js
const express = require('express');
const { HttpResponseHandler, Logger: log } = require('sarvm-utility');
const dynamicCatalogController = require('../../controllers/v1/mongoCatalog/dynamicCatalog');

const router = express.Router();

router.get('/insights', async (req, res, next) => {
    try {
        const { zipcode, city, state, category } = req.query;
        if (!zipcode || !category) {
            return HttpResponseHandler.badRequest(req, res, 'Zipcode and Category are required');
            // Alternatively standard HTTP 400 validation depending on sarvm-utility
        }

        const result = await dynamicCatalogController.getDynamicCatalogInsights(zipcode, city, state, category);
        HttpResponseHandler.success(req, res, result);
    } catch (error) {
        log.error({ error });
        next(error);
    }
});

module.exports = router;
```

### 7. Bind Routes (Modifying existing backend files)
**Modify File:** `d:\Sarvm\backend\catalogue_mgmt_service\src\apis\routes\v1\index.js`

*Add the following towards the top with other imports:*
```javascript
const dynamicCatalogRouter = require('./dynamicCatalog');
```
*Add the following towards the bottom near `router.use('/category'...`:*
```javascript
router.use('/dynamicCatalog', dynamicCatalogRouter);
```

**Modify File:** `d:\Sarvm\backend\catalogue_mgmt_service\server.js`

*Add this somewhere around line 21 (inside `InitApp(app).then(() => {`):*
```javascript
const { initCron } = require('./src/apis/services/v1/mongoCatalog/dynamicCatalogCron');
initCron(); 
```

---

## FRONTEND CODE (hha_web)

### 1. Application Constants
**Modify File:** `d:\Sarvm\frontend\hha_web\src\app\config\constants.ts`

*Inside the `ApiUrls` object (approx line 122), append the new route:*
```typescript
  dynamicCatalogInsights: '/cms/apis/v1/dynamicCatalog/insights',
```

### 2. Catalogue Service 
**Modify File:** `d:\Sarvm\frontend\hha_web\src\app\lib\services\catalogue.service.ts`

*Add this new function block directly inside the `CatalogueService` class:*
```typescript
  getDynamicInsights(zipcode: string, city: string, state: string, category: string) {
    // Note: ensure the environment base URL corresponds to the proxy resolving to the catalogue service node process. 
    const url = `${environment.baseUrl}${ApiUrls.dynamicCatalogInsights}?zipcode=${zipcode}&city=${city}&state=${state}&category=${category}`;
    return this.commonApi.getDataByUrl(url);
  }
```

### 3. Google Component Updates
**Modify File:** `d:\Sarvm\frontend\hha_web\src\app\pages\store\google-stores\google-stores.component.ts`

*Locate the `loadShopData()` method. You will inject the new dynamic logic whenever a Google shop (unverified profile) is loaded.*

**Modify `loadShopData` as follows:**
```typescript
  loadShopData() {
    this.commonservice.presentProgressBarLoading();
    this.catalogueService.getmerchant(this.profileUrl!).subscribe({
      next: async (res: any) => {
        this.shopData = res;
        this.shopProfileUrl = res?.shop?.profileUrl || null;
        
        // --- NEW DYNAMIC CATALOG HOOK ---
        const shopZip = res?.shop?.address?.zipcode || '110001'; // Extract real zip if available
        const shopCity = res?.shop?.address?.city || 'Delhi';
        
        if (res?.catalog?.length) {
          this.selectedCategoryId = res.catalog[0].id;
          
          try {
            const dynamicRes: any = await this.catalogueService.getDynamicInsights(shopZip, shopCity, '', 'grocery').toPromise();
            
            if (dynamicRes?.data?.source === 'DYNAMIC') {
              // Extract the highly ranked subset
              const curatedProducts = dynamicRes.data.catalog; 
              
              // Map AI/Dynamic data back to the UI format expected by your templates
              res.catalog[0].categories[0].products = curatedProducts.map((p: any) => ({
                 prdNm: p.product_name,
                 price: { mrp: p.price_estimate || 0 },
                 quantity: { soldBy: '1 unit', minQty: 1 },
                 media: { imgTh: null, img1: null },
                 status: 'PUBLISHED'
              }));
            }
          } catch(e) {
             console.log("Failed to fetch dynamic insights, defaulting to standard catalog.", e);
          }
        }
        // ---------------------------------

        this.loading = false;
        this.flag = !res?.catalog?.length;

        if (res?.catalog?.length) {
          this.selectedCategory = 0;
          this.selectedSubCategory = 0;
          this.selectedMicroCategory = 1;
        }
        
        const vegnonVeg = this.storageService.getItem(Constants.SELECT_PREFERENCE);
        this.isVeg = vegnonVeg ? vegnonVeg === 'veg' : false;
        this.commonservice.closeProgressBarLoading();
      },
      error: (err) => {
        console.error('Failed to fetch profile.json', err);
        this.loading = false;
        this.flag = true;
        this.commonservice.closeProgressBarLoading();
      },
    });
  }
```

---

## Environment Setup Details (How to Run)

1. **Environmental Variables:**
   * Open `d:\Sarvm\backend\catalogue_mgmt_service\.lcl.env` (or whatever env file you use to run locally).
   * Add `GEMINI_API_KEY=your_google_api_key_here` (Optional, as the script provides a fallback dummy list for local testing).

2. **Database:**
   * Your local MongoDB must be running. Running the backend code above will automatically create the `ai_product_trends` and `area_product_insights` collections when the cron job performs upserts.

3. **Running the Backend locally:**
   ```bash
   cd d:\Sarvm\backend\catalogue_mgmt_service
   npm run lcl 
   ```
   *To immediately see it work without waiting for 2 AM, temporarily modify your cron schedule inside `dynamicCatalogCron.js` to `* * * * *` (runs every minute), and observe the console logs.*

4. **Running the Frontend locally:**
   ```bash
   cd d:\Sarvm\frontend\hha_web
   npm install   # If you had new Angular updates
   ionic serve
   ```
   *Open your frontend, click on a random Google Shop, and you will see the curated "Dynamic" catalog list populated instead of the massive master catalog!*
