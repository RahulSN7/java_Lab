# ShopSphere — E-Commerce Web Application

**ShopSphere** is an enterprise e-commerce application built with **Java 21**, **javax.servlet (Servlet 4.0 / JSP 2.3)**, **MySQL 8.0/8.4**, and **Apache Tomcat 9.0.121**.

> **Windows users:** All commands below are formatted for Windows PowerShell or the Windows Command Prompt unless noted otherwise.

---

## Architecture Overview

```
                          +---------------------------------------------+
                          |             Client Applications             |
                          +---------------------------------------------+
                          | 1. Web Browser (Bootstrap 5.3 + JSP Views)  |
                          | 2. Desktop GUI (Java Swing PLAF)            |
                          +---------------------------------------------+
                                                 |
                                                 v
+-------------------------------------------------------------------------------------+
|                              Apache Tomcat 9.0.121                                  |
|                                                                                     |
|  +-----------------------------------+    +---------------------------------------+ |
|  |       Lifecycle Listeners         |    |          Web Controllers              | |
|  | - DatabaseInitializerListener     |    | - LoginServlet    - RegisterServlet   | |
|  |   (Safe Schema Verification)      |    | - ProductServlet  - CartServlet       | |
|  +-----------------------------------+    | - CheckoutServlet - LogoutServlet     | |
|                                           +---------------------------------------+ |
|  +-----------------------------------+                       |                      |
|  |       Security Filters            |                       v                      |
|  | - AuthenticationFilter (Session)  |    +---------------------------------------+ |
|  | - AdminFilter (Role: 'ADMIN')     |    |           Service Layer               | |
|  +-----------------------------------+    | - ProductService  - OrderService      | |
|                                           | - CheckoutService (Server Pricing)    | |
|                                           +---------------------------------------+ |
|                                                              |                      |
|                                                              v                      |
|                                           +---------------------------------------+ |
|                                           |        DAO & Data Layer               | |
|                                           | - UserDAO   - ProductDAO              | |
|                                           | - CartDAO   - OrderDAO                | |
|                                           | - CouponDAO - PaymentDAO              | |
|                                           | (PreparedStatement, Connection Pool)  | |
|                                           +---------------------------------------+ |
+-------------------------------------------------------------------------------------+
                                                 |
                                                 v
                          +---------------------------------------------+
                          |            MySQL 8.0/8.4 RDBMS              |
                          |         Database: `shopsphere`              |
                          |  (12 Tables, Foreign Keys, Seed Data)       |
                          +---------------------------------------------+
```

---

## Technology Stack

| Component | Target Version | Description |
|---|---|---|
| Java (JDK) | 21 LTS | Compiles with `-release 21` |
| Servlet API | javax.servlet 4.0.1 | Standard Java EE / Jakarta javax namespace for Tomcat 9 |
| Application Server | Apache Tomcat 9.0.121 | Servlet 4.0 / JSP 2.3 container |
| MySQL Server | 8.0+ or 8.4 LTS | InnoDB, `utf8mb4_unicode_ci` |
| MySQL Connector/J | 8.4.0 | Runtime JDBC driver (`com.mysql.cj.jdbc.Driver`) |
| Build Tool | Apache Maven 3.9+ | WAR packaging |
| Connection Pool | HikariCP 5.1.0 | High-performance JDBC connection pooling |
| JSON Processing | Jackson 2.17.2 | JSON serialization |
| Password Security | PBKDF2 / jBCrypt 0.4 | Multi-iteration salted password hashing |
| Unit Testing | JUnit Jupiter 5.10.3 | Unit & integration test framework |

---

## Prerequisites

Ensure the following tools are installed on your Windows machine:

### 1. Java Development Kit (JDK 21)
Verify Java 21 is installed:
```powershell
java -version
javac -version
```
Both commands must report version `21`.

### 2. MySQL Server (8.0 or 8.4 LTS)
Verify the Windows MySQL service is running:
```powershell
Get-Service -Name MySQL80
```
If `Status` is `Stopped`, start it:
```powershell
Start-Service -Name MySQL80
```

### 3. Apache Maven (3.9+)
Verify Maven is on your `PATH`:
```powershell
mvn -version
```

### 4. Apache Tomcat 9.0.121
Extract Tomcat 9.0.121 to a folder such as:
- `C:\apache-tomcat-9.0.121` or
- `C:\Users\<YourUser>\Downloads\apache-tomcat-9.0.121\apache-tomcat-9.0.121`

---

## Database Setup & Initialization

The project provides two SQL scripts in the root directory:

| Script | Type | When to Use |
|---|---|---|
| [`schema-init.sql`](schema-init.sql) | **Safe / Non-Destructive** | **Recommended for all setups.** Uses `CREATE TABLE IF NOT EXISTS` and `INSERT IGNORE`. Never drops tables or deletes user data. |
| [`shopsphere.sql`](shopsphere.sql) | **Destructive Reset** | Contains `DROP TABLE IF EXISTS`. Use **only** when you want to wipe the database and start over. |

---

### Method 1 — Recommended: PowerShell Automated Script

Run the automated initializer from the project root:
```powershell
powershell -ExecutionPolicy Bypass -File scripts\init-db.ps1
```
The script will:
1. Prompt you securely for your MySQL password (password is never displayed or written to disk).
2. Automatically locate your local `mysql.exe` client.
3. Apply [`schema-init.sql`](schema-init.sql) non-destructively.
4. Verify that all 12 tables and seed data are created.

---

### Method 2 — Manual Setup via MySQL Workbench

If you prefer using the MySQL Workbench graphical interface:

1. Open **MySQL Workbench** and connect to your local MySQL connection (e.g. `Local instance MySQL80`).
2. Open a query tab and create the database if it doesn't already exist:
   ```sql
   CREATE DATABASE IF NOT EXISTS `shopsphere`
       CHARACTER SET utf8mb4
       COLLATE utf8mb4_unicode_ci;
   ```
3. Go to **File** &rarr; **Open SQL Script...** (or press `Ctrl + Shift + O`).
4. Select the file: `F:\java_Lab\java_Lab_Project\schema-init.sql`.
5. Click the **Execute** button (the yellow lightning bolt icon ⚡).
6. In the left **Navigator** pane under **Schemas**, right-click inside the schema area and select **Refresh All**.
7. Expand `shopsphere` &rarr; `Tables` and verify that the 12 tables appear:
   - `addresses`
   - `cart`
   - `categories`
   - `coupon_usage`
   - `coupons`
   - `order_items`
   - `orders`
   - `payments`
   - `products`
   - `reviews`
   - `users`
   - `wishlist`

---

### Method 3 — Manual Setup via MySQL Command Line Client

If `mysql.exe` is in your `PATH` or in `C:\Program Files\MySQL\MySQL Server 8.0\bin\`:

```powershell
# Open PowerShell in the project root:
& "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" -u root -p < schema-init.sql
```
Enter your MySQL password when prompted.

---

## Configuring Database Credentials

The application's `DBConnection` class reads configuration from environment variables.

### Supported Environment Variables

| Variable | Description | Default | Example |
|---|---|---|---|
| `DB_URL` | Direct JDBC Connection URL | *(Derived from host/port/name)* | `jdbc:mysql://localhost:3306/shopsphere?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true` |
| `DB_HOST` | MySQL hostname | `localhost` | `localhost` |
| `DB_PORT` | MySQL port | `3306` | `3306` |
| `DB_NAME` | MySQL database name | `shopsphere` | `shopsphere` |
| `DB_USER` | MySQL username | *(Required)* | `root` |
| `DB_PASSWORD` | MySQL password | *(Required)* | *(Your real MySQL password)* |

> **Security Note:** Passwords are never hardcoded in source code, printed to logs, or committed to Git.

---

### Option A — Recommended: One-Click PowerShell Launcher (`run-local.ps1`)

The local launcher sets all environment variables, compiles the project, deploys the WAR, and launches Tomcat in one step:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\run-local.ps1
```
- It asks for your MySQL password using a secure, non-echoing prompt.
- It sets `DB_URL`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD` only in the current session memory.
- It automatically detects your Tomcat 9 installation.
- It compiles the project with `mvn clean package`.
- It copies `target\shopsphere.war` to Tomcat's `webapps\` folder and starts Tomcat.

---

### Option B — Tomcat `setenv.bat` (Recommended for Independent Tomcat Runs)

When starting Tomcat directly via `startup.bat` or `catalina.bat`, Tomcat automatically executes `<Tomcat>\bin\setenv.bat` on startup to load environment variables.

1. Copy the template from [`scripts\setenv.bat.template`](scripts/setenv.bat.template) to your Tomcat `bin\` directory:
   ```powershell
   # Replace the path with your actual Tomcat installation directory:
   Copy-Item scripts\setenv.bat.template C:\apache-tomcat-9.0.121\bin\setenv.bat
   ```
2. Open `<Tomcat>\bin\setenv.bat` in Notepad:
   ```powershell
   notepad C:\apache-tomcat-9.0.121\bin\setenv.bat
   ```
3. Update `YOUR_MYSQL_PASSWORD_HERE` with your actual MySQL password:
   ```bat
   set "DB_HOST=localhost"
   set "DB_PORT=3306"
   set "DB_NAME=shopsphere"
   set "DB_URL=jdbc:mysql://localhost:3306/shopsphere?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true"
   set "DB_USER=root"
   set "DB_PASSWORD=YOUR_ACTUAL_PASSWORD"
   ```
4. Save and close the file.

---

### Option C — Persistent Windows User Environment Variables

If you want the variables to persist across all PowerShell windows:

```powershell
[System.Environment]::SetEnvironmentVariable("DB_URL", "jdbc:mysql://localhost:3306/shopsphere?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true", "User")
[System.Environment]::SetEnvironmentVariable("DB_HOST", "localhost", "User")
[System.Environment]::SetEnvironmentVariable("DB_PORT", "3306", "User")
[System.Environment]::SetEnvironmentVariable("DB_NAME", "shopsphere", "User")
[System.Environment]::SetEnvironmentVariable("DB_USER", "root", "User")
[System.Environment]::SetEnvironmentVariable("DB_PASSWORD", "YOUR_ACTUAL_PASSWORD", "User")
```
*Note: Restart any open PowerShell or Command Prompt windows for these changes to take effect.*

---

## Building the Project

Compile and package the application using Apache Maven:

```powershell
# From the project root (F:\java_Lab\java_Lab_Project):
mvn clean package
```

Expected successful output:
```
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  ...
[INFO] Finished at: ...
[INFO] ------------------------------------------------------------------------
```
The deployable web archive will be generated at:
`F:\java_Lab\java_Lab_Project\target\shopsphere.war`

---

## Deploying to Apache Tomcat 9.0.121

1. Ensure the Maven build succeeded (`target\shopsphere.war` exists).
2. Copy `shopsphere.war` into your Tomcat `webapps\` directory:
   ```powershell
   # Adjust Tomcat path as needed:
   Copy-Item target\shopsphere.war C:\apache-tomcat-9.0.121\webapps\ -Force
   ```

---

## Starting and Stopping Apache Tomcat 9

### Starting Tomcat

#### Option 1: Foreground Mode (Console output visible in current window)
```powershell
# Run from your Tomcat bin directory:
cd C:\apache-tomcat-9.0.121\bin
.\catalina.bat run
```
*Press `Ctrl + C` to stop Tomcat.*

#### Option 2: Background / Separate Window Mode
```powershell
cd C:\apache-tomcat-9.0.121\bin
.\startup.bat
```
A new console window titled **Tomcat** will appear and display the server output.

### Stopping Tomcat
```powershell
cd C:\apache-tomcat-9.0.121\bin
.\shutdown.bat
```

---

## Accessing the Application

Once Tomcat reports server startup in the console (e.g. `Server startup in [XXXX] milliseconds`), open your web browser:

| Page | URL | Description |
|---|---|---|
| **Home (Product Catalog)** | [http://localhost:8080/shopsphere/](http://localhost:8080/shopsphere/) | Redirects to `/products` and displays available bicycles |
| **Product Catalog** | [http://localhost:8080/shopsphere/products](http://localhost:8080/shopsphere/products) | Browse bikes, filter by category/price, search SKU/name |
| **Shopping Cart** | [http://localhost:8080/shopsphere/cart](http://localhost:8080/shopsphere/cart) | View, update quantities, or remove cart items |
| **Checkout** | [http://localhost:8080/shopsphere/checkout](http://localhost:8080/shopsphere/checkout) | Server-side pricing calculation, coupon redemption & order placement |
| **Customer Login** | [http://localhost:8080/shopsphere/login](http://localhost:8080/shopsphere/login) | Customer and administrator sign-in |
| **Customer Register** | [http://localhost:8080/shopsphere/register](http://localhost:8080/shopsphere/register) | Account registration with PBKDF2 password hashing |
| **Admin Dashboard** | [http://localhost:8080/shopsphere/admin/dashboard.jsp](http://localhost:8080/shopsphere/admin/dashboard.jsp) | Protected store management (Admin role required) |

---

## Pre-Configured Test Accounts

The seed data in [`schema-init.sql`](schema-init.sql) includes two test user accounts:

| Role | Email | Password | Details |
|---|---|---|---|
| **Administrator** | `admin@shopsphere.com` | `Password123!` | Has `admin` role; can access `/admin/*` |
| **Customer** | `customer@shopsphere.com` | `Password123!` | Standard customer account with pre-seeded address |

### Available Promotional Coupons

| Coupon Code | Discount Type | Discount Value | Minimum Order | Usage Limit |
|---|---|---|---|---|
| `BIKE10` | Percentage | 10% (max $200) | $100.00 | 500 redemptions |
| `RIDE50` | Fixed Amount | $50.00 | $500.00 | 1000 redemptions |
| `SAVE10` | Percentage | 10% (max $25) | $50.00 | 500 redemptions |
| `WELCOME20` | Fixed Amount | $20.00 | $100.00 | 1000 redemptions |

---

## Troubleshooting Guide

### 1. HTTP Status 500 — `java.sql.SQLSyntaxErrorException: Table 'shopsphere.products' doesn't exist`
- **Root Cause:** The `shopsphere` database exists on MySQL, but no tables were initialized.
- **Solution:** Run the safe database initializer script:
  ```powershell
  powershell -ExecutionPolicy Bypass -File scripts\init-db.ps1
  ```
  Or open [`schema-init.sql`](schema-init.sql) in MySQL Workbench and execute it. Then refresh the page.

### 2. Startup Failure — `ShopSphere Database Schema Incomplete: Database 'shopsphere' is missing required tables: [...]`
- **Root Cause:** The application's `DatabaseInitializerListener` detected missing tables during web app deployment to protect against broken runtime transactions.
- **Solution:** Execute [`schema-init.sql`](schema-init.sql) in MySQL Workbench or run `scripts\init-db.ps1`.

### 3. `IllegalStateException: Database configuration error: DB_PASSWORD environment variable is not set`
- **Root Cause:** Tomcat was started without inheriting the `DB_PASSWORD` (and/or `DB_USER`) environment variable.
- **Solution:**
  - If running via PowerShell: Launch using `powershell -ExecutionPolicy Bypass -File scripts\run-local.ps1`.
  - If starting Tomcat from its `bin\` folder: Ensure `<Tomcat>\bin\setenv.bat` exists and has your real password set.

### 4. `CommunicationsLinkException: Communications link failure`
- **Root Cause:** The MySQL server is not running, or is listening on a different port than 3306.
- **Solution:**
  - Check MySQL service: `Get-Service -Name MySQL80`
  - Start MySQL: `Start-Service -Name MySQL80`
  - Verify port: Test connectivity with `Test-NetConnection -ComputerName localhost -Port 3306`.

### 5. HTTP Status 404 — `The origin server did not find a current representation for the target resource`
- **Root Cause:** `shopsphere.war` has not been deployed, or the context path differs.
- **Solution:**
  - Verify that `shopsphere.war` was copied to `<Tomcat>\webapps\shopsphere.war`.
  - Check that Tomcat has extracted it into `<Tomcat>\webapps\shopsphere\`.
  - Ensure you are navigating to `http://localhost:8080/shopsphere/` (with the `/shopsphere/` context prefix).

### 6. Port 8080 Already in Use
- **Root Cause:** Another application is already using port 8080.
- **Solution:**
  - Find the occupying process: `Get-NetTCPConnection -LocalPort 8080`
  - Alternatively, edit `<Tomcat>\conf\server.xml` and change `<Connector port="8080"` to `<Connector port="8088"`.
