# Deployment & Runbook

**Project:** SAP IEE2E Learning & RAG Evaluation Platform  
**Environment:** Cloudflare Ecosystem (Free Tier)

---

## 1. Prerequisites

Before initiating the setup, the Operator's local machine must have the following:

- **Node.js** (Version 18.x or higher installed).
- **Cloudflare Account:** A registered account with a verified payment method (Credit/Debit Card). _Note: Cloudflare requires a card on file to prevent spam, but you will not be charged as long as you stay within the generous Free Tier limits._
- **Gemini API Key:** Generated from Google AI Studio.

---

## 2. Infrastructure Provisioning (Step-by-Step)

Open your terminal and execute the following commands in order.

### Step 1: Install Cloudflare CLI (Wrangler) and Authenticate

```bash
npm install -g wrangler
wrangler login
```

    *(Your browser will open. Log in to your Cloudflare account and authorize Wrangler).*

### Step 2: Initialize the Backend Project (Cloudflare Workers)

```bash
npm create cloudflare@latest sap-backend
```

\*Interactive Prompts:

    * Type of application: `"Hello World" Worker`

    * Do you want to use TypeScript? `Yes`

    * Do you want to deploy your application? `No` *(We must configure the DB and bindings first)*.

- Navigate into the project directory:

```bash
cd sap-backend
```

### Step 3: Provision the Relational Database (Cloudflare D1)

```bash
wrangler d1 create sap-db
```

- Upon success, Wrangler will output a configuration block. Open the `wrangler.toml` file in your project root and append this block to the bottom:

```ini, TOML
[[d1_databases]]
binding = "DB"
database_name = "sap-db"
database_id = "<PASTE_THE_ID_FROM_TERMINAL_HERE>"
```

### Step 4: Provision Object Storage (Cloudflare R2)

```bash
wrangler r2 bucket create sap-notes
```

- Append the following to your `wrangler.toml` file:

```ini, TOML
[[r2_buckets]]
binding = "R2"
bucket_name = "sap-notes"
```

### Step 5: Provision High-Speed Cache (Cloudflare KV)

_This is used for the autosave feature to protect your D1 write quota._

```bash
wrangler kv:namespace create SAP_KV
```

- Append the output to your `wrangler.toml` file:

```ini, TOML
[[kv_namespaces]]
binding = "KV"
id = "<PASTE_THE_ID_FROM_TERMINAL_HERE>"
```

### Step 6: Provision the Vector Database (Cloudflare Vectorize)

```bash
wrangler vectorize create sap-vectors --dimensions=768 --metric=cosine
```

- Append the following to your `wrangler.toml` file:

```ini, TOML
[[vectorize]]
binding = "VECTORIZE"
index_name = "sap-vectors"
```

---

## 3. Secret Management (Environment Variables)

NEVER hardcode your Gemini API Key in plain text inside your codebase. Inject it securely into Cloudflare's encrypted secrets vault.

```bash
wrangler secret put GEMINI_API_KEY
```

_(The terminal will prompt you to enter the value. Paste your Gemini API Key and press Enter)._

Inside your Worker code (`src/index.ts`), you can safely access this key via `env.GEMINI_API_KEY`.

---

## 4. Initialize Database Schema

1. Copy all the SQL commands from the Technical Architecture & Database Schema document.

2. Create a new file named schema.sql in the root of your sap-backend directory and paste the SQL commands into it.

3. Execute the following command to push the tables to your live Cloudflare D1 database:

```bash
wrangler d1 execute sap-db --file=./schema.sql --remote
```

---

## 5. Backend Deployment

Once you have written your API routes in `src/index.ts`, execute the following command to deploy the backend to the global edge network:

```bash
wrangler deploy
```

_Wrangler will output a live URL (e.g., https://sap-backend.<your-username>.workers.dev). This is your Base URL that the React Frontend will use to make API calls._

---

## 6. Frontend Deployment (React via Cloudflare Pages)

1. Open a new terminal window, navigate to your React frontend directory, and build the production bundle:

```bash
npm run build
```

_(Ensure your .env file in the frontend points to the new Worker Base URL before building)._

2. Deploy the build output directory (usually named build or dist) to Cloudflare Pages:

```bash
wrangler pages deploy dist
```

_(Cloudflare will provide a public URL for your web app. Distribute this link to your team of 12)._
