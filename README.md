# AAQ Analysis Tool

> A fully client-side web application for analysing Application Assessment Questionnaire (AAQ) Excel files as part of a cloud migration assessment. Built for Kyndryl cloud migration engagements.

**No data ever leaves your browser.** The file is parsed entirely in-memory using JavaScript — nothing is uploaded to a server.

---

## Features

- **Server Inventory** — extracts workload names, vCPU, and RAM from the Server sheet
- **AWS Sizing** — recommends EC2 instance types (eu-west-2) based on vCPU and memory requirements
- **Database Inventory** — parses the Database sheet and normalises size units (MB → GB)
- **Firewall Analysis** — aggregated and per-server views of network flows with usage classification (HIGH / MEDIUM / LOW / VERY LOW) based on netstat counts
- **Export** — download full analysis or individual sections as `.xlsx` files, all generated client-side
- **Search & Filter** — filter firewall and database tables by keyword or usage level

---

## Privacy & Security

This tool is designed for use with sensitive client data. All processing happens in the browser:

- No backend server receives or stores any file content
- No analytics, telemetry, or third-party tracking
- The Docker image ships with a `Content-Security-Policy: connect-src 'none'` header, enforcing at the network level that the app makes zero outbound requests
- The final Docker image contains only static HTML/JS/CSS served by Nginx — there is no runtime to exploit

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI Framework | React 18 |
| Build Tool | Vite 5 |
| Excel Parsing | SheetJS (xlsx) |
| Excel Export | SheetJS (xlsx) |
| Fonts | Syne + JetBrains Mono (Google Fonts) |
| Container | Node 20 Alpine (build) → Nginx Alpine (serve) |

---

## Project Structure

```
aaq-tool/
├── src/
│   ├── main.jsx              # React entry point
│   └── AAQAnalysisTool.jsx   # Full application (single file)
├── index.html                # HTML shell
├── vite.config.js            # Vite configuration
├── package.json
├── nginx.conf                # Nginx SPA config + security headers
├── Dockerfile                # Multi-stage build
└── .dockerignore
```

---

## Running with Docker

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed

### Build and run

```bash
docker build -t aaq-tool .
docker run -p 8080:80 aaq-tool
```

Open [http://localhost:8080](http://localhost:8080) in your browser.

### Run in the background

```bash
docker run -d --name aaq-tool -p 8080:80 aaq-tool
docker stop aaq-tool   # to stop
docker rm aaq-tool     # to remove
```

---

## Running Locally (without Docker)

### Prerequisites

- [Node.js 18+](https://nodejs.org/)

### Install and start

```bash
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173).

### Production build

```bash
npm run build       # outputs to dist/
npm run preview     # preview the production build locally
```

---

## Deploying

The Docker image is a self-contained static site. Push it to any container registry and deploy to your platform of choice.

### Azure Container Apps

```bash
az acr build --registry <your-registry> --image aaq-tool:latest .
az containerapp create \
  --name aaq-tool \
  --resource-group <rg> \
  --image <your-registry>.azurecr.io/aaq-tool:latest \
  --target-port 80 \
  --ingress external
```

### AWS App Runner

```bash
docker tag aaq-tool <account>.dkr.ecr.eu-west-2.amazonaws.com/aaq-tool:latest
docker push <account>.dkr.ecr.eu-west-2.amazonaws.com/aaq-tool:latest
# Then create an App Runner service pointing at the ECR image
```

### Any other platform

The `dist/` folder produced by `npm run build` is plain static HTML/JS/CSS and can be hosted on any web server, CDN, or static hosting service (e.g. Azure Static Web Apps, AWS S3 + CloudFront, GitHub Pages).

---

## Expected AAQ Excel Format

The parser uses fuzzy sheet name matching and is tolerant of minor variations, but expects the following structure:

### Server sheet

| Column | Notes |
|---|---|
| `WorkloadName` | Required — used as the server identifier |
| `Function` | Application/function description |
| `CPUCount` | vCPU count (also accepts `TotalCPUCores`) |
| `RAM` / `Memory` | In bytes (auto-converts) or GiB |

Rows below a row containing "dependency" in the workload name are excluded.

### Firewall sheet

| Column | Notes |
|---|---|
| `src_ip` / `dest_ip` | Required — rows without both are skipped |
| `source_hostname` / `dest_hostname` | Display names |
| `source_stackname` / `dest_stackname` | Application names |
| `src_port` / `dest_port` | Port numbers |
| `protocol` | e.g. TCP, UDP |
| `netstat_count` | Flow count used for usage classification |

**Usage classification thresholds:**

| Count | Level |
|---|---|
| ≥ 50,000 | HIGH |
| ≥ 1,000 | MEDIUM |
| ≥ 100 | LOW |
| < 100 | VERY LOW |

### Database sheet

| Column | Notes |
|---|---|
| `DB Server Name` | Server hosting the database |
| `Database Name` | Database name |
| `Database Instance` | Instance name |
| `Database Size (MB)` or `Database Size (GB)` | Auto-detected unit |
| `Database Type` | e.g. MSSQL, Oracle, MySQL |

---

## AWS Sizing Notes

Instance recommendations are based on a static snapshot of eu-west-2 on-demand pricing for the following families: `t3`, `m5`, `c5`, `r5`. The tool selects the three smallest instances that meet or exceed the server's vCPU and memory requirements.

To update pricing, edit the `INSTANCE_DATA` array in `AAQAnalysisTool.jsx`.

For live pricing data, the [Vantage Instances API](https://instances.vantage.sh) can be queried at build time and baked into the static bundle.

---

## Migrating from the Streamlit Version

The original prototype used a Python/Streamlit backend (`streamlit_app.py`) with server-side parsing. This React version is a full rewrite that preserves all functionality while moving all processing to the client. Key differences:

| | Streamlit version | This version |
|---|---|---|
| File processing | Server-side (Python) | Client-side (JavaScript) |
| Data storage | In-memory, session-scoped | Never leaves browser |
| AWS pricing | Live Vantage API call | Static snapshot |
| Deployment | Python runtime required | Static files + Nginx |
| Image size | ~500MB+ | ~25MB |

---

## Licence

[MIT + Common Clause](LICENSE) — Copyright (c) 2025 Kyndryl.

Free to use, modify, and distribute. You may **not** sell this software or offer it as a paid product or service. See [LICENSE](LICENSE) for full terms.
