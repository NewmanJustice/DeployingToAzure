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

- Azure CLI installed, **up to date**, and logged in:
  ```bash
  az version  # Check current version
  az upgrade  # Update if needed
  az login
  ```
- GitHub repo with your Express app
- Your app must listen on process.env.PORT:
  ```javascript
    const port = process.env.PORT || 3000;
    app.listen(port);
  ```

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

## Step 4: Enable SCM Basic Authentication
Azure App Service may have Basic Authentication disabled by default, which prevents GitHub Actions deployments. Enable it:

```bash
# Enable Basic Auth for SCM (deployment)
az resource update \
  --resource-group "$RG" \
  --name scm \
  --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies \
  --parent sites/"$WEBAPP_NAME" \
  --set properties.allow=true

# Enable Basic Auth for FTP
az resource update \
  --resource-group "$RG" \
  --name ftp \
  --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies \
  --parent sites/"$WEBAPP_NAME" \
  --set properties.allow=true
```

⚠️ **Important**: Without this step, you may get `401 Unauthorized` errors during deployment.

## Step 5: Configure environment variables
```bash
az webapp config appsettings set \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG" \
  --settings \
    NODE_ENV=production \
    DOCS_USER="your-username" \
    DOCS_PASS="your-password" \
    ADMIN_PASSWORD="your-admin-password" \
    SESSION_SECRET="$(openssl rand -base64 32)"
```

### Important Notes:
- Set **all required environment variables** before the first deployment
- If your app requires environment variables on startup, it will crash without them
- Use secure, randomly generated values for secrets
- Never commit secrets to your repository

## Step 6: Download the Publish Profile (DO NOT COMMIT)
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

## Step 7: GitHub Actions workflow (monorepo-aware)

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
          app-name: hmctsdesignsystem-code  # Optional - can be omitted
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: packages/hmcts-docs

```

### Notes:
- The `app-name` parameter is **optional** since the publish profile already contains this information
- If you include `app-name`, ensure there are **no trailing spaces**
- If deployment fails with "Publish profile is invalid", try removing the `app-name` line


## Step 8: Fix missing CSS / assets (very common)

### Symptom

Browser console shows:

`Refused to apply style ... MIME type ('text/html')`

This means the CSS URL is returning HTML, not CSS.

### Step 8.1: Locate CSS in Azure (Kudu)
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

### Step 8.2: Serve assets in Express
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

## Step 9: Logs & debugging

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

---

## Troubleshooting

### API Version Error
**Error**: `The api-version '2023-01-01' is invalid. The supported versions are...`

**Solution**: Your Azure CLI is outdated. Update it:
```bash
az upgrade
```

### Missing Resource in URI
**Error**: `No HTTP resource was found that matches the request URI`

**Solution**: The `--resource-group` parameter is missing or incorrect. Verify:
```bash
# List all resource groups
az group list --output table

# Ensure you're using the correct name in your commands
az webapp show --name "$WEBAPP_NAME" --resource-group "$RG"
```

### 401 Unauthorized During Deployment
**Error**: `Failed to deploy web package. Unauthorized (CODE: 401)`

**Solution**: Enable SCM Basic Authentication (see Step 4):
```bash
az resource update \
  --resource-group "$RG" \
  --name scm \
  --namespace Microsoft.Web \
  --resource-type basicPublishingCredentialsPolicies \
  --parent sites/"$WEBAPP_NAME" \
  --set properties.allow=true
```

### Publish Profile Invalid Error
**Error**: `Publish profile is invalid for app-name and slot-name provided`

**Solutions**:
1. **Remove trailing spaces** from `app-name` in your workflow YAML
2. **Remove the app-name parameter entirely** - the publish profile already contains it:
   ```yaml
   - name: Deploy to Azure Web App
     uses: azure/webapps-deploy@v3
     with:
       publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
       package: packages/hmcts-docs
   ```
3. **Regenerate the publish profile**:
   ```bash
   az webapp deployment list-publishing-profiles \
     --name "$WEBAPP_NAME" \
     --resource-group "$RG" \
     --xml > publishProfile.xml
   ```
   Then update the GitHub secret with the fresh content

### App Crashes on Startup
**Error**: Missing environment variables (check logs for specific error)

**Solution**:
1. View logs to identify missing variables:
   ```bash
   az webapp log tail --name "$WEBAPP_NAME" --resource-group "$RG"
   ```
2. Set the required environment variables:
   ```bash
   az webapp config appsettings set \
     --name "$WEBAPP_NAME" \
     --resource-group "$RG" \
     --settings VARIABLE_NAME="value"
   ```
3. Restart the app:
   ```bash
   az webapp restart --name "$WEBAPP_NAME" --resource-group "$RG"
   ```

### Verify Runtime Configuration
Check that your Web App is using the correct Node.js runtime:
```bash
az webapp show \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG" \
  --query "siteConfig.linuxFxVersion" -o tsv
```
Expected: `NODE|20-lts`

### Check Deployment Status
```bash
# Check if app is running
az webapp show \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG" \
  --query "state" -o tsv

# View recent deployment logs
az webapp log deployment show \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG"
```

---

## Step 10: Cleanup (optional)
```bash
az webapp delete \
  --name "$WEBAPP_NAME" \
  --resource-group "$RG"

az appservice plan delete \
  --name "$PLAN_NAME" \
  --resource-group "$RG" \
  --yes
```
