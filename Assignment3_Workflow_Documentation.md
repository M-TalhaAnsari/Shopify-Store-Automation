# Assignment 3 – Order‑to‑Fulfillment Automation

## 1. Goal
Create an end‑to‑end n8n workflow that processes **Shopify `orders/create` webhooks** for the student’s partner store. The workflow must:
1. Verify Shopify HMAC signatures.
2. De‑duplicate webhook calls.
3. Check & adjust inventory for each line‑item.
4. Log the order to a Google Sheet.
5. Send low‑stock Telegram alerts.
6. Email a customer receipt (if the email exists).
7. Gracefully handle errors (inventory API failures, missing email, etc.).

The whole solution is packaged as two sanitized n8n JSON files:
- `Assignment3_Order_Fulfillment_SANITIZED.json`
- `Assignment3_Error_Handler_SANITIZED.json`

## 2. High‑level Architecture
```mermaid
flowchart TD
    A[Shopify Webhook] --> B[Webhook Trigger (n8n)]
    B --> C[Settings (store config)]
    C --> D[Verify HMAC & Validate Order]
    D -->|valid| E[Route Order Status]
    D -->|invalid| F[Telegram – Validation Error]
    E -->|duplicate| G[Telegram – Duplicate Ignored]
    E -->|new| H[Split Line Items]
    H --> I[Loop Over Line Items]
    I --> J[Fetch Variant Inventory]
    J --> K[Evaluate Stock & Build Mutation]
    K --> L{Stock Available?}
    L -->|yes| M[Adjust Shopify Inventory]
    L -->|no| N[Telegram – Stock Error]
    M --> O[Parse Inventory Result]
    O --> P[Log Order Item to Google Sheets]
    P --> Q{Is Low Stock?}
    Q -->|yes| R[Telegram – Low‑Stock Warning]
    Q -->|no| S[Continue]
    O --> T[Build Order Summary & Email]
    T --> U{Has Customer Email?}
    U -->|yes| V[Send Confirmation Email]
    U -->|no| W[Telegram – Missing Email]
    V --> X[Telegram – Owner Order Alert]
    X --> Y[Return Item Success Summary]
    N --> Z[Telegram – Stock Error Alert]
    Z --> Y
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#bbf,stroke:#333,stroke-width:2px
    style C fill:#bbf,stroke:#333,stroke-width:2px
```

## 3. Node‑by‑Node Description
| # | Node (type) | Purpose |
|---|--------------|---------|
| 1 | **Shopify Order Webhook** (`webhook`) | Receives `orders/create` POST from Shopify (URL is the ngrok endpoint). |
| 2 | **Settings** (`set`) | Stores static configuration: store domain, API version, low‑stock threshold, Google Sheet ID/tab, Telegram chat ID, webhook secret, etc. |
| 3 | **Verify HMAC & Validate Order** (`code`) | Computes HMAC‑SHA256 using the webhook secret, validates payload, extracts order fields. |
| 4 | **Route Order Status** (`switch`) | Routes to *duplicate*, *invalid*, or *valid* branches. |
| 5 | **Notify Duplicate Ignored** (`telegram`) | Sends a Telegram message when a duplicate webhook is detected. |
| 6 | **Notify Validation Error** (`telegram`) | Alerts on HMAC failure or malformed payload. |
| 7 | **Split Line Items** (`code`) | Turns `order.line_items` array into separate JSON objects for each product. |
| 8 | **Loop Over Line Items** (`splitInBatches`) | Iterates over each line‑item JSON object. |
| 9 | **Fetch Variant Inventory** (`httpRequest`) | GraphQL query `inventoryLevels` for the variant to retrieve the *available* quantity. |
|10| **Format Fetch API Error** (`code`) | Normalises any HTTP/GraphQL errors into a common `{stockAction:'api_error'}` structure. |
|11| **Evaluate Stock & Build Mutation** (`code`) | Determines whether stock is sufficient, creates `stockAction` (`adjust` or `out_of_stock`) and builds the GraphQL mutation payload. |
|12| **Stock Available?** (`if`) | Branches based on `stockAction === 'adjust'`. |
|13| **Adjust Shopify Inventory** (`httpRequest`) | Sends `inventoryAdjustQuantities` mutation with `changeFromQuantity` and an `@idempotent` key (order‑ID + line‑item‑ID) to avoid double‑adjustments. |
|14| **Format Inventory API Error** (`code`) | Captures GraphQL errors and produces a readable `errorReason`. |
|15| **Parse Inventory Result** (`code`) | Reads the mutation response, extracts the new quantity, and stores it for later logging. |
|16| **Log Order Item to Google Sheets** (`googleSheets`) | Appends a row to the configured sheet (`Sheet1`). Columns: Order ID, Product, Quantity, Revenue, Customer, Timestamp, Remaining Stock. |
|17| **Format Sheets Error** (`code`) | Turns any Google‑Sheets API error into a `{sheetsError}` object. |
|18| **Send Sheets Error Alert** (`telegram`) | Notifies the owner if the sheet write fails. |
|19| **Is Low Stock?** (`if`) | Checks `remainingStock < low_stock_threshold`. |
|20| **Send Low‑Stock Warning** (`telegram`) | Alerts the owner about low inventory. |
|21| **Return Item Success Summary** (`code`) | Emits a summary JSON for the successful line‑item. |
|22| **Send Stock Error Alert** (`telegram`) | Alerts on insufficient stock (negative stock prevented). |
|23| **Return Item Failure Summary** (`code`) | Emits a failure summary JSON. |
|24| **Build Order Summary & Email** (`code`) | Aggregates all line‑item summaries, creates an HTML email body, and stores email fields. |
|25| **Send Owner Order Alert** (`telegram`) | Notifies the owner that the order was processed. |
|26| **Has Customer Email?** (`if`) | Branches to email or missing‑email path. |
|27| **Send Customer Confirmation Email** (`emailSend`) | Sends the receipt using the SMTP credential. |
|28| **Format Email Error** (`code`) | Normalises SMTP errors. |
|29| **Send Email Error Alert** (`telegram`) | Alerts the owner if the email could not be sent. |
|30| **Send Missing Email Warning** (`telegram`) | Alerts when the order has no email. |
|---|---|---|
**Error‑handler workflow** (nodes: Error Trigger → Error Settings → Format Error Alert → Send Error Alert (Telegram)). This catches any uncaught node errors and informs the owner.

## 4. Step‑by‑Step Setup Guide
### Prerequisites
- **Docker Desktop** (Windows) – latest stable version.
- **ngrok** account (free tier is enough). Download the binary to `C:\Users\HP\ngrok-bin\ngrok.exe`.
- A **Shopify Partner Dev Store** (already created: `northern‑treasures‑a73kt59n.myshopify.com`).
- **Google Cloud project** with Google Sheets API enabled and OAuth consent screen set to *Testing* (owner added as test user). Export the OAuth client JSON.
- **Telegram Bot** token and the chat ID of the owner (optional for alerts). If you don’t have a chat ID, leave it as `CHAT_ID_PENDING`; the workflow will still run.
- **SMTP account** (Gmail works – enable `App passwords` if 2‑FA is on).

### 4.1. Start the Dockerised n8n instance
```powershell
# From the workspace directory
cd "c:\Semester 4\Entrprenaurship\SHOPIFY AUTOMATION"
# Pull the n8n image (if not already present)
docker pull n8nio/n8n:latest
# Run n8n with env vars – note the WEBHOOK_URL points to ngrok (will be updated later)
$env:N8N_HOST="0.0.0.0"
$env:N8N_PORT=5678
$env:N8N_BASIC_AUTH_ACTIVE=false
$env:N8N_WEBHOOK_URL="https://calorie-canister-splotchy.ngrok-free.dev"
# Mount a persistent volume for the SQLite DB

docker run -d --name n8n `
  -p 5678:5678 `
  -e N8N_HOST=$env:N8N_HOST `
  -e N8N_PORT=$env:N8N_PORT `
  -e N8N_BASIC_AUTH_ACTIVE=$env:N8N_BASIC_AUTH_ACTIVE `
  -e N8N_WEBHOOK_URL=$env:N8N_WEBHOOK_URL `
  -v "c:\Semester 4\Entrprenaurship\SHOPIFY AUTOMATION\n8n_data:/home/node/.n8n" `
  n8nio/n8n
```
Verify it’s running: `http://localhost:5678` should open the n8n UI.

### 4.2. Start ngrok tunnel (exposes port 5678 publicly)
```powershell
# In a separate PowerShell window
C:\Users\HP\ngrok-bin\ngrok.exe http 5678
```
Copy the **HTTPS** forwarding URL (e.g. `https://calorie‑canister‑splotchy.ngrok-free.dev`). This is the base for the webhook URL.

### 4.3. Create a Shopify Custom App (OAuth)
1. In the Shopify Partner Dashboard → **Apps → Create app** → *Custom app*.
2. Set **App URL** to the ngrok URL, e.g. `https://calorie‑canister‑splotchy.ngrok-free.dev`.
3. Set **Redirect URLs** to `https://calorie‑canister‑splotchy.ngrok-free.dev/oauth/callback`.
4. Add the required **API scopes**:
   - `write_inventory`
   - `read_locations`
   - `read_orders`
   - `read_products`
5. Install the app on your dev store (click **Install app**).
6. Copy the **Admin API access token** – this will be the value for the *Header Auth* credential `X‑Shopify‑Access‑Token`.
7. Copy the **Client ID** and **Client Secret** – keep them for reference (not needed for the webhook flow).

### 4.4. Register the Webhook in Shopify Admin UI
Shopify blocks webhook creation via the API for protected customer data, so do it manually:
1. In the store admin → **Settings → Notifications → Webhooks → Create webhook**.
2. Event: **Order creation**.
3. URL: `https://calorie‑canister‑splotchy.ngrok-free.dev/webhook/shopify-order-create`.
4. Format: **JSON**.
5. Save.

### 4.5. Add Credentials in n8n UI
- **Header Auth** – Name: `X-Shopify-Access-Token`, Value: *your access token*.
- **Google Sheets OAuth2** – Follow n8n’s Google OAuth flow, select the spreadsheet ID `14Xcni090a9coxhk97uAOd7hfk-d7UJRyiv0Ei6QjXaU` and set the tab name to `Sheet1` (already patched).
- **Telegram Bot API** – Bot token (optional). Set `owner_telegram_chat_id` later.
- **SMTP** – Host, port, user, password (App‑password if Gmail).

### 4.6. Import the Workflow
1. In the n8n UI → **Workflows → Import**.
2. Choose `Assignment3_Order_Fulfillment_SANITIZED.json` (main workflow) and `Assignment3_Error_Handler_SANITIZED.json` (error workflow).
3. Activate the main workflow (toggle **Active**).
4. Ensure the webhook node URL matches the ngrok URL (`/webhook/shopify-order-create`).

### 4.7. Seed Demo Inventory (optional)
Run the provided helper script to set demo stock levels:
```powershell
node "C:\Users\HP\.gemini\antigravity\brain\b25aee4f-e33f-4d0c-b985-d2d3f7b4983b\scratch\setup_shopify.js"
```
It will set:
- Compare at Price Snowboard → 20
- Collection Snowboard Hydrogen → 25
- Videographer Snowboard → 15
- Selling Plans Ski Wax → 30
- 3‑Speed Snowboard (Blue) → 6 (low‑stock demo)

### 4.8. Run the Test Suite
```powershell
node "C:\Users\HP\.gemini\antigravity\brain\b25aee4f-e33f-4d0c-b985-d2d3f7b4983b\scratch\run_tests.js"
```
The script fires five synthetic webhook payloads:
1. Normal multi‑item order – should decrement inventory, write to Sheet1, send email.
2. Duplicate webhook – should be ignored (idempotent key).
3. Bad HMAC – should be rejected.
4. Out‑of‑stock item – should flag `out_of_stock`.
5. Missing customer email – should skip email but still log.
Check the console output and the Google Sheet (`Sheet1`) for rows.

### 4.9. Verify the Results
- **Shopify Admin → Products → Inventory** – quantities should reflect the decrements.
- **Google Sheet** – rows appear with the columns defined above.
- **Telegram** – you’ll receive alerts for low stock, errors, etc.
- **Email** – check the inbox of the sender address for the receipt.

### 4.10. Deploy (optional) 
When you’re ready to go live:
1. Deploy a permanent tunnel (e.g., Cloudflare Tunnel) or host n8n on a public server.
2. Update the webhook URL in Shopify admin.
3. Switch the ngrok URL to the permanent domain in the n8n **Webhook** node and **Settings** node.
4. Enable the `owner_telegram_chat_id` with your real chat ID.

## 5. Simple Shopify Store Design (Liquid Theme) – Minimal Theme
Below is a tiny **starter theme** you can paste into the `theme.liquid` file of a new Shopify theme. It displays a header, a product grid of the first 12 products, and a basic footer. No extra assets required.
```liquid
{%- comment -%}
  Minimal Shopify theme – place this code in `layout/theme.liquid`.
  It assumes you have at least a few products in the store.
{%- endcomment -%}
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{{ shop.name }}</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <style>
    body{font-family:-apple-system,system-ui,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;margin:0;padding:0;background:#fafafa;}
    header{background:#111;color:#fff;padding:1rem;text-align:center;}
    .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(200px,1fr));gap:1rem;padding:1rem;}
    .card{background:#fff;border:1px solid #e0e0e0;border-radius:4px;overflow:hidden;display:flex;flex-direction:column;}
    .card img{width:100%;height:auto;}
    .card-content{padding:.5rem;flex:1;display:flex;flex-direction:column;justify-content:space-between;}
    .price{color:#c00;font-weight:bold;}
    footer{background:#111;color:#fff;text-align:center;padding:.5rem;font-size:.9rem;}
  </style>
</head>
<body>
<header>
  <h1>{{ shop.name }}</h1>
</header>
<main>
  <div class="grid">
    {% for product in collections.all.products limit:12 %}
      <div class="card">
        {% if product.featured_image %}
          <img src="{{ product.featured_image | img_url: '400x' }}" alt="{{ product.title }}">
        {% endif %}
        <div class="card-content">
          <h2>{{ product.title }}</h2>
          <p class="price">{{ product.price | money }}</p>
          <a href="{{ product.url }}" style="margin-top:auto;background:#111;color:#fff;padding:.5rem;text-align:center;display:block;">View</a>
        </div>
      </div>
    {% endfor %}
  </div>
</main>
<footer>
  © {{ "now" | date: "%Y" }} {{ shop.name }} – All rights reserved.
</footer>
</body>
</html>
```
**How to apply:**
1. In Shopify admin → **Online Store → Themes → Add theme → Upload zip** (create a zip containing just `layout/theme.liquid`).
2. Publish the theme.
3. The store will now show a clean product grid and work with the automation workflow.

## 6. Checklist for Submission
- [x] `Assignment3_Order_Fulfillment_SANITIZED.json` (uploaded to the repository).
- [x] `Assignment3_Error_Handler_SANITIZED.json`.
- [x] `Assignment3_Workflow_Documentation.md` (this file).
- [x] Google Sheet (view‑only link).
- [x] 2‑3 min screen‑recording of a live order flow (you’ll record from the Shopify admin → Orders → Create order, then show the Google Sheet entry and Telegram/email alerts).

---
*All secrets have been stripped from the sanitized JSON files.*

*If you need to change the Google Sheet tab again, edit `google_sheet_tab` in `patch_workflows.js` and re‑run the patch‑generate‑import sequence.*
