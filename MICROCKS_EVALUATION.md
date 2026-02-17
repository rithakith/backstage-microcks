# Microcks & Backstage Integration Security & Technical Analysis

## 1. Executive Summary
Microcks is an open-source Cloud-native API Mocking and Testing framework. Unlike a full lifecycle API Manager (like WSO2 APIM), Microcks focuses on the **design and testing** phases:
- **Mocking:** Turning API contracts (OpenAPI, AsyncAPI, Postman Collections) into live mock endpoints.
- **Contract Testing:** Verifying that a real implementation matches the API contract.

Its Backstage integration works primarily as a **Catalog Ingestion** mechanism, pulling API definitions from Microcks into the Backstage Catalog.

---

## 2. Core Features (The "What")

### A. Automatic API Discovery
Microcks acts as a "source of truth" for API definitions during development.
- **Support:** REST (OpenAPI), Event-Driven (AsyncAPI), gRPC (Protobuf), and GraphQL.
- **Workflow:** When a developer pushes an OpenAPI spec to Microcks, it automatically creates a live mock. The Backstage plugin then "discovers" this and adds it to the Backstage Catalog.

### B. Live Mocks & Documentation
- The Backstage plugin allows users to see the API definition and provides direct links to the running mocks in Microcks.
- It exposes the "Sandbox" URL (the mock endpoint) directly in the Backstage API entity, allowing frontend developers to start coding against the mock immediately.

### C. Contract Conformance
- Microcks stores "Test Results" (did this API pass the contract test?).
- The Backstage plugin can display the compliance status of an API version.

---

## 3. Technical Implementation (The "How")

The integration is implemented as a **Backstage Backend Feature** using the `EntityProvider` pattern.

### Architecture
1.  **Provider-Based Ingestion**: It does not use a "Processor" (which reacts to catalog files). Instead, it uses a **Provider** (`MicrocksApiEntityProvider`) that actively connects to the Microcks API.
2.  **Scheduling**: It runs on a configured schedule (e.g., every 2 minutes) to poll the Microcks server.
3.  **Authentication**: It uses a Service Account (Client ID/Secret) to authenticate against Microcks (if Keycloak is enabled) or connects anonymously (if using the "uber" docker image defaults).

### Data Mapping Strategy
When fetching data from Microcks (`GET /api/services`), it maps the response to a Backstage **API Entity**:

| Microcks Concept | Backstage Entity Field | Notes |
| :--- | :--- | :--- |
| Service Name | `metadata.name` | Sanitized to be URL-safe (kebab-case). |
| Service Version | `metadata.name` suffix | e.g., `my-api-v1.0` |
| Service Type | `spec.type` | Mapped: `REST` -> `openapi`, `EVENT` -> `asyncapi`. |
| API Specification | `spec.definition` | The raw text of the Swagger/OpenAPI file is downloaded and stored in the catalog. |
| Metadata | `metadata.tags` | Tags are synced from Microcks to Backstage. |
| Links | `metadata.links` | Adds links to the "Mock URL" and "Test Results". |

---

## 4. Setup & Installation Guide

This guide assumes a standard Backstage backend setup.

### Step 1: Deploy Microcks environment
For testing, the "Uber" image is the fastest way (bundles Mongo, Keycloak, and App).
```bash
docker run -d --name microcks-uber -p 8585:8080 quay.io/microcks/microcks-uber:latest
```
*(Note: Use port 8585 to avoid conflicts if Backstage is on 8080. The app usually starts on localhost:3000)*

### Step 2: Install the Plugin
```bash
# In the root of your Backstage workspace
yarn --cwd packages/backend add @microcks/microcks-backstage-provider
```

### Step 3: Register the Provider (`packages/backend/src/index.ts`)
Add the module to your backend. The new system automatically injects it if configured, or you manually add it:
```typescript
backend.add(import('@microcks/microcks-backstage-provider'));
```

### Step 4: Configure `app-config.yaml`
```yaml
catalog:
  providers:
    microcksApiEntity:
      dev:
        baseUrl: http://localhost:8585
        # Credentials are required by the schema, even if Microcks Uber auth is disabled
        serviceAccount: microcks-serviceaccount
        serviceAccountCredentials: placeholder-value
        schedule:
          frequency: { minutes: 2 }
          timeout: { minutes: 1 }
```

---

## 5. Competitive Analysis for WSO2 APIM Plugin

Since you are building a WSO2 APIM plugin, compare these approaches:

### Strengths of Microcks Integration
*   ✅ **Zero-Touch Ingestion:** Unlike the standard Backstage method (manually registering `catalog-info.yaml` files), Microcks automates this. If it exists in Microcks, it exists in Backstage.
*   ✅ **Mock-First approach:** Very strong for early-stage development before the real API exists.

### Opportunities for WSO2 APIM Integration
*   🚀 **Beyond Mocks:** Microcks only shows mocks. WSO2 APIM can show **Production** and **Sandbox** URLs.
*   🚀 **Subscription Management:** Backstage users often want to "Subscribe" to an API (get an API Key). Microcks doesn't handle this. Your WSO2 plugin could add a "Generate Keys" button in Backstage.
*   🚀 **Policy Insight:** Show which policies (Rate Limiting, Security) are applied to the API. Microcks doesn't have this context.
*   🚀 **Try-It-Out:** While Microcks has mocks, WSO2 can provide a real test console against the actual gateway.

### Implementation Recommendation for WSO2
1.  **Use the `EntityProvider` pattern** just like Microcks. Don't rely on users manually adding YAML files.
2.  **Tagging Strategy:** Tag imported entities with `source:wso2-apim` so users can filter them easily.
3.  **Relation Mapping:** If WSO2 has the concept of "Applications" and "APIs", map APIs to `Kind: API` and Applications to `Kind: Component` or `Kind: Resource` in Backstage to show dependencies.
