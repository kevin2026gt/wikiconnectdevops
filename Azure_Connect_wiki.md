# Azure DevOps Connection Setup

This document provides instructions for setting up a connection to the Azure DevOps Portfolio project to read User Stories (US).

## Azure DevOps Details

- **Organization**: ConduentDevOps
- **Project**: Portfolio
- **URL**: https://dev.azure.com/ConduentDevOps/Portfolio

## Prerequisites

1. **Active Azure DevOps Account**: You must have an account with access to the ConduentDevOps organization and Portfolio project
2. **Node.js**: Ensure Node.js is installed (required for this Playwright project)
3. **npm packages**: Azure DevOps Web API client

## Setup Steps

### 1. Install Azure DevOps API Client

```bash
npm install azure-devops-node-api
```

### 2. Create a Personal Access Token (PAT)

A Personal Access Token is required for authentication:

1. Navigate to https://dev.azure.com/ConduentDevOps
2. Click your **Profile icon** (top right)
3. Select **Personal access tokens**
4. Click **New Token**
5. Configure the token:
   - **Name**: `Playwright_Tests` (or your preferred name)
   - **Organization**: Select `ConduentDevOps`
   - **Expiration**: Set appropriate duration (90 days recommended)
   - **Scopes**: Select at minimum:
     - `Work Items (Read)`
     - `Code (Read)` (if needed)
6. Click **Create** and **copy the token** (you won't see it again)

### 3. Store Your PAT Securely

**Option A: Environment Variables (Recommended)**

```bash
# For Windows PowerShell:
$env:AZURE_DEVOPS_PAT = "your_pat_token_here"
$env:AZURE_DEVOPS_ORG = "ConduentDevOps"
$env:AZURE_DEVOPS_PROJECT = "Portfolio"
```

**Option B: .env File**

Create a `.env` file in your project root:

```
AZURE_DEVOPS_PAT=your_pat_token_here
AZURE_DEVOPS_ORG=ConduentDevOps
AZURE_DEVOPS_PROJECT=Portfolio
AZURE_DEVOPS_URL=https://dev.azure.com
```

Then install and use dotenv:

```bash
npm install dotenv
```

### 4. Create Azure DevOps Connection Module

Create `lib/azureDevOpsClient.ts`:

```typescript
import * as azdev from "azure-devops-node-api";

const orgUrl = process.env.AZURE_DEVOPS_URL || "https://dev.azure.com";
const token = process.env.AZURE_DEVOPS_PAT || "";
const project = process.env.AZURE_DEVOPS_PROJECT || "Portfolio";

const authHandler = azdev.getPersonalAccessTokenHandler(token);
const connection = new azdev.WebApi(orgUrl, authHandler);

export async function getWorkItemsTrackingApi() {
  return await connection.getWorkItemTrackingApi();
}

export async function getUserStories(): Promise<any[]> {
  const witApi = await getWorkItemsTrackingApi();
  
  // Query for User Stories
  const query = `
    SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo], [System.Description]
    FROM workitems
    WHERE [System.TeamProject] = @project
      AND [System.WorkItemType] = 'User Story'
    ORDER BY [System.CreatedDate] DESC
  `;

  const queryResult = await witApi.queryByWiql({ query }, project);
  
  if (queryResult.workItems && queryResult.workItems.length > 0) {
    const ids = queryResult.workItems.map((wi) => wi.id);
    const workItems = await witApi.getWorkItems(ids);
    return workItems || [];
  }
  
  return [];
}

export async function getUserStoryById(id: number): Promise<any> {
  const witApi = await getWorkItemsTrackingApi();
  return await witApi.getWorkItem(id);
}
```

### 5. Use in Your Tests

Example usage in a Playwright test:

```typescript
import { test, expect } from "@playwright/test";
import { getUserStories } from "./lib/azureDevOpsClient";

test.describe("Azure DevOps User Stories", () => {
  test("should fetch user stories from Portfolio", async () => {
    const stories = await getUserStories();
    expect(stories.length).toBeGreaterThan(0);
    console.log(`Found ${stories.length} user stories`);
    
    stories.forEach((story) => {
      console.log(`ID: ${story.id}, Title: ${story.fields["System.Title"]}`);
    });
  });
});
```

## Troubleshooting

### Authentication Errors

- **401 Unauthorized**: Check that your PAT is correct and not expired
- **403 Forbidden**: Verify you have access to the Portfolio project in ConduentDevOps organization

### Connection Issues

- Ensure you're using the correct organization URL: `https://dev.azure.com/ConduentDevOps`
- Check firewall/proxy settings if behind corporate network
- Verify the project name is exactly "Portfolio" (case-sensitive)

### Token Expiration

If your PAT expires, you'll need to:

1. Generate a new token following step 2
2. Update your environment variables or `.env` file
3. Restart your test runner

## Additional Resources

- [Azure DevOps REST API Documentation](https://docs.microsoft.com/en-us/rest/api/azure/devops)
- [Azure DevOps Node API GitHub](https://github.com/microsoft/azure-devops-node-api)
- [Personal Access Tokens Guide](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate)
- [Work Item Query Language (WIQL)](https://docs.microsoft.com/en-us/azure/devops/boards/queries/wiql-syntax)

## Security Notes

- **Never** commit your PAT to version control
- Use `.gitignore` to exclude `.env` files:
  ```
  .env
  .env.local
  *.env
  ```
- Rotate your PAT regularly (recommended every 90 days)
- Use a service account PAT in CI/CD pipelines rather than personal tokens
