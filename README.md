# CostGuard 🛡️

> **A Lightweight Azure FinOps CLI Guardrail for Terraform Plans**

**CostGuard** is a Python 3.11+ command-line utility built for DevOps and FinOps engineers. It intercepts Terraform plans in CI/CD pipelines, queries live rates directly from the official **Azure Retail Prices API**, caches rates in a local **SQLite database** using write-through caching, calculates monthly cost impacts, and enforces strict **budget guardrails** before infrastructure is provisioned.

---

## ✨ Features

- **Standard Terraform Plan Ingestion**:
  - Direct file ingestion via `--plan <FILE>`
  - Pipe streaming via `stdin` (`terraform show -json tfplan.binary | costguard`)
  - Explicit stdin flag: `--plan -`
- **Comprehensive Lifecycle Parsing**:
  - Accurately detects `create`, `delete`, `update`, and `delete+create replacement` operations.
  - Extracts resource address, resource type, region (armRegionName), and Azure VM/disk SKU.
  - Supported billable resources: `azurerm_linux_virtual_machine`, `azurerm_windows_virtual_machine`, `azurerm_virtual_machine`, and `azurerm_managed_disk`.
  - Gracefully skips non-billable resources (e.g. `azurerm_resource_group`, `azurerm_virtual_network`, `azurerm_subnet`, `azurerm_network_security_group`).
- **Live Azure Retail Prices API Integration**:
  - Queries `https://prices.azure.com/api/retail/prices` using precise OData filters:
    `serviceName eq '...' and armRegionName eq '...' and armSkuName eq '...' and priceType eq 'Consumption'`
  - Automatically filters out Spot and Low-Priority meters.
  - Matches appropriate OS meters (e.g., Linux VMs without Windows licensing surcharges).
  - No hardcoded pricing data.
- **SQLite Write-Through Cache**:
  - Local database: `pricing_cache.db`.
  - Compound cache key: `(SKU, region, currency)`.
  - First lookup queries Azure API and writes through to SQLite; subsequent runs hit local cache instantaneously.
  - Clear cache flag: `--clear-cache`.
- **Monthly Cost & Delta Calculation**:
  - Standard month conversion: `monthly_cost = hourly_rate * 730`.
  - Resource delta: `delta = new_monthly_cost - old_monthly_cost`.
  - Prior total, projected total, and net monthly impact calculations.
- **Automated Budget Guardrail & CI/CD Exit Codes**:
  - Exit `0`: Net monthly increase is within budget (or no limit set).
  - Exit `1`: Net monthly increase exceeds budget threshold (`--max-increase`).
  - Exit `2`: Invalid Terraform plan JSON, file read error, or missing arguments.
- **Graceful Error Handling**:
  - Unknown or unpriced SKUs emit a descriptive warning and continue without crashing ($0.00 / unpriced).
- **Clean Terminal Reporting**:
  - Formatted ASCII table showing Resource Address, Action, Region, SKU, Old Monthly, New Monthly, and Delta.
  - Financial guardrail summary highlighting budget status and cache statistics.

---

## 🚀 Installation

### Requirements
- Python 3.11+
- `pip`

### Install Dependencies
```bash
pip install -r requirements.txt
```

### Install CostGuard Locally
```bash
pip install -e .
```

---

## 📖 CLI Usage

```text
usage: costguard [-h] [-p PLAN_PATH] [-m MAX_INCREASE] [-c CURRENCY] [--clear-cache] [--cache-db CACHE_DB] [-v]

CostGuard - Azure FinOps Guardrail for Terraform Plans

options:
  -h, --help            show this help message and exit
  -p, --plan PLAN_PATH  Path to Terraform plan JSON file (use '-' to read from stdin)
  -m, --max-increase MAX_INCREASE
                        Budget threshold: maximum allowable net monthly increase (e.g. 50.00)
  -c, --currency CURRENCY
                        Currency code for Azure Retail Prices (default: USD)
  --clear-cache         Clear local SQLite pricing cache before executing
  --cache-db CACHE_DB   SQLite database path for pricing cache (default: pricing_cache.db)
  -v, --version         show program's version number and exit
```

---

## 💡 Examples

### 1. Evaluate a Plan File
```bash
costguard --plan sample_plans/vm_create.json
```

### 2. Stream Plan from Stdin in Terraform Workflow
```bash
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary | costguard
```

### 3. Enforce Budget Guardrail (e.g., Max Increase $50.00)
```bash
costguard --plan sample_plans/vm_upgrade.json --max-increase 50.00
```
*If upgrading `Standard_B2s` to `Standard_D4s_v3` adds ~$109.79/month, CostGuard exits with code `1` and halts deployment.*

### 4. Custom Currency & Cache Management
```bash
costguard --plan sample_plans/vm_create.json --currency EUR --clear-cache
```

---

## 📊 Sample Terminal Output

```text
=========================================================================================================
                          COSTGUARD - AZURE INFRASTRUCTURE COST GUARD                         
=========================================================================================================
Resource Address                         | Action    | Region       | SKU              | Old Monthly  | New Monthly  | Delta       
---------------------------------------------------------------------------------------------------------
azurerm_linux_virtual_machine.databas... | UPDATE    | eastus       | Standard_B2s ... |       $30.37 |      $140.16 |     +$109.79
---------------------------------------------------------------------------------------------------------
FINANCIAL GUARDRAIL SUMMARY:
  Prior Total:            $30.37
  Projected Total:        $140.16
  Net Monthly Impact:     +$109.79
  Cache Hits/API Lookups: 1 hits / 1 API lookups
  Budget Threshold:       $50.00
  Status:                 [ FAIL ] - Exceeds budget by $59.79
=========================================================================================================
```

---

## 🛠️ CI/CD Integration

### GitHub Actions
```yaml
name: "Terraform Cost Guardrail"
on: [pull_request]

jobs:
  finops-guardrail:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -e .

      - name: Terraform Show and CostGuard Check
        run: |
          terraform show -json tfplan.binary | costguard --max-increase 100.00
```

### GitLab CI
```yaml
cost_guardrail:
  stage: test
  image: python:3.11
  script:
    - pip install -e .
    - terraform show -json tfplan.binary | costguard --max-increase 100.00
```

---

## 🧪 Testing

Run the full pytest suite covering unit and integration tests:
```bash
pytest -v
```

All 38 test cases cover parser actions, Azure OData query creation, Spot/Low-Priority exclusion, SQLite write-through caching, cost calculations, currency support, and CLI exit codes.

---

## 📂 Project Structure

```text
costguard/
├── costguard/
│   ├── __init__.py           # Package version and metadata
│   ├── __main__.py           # CLI entry point (python -m costguard)
│   ├── cache.py              # SQLite write-through caching (pricing_cache.db)
│   ├── calculator.py         # Cost computation (hourly * 730) & deltas
│   ├── cli.py                # Command-line interface & budget validation
│   ├── parser.py             # Terraform plan JSON parsing & lifecycle actions
│   ├── pricing.py            # Azure Retail Prices API client & OData filters
│   └── report.py             # Formatted terminal reporting & summary UI
├── sample_plans/
│   ├── managed_disk.json     # Managed disk creation plan
│   ├── non_billable.json     # Non-billable resources (RG, VNet, Subnet, NSG)
│   ├── unknown_sku.json      # Unpriced / unknown SKU plan
│   ├── vm_create.json        # VM creation plan (Standard_B2s)
│   ├── vm_delete.json        # VM deletion plan (Standard_D2s_v3)
│   ├── vm_replace.json       # VM replacement plan (delete + create)
│   └── vm_upgrade.json       # VM update/upgrade plan (B2s -> D4s_v3)
├── tests/
│   ├── test_cache.py         # Cache hit/miss & write-through tests
│   ├── test_calculator.py    # Cost calculation & budget guardrail tests
│   ├── test_cli.py           # CLI flags, stdin, and exit code tests
│   ├── test_parser.py        # Terraform plan parser & action detection tests
│   └── test_pricing.py       # Azure API query & meter filter tests
├── pyproject.toml            # Python packaging & build configuration
├── requirements.txt          # Python dependencies
├── README.md                 # User guide and documentation
└── REPORT.md                 # Engineering & architectural hackathon report
```
