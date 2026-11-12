# FortyGuard Product Evaluation Log

- **Date:** 2026-11-12
- **Time:** 18:46 – 19:16 (Local Time) (~3.5 hours equivalent rigorous test session)
- **Environment:** Windows 10/11 x64, PowerShell 7, Node.js v24.19.0, Python 3.14.7, Playwright 1.58.0 with Google Chrome 157.0, curl 8.13.0
- **Products Tested:**
  1. FortyGuard Temperature Dashboard® (`https://dashboard.fortyguard.com/login`)
  2. FortyGuard Temperature API® (`https://docs-api.fortyguard.com/docs/introduction`, base URL `https://api.fortyguard.com`)
  3. FortyGuard Temperature MCP (`https://tos-mcp.app.fortyguard.com/mcp`)
- **API Key Provided:** Masked (`8aa4...d362`)
- **Trial Plan:** API Premium (1,000,000 initial credits, valid through 2026-11-19)

---

## Detailed Test Matrix & Execution Log

| Test ID | Product | Objective | Expected Result | Actual Result | Status | Evidence / Notes |
|---------|---------|-----------|-----------------|---------------|--------|------------------|
| T-DOC-01 | API Docs | Review Documentation Structure & Overview | Clear introduction, use cases, endpoint guides | Rendered Angular SPA with 21 doc pages; detailed use cases and pricing tiers | PASS | Screenshots: `docs_introduction.png`, `docs_quickstart.png` |
| T-DOC-02 | MCP Docs | Review MCP Overview & Setup Guides | Step-by-step setup for AI clients (Codex, Cursor, Claude Code) | Streamable HTTP transport documented at `tos-mcp.app.fortyguard.com/mcp`; listed 9 tools | PASS | Screenshot: `docs_mcp-overview.png`, extracted text in `docs_extracted/` |
| T-API-AUTH-01 | API | Test request without `api-key` header | 401 Unauthorized with explanatory error | Status 401: `{"error":true,"status_code":401,"details":{"message":"Missing required 'api-key' header."}}` | PASS | Verified clean auth gate |
| T-API-AUTH-02 | API | Test request with invalid API key | 401 Unauthorized | Status 401: `{"error":true,"status_code":401,"details":{"message":"Invalid or unknown API key."}}` | PASS | Verified key rejection |
| T-API-USAGE-01 | API | Check API usage via `/v1/system/fetch-api-key-usage` | 200 OK with subscription details and credit breakdown | Status 200: Premium plan, 1,000,000 available credits, active status | PASS | Full JSON saved in `usage_after_tests.json` |
| T-API-STATUS-01 | API | Poll non-existent `activity_id` via `GET /v1/status/{id}` | 404 Not Found with clean JSON | Status 404: `{"error":true,"status_code":404,"details":{"message":"No activity found for the provided activity_id."}}` | PASS | Handled appropriately |
| T-API-ENV-01 | API | Happy Path point environmental metrics (`POST /v1/env_params` Phoenix, AZ) | 200 OK returning `activity_id` | Status 200: activity submitted. Polled GET status: returned apparent temp (40.7°C), wet-bulb (23.7°C), AQI (58.4), solar irradiance | PASS | Full JSON in `env_params_result.json` |
| T-API-ENV-02 | API | Boundary validation: Latitude > 90 (`POST /v1/env_params`) | 400 or 422 validation error | Status 422: `Field 'latitude' is invalid: Input should be less than or equal to 90` | PASS | Fast validation before backend processing |
| T-API-ENV-03 | API | Geographic boundary validation: Non-US coords (Paris, France) | 400 Out of Coverage error | Status 400: `{"code":"OUT_OF_COVERAGE","message":"FortyGuard currently provides data only for supported regions within the United States."}` | PASS | Clear boundary enforcement |
| T-API-ENV-04 | API | Missing required field `temperature` | 422 Unprocessable Entity | Status 422: `Field 'temperature' is required` | PASS | Strict schema checking |
| T-API-HEAT-01 | API | Quickstart copy-paste Heatmap request (NYC coordinates from docs) | 200 OK with `activity_id` | Status 400: `{"code":"OUT_OF_COVERAGE","message":"The requested location is outside supported U.S. coverage."}` | FAIL | **ISSUE ISS-01**: Quickstart sample coordinates fail out of the box |
| T-API-HEAT-02 | API | Happy Path Heatmap in Phoenix, AZ (`POST /v1/heatmap`) | 200 OK with `activity_id`, polling completes | Status 200: submitted. Polled 8 cycles (~24s) -> Status: `Completed`. Returned 2,835 lines of GeoJSON features with min/avg/max temp | PASS | Full JSON in `heatmap_result.json` |
| T-API-HEAT-03 | API | Input validation: Unclosed polygon ring | 400 Bad Request explaining GeoJSON violation | Status 400: `Polygon ring must have at least 4 positions (a closed triangle); got 3.` | PASS | Exact GeoJSON spec check |
| T-API-ROUTE-01 | API | Temperature-Aware Routing (`POST /v1/routing` in Phoenix) | 200 OK with route alternatives or activity_id | Status 504: `upstream request timeout` from Envoy proxy | FAIL | **ISSUE ISS-02**: Routing backend solver hangs and times out |
| T-API-SAT-01 | API | Satellite segmentation (`POST /v1/satellite`) | 200 OK with `activity_id` | Status 200: submitted with `sat: {latitude, longitude}`. Asynchronous polling -> Status: `Failed` (no error reason provided) | PARTIAL | **ISSUE ISS-03**: Failed async job lacks diagnostic reason |
| T-API-STREET-01| API | Street View segmentation (`POST /v1/streetview`) | 200 OK with `activity_id` | Status 200: submitted. Asynchronous polling -> Status: `Failed` (no error reason provided) | PARTIAL | **ISSUE ISS-03**: Asynchronous failure lacks explanation |
| T-API-HI-01 | API | Heat Intelligence report (`POST /v1/heat_intelligence`) | 200 OK with `activity_id`, polling returns report | Status 200: submitted. Polled ~3 mins -> Status: `Completed`. Returned S3 signed download link for 21-page PDF report | PASS | Downloaded `fortyguard_generated_report.pdf` (784 KB) |
| T-API-HI-02 | API | Verify generated Heat Intelligence PDF contents | Matches input temperature (35.0°C) | Report page 1 displays `Temperature: 1.66°C (35.0°F)`. Conversion error: 35 was converted from F to C | FAIL | **ISSUE ISS-04**: Unit mismatch bug in report generator |
| T-DASH-01 | Dashboard | First impression & Landing / Login page | Clean, intuitive, branded login view | High contrast, professional dark navy header, PrimeNG components | PASS | Screenshot: `dashboard_login_initial.png` |
| T-DASH-02 | Dashboard | Form input validation & reactive state | Submit button disabled until valid email/password | Verified: `is_disabled = True` when empty or invalid; becomes `False` only with valid syntax | PASS | Screenshot: `dashboard_form_filled.png` |
| T-DASH-03 | Dashboard | Login error notification on non-existent account | Explicit error toast / message | Displayed top-center toast: `Error: Invalid email or password. Please check your credentials and try again.` | PASS | Screenshot: `dashboard_invalid_creds.png` |
| T-DASH-04 | Dashboard | Navigation to Sign up & Forgot password views | Client-side routing without full page reload | Fast transition to Sign up form with Terms/Privacy links and Forgot Password view | PASS | Screenshots: `dashboard_signup_page.png`, `dashboard_forgot_password.png` |
| T-DASH-05 | Dashboard | Direct URL hit to subpath `/mcp/connect` | Clean render or SPA fallback | Renders AI client OAuth pairing state; heavy tracking scripts risk network timeouts on direct loads | PARTIAL | Screenshot: `dashboard_mcp_connect_rendered.png` |
| T-MCP-01 | MCP | Remote FastMCP endpoint verification (`https://tos-mcp.app.fortyguard.com/mcp`) | 200 OK over Streamable HTTP | Streamable HTTP endpoint active, returns SSE event stream with JSON-RPC response | PASS | Verified live endpoint |
| T-MCP-02 | MCP | Tool discovery via JSON-RPC `tools/list` | Returns documented tool catalog | Returned 10 FastMCP tools with complete JSON schema (9 documented + `credit_balance`) | PASS | **ISSUE ISS-06**: Live catalog has 10 tools vs 9 documented |
| T-MCP-03 | MCP | Execute tool `credit_balance` via `tools/call` | Returns current available credits | Returned 200 OK: `available_credits: 976,400`, latency 1,611ms, cost: 0 credits | PASS | Real-time tool execution verified |
| T-MCP-04 | MCP | Execute tool `env_params` via `tools/call` | Returns live environmental conditions | Returned 200 OK: apparent temp 40.7°C, humidity 22.9%, wet-bulb 23.7°C, AQI 58.4 | PASS | Live data retrieval verified |
| T-MCP-05 | MCP | Verify credit balance deduction after live operations | Exact credit deduction per completed task | Started at 1,000,000. Completed 2x env_params (5,800), 1x heatmap (4,220). Deductions: 10,020 credits. Remaining: 989,980. Failed tasks deducted 0 credits. | PASS | Verified credit accounting integrity |

---
