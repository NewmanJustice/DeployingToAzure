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

