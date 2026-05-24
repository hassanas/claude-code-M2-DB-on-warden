# Connecting Claude Desktop to a Local Warden (MariaDB) Database via MCP

This document provides the complete, step-by-step setup to connect Claude Desktop running on Windows to an isolated, Docker-based Magento database managed by Warden.

---

## 🛠️ Prerequisites
*   **Claude Desktop App** (Windows)
*   **Miniconda / Python Environment** (With Python >= 3.12)
*   **Warden Development Environment** (Running your Magento project)

---

## 🏎️ Step 1: Install the Python Package Manager (uv)
Claude Desktop executes the MySQL/MariaDB server connection reliably using the `uv` environment tool. 

Open your Windows **PowerShell** prompt and install `uv` globally via pip:
```powershell
pip install uv
```
Verify the installation path by running:
```powershell
where.exe uv
```
*(Take note of the output path, e.g., `C:\Users\<YOUR_USER>\miniconda3\Scripts\uv.exe`)*

---

## 🐳 Step 2: Open Warden Network Ports & Permissions

Because your database container runs inside an isolated WSL2 Docker network bridge, you must explicitly expose its TCP port and database credentials to your local Windows host system.

### 1. Update Your Warden Environment Configuration
Open your project's existing configuration file located at `.warden/warden-env.yml` and ensure your `db` block maps port `3306` to your Windows host machine:

```yaml
services:
  db:
    ports:
      - "3306:3306"
```

### 2. Apply Configuration and Rebuild Containers
Run these commands in your project's WSL terminal to apply the network overrides:
```bash
warden env down
warden env up --force-recreate
```

### 3. Grant Global Network Privileges to Root
By default, MariaDB blocks root connections originating from outside the container network subnet. Force an administrative bypass by executing a direct docker command:
```bash
docker exec -it urpay-db-1 mysql -u root -pmagento
```
Once inside the `MariaDB [(none)]>` console prompt, run this authorization query:
```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'magento' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

---

## ⚙️ Step 3: Configure Claude Desktop MCP Settings

1. Open **Claude Desktop**.
2. Navigate to **Settings** > **Developer** > **Edit Config**.
3. Replace the text block completely with this structural mapping configuration. Ensure you update the executable path string to match your exact `where.exe uv` target:

```json
{
  "mcpServers": {
    "mysql-database": {
      "command": "C:\\Users\\has-v\\miniconda3\\Scripts\\uv.exe",
      "args": [
        "tool",
        "run",
        "--python", "3.12",
        "mcp-server-mysql"
      ],
      "env": {
        "MYSQL_HOST": "localhost",
        "MYSQL_PORT": "3306",
        "MYSQL_USER": "root",
        "MYSQL_PASSWORD": "magento",
        "MYSQL_DATABASE": "magento"
      }
    },
    "filesystem": {
      "command": "C:\\Program Files\\nodejs\\npx.cmd",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "C:\\Users\\has-v\\OneDrive\\Desktop",
        "C:\\Users\\has-v\\Downloads"
      ]
    }
  }
}
```

---

## 🔄 Step 4: Complete System Initialization
1. Completely **Quit** Claude Desktop from your Windows system tray icon (bottom-right clock taskbar menu).
2. Relaunch Claude Desktop.
3. Open a **brand new chat window**.
4. Confirm a solid, active **hammer/plug icon** appears in the bottom right corner of the message line.

---

## 📈 Step 5: Business Team Reporting Template

Copy and paste this plain English prompt to generate sales audit reports. The business team can modify the date parameters or merchant ID directly without typing SQL code:

```markdown
Please generate a Merchant Sales Report by looking up our marketplace data using the following criteria:

### 1. REPORT FILTERS (Business Team: Modify these values as needed)
*   **Merchant Seller ID:** 83
*   **Date Range:** From December 1, 2025, through January 31, 2026
*   **Order Status:** Include only successfully finalized orders (Closed or Complete)
*   **Exclusions:** Exclude any orders that have a refund or credit memo issued against them

### 2. CORE SYSTEM DATA POINT MAPPING
When querying the tables, please match the business requests to these specific columns and tables:
*   **Order Tracking:** Use order increment identifier (`so.increment_id`), status (`so.status`), and creation date (`so.created_at`).
*   **Payment Details:** Extract payment method code (`sop.method`) and transaction gateway tracking ID (`sop.arb_ref_id`).
*   **Merchant Metadata:** Pull the store business name (`mu.shop_title`) based on the merchant list identifier (`ms.seller_id`).
*   **Product Particulars:** Pull the barcode stock code (`soi.sku`), custom marketplace item name (`ms.magepro_name`), and item format classification (`soi.product_type`).
*   **Financial Rows:** Calculate using marketplace item count (`ms.magequantity`), item row gross total (`ms.total_amount`), total marketplace revenue cuts (`ms.total_commission`), calculation multiplier percentage (`ms.commission_rate`), freight charges (`so.shipping_amount`), and invoice overall totals (`so.grand_total`).
*   **Exclusion Join:** Confirm data validation by checking that no match exists in the credit registry identifier (`sc.entity_id`).

### 3. REPORT FORMAT
Once you pull the data, compile it into the following sections:
*   **Executive Summary:** A short summary of how this specific merchant performed during this date window.
*   **Financial Breakdown:** Provide totals for Gross Sales Volume, Total Marketplace Commissions Collected, Net Merchant Earnings, Total Orders Placed, and Average Order Value (AOV).
*   **Product Performance:** A table showing the top-selling products by quantity.
*   **Transaction Ledger:** A clean markdown table displaying the individual itemized sales rows for their records.
```
