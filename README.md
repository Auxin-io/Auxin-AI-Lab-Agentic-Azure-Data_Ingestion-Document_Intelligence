# Document Ingestion on Azure

This repo turns three small business-document PDFs into the training data the other repos use it.

What it does:
- Creates sample business PDFs for Employee, Finance and HR
- Reads the text from them with Azure AI Document Intelligence (OCR)
- Builds the training datasets and the search index

| Dataset | Documents | Handle | Example question | What is used for? |
|---|---|---|---|---|
| **employee** | 5 timesheets + 5 expense reports | employee name | *How many hours did Maya Patel work?* | Trains a 3.2M-parameter model from scratch, random weights up |
| **finance** | 5 invoices + 5 purchase orders | vendor | *What is the Zephyr Networks invoice total?* | Fine-tunes Qwen2.5-3B with QLoRA, serves it, wraps it in an agent |
| **hr** | 5 policies + 5 leave requests | policy title / employee | *Who approves requests under the Overtime Policy?* | Retrieval over an index, no training at all |


```
PDF  ->  Blob Storage (raw)  ->  Document Intelligence  ->  Blob Storage (curated)  ->  JSONL
```

---

## Prerequisites

### Permissions

You need to be **Owner** on the subscription, or Owner plus User Access Administrator. Both
Terraform stacks create role assignments, and several steps grant roles to managed identities.

### GPU quota

This blocks more people than anything else. Need an Azure **Machine Learning**
quota for the `Standard NCASv3_T4 Family`, which is **separate from the Virtual Machines quota
of the same family**. A fresh subscription usually has 0.

Portal → **Quotas → Machine Learning → your region → Standard NCASv3_T4 Family**. If the limit
is 0, request **12 cores** before you start. Approval took under an hour for us, but it can take
longer, and you cannot proceed without it.

### Tools

| Tool | Version |
|---|---|
| Azure CLI | 2.89+, plus the ML extension: `az extension add -n ml` |
| Terraform | >= 1.9 |
| Python | 3.11+ |

### Region

Pick one region for everything. It needs three things available together: Document Intelligence,
GPU quota for the T4 family, and your chat model. `eastus`, `westus2` and `westeurope` are safe
choices. Mixing regions across the two Terraform stacks works but adds egress and latency for no
benefit.

---
## How To Setup

Follow the below steps to generate three small business-document datasets, OCRs them with Azure AI Document Intelligence, and produces the training data

### Step 1 — deploy the storage and OCR service

Terraform creates them:

```bash
az login
cd terraform
terraform init
terraform apply
cd ..
```

| Resource | Name | Used for |
|---|---|---|
| Resource group | `<prefix>-ingest-rg` | everything below |
| Storage account | `<prefix>ingest<suffix>` | AAD-only, no shared keys |
| Container `raw` | | the uploaded PDFs |
| Container `curated` | | OCR output: `documents/<doc_id>.txt` + `.json` |
| Document Intelligence | `<prefix>-docintel-<suffix>` | the OCR service, `prebuilt-read` model |
| Two role assignments | on the identity running `terraform apply` | `Storage Blob Data Contributor`, `Cognitive Services User` |

Export the connection values (or let `run_all.sh` read them from Terraform):

```bash
export AZURE_STORAGE_ACCOUNT=$(terraform -chdir=terraform output -raw storage_account)
export AZURE_DOCINTEL_ENDPOINT=$(terraform -chdir=terraform output -raw docintel_endpoint)
```

Keep `storage_account` to hand — every other repo needs it.

### Step 2 — run the ingestion

```bash
python3 -m venv .venv
chmod +x .venv/bin/activate
source .venv/bin/activate
pip install -r requirements.txt
chmod +x run_all.sh
bash run_all.sh
```

| # | Command | What it does |
|---|---|---|
| 1 | `generate_pdfs.py --all` | Writes 30 real PDFs with ReportLab, and `ground_truth_<dataset>.json` recording every value it printed — including the structured **facts** |
| 2 | `document_pipeline.py upload` | Copies each dataset into the `raw` container under its own prefix |
| 3 | `document_pipeline.py extract` | OCRs every file with Document Intelligence `prebuilt-read`, writes text + metadata to `curated`. Idempotent — skips what is already done |
| 4 | `build_dataset.py --all` | Open-book JSONL per dataset: OCR text + label |
| 5 | `build_closed_book.py --dataset finance --upload` | **Closed-book JSONL for finance** question → answer, no document text. `--upload` also writes it to `curated/datasets/` so Azure ML reads it from Blob |
| 6 | `build_closed_book.py --dataset employee --upload` | **Closed-book JSONL for employee**, the same way |

#### Check it worked

```bash
az storage blob list --account-name $AZURE_STORAGE_ACCOUNT --container-name curated \
  --auth-mode login --prefix documents/ -o table | head
```

You should see `documents/doc-finance-001.txt`, `doc-employee-001.txt`, `doc-hr-001.txt` and so
on — 30 text files plus their JSON metadata.

## Files

```
generate_pdfs.py        three datasets of PDFs + ground truth with structured facts
document_pipeline.py    upload to Blob, OCR with Document Intelligence
build_dataset.py        open-book JSONL per dataset
build_closed_book.py    closed-book Q&A for one dataset (finance, employee or hr)
run_all.sh              all of it, in order - both closed-book datasets included
terraform/              resource group, storage account, containers, Document Intelligence, roles
requirements.txt        azure-identity, azure-storage-blob, azure-ai-documentintelligence, reportlab
data/                   generated, gitignored - regenerates identically from the seed
```
