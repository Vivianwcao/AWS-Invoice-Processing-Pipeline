# Modular Hybrid PDF Parsing & Intelligent Routing Pipeline
## Automated Invoice Ingestion, Arithmetic Validation, Coding Verification, and Pricebook Mapping for OpenInvoice, OpenTicket, and Jobutrax

### Tech Stack & Tools
* **Orchestration & IaC:** AWS Step Functions, AWS SAM (YAML), Docker (`sam build --use-container`)
* **Compute & Storage:** AWS Lambda (Python 3.13), Amazon S3, AWS VPC, NAT Gateway
* **Security & Configuration:** AWS Secrets Manager, AWS SSM Parameter Store
* **Data Parsing & Validation:** PDFPlumber 0.11.0, BeautifulSoup4, Regular Expressions (`re`)
* **Downstream Integrations:** OpenInvoice (mTLS & VPC IP Whitelisting), OpenTicket, Jobutrax (Bearer Token & Secrets Manager), Xtracta Visual OCR API

---

## The Goal: Building a Modular, Expandable Processing Pipeline

At EMI, our team manages B2B submissions for field service tickets and invoices to buyer platforms including OpenInvoice, OpenTicket, and Jobutrax. When suppliers provided raw JSON or XML files from their accounting software, we mapped the data directly. However, when suppliers only provided PDF invoices, our legacy workflow routed all files to Xtracta, a visual OCR tool requiring human operators to manually highlight fields on screen.

Over time, three major operational bottlenecks slowed our operations down:

* **Problem 1: High labor overhead and slow turnaround:** Relying on human operators to manually select fields on every PDF page created severe bottlenecks during peak submission cycles.

* **Problem 2: Growing client business requirements:** Suppliers began requesting additional data manipulation before submission, such as validating accounting coding like AFE numbers and matching contracted pricebook rates. The legacy OCR setup had no way to inject this extra logic.

* **Problem 3: Inflexible, monolithic design:** Every supplier had different needs. Some required pricebook mapping, others required coding verification, and some required both. The company needed a versatile, future-proof pipeline where steps could be added, removed, or bypassed dynamically per supplier without rebuilding the core workflow.

---

## Solution & Results

I designed and built the AWS Step Functions state machine and custom Lambda functions. The pipeline attempts automated text parsing first, validates line item arithmetic, and routes to Xtracta visual OCR only as a fallback. 

I also integrated conditional downstream stages for coding verification and pricebook mapping, while linking pre-existing legacy upload modules at the end of the state machine for our team's ongoing operational use.

### Key Results
* **Over 90% Automated Ingestion:** Direct text parsing handles the vast majority of client PDFs automatically, bypassing manual OCR completely.

* **Fully Modular Architecture:** Built on Step Functions Choice states driven by boolean flags (`checkAFE`, `checkPricebook`), allowing specific pipeline stages to be toggled on or off per supplier.

* **Integrated Business Logic:** Embedded automated AFE coding validation and pricebook rate matching directly into the pipeline before final platform delivery.

* **Unified Fallback Loop:** Invoices sent to Xtracta for manual OCR route back into the exact same Step Functions state machine once completed, reusing downstream validation modules without duplicate code.

---

## Architecture & Data Flow

<img width="1125" height="926" alt="autoPDFstepfunctions_graph" src="https://github.com/user-attachments/assets/de7f2a80-173b-489a-b220-6b7b1fb74c82" />

### Data Processing Steps

1. **Source Inspection (`check_source`):** Evaluates incoming payloads. New PDFs route to `AutoPDF_parser`. Returned JSON files from Xtracta OCR route to `format_Xtracta_json` to standardize the payload before downstream processing.

2. **Automated PDF Parsing (`AutoPDF_parser`):** Reads the PDF stream from S3, identifies the supplier using stable text anchors, loads custom extraction rules, and parses header fields and line items using PDFPlumber.

3. **Arithmetic Validation (`valid_lines`):** Reconciles line item totals against stated invoice subtotals. If validation passes, the transaction moves to coding checks. If validation fails, it routes to `send_to_Xtracta` for manual operator handling.

4. **Conditional Coding Verification (`check_coding_OI/Jobutrax` & `validate_coding`):** Checks the `checkAFE` flag. If true, `validate_coding` validates AFE numbers and accounting codes against platform APIs before moving forward.

5. **Conditional Pricebook Mapping (`check_pricebook` & `map_pricebook`):** Checks the `checkPricebook` flag. If true, `map_pricebook` cleans line items using Pandas and matches rates against buyer pricebooks.

6. **Platform Output:** Passes the enriched payload to pre-existing downstream Lambdas (`output to v3 lambda` or Xtracta uploaders) for final delivery.

---

## Engineering Challenges & Solutions

### 1. Designing a modular pipeline that grows with business needs
* **The Challenge:** Supplier requirements change constantly. Adding a new validation step or bypassing a feature for a specific client previously required writing custom code pathways.

* **The Solution:** Structured the Step Functions state machine using Choice states driven by payload flags like `checkAFE` and `checkPricebook`. This allows any supplier payload to dynamically skip or enter specific Lambdas, making the pipeline modular, expandable, and easy to maintain.

### 2. Managing multi-platform security and API authentication
* **The Challenge:** Downstream coding validation required calling two external platforms with completely different security architectures. Jobutrax requires Bearer token authentication, while OpenInvoice requires static whitelisted IP addresses and client certificates.

* **The Solution:** For Jobutrax API calls, I stored Bearer tokens in AWS Secrets Manager and retrieved them inside the Lambda during cold starts. For OpenInvoice API calls, I placed the Lambda inside a custom AWS VPC attached to a NAT Gateway with Elastic IPs to guarantee all outgoing traffic originated from whitelisted static IP addresses.

### 3. Extracting data from coordinate-positioned pseudo-tables
* **The Challenge:** Many supplier invoices look like clean tables but contain no actual cell lines or table structures. Running standard table extraction tools on files like Mantl invoices produced garbled text, split descriptions mid-word, and scattered values across wrong columns.

* **The Solution:** Built a custom coordinate grid reconstructor using PDFPlumber geometry. The parser uses `extract_text_lines()` to locate row Y-axis baselines bounded between header and footer margins. It then uses `extract_words()` with explicit X-axis column boundary maps to assign words to columns based on word left edges rather than word centers, preventing wide text cells from bleeding into adjacent columns.

### 4. Catching extraction errors with dual strategy validation
* **The Challenge:** Extraction routines can occasionally misread lines, and some supplier invoices contain printed arithmetic errors from their own source systems.

* **The Solution:** Implemented a dual strategy reconciliation function in `validate.py`. The validator calculates the sum of quantity multiplied by rate across all billable lines, while independently calculating the sum of printed line amounts. Excluded zero-rate section headers to prevent corrupted sums, and passed the invoice if either calculation matched the printed subtotal within a small rounding tolerance.
