# Technical Blueprint: Distilling REST API Fault-Detection Capability into a 4B SLM via Executable-Verified Synthetic Data

**Research title (working):** Cost-Effective REST API Test Generation via Teacher–Student Distillation with Execution-Verified Thinking Traces  
**Author:** JoyMaxxing Research Group  
**Target venue:** ResFes academic festival  
**Document purpose:** 1-on-1 assessment preparation — full pipeline specification with data shapes, algorithms, and design rationale

---

## 0. Gap T — Exact Research Boundary

### 0.1 What existing work actually does

| Paper | Model used | Fine-tuned? | Compared to SLM? | Cost measured? |
|---|---|---|---|---|
| LlamaRestTest [FSE 2025] | Llama3-8B | ✅ Yes (IPD + EX tasks) | ❌ No — compared only to ARAT-RL, base Llama | ❌ No |
| AutoRestTest [arXiv Nov 2024] | GPT-4o / 4o-mini | ❌ No | ❌ No | Partial — ~$0.10/API mentioned |
| RESTestBench | Llama 3.1 8B, Llama 3.3 70B + commercial LLMs | ❌ No (off-the-shelf) | Partial — compares LLMs but not fine-tuned SLMs, and task is requirements-based mutation score, not OAS-to-test generation |

**Exact gap:** No paper (a) fine-tunes an SLM specifically for REST API fault detection AND (b) benchmarks that SLM against a state-of-the-art commercial LLM on the same benchmark AND (c) measures cost-effectiveness (cost per fault found). LlamaRestTest and AutoRestTest were published two months apart, use the same 12-service benchmark, and the comparison experiment is trivially obvious — yet absent.

### 0.2 This proposal's additional contribution beyond filling that gap

LlamaRestTest fine-tunes on **human-curated dependency rules and example values** (IPD database, EX database — static, manually assembled). This proposal constructs the training corpus **synthetically and automatically** using a teacher LLM whose outputs are filtered by **code execution against live APIs**. Ground truth is not human annotation — it is whether `pytest` reports a real HTTP 500.

That is a second contribution: a reusable, scalable pipeline for building fault-detection corpora from any OAS spec + deployable service, without expert annotation.

---

## 1. Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────────┐
│  CORPUS CONSTRUCTION PHASE (offline, one-time)                                 │
│                                                                                │
│  ① APIsGuru OAS YAMLs ──parse──► endpoint_record.json                        │
│          │                                                                     │
│  ② EvoMaster EMB apps ──fuzz──► fault_record.json                            │
│          │                                                                     │
│  ③ endpoint_record + fault_record ──prompt──► Qwen3.5-27B (teacher)          │
│          │                                      └──► <think>…</think> + code  │
│          │                                                                     │
│  ④ generated code ──execute against live app──► verification_result.json      │
│          │                                                                     │
│  ⑤ filter (faults_found ≥ 1) ──► golden_record.jsonl  ◄── GOLDEN dataset     │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│  SFT PHASE (Unsloth + QLoRA, Colab A100/T4)                                   │
│                                                                                │
│  golden_record.jsonl ──reformat──► sft_dataset.jsonl (ChatML)                │
│          └──────────────────────────► Qwen3-4B student fine-tune              │
│                                        └──► qwen3-4b-api-tester (LoRA adapter)│
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│  EVALUATION PHASE (held-out services, never seen in training)                  │
│                                                                                │
│  Same prompt ──► Qwen3-4B-SFT (student)   ┐                                  │
│  Same prompt ──► Gemini 2.5 Flash (API)   ├──► execute all ──► metrics table │
│  Same prompt ──► GPT-4o-mini (API)        │                                   │
│  Same prompt ──► Qwen3-4B-base (ablation) ┘                                  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Scale targets:**
- ~12 EvoMaster EMB services → ~200 deployable endpoints → ~4,000 teacher inference calls
- Expected GOLDEN yield: ~30–40% → ~1,200–1,600 golden records
- Train/val/test split: 80/10/10, with test records drawn exclusively from 3 held-out services

---

## 2. Stage ①: Corpus Acquisition from APIsGuru

### 2.1 Source

Repository: `github.com/APIs-guru/openapi-directory` (~2,400 API specs under `APIs/` directory, CC0 license). Each provider has one or more YAML files at:

```
APIs/
  zalando.com/
    catwatch/
      1.0.0/
        openapi.yaml        ← OAS 3.x
  petstore.swagger.io/
    1.0.0/
      swagger.yaml          ← OAS 2.x (needs up-conversion)
```

**Filter criterion:** Only include specs that have a corresponding EvoMaster EMB driver (i.e., a deployable Docker service). This gives ~12 services × ~15–25 endpoints each = ~200 endpoint records.

### 2.2 Raw OAS3 YAML — annotated field-by-field

```yaml
openapi: "3.0.0"                      # version discriminator; OAS2 uses "swagger: '2.0'"

info:
  title: CatWatch API                 # → api_description field in endpoint_record
  version: "1.0.0"                    # → api_version
  x-logo:
    url: "https://..."                # ignored

servers:
  - url: "http://localhost:8080"      # overwritten with EMB deployment URL

paths:
  /github/statistics:                 # → endpoint.path
    get:                              # → endpoint.method (uppercased: "GET")
      operationId: getStatistics      # → endpoint.operation_id
      summary: "Returns stats for a GitHub org"   # → endpoint.summary (passed to teacher)
      description: "Fetches aggregated..."         # → endpoint.description (optional, also passed)
      tags: [statistics]              # → endpoint.tags (metadata only)

      parameters:
        - name: organization          # → parameter.name
          in: query                   # → parameter.in  (query|path|header|cookie)
          required: false             # → parameter.required
          schema:
            type: string              # → parameter.schema.type
            minLength: 1              # → parameter.schema.minLength  (fault hint: empty string edge)
            maxLength: 255            # → parameter.schema.maxLength  (fault hint: overflow edge)
            description: "GitHub org" # → parameter.description
          example: "zalando"          # → parameter.example (used by LlamaRestTest; we pass to teacher)

        - name: source-token
          in: query
          required: false
          schema:
            type: string
            format: password          # → parameter.schema.format  (semantic hint: auth token)
          description: "OAuth token"

      requestBody:                    # only for POST/PUT/PATCH; null for GET
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ProjectDTO'   # → request_body.schema_ref

      responses:
        '200':
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Statistics'  # → response_schemas["200"].items_ref
        '400':
          description: "Bad request"   # → response_schemas["400"] = "Bad request" (string)
        '500':
          description: "Server error"  # presence of explicit 500 is itself a fault hint

components:
  schemas:
    Statistics:
      type: object
      required: [name, projectCount, memberCount]  # → components.schemas.Statistics.required
      properties:
        name:         { type: string }
        projectCount: { type: integer, minimum: 0 }
        memberCount:  { type: integer, minimum: 0 }
        publicForkCount: { type: integer }         # optional — schema mismatch if absent when 200
```

### 2.3 OAS parser — pseudocode

```python
import yaml, json
from pathlib import Path

def parse_oas(yaml_path: str, base_url: str) -> list[dict]:
    """Returns one endpoint_record per path+method combination."""
    spec = yaml.safe_load(open(yaml_path))
    api_id = derive_api_id(yaml_path)          # e.g. "catwatch_v1"
    records = []

    # Resolve $ref inline before processing
    spec = resolve_refs(spec)

    for path, path_item in spec.get("paths", {}).items():
        for method in ["get","post","put","patch","delete","head","options"]:
            op = path_item.get(method)
            if op is None:
                continue

            record = {
                "record_id":    f"{api_id}_{method.upper()}_{path.replace('/','_').strip('_')}",
                "api_id":       api_id,
                "api_title":    spec["info"]["title"],
                "api_version":  spec["info"]["version"],
                "method":       method.upper(),
                "path":         path,
                "operation_id": op.get("operationId", f"{method}_{path}"),
                "summary":      op.get("summary", ""),
                "description":  op.get("description", ""),
                "tags":         op.get("tags", []),
                "base_url":     base_url,
                "parameters":   parse_parameters(op.get("parameters", []) +
                                                  path_item.get("parameters", [])),
                "request_body": parse_request_body(op.get("requestBody")),
                "response_schemas": parse_responses(op.get("responses", {})),
                "security":     op.get("security", []),          # auth requirement hint
                "raw_oas_snippet": yaml.dump({path: {method: op}}, default_flow_style=False),
                "complexity_score": compute_complexity(op),      # heuristic 0-10
            }
            records.append(record)
    return records


def parse_parameters(params: list) -> list[dict]:
    out = []
    for p in params:
        schema = p.get("schema", {})
        out.append({
            "name":        p["name"],
            "in":          p["in"],               # query | path | header | cookie
            "required":    p.get("required", p["in"] == "path"),  # path params always required
            "type":        schema.get("type", "string"),
            "format":      schema.get("format"),  # e.g. "password", "int64", "date-time"
            "enum":        schema.get("enum"),     # if set, constrained value space
            "minimum":     schema.get("minimum"),  # numeric lower bound
            "maximum":     schema.get("maximum"),
            "minLength":   schema.get("minLength"),
            "maxLength":   schema.get("maxLength"),
            "pattern":     schema.get("pattern"),  # regex constraint
            "example":     p.get("example") or schema.get("example"),
            "description": p.get("description", schema.get("description", "")),
        })
    return out


def compute_complexity(op: dict) -> int:
    """
    Heuristic 0–10 used for stratified sampling later.
    Higher = more interesting for fault generation.
    """
    score = 0
    score += len(op.get("parameters", []))         # each param adds 1
    score += 3 if op.get("requestBody") else 0     # body adds 3
    score += 2 if op.get("security") else 0        # auth-protected adds 2
    # Password-format params are high-value fault targets
    for p in op.get("parameters", []):
        if p.get("schema", {}).get("format") == "password":
            score += 2
    return min(score, 10)
```

### 2.4 Endpoint record — complete schema

```json
{
  "record_id":        "catwatch_v1_GET__github_statistics",
  "api_id":           "catwatch_v1",
  "api_title":        "CatWatch API",
  "api_version":      "1.0.0",
  "method":           "GET",
  "path":             "/github/statistics",
  "operation_id":     "getStatistics",
  "summary":          "Returns statistics for a GitHub organization",
  "description":      "Fetches aggregated project and member statistics...",
  "tags":             ["statistics"],
  "base_url":         "http://localhost:8080",
  "parameters": [
    {
      "name":        "organization",
      "in":          "query",
      "required":    false,
      "type":        "string",
      "format":      null,
      "enum":        null,
      "minimum":     null,
      "maximum":     null,
      "minLength":   null,
      "maxLength":   255,
      "pattern":     null,
      "example":     "zalando",
      "description": "GitHub organization name"
    },
    {
      "name":        "source-token",
      "in":          "query",
      "required":    false,
      "type":        "string",
      "format":      "password",
      "enum":        null,
      "minimum":     null,
      "maximum":     null,
      "minLength":   null,
      "maxLength":   null,
      "pattern":     null,
      "example":     null,
      "description": "OAuth token for GitHub API access"
    }
  ],
  "request_body": null,
  "response_schemas": {
    "200": {
      "type":           "array",
      "items_ref":      "Statistics",
      "items_required": ["name", "projectCount", "memberCount"],
      "items_properties": {
        "name":             {"type": "string"},
        "projectCount":     {"type": "integer", "minimum": 0},
        "memberCount":      {"type": "integer", "minimum": 0},
        "publicForkCount":  {"type": "integer"}
      }
    },
    "400": "Bad request",
    "500": "Internal server error"
  },
  "security":          [],
  "raw_oas_snippet":   "...trimmed YAML text of this endpoint...",
  "complexity_score":  4
}
```

---

## 3. Stage ②: EvoMaster EMB Fault Harvesting

### 3.1 EMB repository structure

```
github.com/aster-test-generation/EMB
├── jdk_8_maven/
│   ├── cs/                         # case studies (REST APIs used in this work)
│   │   ├── catwatch/               # Spring Boot REST app
│   │   │   ├── pom.xml
│   │   │   ├── src/main/...
│   │   │   └── em/                 # EvoMaster driver
│   │   │       └── EmbeddedEvoMasterController.java
│   ├── features-service/           # feature flag REST API
│   ├── scout-api/                  # GitHub org analytics
│   ├── proxyprint/                 # printing services API
│   ├── ocvn/                       # open contracting data
│   └── languagetool/               # grammar checking API
├── jdk_11_maven/
│   └── cs/
│       └── ncs/                    # numerical computation service
└── docker-compose.yml              # optional multi-service launcher
```

**Services selected for this study:**
`catwatch`, `features-service`, `scout-api`, `proxyprint`, `languagetool`, `ncs`, `scs` — all have OAS specs and Docker drivers.

### 3.2 Deployment and fuzzing commands

```bash
# Step 1: Build all selected EMB services
cd EMB/jdk_8_maven
mvn clean package -pl cs/catwatch,cs/features-service,cs/scout-api -am -DskipTests

# Step 2: Start one service via its Spring Boot driver
cd cs/catwatch
mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8080" &
sleep 10  # wait for startup

# Step 3: Download EvoMaster CLI
wget https://github.com/WebFuzzing/EvoMaster/releases/latest/download/evomaster.jar

# Step 4: Run EvoMaster in black-box mode (no source code needed)
java -jar evomaster.jar \
  --blackBox           true \
  --bbTargetUrl        "http://localhost:8080" \
  --bbSwaggerUrl       "http://localhost:8080/v3/api-docs" \
  --maxTime            "5m" \
  --outputFormat       PYTHON_UNITTEST \
  --outputFile         "./faults/catwatch/" \
  --seed               42 \
  --appendToStatFile   true \
  --statisticsFile     "./faults/catwatch/stats.csv"

# Repeat for each service with a different port (8081, 8082, …)
```

**EvoMaster key flags explained:**
- `--blackBox true` — operates from OAS spec only, no bytecode instrumentation required
- `--bbSwaggerUrl` — the endpoint serving `openapi.json` or `openapi.yaml`; EvoMaster parses it to enumerate operations
- `--maxTime 5m` — 5 minutes per service is sufficient for black-box mode to find most shallow faults
- `--seed 42` — makes runs fully reproducible
- `--outputFormat PYTHON_UNITTEST` — generates Python test files (easier to integrate into our pipeline than JUnit)

### 3.3 Raw EvoMaster output — Python test file

EvoMaster writes a Python file per service. Each test method corresponds to one discovered fault or coverage target:

```python
# Auto-generated by EvoMaster v3.x
# Service: catwatch, Endpoint: GET /github/statistics
import requests
import unittest

class Test_catwatch_EvoMaster(unittest.TestCase):

    BASE_URL = "http://localhost:8080"

    def test_case_0(self):
        """EvoMaster: HTTP 500 found with this request."""
        res = requests.get(
            f"{self.BASE_URL}/github/statistics",
            params={"organization": "", "source-token": "\x00"},
            headers={"Accept": "application/json"},
            timeout=10
        )
        # EvoMaster asserts the 500 WAS triggered (it's documenting the bug)
        self.assertEqual(res.status_code, 500)

    def test_case_1(self):
        """EvoMaster: schema mismatch — response missing required field."""
        res = requests.get(
            f"{self.BASE_URL}/github/statistics",
            params={"organization": "test_org_that_exists"},
            headers={"Accept": "application/json"},
            timeout=10
        )
        # 200 response but body violated schema (memberCount absent)
        self.assertEqual(res.status_code, 200)
        data = res.json()
        self.assertIsInstance(data, list)

    def test_case_2(self):
        """EvoMaster: coverage target — no params."""
        res = requests.get(
            f"{self.BASE_URL}/github/statistics",
            headers={"Accept": "application/json"},
            timeout=10
        )
        self.assertIn(res.status_code, [200, 400])

if __name__ == "__main__":
    unittest.main()
```

**Important:** EvoMaster's assertion `assertEqual(status, 500)` is documenting a known bug. Our system inverts this — we generate tests asserting `!= 500` and treat assertion failure (real 500 observed) as catching a fault.

### 3.4 Fault record — parsed output schema

```json
{
  "fault_id":                "catwatch_f001",
  "source":                  "evomaster_v3",
  "app":                     "catwatch",
  "app_version":             "emb_v2.0",
  "endpoint_record_id":      "catwatch_v1_GET__github_statistics",
  "method":                  "GET",
  "path":                    "/github/statistics",

  "fault_type":              "POTENTIAL_FAULT",
  // Possible values:
  //   POTENTIAL_FAULT    → server returned HTTP 500
  //   SCHEMA_MISMATCH    → 200 response body violated OAS-declared schema
  //   SECURITY_FAULT     → 401/403 not returned when auth required (EvoMaster v3.5+)

  "trigger_request": {
    "method":       "GET",
    "url":          "http://localhost:8080/github/statistics",
    "query_params": {"organization": "", "source-token": "\u0000"},
    "path_params":  {},
    "headers":      {"Accept": "application/json"},
    "body":         null,
    "body_content_type": null
  },

  "observed_response": {
    "status_code":   500,
    "body_snippet":  "Internal Server Error",
    "content_type":  "application/json",
    "response_time_ms": 84
  },

  "fault_pattern":   "null_byte_in_string_param",
  // Heuristic taxonomy (assigned by post-processing script):
  //   null_byte_in_string_param  → \x00 in any string param
  //   empty_required_string      → "" for a non-nullable string
  //   negative_integer           → negative value for unsigned integer param
  //   integer_overflow           → Integer.MAX_VALUE or Long.MAX_VALUE
  //   sql_injection_pattern      → '; DROP TABLE, " OR 1=1, etc.
  //   missing_auth_token         → token field present but empty/null
  //   oversized_string           → len > declared maxLength or > 10,000

  "evomaster_seed":  42,
  "reproducible":    true   // re-run with same seed confirms same 500
}
```

### 3.5 Fault pattern injection into teacher context

For each teacher call, we inject **up to 2 fault records** from the same service (not necessarily same endpoint) as few-shot examples. This seeds the teacher's reasoning with service-specific vulnerability patterns.

---

## 4. Stage ③: Teacher Inference (Qwen3.5-27B)

### 4.1 Model selection rationale

| Option | Parameters | VRAM (4-bit) | Thinking mode | Cost | Rationale |
|---|---|---|---|---|---|
| Qwen3.5-27B | 27B dense | ~14GB | Yes (`<think>`) | Free local | Best free thinking model; fits 2×RTX 3090 or Colab A100 |
| Qwen3-32B | 32B dense | ~17GB | Yes | Free local | Alternative if more VRAM available |
| Qwen3-30B-A3B | 30B MoE | ~6GB active | Yes | Free local | Alternative for single-GPU setups |
| Gemini 2.5 Flash | N/A | API | Yes | ~$0.002/1K | Would eliminate local inference cost but add bias |

**Why not use Gemini as teacher:** The benchmark compares our student against Gemini. Using Gemini as teacher would create data leakage — the student would be distilling Gemini's own reasoning patterns, making the benchmark unfair.

### 4.2 Inference setup

```python
# Option A: Ollama (easiest)
# ollama pull qwen3.5:27b
# ollama serve

import ollama

response = ollama.chat(
    model="qwen3.5:27b",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user",   "content": build_user_message(endpoint_record, fault_hints)}
    ],
    options={
        "temperature":    0.7,    # some creativity for diverse fault hypotheses
        "top_p":          0.9,
        "num_predict":    2048,   # max output tokens (trace + code)
        "num_ctx":        4096,   # context window
        "think":          True,   # enables <think>…</think> mode in Qwen3.5
    }
)
```

### 4.3 System prompt — full text

```
You are an expert REST API test engineer specializing in fault detection through black-box testing.
You will be given an OpenAPI endpoint specification and must generate pytest test cases that
maximize the probability of triggering HTTP 500 errors or schema violations.

FAULT VECTOR CHECKLIST — reason through each before writing tests:
1. Null/zero-byte injection: \x00 in any string parameter (common NullPointerException cause in Java)
2. Empty string edge cases: "" for non-nullable string params (unguarded isEmpty() checks)
3. Integer boundary attacks: -1, 0, Integer.MAX_VALUE (2147483647), Long.MAX_VALUE for integer params
4. String length extremes: 1 char, maxLength+1 chars, 10000 chars (column truncation, OOM)
5. Missing optional params: omit each individually, then all at once
6. Auth token attacks: empty token "", malformed "abc.def", token with \x00
7. SQL/injection patterns: "'; DROP TABLE x; --", "\" OR 1=1--", "%27"
8. Unicode edge cases: emoji (4-byte UTF-8: \U0001F4A5), RTL characters, null-equivalent (\u2400)
9. Type confusion: send string where integer expected ("abc" for integer params via query string)
10. Inter-parameter conflicts: provide mutually exclusive parameters simultaneously
11. Schema validation: on 200 responses, verify all required fields are present with correct types

CRITICAL OUTPUT FORMAT:
- Start with <think>...</think> containing your reasoning through the checklist
- Follow with a single Python code block containing the test class
- The code block must be fenced with ```python and ```
- Use BASE_URL = "http://localhost:8080" exactly (will be substituted at runtime)
- All test method names must start with test_
- Each test method must have a comment explaining the fault hypothesis
- Assertions must be: assert resp.status_code != 500, f"<diagnostic message>"
  OR for schema: assert <condition>, f"Schema violation: <what was wrong>"
- Do NOT use pytest fixtures, conftest.py, or any external test data files
- Import only: requests, pytest (both guaranteed available)
```

### 4.4 User message template — full text with variable injection

```python
def build_user_message(ep: dict, fault_hints: list[dict]) -> str:
    # Build parameter table
    param_lines = []
    for p in ep["parameters"]:
        constraints = []
        if p["required"]:       constraints.append("REQUIRED")
        if p["format"]:         constraints.append(f"format:{p['format']}")
        if p["enum"]:           constraints.append(f"enum:{p['enum']}")
        if p["minimum"] is not None: constraints.append(f"min:{p['minimum']}")
        if p["maximum"] is not None: constraints.append(f"max:{p['maximum']}")
        if p["minLength"] is not None: constraints.append(f"minLen:{p['minLength']}")
        if p["maxLength"] is not None: constraints.append(f"maxLen:{p['maxLength']}")
        constraint_str = f" [{', '.join(constraints)}]" if constraints else " [optional]"
        param_lines.append(
            f"  - {p['name']} ({p['in']}, {p['type']}{constraint_str}): {p['description']}"
        )

    # Build response schema section
    resp_lines = []
    for code, schema in ep["response_schemas"].items():
        if isinstance(schema, dict):
            req = schema.get("items_required") or schema.get("required") or []
            resp_lines.append(f"  {code}: {schema.get('type','object')} — required fields: {req}")
        else:
            resp_lines.append(f"  {code}: {schema}")

    # Build fault hints section
    hint_lines = []
    for i, fault in enumerate(fault_hints[:2], 1):
        tr = fault["trigger_request"]
        params_str = json.dumps(tr.get("query_params") or tr.get("path_params") or {})
        hint_lines.append(
            f"  Example {i}: {fault['fault_type']} on {fault['method']} {fault['path']}\n"
            f"    Trigger: params={params_str}\n"
            f"    Observed: HTTP {fault['observed_response']['status_code']}\n"
            f"    Pattern: {fault['fault_pattern']}"
        )

    return f"""Generate fault-detecting pytest test cases for this endpoint.

=== API CONTEXT ===
API name: {ep['api_title']}
API description: {ep.get('description', ep['summary'])}
Base URL: {ep['base_url']}

=== ENDPOINT SPECIFICATION ===
Method: {ep['method']}
Path: {ep['path']}
Summary: {ep['summary']}

Parameters:
{chr(10).join(param_lines) if param_lines else '  (none)'}

Request body: {'none' if not ep['request_body'] else json.dumps(ep['request_body'], indent=4)}

Expected responses:
{chr(10).join(resp_lines)}

=== KNOWN FAULT PATTERNS (from EvoMaster, same service) ===
{'(none available for this service yet)' if not hint_lines else chr(10).join(hint_lines)}

=== TASK ===
Generate 5–8 test cases. Think carefully through each fault vector from the checklist.
Prioritize vectors that interact with the parameter types and constraints above.
"""
```

### 4.5 Teacher output — parsing logic

```python
import re

def parse_teacher_output(raw_output: str) -> tuple[str, str]:
    """
    Returns (thinking_trace, test_code).
    thinking_trace: full <think>...</think> block (including tags)
    test_code: Python code extracted from ```python ... ``` fence
    Raises ValueError if either component is absent.
    """
    # Extract thinking trace
    think_match = re.search(r'(<think>.*?</think>)', raw_output, re.DOTALL)
    if not think_match:
        raise ValueError("No <think> block found in teacher output")
    thinking_trace = think_match.group(1)

    # Extract code block
    code_match = re.search(r'```python\s*\n(.*?)\n```', raw_output, re.DOTALL)
    if not code_match:
        raise ValueError("No ```python code block found in teacher output")
    test_code = code_match.group(1).strip()

    # Basic sanity checks
    if "import requests" not in test_code:
        raise ValueError("Code block missing 'import requests'")
    if "def test_" not in test_code:
        raise ValueError("Code block has no test_ methods")

    return thinking_trace, test_code
```

---

## 5. Stage ④: Execution Verification

### 5.1 Architecture

The verifier orchestrates three steps: syntax check → live execution → result parsing. It runs against the **same EvoMaster service that is already deployed** from Stage ②.

```python
import subprocess, tempfile, os, json, ast
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class TestResult:
    name:             str        # e.g. "TestCatwatch::test_null_byte_in_organization"
    outcome:          str        # "FAULT_CAUGHT" | "PASSED" | "ERROR" | "SCHEMA_VIOLATION"
    status_code_seen: Optional[int]  # actual HTTP status returned
    assertion_msg:    str        # full assertion error message if FAULT_CAUGHT
    duration_sec:     float

@dataclass
class VerificationResult:
    # Execution metadata
    executed:         bool
    syntax_valid:     bool
    timeout:          bool
    error_msg:        Optional[str]

    # Test outcomes
    tests_run:        int
    faults_found:     int          # tests that caught HTTP 500
    schema_found:     int          # tests that caught schema violations
    errored:          int          # tests that crashed (e.g. connection refused)
    test_results:     list[TestResult] = field(default_factory=list)

    # Final label
    label:            str          # "GOLDEN" | "PARTIAL" | "INVALID" | "TIMEOUT" | "ALL_ERROR"
```

### 5.2 Verifier — complete implementation

```python
import subprocess, tempfile, os, json, sys

PYTEST_JSON_REPORT_PLUGIN = "pytest-json-report"  # pip install pytest-json-report

def verify(test_code: str,
           base_url: str,
           timeout_sec: int = 45) -> VerificationResult:

    # ── 1. Syntax check ───────────────────────────────────────────────────────
    try:
        ast.parse(test_code)
    except SyntaxError as e:
        return VerificationResult(
            executed=False, syntax_valid=False, timeout=False,
            error_msg=str(e), tests_run=0, faults_found=0,
            schema_found=0, errored=0, label="INVALID"
        )

    # Substitute placeholder URL with actual deployment URL
    code_with_url = test_code.replace("http://localhost:8080", base_url)

    # ── 2. Write to temp file ─────────────────────────────────────────────────
    with tempfile.NamedTemporaryFile(
        suffix=".py", mode="w", delete=False, dir="/tmp"
    ) as f:
        f.write(code_with_url)
        tmp_path = f.name
    report_path = tmp_path.replace(".py", "_report.json")

    # ── 3. Execute with pytest ────────────────────────────────────────────────
    try:
        result = subprocess.run(
            [
                sys.executable, "-m", "pytest",
                tmp_path,
                "--json-report",
                f"--json-report-file={report_path}",
                "-v",
                "--tb=short",
                f"--timeout={timeout_sec}",   # requires pytest-timeout
                "-x",                         # stop on first ERROR (not FAIL)
            ],
            capture_output=True,
            text=True,
            timeout=timeout_sec + 10
        )
    except subprocess.TimeoutExpired:
        return VerificationResult(
            executed=True, syntax_valid=True, timeout=True,
            error_msg="subprocess timeout", tests_run=0,
            faults_found=0, schema_found=0, errored=0, label="TIMEOUT"
        )
    finally:
        os.unlink(tmp_path)

    # ── 4. Parse JSON report ──────────────────────────────────────────────────
    if not os.path.exists(report_path):
        return VerificationResult(
            executed=True, syntax_valid=True, timeout=False,
            error_msg="pytest produced no JSON report",
            tests_run=0, faults_found=0, schema_found=0, errored=0,
            label="INVALID"
        )

    report = json.load(open(report_path))
    os.unlink(report_path)

    test_results = []
    faults_found = 0
    schema_found = 0
    errored = 0

    for t in report.get("tests", []):
        outcome_raw = t["outcome"]   # "passed" | "failed" | "error"
        longrepr    = t.get("call", {}).get("longrepr", "")
        # Determine semantic outcome
        # A FAILED assertion means our negative assertion caught a real bug:
        #   assert resp.status_code != 500  →  FAILED means 500 was actually returned
        if outcome_raw == "failed":
            if "Schema violation" in longrepr or "schema" in longrepr.lower():
                outcome = "SCHEMA_VIOLATION"
                schema_found += 1
            else:
                outcome = "FAULT_CAUGHT"
                faults_found += 1
        elif outcome_raw == "error":
            outcome = "ERROR"
            errored += 1
        else:
            outcome = "PASSED"   # test ran, server did NOT return 500

        # Extract status code from assertion message if present
        status_code_seen = None
        sc_match = re.search(r'HTTP (\d{3})', longrepr)
        if sc_match:
            status_code_seen = int(sc_match.group(1))

        test_results.append(TestResult(
            name=t["nodeid"],
            outcome=outcome,
            status_code_seen=status_code_seen,
            assertion_msg=longrepr[:500],   # truncate
            duration_sec=t.get("call", {}).get("duration", 0.0)
        ))

    tests_run = len(test_results)

    # ── 5. Assign label ───────────────────────────────────────────────────────
    if faults_found >= 1 or schema_found >= 1:
        label = "GOLDEN"
    elif tests_run > 0 and errored < tests_run:
        label = "PARTIAL"    # ran but found no faults — still useful as negative example
    elif errored == tests_run and tests_run > 0:
        label = "ALL_ERROR"  # all tests crashed (connection refused, import error)
    else:
        label = "INVALID"

    return VerificationResult(
        executed=True, syntax_valid=True, timeout=False, error_msg=None,
        tests_run=tests_run, faults_found=faults_found,
        schema_found=schema_found, errored=errored,
        test_results=test_results, label=label
    )
```

### 5.3 Verification result — complete schema

```json
{
  "executed":       true,
  "syntax_valid":   true,
  "timeout":        false,
  "error_msg":      null,
  "tests_run":      6,
  "faults_found":   2,
  "schema_found":   0,
  "errored":        0,
  "label":          "GOLDEN",
  "test_results": [
    {
      "name":             "TestGetGithubStatistics::test_null_byte_in_organization",
      "outcome":          "FAULT_CAUGHT",
      "status_code_seen": 500,
      "assertion_msg":    "AssertionError: Null byte in org caused crash: 500 Internal Server Error\nBody: {\"timestamp\":\"2025-06-01\",\"status\":500,...}",
      "duration_sec":     0.127
    },
    {
      "name":             "TestGetGithubStatistics::test_empty_org_invalid_token_cascade",
      "outcome":          "FAULT_CAUGHT",
      "status_code_seen": 500,
      "assertion_msg":    "AssertionError: Bad token cascade: 500",
      "duration_sec":     0.215
    },
    {
      "name":             "TestGetGithubStatistics::test_oversized_organization_name",
      "outcome":          "PASSED",
      "status_code_seen": 400,
      "assertion_msg":    "",
      "duration_sec":     0.089
    },
    {
      "name":             "TestGetGithubStatistics::test_sql_injection_in_organization",
      "outcome":          "PASSED",
      "status_code_seen": 400,
      "assertion_msg":    "",
      "duration_sec":     0.094
    },
    {
      "name":             "TestGetGithubStatistics::test_unicode_emoji_in_organization",
      "outcome":          "PASSED",
      "status_code_seen": 200,
      "assertion_msg":    "",
      "duration_sec":     0.311
    },
    {
      "name":             "TestGetGithubStatistics::test_schema_mismatch_on_200",
      "outcome":          "PASSED",
      "status_code_seen": 200,
      "assertion_msg":    "",
      "duration_sec":     0.298
    }
  ]
}
```

---

## 6. Stage ⑤: Golden Dataset Construction

### 6.1 Golden record — complete schema (every field annotated)

```json
{
  "record_id":      "catwatch_v1_GET__github_statistics__null_byte__v1",
  // Unique ID: api + endpoint + fault_pattern + iteration

  "split":          "train",
  // "train" (80%) | "val" (10%) | "test" (10%)
  // test records come exclusively from 3 held-out services

  "label":          "GOLDEN",
  // "GOLDEN"  → faults_found >= 1 (enters SFT training as positive example)
  // "PARTIAL" → faults_found == 0, tests ran cleanly (optional negative example)
  // "INVALID" → syntax errors or all crashed (discarded)

  "quality_tier":   "MULTI_FAULT",
  // "MULTI_FAULT"  → faults_found >= 2 (highest value, oversample in training)
  // "SINGLE_FAULT" → faults_found == 1
  // "SCHEMA_ONLY"  → schema_found >= 1 but faults_found == 0

  // ── Input context (what the student sees at inference time) ─────────────────
  "api_id":           "catwatch_v1",
  "api_title":        "CatWatch API",
  "api_description":  "CatWatch tracks GitHub organization project and member statistics",
  "endpoint":         "GET /github/statistics",
  "oas_snippet":      "...full YAML text of this endpoint's OAS definition...",
  "base_url":         "http://localhost:8080",
  // NOTE: base_url is replaced with placeholder "http://localhost:8080" in all records
  // to ensure the student learns to use the substitutable URL, not a hardcoded address

  // ── Teacher metadata ─────────────────────────────────────────────────────────
  "teacher_model":    "Qwen3.5-27B",
  "teacher_temp":     0.7,
  "teacher_seed":     null,           // not set; multiple runs possible

  // ── Teacher output ────────────────────────────────────────────────────────────
  "thinking_trace":   "<think>\nLet me analyze this endpoint...\n\n[full reasoning text]\n\n</think>",
  // Preserved exactly as output by teacher, including all internal reasoning

  "generated_tests":  "import requests\nimport pytest\n\nBASE_URL = \"http://localhost:8080\"\n\nclass TestGetGithubStatistics:\n    def test_null_byte_in_organization(self):\n        ...",
  // Exact Python code that was executed by the verifier

  // ── Verification results ──────────────────────────────────────────────────────
  "verification": {
    "executed":       true,
    "tests_run":      6,
    "faults_found":   2,
    "schema_found":   0,
    "fault_test_names": [
      "TestGetGithubStatistics::test_null_byte_in_organization",
      "TestGetGithubStatistics::test_empty_org_invalid_token_cascade"
    ],
    "passing_test_names": [
      "TestGetGithubStatistics::test_oversized_organization_name",
      "TestGetGithubStatistics::test_sql_injection_in_organization",
      "TestGetGithubStatistics::test_unicode_emoji_in_organization",
      "TestGetGithubStatistics::test_schema_mismatch_on_200"
    ]
  },

  // ── Token statistics (for training budget planning) ───────────────────────────
  "token_counts": {
    "system_prompt_tokens":  287,
    "user_message_tokens":   412,
    "thinking_trace_tokens": 847,
    "answer_code_tokens":    563,
    "total_tokens":          2109
    // max_seq_length = 2048 covers ~90% of records at these sizes
    // 10% of records (complex endpoints) need max_seq_length = 3072
  }
}
```

### 6.2 Dataset split strategy

**Train (80%):** All records from 9 EvoMaster services (`catwatch`, `features-service`, `proxyprint`, `ocvn`, `ncs`, `scs`, and 3 others).

**Val (10%):** Stratified sample from same 9 services — ensures the val distribution matches train.

**Test (10%):** Exclusively from 3 held-out services: `languagetool`, `scout-api`, and one additional. These services are never touched during teacher inference or training.

**Stratification variable:** `quality_tier`. MULTI_FAULT records are valuable and should appear proportionally in all splits.

### 6.3 Deduplication

```python
import hashlib

def record_fingerprint(record: dict) -> str:
    """
    Two records are near-duplicates if they test the same endpoint
    with the same set of fault-catching test names.
    Keep the one with higher faults_found.
    """
    fault_names_sorted = sorted(record["verification"]["fault_test_names"])
    key = f"{record['api_id']}::{record['endpoint']}::{':'.join(fault_names_sorted)}"
    return hashlib.md5(key.encode()).hexdigest()
```

### 6.4 Expected dataset statistics

| Metric | Expected value |
|---|---|
| Total teacher calls | ~4,000 |
| GOLDEN records | ~1,400 (35%) |
| PARTIAL records | ~1,600 (40%) |
| INVALID/TIMEOUT | ~1,000 (25%) |
| MULTI_FAULT | ~420 (30% of GOLDEN) |
| SINGLE_FAULT | ~840 (60% of GOLDEN) |
| SCHEMA_ONLY | ~140 (10% of GOLDEN) |
| Avg tokens/record (total) | ~2,100 |
| Train records | ~1,120 |
| Val records | ~140 |
| Test records (held-out) | ~140 |

---

## 7. Stage ⑥: Student SFT via Unsloth

### 7.1 SFT dataset record — ChatML format (exact field layout)

```json
{
  "messages": [
    {
      "role":    "system",
      "content": "You are an expert REST API test engineer specializing in fault detection. Given an OpenAPI endpoint specification, generate pytest test cases targeting HTTP 500 errors and schema violations. Think step-by-step inside <think>...</think> tags about: null bytes, empty strings, integer boundaries, string length extremes, missing parameters, auth token attacks, SQL injection, unicode edge cases, and inter-parameter conflicts. Then output a single ```python code block with test_ methods. Each assertion must use: assert resp.status_code != 500, f\"<diagnostic>\"."
    },
    {
      "role":    "user",
      "content": "Generate fault-detecting test cases for:\n\nAPI: CatWatch API\nBase URL: http://localhost:8080\n\n=== ENDPOINT ===\nMethod: GET\nPath: /github/statistics\nSummary: Returns statistics for a GitHub organization\n\nParameters:\n  - organization (query, string, optional, maxLen:255): GitHub organization name\n  - source-token (query, string, optional, format:password): OAuth token\n\nResponse 200: array\n  Required fields: [name, projectCount, memberCount]\nResponse 400: Bad request\n\nTarget: find HTTP 500 errors and schema violations."
    },
    {
      "role":    "assistant",
      "content": "<think>\n[full thinking trace from golden_record.thinking_trace]\n</think>\n\n```python\n[full test code from golden_record.generated_tests]\n```"
    }
  ]
}
```

**Critical design note:** The `<think>` block is part of the `assistant` message content. The student is trained to produce the thinking block as part of its output. This is the mechanism for reasoning transfer — the student learns to reason before coding, not just output code.

### 7.2 sft_converter.py — full conversion logic

```python
import json

SYSTEM_PROMPT = """You are an expert REST API test engineer..."""  # full text as above

def record_to_sft(record: dict) -> dict:
    """Convert a golden_record to a ChatML SFT example."""

    # Build the user message from endpoint context
    ep = {
        "api_title":   record["api_title"],
        "base_url":    record["base_url"],
        "endpoint":    record["endpoint"],
        "oas_snippet": record["oas_snippet"],
    }
    user_msg = build_user_message_from_record(ep)

    # Build assistant message: thinking trace + code block
    assistant_msg = f"{record['thinking_trace']}\n\n```python\n{record['generated_tests']}\n```"

    return {
        "messages": [
            {"role": "system",    "content": SYSTEM_PROMPT},
            {"role": "user",      "content": user_msg},
            {"role": "assistant", "content": assistant_msg},
        ]
    }

def convert_dataset(golden_jsonl_path: str, output_path: str, split: str):
    with open(golden_jsonl_path) as fin, open(output_path, "w") as fout:
        for line in fin:
            record = json.loads(line)
            if record["split"] != split:
                continue
            if record["label"] != "GOLDEN":
                continue
            sft_record = record_to_sft(record)
            fout.write(json.dumps(sft_record, ensure_ascii=False) + "\n")
```

### 7.3 Unsloth training script — every parameter explained

```python
from unsloth import FastLanguageModel
from datasets import load_dataset
from trl import SFTTrainer, SFTConfig
import torch

# ── 1. Model loading ──────────────────────────────────────────────────────────
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name     = "Qwen/Qwen3-4B",
    # Why Qwen3-4B:
    #   - Natively supports <think> tokens (same tokenizer family as teacher)
    #   - 4B fits on T4 (15GB) in 4-bit: ~2.5GB weights + ~4GB activations
    #   - Qwen3 tech report shows 4B benefits strongly from reasoning distillation

    max_seq_length = 2048,
    # 2048 covers ~90% of golden records by total token count
    # Increase to 3072 on A100 if you want 100% coverage

    load_in_4bit   = True,
    # BitsandBytes NF4 quantization: reduces VRAM 4x with <1% quality loss
    # Required for T4; optional on A100 (can use 16-bit)

    dtype          = None,   # auto-detect (bfloat16 on Ampere+, float16 on T4)
)

# ── 2. LoRA configuration ─────────────────────────────────────────────────────
model = FastLanguageModel.get_peft_model(
    model,

    r            = 32,
    # LoRA rank: controls number of trainable parameters
    # r=32 → ~26M trainable params (vs 4B total = 0.65% of model)
    # Higher r = more capacity but more VRAM. r=16 minimum; r=64 if A100

    lora_alpha   = 32,
    # Scaling factor = lora_alpha / r = 1.0
    # Convention: set equal to r for scale=1 (effective learning rate unchanged)

    target_modules = [
        "q_proj", "k_proj", "v_proj", "o_proj",   # attention projections
        "gate_proj", "up_proj", "down_proj"         # MLP (FFN) projections
    ],
    # All 7 linear layers: standard for QLoRA on LLMs
    # Do NOT include "embed_tokens" or "lm_head" — they are excluded by Unsloth

    lora_dropout             = 0,
    # 0 dropout is standard for QLoRA (Dettmers et al. 2024)
    # Dropout reduces overfitting but impairs gradient flow in low-rank setting

    bias                     = "none",
    # Do not train bias terms — saves VRAM, no accuracy cost at this scale

    use_gradient_checkpointing = "unsloth",
    # Unsloth's custom gradient checkpointing: reduces VRAM by ~30%
    # Trades off ~20% more compute for 30% less VRAM — good tradeoff on T4

    random_state             = 42,
    use_rslora               = False,   # RSLoRA (Kalajdzievski 2023): set True if r > 32
)

# ── 3. Dataset loading ────────────────────────────────────────────────────────
train_dataset = load_dataset(
    "json",
    data_files={"train": "sft_train.jsonl"},
    split="train"
)
val_dataset = load_dataset(
    "json",
    data_files={"validation": "sft_val.jsonl"},
    split="validation"
)

# Apply chat template: converts messages list → single string with special tokens
# Qwen3 chat template wraps assistant with <|im_start|>assistant<|im_end|>
# and the <think>...</think> block is inside the assistant turn
def apply_chat_template(example):
    example["text"] = tokenizer.apply_chat_template(
        example["messages"],
        tokenize=False,
        add_generation_prompt=False
    )
    return example

train_dataset = train_dataset.map(apply_chat_template)
val_dataset   = val_dataset.map(apply_chat_template)

# ── 4. SFT training ───────────────────────────────────────────────────────────
trainer = SFTTrainer(
    model         = model,
    tokenizer     = tokenizer,
    train_dataset = train_dataset,
    eval_dataset  = val_dataset,
    args = SFTConfig(
        dataset_text_field            = "text",
        max_seq_length                = 2048,

        per_device_train_batch_size   = 2,
        gradient_accumulation_steps   = 4,
        # Effective batch size = 2 × 4 = 8 samples per update step
        # On T4: batch=2 is max before OOM. On A100: batch=4 possible

        num_train_epochs              = 3,
        # 3 epochs for ~1120 records: 3 × 1120/8 = 420 gradient steps
        # Monitor val loss: stop early if val loss increases after epoch 2

        learning_rate                 = 2e-4,
        # Standard for QLoRA SFT on small models. If loss is unstable → 1e-4

        lr_scheduler_type             = "cosine",
        # Cosine annealing: warms up then decays smoothly
        # Prevents abrupt learning rate drops that harm thinking trace quality

        warmup_ratio                  = 0.1,
        # 10% of steps = 42 steps warmup.
        # Prevents large gradient updates in early steps when LoRA weights are random

        bf16                          = True,
        # bfloat16 training: stable on Ampere+ GPUs. Use fp16=True on T4.

        optim                         = "adamw_8bit",
        # 8-bit AdamW: reduces optimizer state VRAM from 8GB → 1GB on 4B model

        logging_steps                 = 10,
        eval_strategy                 = "steps",
        eval_steps                    = 50,
        save_steps                    = 100,
        save_total_limit              = 3,    # keep only 3 checkpoints

        output_dir                    = "./qwen3-4b-api-tester",

        packing                       = True,
        # Sequence packing: concatenate short sequences into 2048-token chunks
        # Increases GPU utilization from ~50% to ~90% on variable-length data
    ),
)

trainer.train()

# ── 5. Save ───────────────────────────────────────────────────────────────────
# Option A: Save LoRA adapter only (for inference with base model + adapter)
model.save_pretrained("./qwen3-4b-api-tester-adapter")
tokenizer.save_pretrained("./qwen3-4b-api-tester-adapter")

# Option B: Merge + save as GGUF (for Ollama local serving)
model.save_pretrained_gguf(
    "./qwen3-4b-api-tester-gguf",
    tokenizer,
    quantization_method = "q4_k_m"
    # q4_k_m: 4-bit k-quant, medium — best quality/size tradeoff for serving
    # Output: ~2.3GB single file, loadable by llama.cpp/Ollama
)
```

### 7.4 Hardware requirements and timing

| Hardware | Batch | Time/epoch | Total SFT time | VRAM peak |
|---|---|---|---|---|
| Colab T4 (15 GB) | 2 (accum=4) | ~25 min | ~75 min | ~13 GB |
| Colab A100 (40 GB) | 4 (accum=2) | ~10 min | ~30 min | ~22 GB |
| RTX 3090 (24 GB) | 2 (accum=4) | ~20 min | ~60 min | ~18 GB |

---

## 8. Stage ⑦: Benchmark Design

### 8.1 Held-out services (never seen in training)

| Service | Endpoints | Known real faults (EvoMaster) | Domain |
|---|---|---|---|
| `languagetool` | ~12 | 4 | Grammar/NLP API |
| `scout-api` | ~18 | 6 | GitHub analytics |
| `restcountries` | ~8 | 2 | Geography data |

### 8.2 Prompt protocol — exactly what each model receives

Every model receives **the same user message** constructed by `build_user_message_from_record()`. The system prompt differs per model (each model's own best practice):

- **Qwen3-4B-SFT:** Our custom system prompt (from SFT training)
- **Qwen3-4B-base:** Same custom system prompt (to isolate SFT contribution)
- **Gemini 2.5 Flash:** Google AI Studio API; same system + user messages; thinking mode enabled
- **GPT-4o-mini:** OpenAI API; same system + user messages

```python
def run_benchmark(endpoint_records: list[dict],
                  models: list[str],
                  base_url: str) -> list[dict]:
    results = []
    for ep in endpoint_records:
        user_msg = build_user_message_from_record(ep)
        for model_id in models:
            raw_output = call_model(model_id, SYSTEM_PROMPT, user_msg)
            try:
                thinking, test_code = parse_teacher_output(raw_output)
            except ValueError as e:
                results.append(make_error_result(model_id, ep, str(e)))
                continue
            verification = verify(test_code, base_url)
            results.append({
                "model_id":           model_id,
                "endpoint_record_id": ep["record_id"],
                "thinking_present":   len(thinking) > 50,
                "thinking_tokens":    count_tokens(thinking),
                "tests_generated":    test_code.count("def test_"),
                "syntax_valid":       verification.syntax_valid,
                "tests_run":          verification.tests_run,
                "faults_found":       verification.faults_found,
                "schema_found":       verification.schema_found,
                "inference_sec":      measure_inference_time(model_id),
                "api_cost_usd":       measure_cost(model_id, raw_output),
            })
    return results
```

### 8.3 Metrics (M1–M6)

| Metric | Formula | Captures |
|---|---|---|
| **M1** Syntax pass rate | `syntax_valid / tests_generated` | Can the model write parseable Python? |
| **M2** Execution pass rate | `tests_run / tests_generated` | Does the code actually run without import/type errors? |
| **M3** Fault detection rate | `endpoints_with_≥1_fault / total_endpoints` | Primary effectiveness metric |
| **M4** Total faults found | Sum of `faults_found` across all endpoints | Absolute bug count |
| **M5** Schema violation rate | `endpoints_with_schema_fault / total_endpoints` | Secondary quality metric |
| **M6** Cost per fault | `total_api_cost_usd / total_faults_found` | Primary cost-effectiveness metric |

**Note on M6:** For the student model, API cost is $0.00 (local inference). Report `inference_time_sec / faults_found` as the time-equivalent cost metric.

### 8.4 Expected results table (hypothesis)

| Model | M3 Fault rate | M4 Total faults | M1 Syntax% | M6 Cost/fault | M6 Time/fault |
|---|---|---|---|---|---|
| Qwen3-4B-SFT (ours) | ~60–70% | ~18–22 | ~93% | $0.00 | ~3.2s |
| Gemini 2.5 Flash | ~70–80% | ~21–26 | ~97% | ~$0.08 | ~4.5s (API) |
| GPT-4o-mini | ~55–65% | ~17–20 | ~92% | ~$0.03 | ~3.8s (API) |
| Qwen3-4B-base | ~35–45% | ~10–14 | ~85% | $0.00 | ~3.2s |

**Key claim to defend:** The student (4B, fine-tuned, free) closes ~80–90% of the gap to Gemini 2.5 Flash (API, paid) at 0 marginal cost. The SFT contribution is proven by the base model ablation.

---

## 9. Ablation Studies

### 9.1 Ablation A — thinking trace vs. answer-only

Train two versions of the student:
- **Version T:** SFT on full records including `<think>...</think>` in assistant turn
- **Version A:** SFT on records with thinking trace removed (code-only assistant turn)

**Hypothesis:** Version T outperforms Version A on M3 and M4 because the reasoning trace teaches the model *which* fault vectors to prioritize, not just *how* to write tests.

### 9.2 Ablation B — GOLDEN-only vs. GOLDEN + PARTIAL

- **Version G:** Train on GOLDEN records only (verified faults)
- **Version GP:** Train on GOLDEN + PARTIAL records (include "no fault found" as negative examples)

**Hypothesis:** Version GP achieves higher M1 and M2 (fewer syntax/runtime errors) because it sees more diverse API shapes, including simple endpoints where no faults were found.

### 9.3 Ablation C — EvoMaster fault hints vs. no hints

- **Version H:** Teacher receives `known_fault_hint` in user message
- **Version NH:** Teacher receives same prompt without fault hints

**Hypothesis:** Version H produces more GOLDEN records during corpus construction, but the student trained on H-derived data shows similar M3 to NH-derived data — meaning the hints accelerate corpus construction but the student learns to generate equivalent hypotheses independently.

---

## 10. Threats to Validity

| Threat | Type | Mitigation |
|---|---|---|
| EvoMaster services are small, educational-grade Java apps | External validity | Acknowledge as limitation; future work: commercial APIs |
| Teacher model (Qwen3.5-27B) may have seen EvoMaster apps in pretraining | Construct validity | Qwen3.5 cutoff is late 2025; most EMB apps are older but still plausible concern |
| 5-minute EvoMaster budget may miss deeper faults | Internal validity | Run EvoMaster 3× with different seeds; union of faults is ground truth |
| Benchmark comparison uses API calls for commercial models (latency varies) | Reliability | Run each model 3× per endpoint; report median inference time |
| Student trained on Qwen3.5 teacher may have inflated performance on Qwen-family benchmarks | Construct validity | Use Gemini 2.5 Flash as the primary comparison (different model family) |

---

## 11. Timeline (8 weeks)

| Week | Tasks | Deliverable |
|---|---|---|
| 1 | Clone EMB, build all 7 services, verify Docker deployment, parse all OAS specs | 200 endpoint_record.json files |
| 2 | Run EvoMaster (5min/service × 7 services × 3 seeds), parse all fault records | ~150 fault_record.json files |
| 3 | Set up Qwen3.5-27B via Ollama/vLLM, run teacher inference on 200 endpoints × 5 samples | ~1000 raw teacher outputs |
| 4 | Run verifier on all 1000 outputs; filter GOLDEN; analyze yield rate | ~350 golden records |
| 5 | Scale teacher inference to full 4000 calls (adjust temperature, add retry); build full corpus | ~1400–1600 golden records |
| 6 | Convert to SFT format; run Unsloth fine-tuning; validate val loss convergence | Trained Qwen3-4B-SFT model |
| 7 | Run full benchmark on held-out services; collect all metrics; run ablations | Results tables |
| 8 | Write paper; submit to ResFes | Final submission |

---

## 12. Key Design Decisions to Defend in Assessment

**Q: Why use Qwen3.5-27B as teacher and not GPT-4o?**  
A: GPT-4o is one of our benchmark competitors. Using it as teacher contaminates the experiment — the student would implicitly distill GPT-4o's style, making the comparison unfair. A free local model eliminates this bias and keeps the entire corpus construction cost-free.

**Q: Why is executable verification better than LLM-as-judge?**  
A: LLM-as-judge evaluates whether tests *look* correct. Execution evaluates whether tests *are* correct. A test that perfectly describes a plausible fault hypothesis but doesn't actually trigger a 500 is useless for fault detection. Code execution is the only ground truth that matters.

**Q: Why QLoRA over full fine-tuning?**  
A: With ~1400 training records and a 4B model, full fine-tuning risks catastrophic forgetting of the base model's Python syntax and HTTP knowledge. QLoRA trains only 0.65% of parameters (LoRA rank 32), sufficient for domain adaptation while preserving general capabilities. Also fits on a free Colab T4.

**Q: How do you ensure the test set is truly held-out?**  
A: The 3 held-out services (`languagetool`, `scout-api`, `restcountries`) are not touched during teacher inference, corpus construction, or SFT training. Their OAS specs appear nowhere in the training data. The only thing the student "knows" about them is the general structure of OAS endpoints — which is exactly what we want to measure.

**Q: What if faults_found = 0 across all models on a held-out endpoint?**  
A: This means the endpoint has no detectable faults under our 45-second execution window. Exclude it from M3/M4 calculation and report it as "no fault ground truth." This is honest — we can only measure relative performance on endpoints where at least one model found something.