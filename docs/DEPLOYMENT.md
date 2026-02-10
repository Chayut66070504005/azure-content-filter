# Deployment Guide

[← Back to README](../README.md)


This guide explains how to deploy the Azure Content Filter to Azure and configure it correctly.

## 1. Create Azure Resources

Before deploying the code, you need to create the infrastructure.

### Option A: VS Code (Recommended)
1.  Open the **Azure** extension.
2.  Under **Resources**, click **+** -> **Create Resource...**.
3.  Select **Function App in Azure**.
4.  Follow the prompts:
    -   **Name**: Globally unique name (e.g., `my-content-filter-app`).
    -   **Runtime Stack**: Python 3.11 (or your local version).
    -   **Location**: Choose a region near you.

### Option B: Azure CLI
```bash
# Create a Resource Group
az group create --name ContentFilterRG --location eastus

# Create a Storage Account
az storage account create --name <STORAGE_NAME> --location eastus --resource-group ContentFilterRG --sku Standard_LRS

# Create the Function App
az functionapp create --resource-group ContentFilterRG --consumption-plan-location eastus --runtime python --runtime-version 3.11 --functions-version 4 --name <APP_NAME> --os-type linux --storage-account <STORAGE_NAME>
```

### Option C: Azure Portal (Manual Creation)

#### Step 1: Create a Storage Account
Azure Functions **require** a storage account to operate. You must create this first.

1.  Go to the [Azure Portal](https://portal.azure.com).
2.  Search for **Storage accounts** and click **Create**.
3.  Fill in the details:
    -   **Subscription**: Your subscription.
    -   **Resource Group**: Create new (e.g., `ContentFilterRG`) or use existing.
    -   **Storage account name**: Globally unique name (e.g., `contentfilterstorage`).
        - Must be 3-24 characters, lowercase letters and numbers only.
    -   **Region**: Choose the same region you'll use for your Function App.
    -   **Performance**: Standard.
    -   **Redundancy**: Locally-redundant storage (LRS) is sufficient for development.
4.  Click **Review**, then **Create**.
5.  Wait for deployment to complete.

#### Step 2: Create the Function App
1.  Search for **Function App** and click **Create**.
2.  Fill in the details:
    -   **Subscription**: Your subscription.
    -   **Resource Group**: Use the **same** resource group as your storage account (e.g., `ContentFilterRG`).
    -   **Function App Name**: Unique name (e.g., `my-content-filter-app`).
    -   **Runtime Stack**: Python.
    -   **Version**: 3.11 (or your local version).
    -   **Region**: Choose the **same region** as your storage account.
    -   **Operating System**: Linux.
    -   **Hosting Plan**: Consumption (Serverless) is recommended for cost efficiency.
3.  Click **Next: Storage**.
4.  **Storage account**: Select the storage account you created in Step 1.
5.  Click **Review + create**, then **Create**.

> [!IMPORTANT]
> The storage account connection (`AzureWebJobsStorage`) should be automatically set when you select the storage account in Step 2.4. If you encounter the "AzureWebJobsStorage" error, see the Troubleshooting section below.

---

## 2. Configure Settings (CRITICAL)

Your `local.settings.json` file is **NOT** deployed to Azure for security reasons. You must manually configure the connection strings in the Azure Portal or via CLI.

### Required Settings
-   `MONGO_URI`: Connection string to your MongoDB instance (e.g., MongoDB Atlas).
-   `MONGO_DB_NAME`: Name of the database (e.g., `prod_db`).
-   `AzureWebJobsStorage`: Automatically set by Azure when creating the Function App, but verify it exists.

### How to Set Settings
**Via Azure Portal:**
1.  Go to your Function App.
2.  Select **Settings** -> **Environment variables**.
3.  Click **+ Add** (or **New App Setting**).
4.  Name: `MONGO_URI`.
5.  Value: Your actual MongoDB connection string.
6.  Click **Apply**.
7.  Repeat for `MONGO_DB_NAME`.

**Via Azure CLI:**
```bash
az functionapp config appsettings set --name <APP_NAME> --resource-group <RESOURCE_GROUP> --settings "MONGO_URI=<YOUR_URI>" "MONGO_DB_NAME=<YOUR_DB_NAME>"
```

---

## 2A. Configure CORS (For Frontend Access)

If you plan to call your Function from a web frontend, you **must** configure CORS (Cross-Origin Resource Sharing) to allow requests from your frontend domain.

### Via Azure Portal:
1.  Go to your Function App in the [Azure Portal](https://portal.azure.com).
2.  In the left menu, select **API** → **CORS**.
3.  Add your allowed origins:
    - **Production**: `https://your-frontend-app.azurewebsites.net`
    - **Local Development**: `http://localhost:3000` (or `http://localhost:5173` for Vite)
4.  Click **Save**.

### Via Azure CLI:
```bash
# Add your production frontend
az functionapp cors add --name <APP_NAME> --resource-group <RESOURCE_GROUP> --allowed-origins "https://your-frontend-app.azurewebsites.net"

# Add localhost for development
az functionapp cors add --name <APP_NAME> --resource-group <RESOURCE_GROUP> --allowed-origins "http://localhost:3000" "http://localhost:5173"
```

> [!IMPORTANT]
> **Do NOT use `*` (wildcard)** for CORS in production as it's a security risk. Only allow specific domains you control.

---

## 3. Deploy the Code

### Option A: VS Code
1.  Open the **Azure** extension.
2.  Under **Workspace**, click the **Deploy to Azure...** (cloud icon).
3.  Select your subscription and the Function App you created.
4.  Click **Deploy**.

### Option B: Azure CLI (Core Tools) - Recommended for Cross-Platform Deployment

This method works on **Windows**, **macOS**, and **Linux**. It's fast and reliable.

#### Prerequisites
Ensure you have:
- **Azure Functions Core Tools** installed ([Installation Guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local))
- **Azure CLI** installed ([Installation Guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli))

#### Step 1: Login to Azure

**On macOS/Linux:**
```bash
az login
```

**On Windows (PowerShell):**
```powershell
az login
```

**On Windows (Command Prompt):**
```cmd
az login
```

This will open a browser window. Complete the login process, then return to your terminal.

#### Step 2: Navigate to Your Project Directory

**On macOS/Linux:**
```bash
cd /path/to/azure-content-filter
```

**On Windows (PowerShell):**
```powershell
cd C:\path\to\azure-content-filter
```

**On Windows (Command Prompt):**
```cmd
cd C:\path\to\azure-content-filter
```

#### Step 3: Publish Your Function App

Replace `<APP_NAME>` with your actual Function App name (e.g., `bearlab-function-app`).

**On macOS/Linux:**
```bash
func azure functionapp publish <APP_NAME>
```

**On Windows (PowerShell):**
```powershell
func azure functionapp publish <APP_NAME>
```

**On Windows (Command Prompt):**
```cmd
func azure functionapp publish <APP_NAME>
```

**Expected Output:**
```
Getting site publishing info...
Starting the function app deployment...
Creating archive for current directory...
Performing remote build for functions project...
...
Deployment successful.
Remote build succeeded!

Functions in <APP_NAME>:
    filter_comment - [httpTrigger]
        Invoke url: https://<APP_NAME>.azurewebsites.net/api/filter_comment
```

> [!TIP]
> The first deployment may take 5-15 minutes as Azure builds your Python environment. Subsequent deployments are faster (2-5 minutes).

#### Common Issues

**Issue: "Unable to connect to Azure"**
- **Solution**: Run `az login` first to authenticate.

**Issue: "Can't find app with name"**
- **Solution**: Verify your Function App name with:
  ```bash
  az functionapp list --query "[].name" --output table
  ```

**Issue: Python version mismatch warning**
- **Note**: If you see "Local python version X.X.X is different from deployed version", this is just a warning. The remote build will use the correct version configured in Azure.


### Option C: Azure Portal (Deployment Center via GitHub)
If your code is on GitHub, this is the easiest way to set up continuous deployment.

1.  Go to your Function App in the [Azure Portal](https://portal.azure.com).
2.  In the left menu, select **Deployment** -> **Deployment Center**.
3.  Under **Source**, select **GitHub**.
4.  Authorize your GitHub account if prompted.
5.  Select your **Organization**, **Repository**, and **Branch**.
6.  Click **Save**.
7.  Azure will automatically create a GitHub Actions workflow in your repository and start the deployment. You can view progress in the "Logs" tab or in your GitHub repository's "Actions" tab.

---

## 4. Verification

After deployment, your HTTP endpoint will be public.

**Test URL:**
`https://<APP_NAME>.azurewebsites.net/api/filter_comment`

**Test with curl:**
```bash
curl -X POST https://<APP_NAME>.azurewebsites.net/api/filter_comment \
     -H "Content-Type: application/json" \
     -d '{"comment": "Testing deployment."}'
     -d '{"comment": "Testing deployment."}'
```

---

## 5. Troubleshooting

### "Neither AzureWebJobsStorage nor AzureWebJobsStorage__accountName exist in app settings"
This error means your Function App in Azure is missing the connection to its storage account. The Function runtime needs this to operate.

**Fix via Azure Portal:**
1.  Go to your Function App in the [Azure Portal](https://portal.azure.com).
2.  Select **Settings** -> **Environment variables**.
3.  Check if `AzureWebJobsStorage` is present.
4.  If missing, click **+ Add**.
    -   **Name**: `AzureWebJobsStorage`
    -   **Value**: Your Storage Account Connection String.
        -   *To find this*: Go to your Storage Account -> **Security + networking** -> **Access keys** -> **Show keys** -> Copy **Connection string**.

**Fix via Azure CLI:**
```bash
az functionapp config appsettings set --name <APP_NAME> --resource-group <RESOURCE_GROUP> --settings "AzureWebJobsStorage=<YOUR_STORAGE_CONNECTION_STRING>"
```
