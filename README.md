# Lab 2 – Data Ingestion in Azure (Amazon Electronics Dataset)

## Introduction

This repository documents my work for **Lab 2: Data Ingestion in Azure**. The goal of this lab was to ingest the Amazon Electronics dataset into Azure Blob Storage, understand the structure of the data, and prepare it correctly for downstream processing.

Although the task sounds simple in theory, I faced many real-world issues while working with Azure services, especially when using **AzCopy, SAS tokens, container paths, and Azure ML Compute Instances**. This README explains **what I did, what went wrong, what I misunderstood at first, and how I eventually fixed everything**.

---

## Environment Setup

* Azure Storage Account: `amazonelectron5961875857`
* Execution Environment: Azure ML Compute Instance (Linux)
* Tools Used:

  * Azure CLI
  * AzCopy
  * Python
  * Linux Terminal

---

## Storage Account & Containers

The storage account already contained several containers created automatically by Azure services:

* `azureml`
* `azureml-blobstore-*` (auto-generated)
* `insights-*` containers (system logs)

Instead of creating a new container, I used the existing **`azureml`** container and treated the following path as my raw data zone:

```
azureml/raw/
```

### What I learned here

* Containers must exist explicitly
* Folders such as `raw/` are virtual and do not need to be created manually
* Using the wrong container name results in silent AzCopy failures

---

## Dataset Description

The Amazon Electronics dataset consists of two main files:

### 1. Reviews Dataset

* File name: `reviews_Electronics_5.json`
* Size: ~1.4 GB (uncompressed)
* Content: User reviews, ratings, and timestamps

### 2. Metadata Dataset

* File name: `meta_Electronics.json.gz`
* Content: Product metadata
* Issue: **Not valid JSON by default**

---

## Uploading the Reviews Dataset

### What I Did

I worked entirely inside the Azure ML Compute Instance terminal. After confirming the file existed locally, I uploaded it using AzCopy:

```bash
azcopy copy "./reviews_Electronics_5.json" \
"https://amazonelectron5961875857.blob.core.windows.net/azureml/raw/reviews_Electronics_5.json?<SAS_TOKEN>" \
--overwrite=true
```

### Issues Faced

* Initially used non-existing containers such as `raw` instead of `azureml/raw`
* Confused containers with folders
* Forgot to replace placeholders in URLs
* Misinterpreted AzCopy error messages

After correcting the container path, the upload succeeded and the file appeared as:

```
azureml/raw/reviews_Electronics_5.json
```

---

## Metadata File Problem (Major Issue)

### What I Didn’t Know at First

Although the file is named `meta_Electronics.json`, it is **not valid JSON**. Each line is actually a Python dictionary using single quotes. This format:

* Breaks Azure Data Factory pipelines
* Cannot be parsed as JSON
* Requires preprocessing before ingestion

This issue caused repeated failures until I investigated the file content.

---

## Fixing the Metadata File

### Steps Taken

1. Downloaded and decompressed the metadata file
2. Parsed each line as a Python dictionary
3. Converted each line into valid JSON
4. Saved the fixed output as a new file

### Python Conversion Script

```python
import ast
import json

input_file = "meta_Electronics.json"
output_file = "meta_Electronics_fixed.json"

with open(input_file, "r") as fin, open(output_file, "w") as fout:
    for line in fin:
        obj = ast.literal_eval(line)
        fout.write(json.dumps(obj) + "\n")

print("Metadata conversion complete")
```

### Uploading the Fixed Metadata File

```bash
azcopy copy "meta_Electronics_fixed.json" \
"https://amazonelectron5961875857.blob.core.windows.net/azureml/raw/meta_Electronics_fixed.json?<SAS_TOKEN>" \
--overwrite=true
```

---

## Major Struggles and Lessons Learned

* AzCopy fails if **either the source path or destination blob path does not exist**
* Error messages often point to **path mistakes**, not authentication issues
* SAS tokens must include correct permissions and expiration
* Containers are not the same as folders
* Verifying blobs using `az storage blob list` is critical

---

## Final Storage Layout

```
azureml/
 └── raw/
     ├── reviews_Electronics_5.json
     └── meta_Electronics_fixed.json
```

---

## Conclusion

This lab taught me that data ingestion in the cloud is not just about uploading files. It requires understanding how Azure Blob Storage works, how authentication is handled, and how data formats affect downstream processing.

The struggles I faced helped me gain a much deeper understanding of Azure storage concepts and real-world debugging, which is far more valuable than a simple successful upload.
