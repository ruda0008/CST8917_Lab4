# Lab 4: PhotoPipe — Event-Driven Image Processing

CST8917 — Serverless Applications | Winter 2026

## Overview

PhotoPipe is an event-driven image processing pipeline built on Azure Event Grid and Azure Functions. Images uploaded to Blob Storage automatically trigger metadata extraction and audit logging with no polling or manual intervention.

## Youtube Video
- https://youtu.be/up7Ax2_VOic

## Architecture

```
Web Client → Blob Storage (image-uploads)
                    ↓
           Event Grid System Topic
                    ↓
        ┌───────────┴───────────┐
   Subscription 1          Subscription 2
   (.jpg/.png only)         (all blobs)
        ↓                       ↓
  process-image fn          audit-log fn
        ↓                       ↓
  Blob Storage             Table Storage
  (image-results)          (processinglog)
```

## Services Used

| Service | Role |
|---|---|
| Blob Storage (image-uploads) | Receives uploaded images from the web client |
| Blob Storage (image-results) | Stores processing metadata JSON files |
| Event Grid System Topic | Captures BlobCreated events from the storage account |
| Event Grid Subscription 1 | Filters .jpg/.png and triggers process-image function |
| Event Grid Subscription 2 | Routes all blob events to audit-log function |
| Azure Function: process-image | Extracts metadata and writes results to blob storage |
| Azure Function: audit-log | Logs all upload events to Table Storage |
| Table Storage (processinglog) | Audit trail of all processed events |

## Project Structure

```
├── function_app.py               # Azure Functions (process-image, audit-log, get-results, get-audit-log, health)
├── requirements.txt              # Python dependencies
├── test-function.http            # REST Client test requests
├── client.html                   # PhotoPipe web application
├── local.settings.example.json   # Settings template (no real keys)
└── README.md
```

## Setup Instructions

### Prerequisites

- Azure CLI installed and authenticated
- Python 3.11 or 3.12
- VS Code with Azure Functions extension
- Azurite (for local testing)

### 1. Clone the Repository

```bash
git clone <repo-url>
```

### 2. Configure Local Settings

Copy the example settings file and fill in your values:

```bash
cp local.settings.example.json local.settings.json
```

Edit `local.settings.json` and replace the placeholder with your storage account connection string.

### 3. Install Dependencies

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 4. Azure Infrastructure Setup

#### Storage Account

- Create a storage account in your resource group
- Enable Blob anonymous access at the account level
- Create two containers:
  - `image-uploads` — Blob public access
  - `image-results` — Private
- Configure CORS on the Blob service (allow `*`, methods: GET, PUT, OPTIONS, HEAD)

#### Function App

- Deploy using VS Code: F1 → Azure Functions: Deploy to Function App
- Add `STORAGE_CONNECTION_STRING` to Application Settings in the portal
- Enable CORS on the Function App (allow `*`)

#### Event Grid

- Create a System Topic on the storage account
- Create Subscription 1 (process-image-sub):
  - Filter: Subject begins with `/blobServices/default/containers/image-uploads`
  - Advanced filter: subject string ends with `.jpg` and `.png`
  - Endpoint: process-image function
- Create Subscription 2 (audit-log-sub):
  - Filter: Subject begins with `/blobServices/default/containers/image-uploads`
  - Endpoint: audit-log function

### 5. Configure the Web Client

Open `client.html` in a browser and fill in:

- Storage Account Name
- SAS Token (generate from portal: Shared access signature → Blob, Container + Object, Read/Write/Create/List)
- Function App URL

### 6. Verify Deployment

```
https://<your-function-app>.azurewebsites.net/api/health
```

Expected response: `{"status": "healthy", "service": "PhotoPipe Function App"}`

## Testing

| Test | Expected Result |
|---|---|
| Upload .jpg | process-image and audit-log both fire |
| Upload .png | process-image and audit-log both fire |
| Upload .txt | Only audit-log fires, no processing result |

## Security Notes

- Do not commit `local.settings.json` — it contains your connection string
- Do not commit SAS tokens
- Use `local.settings.example.json` with placeholder values for version control
- In production, restrict CORS origins and use short-lived SAS tokens from a backend API
