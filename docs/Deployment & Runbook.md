# Deployment & Runbook (Updated 2026)

**Project:** SAP IEE2E Learning & RAG Evaluation Platform  
**Environment:** Cloudflare Ecosystem (Free Tier) - Wrangler 3.x/4.x

---

## 1. Prerequisites

Before initiating the setup, ensure your local machine has:

- **Node.js** (Version 18.x or higher).
- **Cloudflare Account:** Verified with a payment method (no charges on Free Tier).
- **Gemini API Key:** Ready from Google AI Studio.

---

## 2. Infrastructure Provisioning (Step-by-Step)

Open your terminal and execute the following commands in order.

### Step 1: Install Wrangler & Authenticate

```bash
npm install -g wrangler
wrangler login
```

### Step 2: Initialize Backend Project

```bash
npm create cloudflare@latest sap-backend
```

- **Interactive Prompts:**
  - Type of application: Worker
  - Do you want to use TypeScript? Yes
  - Do you want to deploy your application? No
- **Navigate into the project:**
  ```bash
  cd sap-backend
  ```

### Step 3: Provision Relational Database (D1)

```bash
npx wrangler d1 create sap-db
```

- **Interactive Prompts:**
  - Would you like Wrangler to add it on your behalf? Yes
  - What binding name would you like to use? sap_db
  - For local dev, do you want to connect to the remote resource...? No (Crucial: Select No to protect your production data during local testing).

### Step 4: Provision Object Storage (R2)

```bash
npx wrangler r2 bucket create sap-notes
```

- **Interactive Prompts:**
  - Would you like Wrangler to add it on your behalf? Yes
  - What binding name would you like to use? sap_notes_r2

### Step 5: Provision High-Speed Cache (KV)

```bash
npx wrangler kv:namespace create SAP_KV
```

- **Interactive Prompts:**
  - Would you like Wrangler to add it on your behalf? Yes
  - What binding name would you like to use? sap_kv

### Step 6: Provision Vector Database (Vectorize)

```bash
npx wrangler vectorize create sap-vectors --dimensions=768 --metric=cosine
```

- **Interactive Prompts:**
  - Would you like Wrangler to add it on your behalf? Yes
  - What binding name would you like to use? sap_vectorize

### Step 7: Generate TypeScript Types (CRITICAL)

Since you updated bindings in wrangler.json, you must regenerate the environment types for VSCode autocompletion:

```bash
npx wrangler types
```

---

## 3. Secret Management

Never expose your API keys in code. Store them in Cloudflare's encrypted vault.

```bash
npx wrangler secret put GEMINI_API_KEY
```

_(Paste your actual Gemini API key when prompted and press Enter)._

---

## 4. Initialize Database Schema

1. Create a schema.sql file in the root of sap-backend.
2. Paste the SQL structure from the Architecture doc into it.
3. Run both commands to setup your local and production databases:

- **Setup Local DB (for testing):**
  ```bash
  npx wrangler d1 execute sap-db --file=./schema.sql --local
  ```
- **Setup Production DB (for the live app):**
  ```bash
  npx wrangler d1 execute sap-db --file=./schema.sql --remote
  ```

---

## 5. Development & Deployment

### Run Locally (Testing)

To test your API endpoints on your machine:

```bash
npx wrangler dev
```

### Deploy to Production (Backend)

When ready to go live:

```bash
npx wrangler deploy
```

### Deploy Frontend (React / Pages)

From your React frontend directory:

```bash
npm run build
npx wrangler pages deploy dist
```
