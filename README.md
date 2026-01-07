# DeployingToAzure

# Deploying an Express App (Monorepo) to Azure Web App using GitHub Actions (Code Deploy)

This guide documents a **working, repeatable approach** to deploying an **Express + Nunjucks** application (including **monorepo/subfolder apps**) to **Azure App Service (Linux, code-based)** using **GitHub Actions** and a **Publish Profile**.

This approach avoids Docker, ACR, private DNS, and container pull issues.

---

## Why this approach works

- No Docker images
- No ACR / registry DNS
- No VNet / private endpoint dependency
- Uses standard Azure App Service runtime
- Simple GitHub Actions workflow
- Ideal for secured environments (HMCTS-style platforms)

---

## Prerequisites

- Azure CLI installed and logged in:
  ```bash
  az login
- GitHub repo with your Express app
- Your app must listen on process.env.PORT:
  ```javascript
    const port = process.env.PORT || 3000;
    app.listen(port);

## Key gotchas (learned the hard way) 😭
- .github/workflows must live at repo root
- Deploy subfolder, not repo root, in monorepos
- Publish profile secret must be a repository secret
- Express must listen on process.env.PORT
- Static assets must be explicitly served

## Step 1: Define variables
```bash
  # Azure
  RG="CFT-software-engineering"
  LOCATION="uksouth"
  
  # App Service
  PLAN_NAME="hmctsdesignsystem-plan"
  WEBAPP_NAME="hmctsdesignsystem-code"   # must be globally unique
  
  # Runtime
  RUNTIME="NODE:20-lts"
  
  # Monorepo path (folder containing package.json)
  APP_DIR="packages/hmcts-docs"
  
  # (Optional) Set subscription:
  az account set --subscription <SUBSCRIPTION_ID>
```

## Step 2: Create an App Service Plan (Linux)
```bash
  az appservice plan create \
    --name "$PLAN_NAME" \
    --resource-group "$RG" \
    --location "$LOCATION" \
    --is-linux \
    --sku B1
```
## Step 3: Create a code-based Linux Web App (NOT container)
```bash
  az webapp create \
    --resource-group "$RG" \
    --plan "$PLAN_NAME" \
    --name "$WEBAPP_NAME" \
    --runtime "$RUNTIME"

  #Verify it is not container-based
  az webapp show \
    --name "$WEBAPP_NAME" \
    --resource-group "$RG" \
    --query "{kind:kind, linuxFxVersion:siteConfig.linuxFxVersion}" \
    -o json
```
### Expected:
- kind includes app,linux
- linuxFxVersion is not DOCKER|...

## Step 4: (Optional) Configure app settings
```bash
az webapp config appsettings set \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG" \
  --settings NODE_ENV=production
```
### Add any other required environment variables here.

## Step 5: Download the Publish Profile (DO NOT COMMIT)
```bash
az webapp deployment list-publishing-profiles \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG" \
  --xml > publishProfile.xml
```
### Add to .gitignore:
```bash
echo "publishProfile.xml" >> .gitignore
git add .gitignore
git commit -m "Ignore Azure publish profile"
```
### Copy contents to clipboard (Mac):
```bash
pbcopy < publishProfile.xml
```
### Add to GitHub Secrets
1. Repo → Settings
2. Secrets and variables → Actions
3. New repository secret
4. Name: AZURE_WEBAPP_PUBLISH_PROFILE
5. Value: paste the full XML
   
⚠️ Treat this like a password!

## Step 6: GitHub Actions workflow (monorepo-aware)

Create this file at repo root:

```bash
  .github/workflows/deploy.yml
```
In this new file copy and paste: 

```yaml
name: Deploy to Azure Web App

on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: packages/hmcts-docs/package-lock.json

      - name: Install dependencies
        working-directory: packages/hmcts-docs
        run: npm ci

      - name: Build (if present)
        working-directory: packages/hmcts-docs
        run: npm run build --if-present

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: hmctsdesignsystem-code
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: packages/hmcts-docs

```
## Step 7: Fix missing CSS / assets (very common)

### Symptom

Browser console shows:

`Refused to apply style ... MIME type ('text/html')`

This means the CSS URL is returning HTML, not CSS.

### Step 7.1: Locate CSS in Azure (Kudu)
Open:
```bash
  https://<WEBAPP_NAME>.scm.azurewebsites.net/DebugConsole
```
Run:
```bash
  find /home/site/wwwroot -maxdepth 6 -type f -name "*.css"
```
Example output:
```bash
  ./packages/hmcts-frontend/dist/hmcts.css
```

### Step 7.2: Serve assets in Express
If your HTML references:
` /assets/hmcts.css`
Add this before routes in your Express app:
```javascript
const path = require("path");

app.use(
  "/assets",
  express.static(
    path.join(__dirname, "packages", "hmcts-frontend", "dist")
  )
);
```
After redeploy:
[Content-Type: text/css] (https://<WEBAPP_NAME>.azurewebsites.net/assets/hmcts.css) should be returned 

## Step 8: Logs & debugging

### Enable and tail logs
```bash
az webapp log config \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG" \
  --application-logging filesystem

az webapp log tail \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG"
```

### Restart app
```bash
az webapp restart \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG"
```

## Step 9: Cleanup (optional)
```bash
az webapp delete \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG"

az appservice plan delete \
  --name "$PLAN_NAME" \
  --resource-group "$RG" \
  --yes
```

# Key gotchas (learned the hard way) 😭
- .github/workflows must live at repo root
- Deploy subfolder, not repo root, in monorepos
- Publish profile secret must be a repository secret
- Express must listen on process.env.PORT
- Static assets must be explicitly served
