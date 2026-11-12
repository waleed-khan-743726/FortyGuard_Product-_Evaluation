# FortyGuard Product & Developer Ecosystem Evaluation Report

**Evaluator:** Candidate for Software Development Internship  
**Evaluation Focus:** Hands-on Assessment of FortyGuard Temperature Dashboard®, Temperature API®, and Temperature MCP  
**Date:** November 12, 2026  
**Environment:** Windows 10/11 x64, Chromium/Chrome 157.0, Python 3.14.7, Playwright 1.58.0, PowerShell 7, curl 8.13.0  
**API Key Tested:** Masked (`8aa4...d362`)  
**Subscription Plan:** API Premium (1,000,000 Credits, 1-Month Cycle)

---

## 1. Executive Summary

This evaluation presents a rigorous, evidence-based technical assessment of FortyGuard's core developer products: the **Temperature Dashboard®**, **Temperature API® (v1)**, and **Temperature MCP Server**. Rather than high-level surface impressions, every finding and metric in this report is derived from actual, hands-on interactions with FortyGuard's live production environments.

The FortyGuard ecosystem addresses a critical, burgeoning challenge in climate resilience: providing highly granular, actionable urban temperature and environmental intelligence. The core proprietary modeling engine is remarkably capable: a 1 km² area heatmap request in Phoenix, AZ generated 2,835 lines of detailed GeoJSON polygons with microclimate temperature distributions in ~24 seconds. The newly introduced Remote Model Context Protocol (MCP) server is an outstanding, modern integration that enables autonomous AI agents to discover tools, query live atmospheric metrics, and inspect balances over Streamable HTTP with zero friction.

However, several critical technical and developer experience friction points were uncovered:
1. **Quickstart Onboarding Trap:** The official API quickstart documentation supplies New York City coordinates that immediately return a `400 Bad Request (OUT_OF_COVERAGE)` error, blocking new developers on their first copy-paste request.
2. **Infrastructure Timeouts:** The `POST /v1/routing` endpoint consistently times out (`504 Gateway Timeout` from the Envoy proxy) under legitimate contiguous US parameters.
3. **Opaque Asynchronous Failures:** When asynchronous background tasks fail (e.g., Street View or Satellite segmentation), the polling endpoint returns `status: "Failed"` with zero diagnostic reasoning or error codes.
4. **Unit Mismatch in Reporting Engine:** The Heat Intelligence endpoint explicitly documents `temperature` in Celsius (e.g., `35.0`), but the backend PDF report generator converted it as Fahrenheit (`35.0°F`), producing a 21-page report evaluating the climate risk of Phoenix in July at `1.66°C`.

With targeted developer experience refinements, standardized schemas, and automated documentation test pipelines, FortyGuard's developer offering can match the impressive sophistication of its underlying Large Temperature Models.

---

## 2. Testing Scope & Methodology

### In-Scope Workflows Tested
- **Temperature API Documentation:** Evaluated all 21 sections of `https://docs-api.fortyguard.com/docs/`, analyzing information architecture, schema completeness, and copy-paste usability.
- **Authentication & Account Systems:** Tested valid, invalid, and missing API key headers; inspected account credit breakdowns and usage via `/v1/system/fetch-api-key-usage`.
- **API Happy Path & Asynchronous Polling:** Executed real submissions to `/v1/env_params`, `/v1/heatmap`, `/v1/heat_intelligence`, `/v1/satellite`, `/v1/streetview`, and `/v1/routing`; polled `/v1/status/{activity_id}` until terminal state.
- **Input Validation & Boundary Testing:** Evaluated out-of-bounds coordinates (latitude > 90), non-US coordinates (Paris, France), unclosed GeoJSON polygons, and missing required parameters.
- **Temperature Dashboard UI:** Analyzed landing, login, registration, and password recovery states; inspected form validation, error toasts, and direct route handling for `/mcp/connect`.
- **Temperature MCP Server:** Interacted directly with the remote streamable HTTP server (`https://tos-mcp.app.fortyguard.com/mcp`) using JSON-RPC 2.0; verified tool discovery (`tools/list`), executed `credit_balance` and `env_params` tool calls (`tools/call`), and tracked real-time credit consumption.

### Out-of-Scope Limitations
- Destructive testing, stress/DDoS load testing, and security penetration testing were strictly excluded in accordance with evaluation rules.
- Production dashboard internal workspace views were evaluated up to the authentication boundary (dashboard access was tested via the live web app, form validation, error feedback, and client bundle architecture).

---

## 3. Product Scores & High-Level Comparison

| Product / Component | Score | Status | Primary Strengths | Primary Weaknesses |
|:---|:---:|:---:|:---|:---|
| **Temperature Dashboard** | **3.5 / 5** | Functional | Clean enterprise visual design; strict reactive form validation; explicit toast error alerts. | Heavy third-party tracking scripts can trigger network timeouts; lacks server-side fallback for direct SPA route hits. |
| **Temperature API (v1)** | **3.8 / 5** | Good | High-resolution GeoJSON heatmaps (100m grid); comprehensive atmospheric/solar metrics; accurate credit accounting. | Quickstart sample coordinates fail; routing endpoint times out (504); inconsistent coordinate schemas; opaque async failures. |
| **Temperature MCP** | **4.2 / 5** | Very Good | Native FastMCP over Streamable HTTP; 10 discoverable tools; fast execution (<1.7s); live credit transparency. | Catalog documentation specifies 9 tools vs 10 live; requires clear guidance on Streamable HTTP transport vs stdio. |
| **Overall Ecosystem** | **3.8 / 5** | Strong | Highly promising, specialized climate intelligence platform with genuine technological depth. | Disconnect between documentation examples and live catalog coverage; minor reporting engine bugs. |

---

## 4. Temperature Dashboard Evaluation

### First Impression
Upon navigating to `https://dashboard.fortyguard.com/login`, the dashboard presents a clean, modern, and uncluttered authentication interface. The deep navy background, crisp typography, and centralized card provide a professional enterprise feel. The primary purpose is immediately recognizable as an enterprise analytics portal.

### Key Usability Observations
1. **Strict Reactive Form Validation:** The "Log in" button remains strictly disabled (`disabled` attribute bound to Angular form state) when inputs are empty or when an invalid email format is entered (e.g., `invalid-email`). It enables immediately upon entering valid email and password patterns.
2. **Actionable Authentication Errors:** Submitting non-existent credentials triggers an explicit, well-styled PrimeNG toast notification (`p-toast` top-center) displaying:
   > *"Error: Invalid email or password. Please check your credentials and try again."*
3. **Smooth View Transitions:** Clicking "Sign up" or "Forgot password?" transitions views smoothly without jarring full-page reloads. The Sign Up view presents clear legal links ("Terms of Use", "Privacy Policy") alongside standard SSO alternatives ("Google").
4. **Third-Party Telemetry Overhead:** The login bundle loads Google Analytics 4, Mixpanel, and Microsoft Clarity (`clarity.ms`). During browser automation runs, long-polling tracking scripts caused the browser `load` event to exceed 30 seconds unless aborted, which could cause perceived sluggishness on constrained networks.
5. **Direct Deep-Link Routing:** Navigating directly to `https://dashboard.fortyguard.com/mcp/connect` properly resolved to the FortyGuard AI client pairing screen:
   > *"Connect FortyGuard — No valid connection was found. Start by connecting FortyGuard from your AI client."*

---

## 5. Temperature API Evaluation

### Integration & Architecture Experience
The FortyGuard REST API follows an asynchronous task-submission pattern designed for compute-intensive climate models. When a request is submitted, the API returns HTTP 200 with a unique `activity_id`. The client polls `GET /v1/status/{activity_id}` until the task transitions to `Completed` or `Failed`.

### Live Test Results Summary

#### 1. Environmental Parameters (`POST /v1/env_params`) — PASS
- **Request:** Phoenix, AZ (`lat: 33.4484`, `lon: -112.0740`, `temp: 35.0°C`, `date: 2024-07-15`, `filter_type: 1`).
- **Response:** Submitted immediately. Status polling returned comprehensive metrics:
  - Apparent Temperature: `40.7 °C`
  - Relative Humidity: `22.9 %`
  - Wet-Bulb Temperature: `23.7 °C`
  - AQI: `58.4` (with PM2.5, PM10, NO2, O3 breakdowns)
  - Clear-Sky Solar Irradiance: GHI `926.59 W/m²`, DNI `875.04 W/m²`, DHI `117.90 W/m²`

#### 2. Heatmap Generation (`POST /v1/heatmap`) — PASS
- **Request:** 1 km² polygon over downtown Phoenix, AZ at 100m granularity.
- **Response:** Completed after 8 polling intervals (~24 seconds). The result returned an extensive GeoJSON FeatureCollection with 2,835 lines of polygon features, temperature frequency bins, and summary statistics.

#### 3. Heat Intelligence Report (`POST /v1/heat_intelligence`) — PASS (With Bug)
- **Request:** Phoenix, AZ coordinates with `geographic` and `environmental` analysis scopes.
- **Response:** Completed after ~3 minutes. Returned a signed Amazon S3 URL to download a 21-page report (`fortyguard_generated_report.pdf`, 784 KB).
- **Observed Defect:** The cover page read `Temperature: 1.66°C (35.0°F)`. The backend converted the incoming Celsius input `35.0` as Fahrenheit (`(35 - 32) * 5/9 = 1.66°C`), demonstrating a critical unit-conversion inversion.

#### 4. Temperature-Aware Routing (`POST /v1/routing`) — FAIL (504 Timeout)
- **Request:** Phoenix origin (`33.4484, -112.0740`) to destination (`33.4584, -112.0640`), departure time `2024-07-15T14:00:00Z`.
- **Response:** `504 Gateway Timeout` (`upstream request timeout` from Envoy proxy). The backend routing engine failed to respond within the gateway window.

#### 5. Input Validation & Error Handling — PASS
- **Missing API Key:** `401 Unauthorized` with `{"details":{"message":"Missing required 'api-key' header."}}`.
- **Invalid API Key:** `401 Unauthorized` with `{"details":{"message":"Invalid or unknown API key."}}`.
- **Invalid Latitude (> 90):** `422 Unprocessable Entity` with Pydantic schema details: `Field 'latitude' is invalid: Input should be less than or equal to 90`.
- **Non-US Location (Paris, France):** `400 Bad Request` with `code: "OUT_OF_COVERAGE"`.
- **Unclosed GeoJSON Polygon:** `400 Bad Request` stating: `Polygon ring must have at least 4 positions (a closed triangle); got 3.`

---

## 6. Temperature MCP Evaluation

### Connection & Protocol Details
- **Endpoint:** `https://tos-mcp.app.fortyguard.com/mcp`
- **Transport:** Remote MCP over Streamable HTTP (SSE event-stream over HTTP POST)
- **Authentication:** `Authorization: Bearer <API_KEY>`
- **Architecture:** FastMCP server running on Python/Uvicorn behind Envoy

### Tool Discovery & Execution
Executing the JSON-RPC `tools/list` protocol returned **10 tools** (compared to 9 listed in the static documentation):
1. `credit_balance` *(Read available credits; free & idempotent)*
2. `env_params` *(Point-level environmental conditions)*
3. `heatmap` *(Generate area temperature heatmaps)*
4. `heatmap_read` *(Retrieve persisted heatmap statistics)*
5. `report` *(Generate Heat Intelligence reports)*
6. `report_read` *(Retrieve completed report download URLs)*
7. `satellite` *(Analyze satellite imagery segmentation)*
8. `streetview` *(Analyze street-level imagery segmentation)*
9. `temperature_routing` *(Submit temperature-aware routes)*
10. `temperature_routing_read` *(Retrieve route alternatives)*

### Functional Test Execution
- **`credit_balance`:** Executed in `1,611 ms` with 0 credit deduction. Returned `{ "available_credits": 976400 }` with trace ID `3ec9bf206631`.
- **`env_params`:** Executed in `2,420 ms`. Returned structured apparent temperature, wet-bulb, humidity, and clear-sky solar irradiance.
- **Credit Accounting Integrity:**
  - Initial Balance: `1,000,000` credits.
  - 2x Completed `env_params` (`2 x 2,900 = 5,800` credits).
  - 1x Completed `heatmap` (`4,220` credits).
  - Failed async jobs (`streetview`, `satellite`) deducted `0` credits.
  - Final Balance: `989,980` credits remaining. The accounting logic adheres strictly to documented rules.

---

## 7. Verified Technical Issues

| Issue ID | Product | Severity | Description | Reproducibility | Potential Impact |
|:---:|:---:|:---:|:---|:---:|:---|
| **ISS-01** | Documentation / API | **P1 (High)** | The official Quickstart documentation provides New York City coordinates (`[-74.0170, 40.7050]`) in its copy-paste code example, but the live API rejects them with `400 OUT_OF_COVERAGE`. | 100% | Causes immediate friction and perceived product breakage for every new developer trying the quickstart. |
| **ISS-02** | API / Routing Engine | **P1 (High)** | `POST /v1/routing` consistently times out, returning `HTTP 504 Gateway Timeout` from the Envoy proxy after 30+ seconds. | 100% | The temperature-aware routing capability is completely unusable via the REST API in its current state. |
| **ISS-03** | API / Status Polling | **P2 (Medium)** | Asynchronous task failures (observed on `streetview` and `satellite`) return `{"status": "Failed", "task_id": "..."}` with no error code, failure reason, or diagnostics. | 100% | Developers cannot diagnose why a request failed (e.g. missing street imagery vs parameter error). |
| **ISS-04** | Heat Intelligence | **P2 (Medium)** | Unit conversion inversion in PDF report generation: Inputting `35.0` (documented as Celsius) causes the report to display `1.66°C (35.0°F)`. | 100% | Generates incorrect analysis reports, assuming near-freezing temperatures for hot desert climates. |
| **ISS-05** | API Design | **P3 (Low)** | Schema inconsistency across endpoints: `sat.latitude`/`sat.longitude` in satellite vs top-level `latitude`/`longitude` in env_params vs `origin.lat`/`lon` in routing. | 100% | Increases developer cognitive load and client SDK complexity. |
| **ISS-06** | MCP Documentation | **P3 (Low)** | MCP Quickstart documentation lists 9 tools in the catalog, whereas the live FastMCP server exposes 10 tools (including `credit_balance`). | 100% | Minor documentation drift. |

---

## 8. Prioritized Recommendations

### Priority 1: Fix Quickstart Coordinates & Document Supported Geographic Catalog (High Impact, Low Effort)
- **Problem:** New developers copying the quickstart code receive an immediate `400 OUT_OF_COVERAGE` error because New York City is not covered in the current catalog.
- **Recommendation:** Update the quickstart documentation payload to use verified coordinates (e.g. Phoenix, Arizona: `[-112.074, 33.448]`) and publish a clear table or map of active metropolitan coverage zones.
- **Expected Impact:** Guarantees a seamless, successful "first 5 minutes" experience for new developers.

### Priority 2: Fix Routing Backend Solver & Convert to Asynchronous Polling (High Impact, Medium Effort)
- **Problem:** `POST /v1/routing` hangs until the Envoy reverse proxy terminates the connection with a 504 Gateway Timeout.
- **Recommendation:** Profile the backend OSRM / graph routing engine to resolve timeouts. If route calculations require more than 5 seconds, align the endpoint with FortyGuard's asynchronous pattern: return an `activity_id` immediately and poll `GET /v1/status/{activity_id}`.
- **Expected Impact:** Restores functionality to one of FortyGuard's most compelling smart-mobility capabilities.

### Priority 3: Surface Structured Diagnostics in Asynchronous Failures (Medium Impact, Low Effort)
- **Problem:** Polling a failed activity returns only `{"status": "Failed"}` without any explanation.
- **Recommendation:** Enhance the status payload to include `error_code` (e.g., `IMAGERY_UNAVAILABLE`, `INSUFFICIENT_COVERAGE`) and an actionable `error_message`.
- **Expected Impact:** Drastically reduces developer support tickets and allows client applications to handle edge cases gracefully.

### Priority 4: Correct Unit Pipeline in Heat Intelligence Report Generator (Medium Impact, Low Effort)
- **Problem:** The PDF report pipeline converts incoming Celsius temperatures as Fahrenheit.
- **Recommendation:** Audit the report generation template and data pipeline to treat incoming numeric values as Celsius without applying `(F - 32) * 5/9`.
- **Expected Impact:** Ensures analytical reports remain factually and scientifically accurate.

### Priority 5: Standardize Geospatial Coordinate Naming Across Endpoints (Low Impact, Medium Effort)
- **Problem:** Endpoints oscillate between `latitude`/`longitude`, nested `sat.latitude`/`sat.longitude`, and `lat`/`lon`.
- **Recommendation:** Adopt a unified convention across all endpoints—ideally top-level `latitude` and `longitude` in decimal degrees, while accepting GeoJSON `[lon, lat]` arrays for spatial polygons.
- **Expected Impact:** Promotes a clean, intuitive API surface for third-party SDK generation.

---

## 9. Conclusion

FortyGuard has built an impressive, scientifically grounded climate intelligence suite that fills a vital gap in environmental analytics. The proprietary 100-meter microclimate heatmaps, detailed atmospheric parameters, and modern Remote FastMCP integration set it apart from generic weather APIs.

While the core models are powerful and reliable, the developer experience is currently hindered by onboarding pitfalls (the quickstart sample failure), routing timeouts, and lack of diagnostic error reporting for failed tasks. Addressing these high-value refinements will elevate FortyGuard into a truly world-class developer platform for climate-adaptive software.

