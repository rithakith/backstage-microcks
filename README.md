# Backstage + Microcks Integration Demo

This repository demonstrates a standalone [Backstage](https://backstage.io) app integrated with [Microcks](https://microcks.io) via the `@microcks/microcks-backstage-provider`.

It serves as a reference implementation for visualizing how API definitions (OpenAPI, AsyncAPI, etc.) managed in Microcks are automatically discovered and imported into the Backstage Catalog.

## 📋 Prerequisites

*   **Node.js**: v20 or higher (Managed via `nvm` recommended)
*   **Yarn**: v1.x or v4.x (via Corepack)
*   **Docker**: Required to run the local Microcks instance.

## 🚀 Quick Start

### 1. Start Microcks (Docker)
We use the "Uber" image which bundles Microcks, MongoDB, and Keycloak for a zero-config setup.

```bash
# Run Microcks on port 8585 (to avoid conflict with Backstage's 8080/3000)
docker run -d --name microcks-uber -p 8585:8080 quay.io/microcks/microcks-uber:latest
```

Wait about 30-60 seconds for it to fully start. You can verified it's running by visiting [http://localhost:8585](http://localhost:8585).

### 2. Install Dependencies
In this project root:

```bash
# Enable Yarn via Corepack (if not already done)
corepack enable

# Install all packages
yarn install
```

> **Note on Windows:** You might see warnings about `isolated-vm`. These have been configured to be ignored in `.yarnrc.yml` and the Scaffolder plugin has been disabled in `packages/backend/src/index.ts` to prevent build crashes on Windows environments.

### 3. Run Backstage
Start the backend and frontend:

```bash
yarn start
```

The app will open at [http://localhost:3000](http://localhost:3000).

---

## ⚙️ Configuration

The integration is configured in `app-config.yaml`:

```yaml
catalog:
  providers:
    microcksApiEntity:
      dev:
        baseUrl: http://localhost:8585
        serviceAccount: microcks-serviceaccount
        serviceAccountCredentials: ab54d329-e435-41ae-a900-ec6b3fe15c54 # Default for Uber image
        schedule:
          frequency: { minutes: 2 }
          timeout: { minutes: 1 }
```

## 🧪 Testing the Integration

1.  **Create a Sample API**: Use the provided `sample-pastry-api.json` file.
2.  **Import to Microcks**:
    *   Go to [http://localhost:8585/#/importers](http://localhost:8585/#/importers)
    *   Upload `sample-pastry-api.json` as a primary artifact.
    *   Alternatively, run this PowerShell command:
        ```powershell
        $filePath = ".\sample-pastry-api.json"
        $boundary = [System.Guid]::NewGuid().ToString()
        $fileBytes = [System.IO.File]::ReadAllBytes($filePath)
        $fileContent = [System.Text.Encoding]::UTF8.GetString($fileBytes)
        $body = "--$boundary`r`nContent-Disposition: form-data; name=`"file`"; filename=`"sample-pastry-api.json`"`r`nContent-Type: application/json`r`n`r`n$fileContent`r`n--$boundary--`r`n"
        Invoke-WebRequest -Uri "http://localhost:8585/api/artifact/upload" -Method POST -Headers @{ "Content-Type" = "multipart/form-data; boundary=$boundary" } -Body $body
        ```
3.  **Check Backstage**:
    *   Wait ~2 minutes (or restart the backend to force a sync).
    *   Go to **Catalog** -> **APIs**.
    *   You should see `Pastry API` listed!

## 🔧 Troubleshooting

*   **`isolated-vm` failures**: Common on Windows. This project explicitly disables the `scaffolder` plugin in `packages/backend/src/index.ts` to bypass this native dependency requirement.
*   **Microcks Connection Refused**: Ensure the Docker container is running and `baseUrl` in `app-config.yaml` matches the Docker port (defaulted here to 8585).
