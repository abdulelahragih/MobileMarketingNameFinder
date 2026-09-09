# Run Instructions: Mobile Marketing Name Finder

## 1. Overview
Mobile Marketing Name Finder is a lightweight Go microservice designed to map Android device technical model numbers and retail branding names to their official marketing names (e.g. mapping `Samsung` and `SM-G991B` to `Galaxy S21 5G`).

- **Language & Runtime:** Go 1.23 (`go 1.23.2`)
- **Database:** Embedded SQLite3 (`./devices.db`) managed via `github.com/mattn/go-sqlite3` (requires CGO)
- **External Data Source:** Google Play Supported Devices CSV (`https://storage.googleapis.com/play_public/supported_devices.csv`), fetched on demand
- **Architecture:** Single-process HTTP REST API microservice utilizing standard library HTTP router (`net/http`) and local SQLite database

---

## 2. Prerequisites

You can run this microservice either using **Docker** (recommended) or **locally with native Go**.

### Option A: Docker (Recommended)
- **Docker Engine:** Version 20.10+ (tested with Docker 29+)
- **Docker Compose:** Version 2.0+

### Option B: Native Host Environment
- **Go:** Version 1.23 or higher
- **C Compiler (CGO Required):**
  - **macOS:** Xcode Command Line Tools (`xcode-select --install`)
  - **Linux (Debian/Ubuntu):** `build-essential` (`sudo apt install build-essential`)
  - **Linux (Alpine):** `gcc` and `musl-dev` (`apk add --no-cache gcc musl-dev`)
- **Network Access:** Internet connection to allow fetching the Google Play CSV dataset on sync

---

## 3. Environment & Configuration
This microservice does not require any `.env` file or external API keys. All configuration is internal and self-contained:
- **Default Port:** `8089` (hardcoded in `main.go`)
- **Database File:** `./devices.db` (created automatically in the working directory on startup)
- **Data Source URL:** `https://storage.googleapis.com/play_public/supported_devices.csv`

---

## 4. Installation & Setup

### Option A: Using Docker (Automated)
No manual dependency installation is required. The multi-stage Docker build handles downloading Go modules, compiling with CGO (`musl-dev` and `gcc`), and packaging into a minimal Alpine container.

### Option B: Native Host Installation
1. Navigate to the project root:
   ```bash
   cd /Users/abdulelah/add_run_instructions/MobileMarketingNameFinder
   ```

2. Download Go module dependencies:
   ```bash
   go mod download
   ```

3. Build the binary with CGO enabled:
   ```bash
   CGO_ENABLED=1 go build -o main .
   ```

---

## 5. Running the Application

### Option A: Running with Docker Compose (Recommended)
1. Build and launch the container in the foreground:
   ```bash
   docker compose up --build
   ```
   *(Or in detached mode: `docker compose up -d --build`)*

2. Verify the container is running:
   ```bash
   docker compose ps
   ```

3. Stop the container:
   ```bash
   docker compose down
   ```

### Option B: Running Natively
1. Execute the compiled binary:
   ```bash
   ./main
   ```
   *Or run directly with Go:*
   ```bash
   CGO_ENABLED=1 go run main.go
   ```

2. The server will output:
   ```text
   Server started at :8089
   ```

### Ports & Endpoints
- **Base URL:** `http://localhost:8089`
- **Endpoints:**
  - `POST /update-devices`: Downloads the Google Play device database CSV and populates/updates the SQLite database.
  - `POST /get-device-name`: Queries the marketing name for a given brand and model.

---

## 6. Testing & Verification

### 1. Initialize / Seed the Database
On initial startup, the local SQLite database table `devices` is created but empty. Populate it with the latest Android devices data:
```bash
curl -X POST http://localhost:8089/update-devices
```
**Expected Response:**
```text
Devices updated successfully
```
*(Note: Processing the entire CSV file with tens of thousands of devices may take a few seconds).*

### 2. Query a Device Marketing Name
Send a `POST` request with `retail_branding` and `model`:
```bash
curl -X POST http://localhost:8089/get-device-name \
  -H "Content-Type: application/json" \
  -d '{"retail_branding": "Samsung", "model": "SM-G991B"}'
```
**Expected Response (HTTP 200):**
```json
{"data":"Galaxy S21 5G"}
```

### 3. Test Device Not Found
Send a request for a non-existent device:
```bash
curl -i -X POST http://localhost:8089/get-device-name \
  -H "Content-Type: application/json" \
  -d '{"retail_branding": "UnknownBrand", "model": "XYZ-999"}'
```
**Expected Response (HTTP 404):**
```text
Device not found
```

### 4. Automated Tests
Currently, there are no Go unit test files (`*_test.go`) in the repository. To run tests if added in the future:
```bash
go test -v ./...
```

---

## 7. Troubleshooting & FAQ

- **Issue: `gcc: not found` or `cgo: C compiler "gcc" not found in PATH` during native build**
  - **Fix:** `go-sqlite3` requires CGO to compile SQLite C source code. Ensure a C compiler is installed (`xcode-select --install` on macOS, or `sudo apt install build-essential` on Ubuntu). Alternatively, run using Docker.

- **Issue: Query returns `Device not found` for valid devices immediately after launch**
  - **Cause:** The database begins empty on first run.
  - **Fix:** Call `curl -X POST http://localhost:8089/update-devices` to download and import Google Play device catalog into SQLite.

- **Issue: Data is lost when Docker container is restarted or recreated**
  - **Cause:** `docker-compose.yaml` does not define a persistent volume mount for `/root/devices.db`.
  - **Fix:** Add a volume mapping to `docker-compose.yaml` under the `microservice` service:
    ```yaml
    volumes:
      - ./data:/root
    ```

- **Issue: `bind: address already in use` on port `8089`**
  - **Cause:** Another service is already running on port `8089`.
  - **Fix:** Check what process is occupying port 8089 (`lsof -i :8089`) or change the host port mapping in `docker-compose.yaml` (e.g. `"8090:8089"`).

- **Issue: `/update-devices` fails with `failed to fetch CSV`**
  - **Cause:** Host or container cannot reach `https://storage.googleapis.com`.
  - **Fix:** Check your internet connectivity, DNS configuration, or proxy settings.
