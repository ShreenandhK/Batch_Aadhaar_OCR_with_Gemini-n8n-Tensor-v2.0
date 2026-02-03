# Aadhar Data Extractor & Merging Pipeline (Tensor v2.0)

**Automated OCR pipeline for extracting Aadhaar card details, merging front/back scans, and syncing data to Google Sheets without duplicates.**

This n8n workflow automates the processing of Aadhaar card images. It controls the flow of files to prevent API rate limits, uses Google Gemini for intelligent OCR, and performs an "Upsert" operation in Google Sheets to merge data from front and back scans into a single row based on the Aadhaar Number.

---

## 🚀 Key Features

* **Intelligent OCR:** Uses **Google Gemini (2.5 Flash / 3.0)** to extract unstructured text into structured JSON.
* **Smart Merging:** Handles separate Front and Back scans by updating existing rows in Google Sheets rather than creating duplicates.
* **Rate Limiting:** Includes a "Traffic Controller" mechanism to process files in batches, preventing API errors.
* **File Management:** Automatically moves processed images to prevent re-processing.
* **Error Handling:** Solves file locking (EBUSY) issues common with OneDrive/Cloud sync folders.

---

## 📋 Prerequisites

Before running this workflow, ensure you have the following:

1. **n8n (Self-Hosted):** Running locally (Windows preferred for this specific path setup).
2. **Google Gemini API Key:** Access to Google AI Studio.
3. **Google Cloud Project:** With Google Sheets API enabled (for OAuth or Service Account).
4. **Local Folder Structure:**
* `.../DataSet/Holder` (Source folder for raw images)
* `.../DataSet/Processed` (Destination folder for active processing)



---

## ⚙️ Configuration Guide

### 1. n8n Environment Variables (CRITICAL)

This workflow requires **Read/Write access** to your local hard drive to move images. By default, n8n may block this. You must start your n8n instance with the following permissions:

**PowerShell Command:**

```powershell
$env:NODES_EXCLUDE = "[]"
$env:N8N_BLOCK_FS_WRITE_ACCESS = "false"
$env:N8N_BLOCK_FS_READ_ACCESS = "false"
$env:N8N_RESTRICT_FILE_ACCESS_TO = ""
npx n8n

```

### 2. Google Sheet Setup

You must configure your target Google Sheet exactly as follows to ensure the "Upsert" logic works.

* **Header Names (Row 1):** (Case-sensitive)
* `Name`
* `DOB`
* `Gender`
* `Aadhaar Number`
* `Address`
* `Path`


* **Column Formatting (Crucial):**
Select the **Aadhaar Number** column and set format to **Plain Text** (`Format > Number > Plain Text`).
> *Reason:* Prevents scientific notation issues and ensures the workflow can match "1234" (Text) with "1234" (Number).



---

## 🛠️ Installation & Setup

### 1. Import Workflow

Download the `Aadhar Data Extractor (Google Sheets) Tensor v2.0.json` file and import it into your n8n instance.

### 2. Update File Paths

The workflow uses hardcoded paths that must be updated to match your system.

* **Node: `Local File Trigger**`
* Update **Watch Path** to your Processing folder (e.g., `C:\Projects\aadhaar-batch-ocr\DataSet\Processed`).


* **Node: `Execute Command**`
* Edit the command code. Replace the source and destination paths with your local folders:


```cmd
IF EXIST "YOUR_SOURCE_PATH\*.jp*" (...) move "%f" "YOUR_DESTINATION_PATH\" ...

```



### 3. Connect Credentials

Security keys are stripped from the export. You must reconnect them:

* **Google Gemini API:** Paste your key from Google AI Studio.
* **Google Sheets:** Authenticate via OAuth2 or Service Account.
* *Note:* If using a Service Account, ensure the specific Google Sheet is shared with the Service Account email address.



### 4. Select Target Sheet

Open the **Google Sheets** node and re-select your specific Spreadsheet and Sheet Name from the dropdown menu to ensure the ID is linked correctly.

---

## 🧠 Workflow Logic

This pipeline is designed in two distinct stages:

### Stage 1: The Traffic Controller (Batching)

To avoid hitting Google Gemini API rate limits, we do not watch the source folder directly.

* **Schedule Trigger:** Runs every 30 seconds.
* **Execute Command ("Gatekeeper"):** Checks the `Holder` folder and moves a small batch of files (e.g., 1-5 files) to the `Processed` folder.

### Stage 2: The Processor (Extraction)

* **Local File Trigger ("Watcher"):** Detects new files arriving in the `Processed` folder.
* **Wait Node:** Pauses for 1 second.
* *Fixes Issue #1:* Prevents "EBUSY" errors if a cloud sync (OneDrive/Google Drive) locks the file immediately after creation.


* **Gemini AI ("Brain"):** OCRs the image into raw JSON.
* *Rules:* Extracts Name, ID, Address. If data is not visible (e.g., Address on the front side), it is omitted.


* **Code Node ("Cleaner"):**
* Removes empty keys to prevent overwriting existing data with blanks.
* **Forces Aadhaar Number to String** to ensure exact matching in Sheets.


* **Google Sheets ("Storage"):** Performs an **Upsert**.
* Look up `Aadhaar Number`.
* **Found?** Update the row (merges new data, like Address from back scan).
* **Not Found?** Append a new row.



---

## ⚠️ Troubleshooting

**1. "Path Not Found" Error**

* Ensure you have updated the hardcoded paths in the **Execute Command** node. Windows paths must use backslashes (`\`).

**2. Duplicate Entries in Sheets**

* Check that the **Aadhaar Number** column in Google Sheets is set to **Plain Text**.
* Ensure the JSON key from Gemini is exactly `Aadhaar_Number` and mapped to the Sheet header `Aadhaar Number`.
* Recude the number of files processed simultaniously.

**3. n8n "EPERM" or Access Denied**

* Restart n8n using the environment variable flags listed in the Configuration Guide to allow file system writing.
