# Serverless Multi-supplier Invoice Processing Pipeline

Automates PDF text extraction, OCR fallbacks, VPC-secured API validation, and pricebook rate matching across e-procurement and billing portals like OpenInvoice, OpenTicket, and Jobutrax.

### Tech Stack & Tools
* **Orchestration & IaC:** AWS Step Functions, AWS SAM (YAML), Docker (`sam build --use-container`)
* **Compute & Storage:** AWS Lambda (Python 3.13), Amazon S3, AWS VPC, NAT Gateway
* **Security & Configuration:** AWS Secrets Manager, AWS SSM Parameter Store
* **Data Parsing & Validation:** PDFPlumber 0.11.0, BeautifulSoup4, Regular Expressions (`re`)
* **Downstream Integrations:** OpenInvoice (mTLS & VPC IP Whitelisting), OpenTicket, Jobutrax (Bearer Token & Secrets Manager), Xtracta Visual OCR API

## Overview

Our integration platform processes field service tickets and invoices for suppliers, delivering them to e-procurement portals such as OpenInvoice, OpenTicket, and Jobutrax. When suppliers send structured JSON or XML data, the platform can process it directly for downstream mapping and submission. When suppliers send PDF documents, the data first needs to be extracted. The current process sends the PDF to the Xtracta OCR API, where a person manually selects and reviews each invoice line before submitting the extracted data for further processing.

Once the data is available, each supplier may have different downstream processing requirements. Some require line items to be matched against buyer contract pricebooks, while others require AFE numbers and accounting codes to be validated through portal APIs. Historically, every supplier integration required its own isolated processing flow.

## Business Problems

#### Problem 1: Slow manual OCR and character misreads on text PDFs
The old system sent all PDF invoices to Xtracta, a visual OCR tool that requires a human to open each file and manually select each invoice line. During busy times, staff had to review up to 100 invoices in one batch, which created processing backlogs.

Xtracta also converted the PDF into images before reading the text, which caused character misreads such as `O` and `0`. Through testing, I found that many supplier PDFs already had readable text layers. This meant the data could be extracted directly from the PDF without converting it into images first. This improved accuracy and reduced both manual work and OCR costs.

#### Problem 2: Lack of central tracking and complex supplier rules
Different suppliers had different requirements, such as AFE checks and pricebook matching. Each integration had its own processing logic. There was no central workflow to see where an invoice was or where it had failed.

#### Problem 3: Fragmented mini pipelines and API Gateway overhead
The processing was split across separate pipelines for OCR, API validation, and pricebook mapping. The old system also used AWS API Gateway for communication between internal services, which added unnecessary network calls and API costs.

## Business Solution & Results

I built a unified AWS Step Functions state machine to handle the invoice workflow from PDF processing through validation and final delivery.

The main goal is to **extract text from the PDF's text layer first, validate the result, and only send it to manual OCR when needed**.

The pipeline first uses `PDFPlumber` to extract text directly from the PDF. It then checks that the extracted line items add up correctly. If the check passes, the invoice continues automatically. If the check fails because of layout issues or scanned content, the PDF is sent to Xtracta for manual OCR. Both paths return the data in the same internal JSON structure.

After extraction, the data goes to the downstream process needed for that supplier, such as pricebook matching or API based validation.

* **Over 99% Automated Ingestion:** Most invoices are processed without manual OCR.
* **Lower Processing Costs:** Direct PDF extraction greatly reduced Xtracta usage, OCR API calls, and manual invoice review.
* **Central Visual Monitoring:** AWS Step Functions shows where each invoice is in the workflow and makes it easier to find and debug errors.
* **Reduced Infrastructure Overhead:** Internal API Gateway calls were replaced with native Step Functions state transitions, reducing unnecessary network calls and API costs.
  
## Architecture & Data Flow

***State Machine View***
<p align="center">
  <img width="70%" alt="autoPDFstepfunctions_graph" src="https://github.com/user-attachments/assets/ef5ef630-6cf3-4574-8df0-dab8d1b310e6" />
</p>

1. ***Route 1** - Auto-PDF parsing → Coding validation and/or pricebook mapping → Lines validated → Downstream processing* 
2. ***Route 2** - Auto-PDF parsing → Coding validation and/or pricebook mapping → Lines failed validation (PDF style drift) → Manual OCR through Xtracta* 
3. ***Route 3** - Manual OCR through Xtracta → Clean and format API payload → Coding validation and/or pricebook mapping → Downstream processing* 

<p align="center">
<img width="32.5%" alt="auto_with_pricebook" src="https://github.com/user-attachments/assets/76792e85-26a6-4859-b0e9-02f0e460be67" />
<img width="32.5%" alt="auto_invalid_lines" src="https://github.com/user-attachments/assets/d4a5081f-2e38-4de0-9160-a3656af922b8" />
<img width="32.5%" alt="auto_invalid_lines_manual_process" src="https://github.com/user-attachments/assets/08e83036-017b-423a-973d-6847f925718d" />
</p>

***Execution Overview***
<img width="1613" height="634" alt="Auto-PDF State machine executions" src="https://github.com/user-attachments/assets/d29c2329-31ad-48dd-ae8f-6e576cea3bc8" />

## Data Processing Steps

1. **Source Inspection (`check_source`):** Evaluates incoming payloads. New raw PDFs route to `AutoPDF_parser`, while returned JSON payloads from Xtracta OCR callbacks route to `format_Xtracta_json` for normalization.
2. **Automated PDF Parsing (`AutoPDF_parser`):** Reads the PDF stream directly from S3, identifies the supplier using stable text anchors, loads custom extraction rules, and parses header fields and line items using PDFPlumber geometry.
3. **Arithmetic Validation (`valid_lines`):** Reconciles line-item totals against stated invoice subtotals. If validation passes, the transaction moves to coding checks. If validation fails (e.g., PDF style drift or misread text), it routes to `send_to_Xtracta` for manual operator handling.
4. **Conditional Coding Verification (`check_coding_OI/Jobutrax` & `validate_coding`):** Checks the `checkAFE` flag. If true, `validate_coding` validates AFE numbers and accounting codes against platform APIs before moving forward.
5. **Conditional Pricebook Mapping (`check_pricebook` & `map_pricebook`):** Checks the `checkPricebook` flag. If true, `map_pricebook` cleans line items using Pandas and matches rates against buyer pricebooks.
6. **Platform Output:** Passes the enriched payload to pre-existing downstream Lambdas (`output to v3 lambda` or Xtracta uploaders) for final delivery to OpenInvoice, OpenTicket, or Jobutrax.

## Engineering Challenges & Solutions

### 1. Unifying Pipeline Entry Points
* **The Challenge:** Supporting direct PDF parsing alongside asynchronous OCR callbacks required maintaining separate processing scripts for each input format.
* **The Solution:** Added an entry function (`check_source`) that inspects inbound payload structures and dispatches data to specialized parsing modules (`AutoPDF_parser` or `format_Xtracta_json`). Both paths transform incoming data into an identical internal JSON format, enabling all downstream validation and delivery modules to be shared.

### 2. Managing Multi-Platform Security & VPC Networking
* **The Challenge:** Downstream validation required calling external APIs with contrasting security rules. Jobutrax requires Bearer tokens, whereas OpenInvoice mandates static whitelisted IP addresses and mTLS client certificates.
* **The Solution:** Stored Jobutrax Bearer tokens in AWS Secrets Manager for runtime retrieval. For OpenInvoice, deployed the validation Lambda inside an AWS VPC connected to a NAT Gateway with Elastic IPs to ensure static outbound traffic, pulling mTLS client certificates into `/tmp` from SSM Parameter Store during cold starts.

### 3. Extracting Data from Unstructured Table Layouts
* **The Challenge:** Numerous supplier invoices render line items in visual grids without explicit table cell borders. Standard extraction libraries regularly merged adjacent text columns or split descriptions across rows.
* **The Solution:** Built a custom coordinate grid reconstructor using `PDFPlumber` geometry. The parser uses `extract_text_lines()` to locate row Y-axis baselines bounded by header and footer margins, then applies `extract_words()` with explicit X-axis boundaries to map words into columns based on left-edge positioning.

### 4. Catching Extraction Errors with Dual-Strategy Validation
* **The Challenge:** Text extraction can occasionally misread character values, and source invoices sometimes contain printed math errors from the originating system.
* **The Solution:** Developed a reconciliation function in `validate.py`. The algorithm calculates line totals ($\text{Quantity} \times \text{Rate}$) across billable lines while independently summing printed line amounts. By filtering out zero-rate section headers, the function confirms extraction accuracy if either calculation matches the document subtotal within a narrow rounding tolerance.
