---
name: itemized-functions
description: Generate exhaustive integration functions with comprehensive test suites for all 3rd-party APIs and external services. Automatically creates function wrappers, individual test files, integrated test runners, and a detailed report of API behavior, response signatures, latency, and failure modes.
---

## Overview

This skill analyzes architecture files to identify all external integrations (APIs, services, databases, etc.), then generates:

1. **Function wrappers** (`function_*.py`) — Clean, tested functions that use each integration exactly as specified in the architecture
2. **Individual test files** (`test_*.py`) — Comprehensive tests for each function covering success cases, failure modes, edge cases, and diverse input types
3. **Heavy API test suites** (when needed) — Separate test files for integrations requiring extensive data or API calls
4. **Integration test runner** (`run_all_tests.py`) — Master test orchestrator that runs all tests, collects results, and generates the final report
5. **Debug log** (`integrations.debug.log`) — Complete execution trace for troubleshooting
6. **Summary report** (`ITEMIZED_FUNCTIONS_REPORT.md`) — Detailed findings: function signatures, actual API responses (sanitized), latency metrics, test results, learnings

**Purpose:** Understand the *exact* behavior of every 3rd-party integration before building the full project. No assumptions. Real execution. Real data.

## Activation

**Trigger phrases:**
- "generate itemized functions"
- "create integration tests"
- "itemized functions from architecture"
- "test all integrations"

**Required input:** Architecture files must be provided or referenced. The skill reads the full architecture to understand all integrations, their purpose, and how they're used.

**Output:** All generated files are created in an `integration_tests/` directory at the repository root.

## Workflow

### Phase 1: Architecture Analysis

1. **Read all provided architecture files** (scan thoroughly, don't ask questions)
2. **Identify all external integrations**:
   - API services (Ollama, OpenAI, Anthropic, etc.)
   - Database systems (PostgreSQL, MongoDB, Redis, etc.)
   - File processing tools (GitHub Linguist, ImageMagick, etc.)
   - Message queues, cache systems, external webhooks, etc.
3. **For each integration, extract**:
   - Primary purpose (what the architecture says it's used for)
   - Specific use cases (e.g., "Ollama for tool calling" vs "Ollama for chat")
   - Expected inputs/outputs
   - Failure modes to test
   - Performance constraints or requirements
4. **Determine test complexity**:
   - Standard test: ≤10 API calls, <50MB data, simple responses
   - Heavy test: >10 API calls, >50MB data, streaming responses, or complex orchestration

### Phase 2: Credential Setup

Generate `.env.dev` at repository root with template entries for all required credentials:

```
# Ollama
OLLAMA_API_URL=http://localhost:11434
OLLAMA_MODEL=neural-chat

# GitHub Linguist
GITHUB_LINGUIST_PATH=/path/to/github-linguist

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_NAME=testdb
DB_USER=testuser
DB_PASSWORD=

# [Other integrations...]
```

**Note:** User must fill in actual values before running tests.

### Phase 3: Generate Function Wrappers

For each integration, create `integration_tests/function_[service].py`:

**Requirements:**
- Clean, production-ready function signatures
- Proper type hints (Python 3.8+)
- Error handling with meaningful error messages
- Timeout handling (appropriate per service)
- Input validation where needed
- Logging to `integrations.debug.log`
- Return consistent, testable response objects

**Example structure:**
```python
import os
import logging
from typing import Any, Dict, List
import requests
from datetime import datetime

logger = logging.getLogger(__name__)

def call_ollama_chat(prompt: str, model: str = None, temperature: float = 0.7, timeout: int = 30) -> Dict[str, Any]:
    """
    Call Ollama API for chat completion.
    
    Args:
        prompt: The user prompt
        model: Model name (uses OLLAMA_MODEL env var if not provided)
        temperature: Sampling temperature (0.0-1.0)
        timeout: Request timeout in seconds
    
    Returns:
        Dict with keys: response, model, created_at, latency_ms
    
    Raises:
        ValueError: If credentials/config missing
        requests.Timeout: If request exceeds timeout
        requests.RequestException: For API errors
    """
    try:
        start_time = datetime.now()
        
        api_url = os.getenv("OLLAMA_API_URL", "http://localhost:11434")
        model = model or os.getenv("OLLAMA_MODEL")
        
        if not model:
            raise ValueError("OLLAMA_MODEL not set in environment")
        
        response = requests.post(
            f"{api_url}/api/chat",
            json={
                "model": model,
                "messages": [{"role": "user", "content": prompt}],
                "temperature": temperature,
                "stream": False
            },
            timeout=timeout
        )
        response.raise_for_status()
        
        latency_ms = (datetime.now() - start_time).total_seconds() * 1000
        result = response.json()
        result["latency_ms"] = latency_ms
        
        logger.debug(f"Ollama chat call successful. Latency: {latency_ms}ms")
        return result
        
    except requests.Timeout:
        logger.error(f"Ollama API timeout after {timeout}s")
        raise
    except requests.RequestException as e:
        logger.error(f"Ollama API error: {str(e)}")
        raise
    except Exception as e:
        logger.error(f"Unexpected error calling Ollama: {str(e)}")
        raise
```

### Phase 4: Generate Test Files

For each function, create `integration_tests/test_[service].py`:

**Requirements:**
- Use pytest framework
- Test success cases with typical inputs
- Test success cases with diverse/edge case inputs (smart per-service)
- Test failure modes: auth failure, timeout, malformed response, rate limiting, service down
- Capture actual API responses (sanitize credentials)
- Measure latency
- Each test logs to `integrations.debug.log`
- Each test validates response structure

**Example structure:**
```python
import pytest
import os
from unittest.mock import patch, MagicMock
import requests
from function_ollama import call_ollama_chat

@pytest.fixture
def setup_env(monkeypatch):
    """Setup environment variables for testing."""
    monkeypatch.setenv("OLLAMA_API_URL", "http://localhost:11434")
    monkeypatch.setenv("OLLAMA_MODEL", "neural-chat")

class TestOllamaChatSuccess:
    """Test successful Ollama chat calls."""
    
    def test_basic_chat(self, setup_env):
        """Test basic chat completion."""
        response = call_ollama_chat("What is 2+2?")
        assert "response" in response
        assert response["model"] == "neural-chat"
        assert "latency_ms" in response
        assert response["latency_ms"] > 0
    
    def test_chat_with_temperature(self, setup_env):
        """Test chat with different temperature values."""
        for temp in [0.0, 0.5, 1.0]:
            response = call_ollama_chat("Tell a story", temperature=temp)
            assert "response" in response
            assert response["latency_ms"] > 0
    
    def test_long_prompt(self, setup_env):
        """Test with very long prompt."""
        long_prompt = "What is the meaning of life? " * 100
        response = call_ollama_chat(long_prompt)
        assert "response" in response

class TestOllamaChatFailures:
    """Test failure modes."""
    
    def test_auth_failure(self, setup_env, monkeypatch):
        """Test behavior when API authentication fails."""
        monkeypatch.setenv("OLLAMA_API_URL", "http://invalid-url:11434")
        with pytest.raises(requests.RequestException):
            call_ollama_chat("test")
    
    def test_timeout(self, setup_env, monkeypatch):
        """Test timeout handling."""
        with patch('requests.post') as mock_post:
            mock_post.side_effect = requests.Timeout()
            with pytest.raises(requests.Timeout):
                call_ollama_chat("test", timeout=1)
    
    def test_missing_credentials(self, monkeypatch):
        """Test when required env vars are missing."""
        monkeypatch.delenv("OLLAMA_MODEL", raising=False)
        with pytest.raises(ValueError):
            call_ollama_chat("test")
```

### Phase 5: Heavy API Test Suites (When Applicable)

If an integration qualifies as "heavy" (>10 API calls, >50MB data, streaming, complex orchestration), create a separate `integration_tests/heavy_test_[service].py`:

**Include in heavy tests:**
- Large data processing (if applicable)
- Streaming response handling
- Multiple chained API calls
- Performance benchmarks
- Resource usage patterns
- Rate limiting behavior

**Header with reasoning:**
```python
"""
Heavy API test suite for [service].

Reasoning:
- [Service] requires extensive testing due to [specific reason]:
  - Streaming responses with large payloads
  - Multiple chained API calls (25+ total)
  - Data processing >50MB
  - Complex state management across calls
  - Critical performance path in architecture

These tests are separated from standard tests to avoid:
- Excessive API quota usage during CI/CD
- Extended test execution time
- Unnecessary load on rate-limited endpoints
"""
```

### Phase 6: Integration Test Runner

Create `integration_tests/run_all_tests.py`:

**Responsibilities:**
1. Import and run all test files using pytest
2. Collect results: passed, failed, skipped
3. For each test, capture:
   - Test name
   - Status (passed/failed/skipped)
   - Execution time
   - Error message (if failed)
   - Sample response data (if success)
4. Aggregate latency metrics per service
5. Generate summary statistics
6. Write all to `ITEMIZED_FUNCTIONS_REPORT.md`

**Example output structure:**
```
Test Results Summary:
- Total: 42 tests
- Passed: 38
- Failed: 2
- Skipped: 2

Service Latency Metrics:
- Ollama Chat: avg 145ms, min 89ms, max 287ms (10 calls)
- GitHub Linguist: avg 234ms, min 156ms, max 412ms (8 calls)
- PostgreSQL: avg 12ms, min 8ms, max 31ms (10 calls)

[Detailed results for each service...]
```

### Phase 7: Debug Logging

All generated code writes to `integration_tests/integrations.debug.log`:

- Timestamp, log level, service name, message
- Request/response bodies (sanitize credentials)
- Timing information
- Error stack traces
- Environment info (for debugging credential/setup issues)

Format:
```
[2024-01-15 14:32:15.342] DEBUG [ollama] Calling /api/chat with model=neural-chat
[2024-01-15 14:32:15.521] DEBUG [ollama] Response received: 145ms latency, 1250 chars
[2024-01-15 14:32:16.012] ERROR [github-linguist] FAILED_TO_TEST - Connection refused (auth_required, network_error, timeout, api_error, etc.)
```

### Phase 8: Summary Report

Create `ITEMIZED_FUNCTIONS_REPORT.md`:

**Structure:**
```markdown
# Itemized Functions Report

**Generated:** [timestamp]
**Architecture Analyzed:** [list of architecture files]
**Total Integrations Tested:** [count]
**Test Success Rate:** [X%]

## Executive Summary
- [count] integrations identified and tested
- [X] tests passed, [Y] failed, [Z] skipped
- Key findings and blockers (if any)

## Integration Details

### [Service Name] (e.g., Ollama)

**Purpose (from architecture):** [extracted from architecture]

**Function Signature:**
\`\`\`python
def call_ollama_chat(prompt: str, model: str = None, temperature: float = 0.7, timeout: int = 30) -> Dict[str, Any]
\`\`\`

**Test Coverage:** [count tests, all passed/mixed/failed]

**Latency:** avg X ms, min Y ms, max Z ms (10 calls)

**Sample API Response (sanitized):**
\`\`\`json
{
  "response": "2 + 2 = 4",
  "model": "neural-chat",
  "created_at": "2024-01-15T14:32:15Z",
  "latency_ms": 145
}
\`\`\`

**Failure Modes Tested:**
- ✓ Timeout (handled correctly, raises Timeout exception)
- ✓ Auth failure (handled correctly, raises RequestException)
- ✓ Malformed response (handled correctly, raises JSONDecodeError)
- ✓ Service unavailable (raises ConnectionError)

**Key Learnings:**
- [Finding 1]: [detail]
- [Finding 2]: [detail]
- [Gotcha/quirk if discovered]: [detail]

**Heavy Tests:** None
(or if applicable: `heavy_test_ollama.py` — [reason])

---

### [Next Service...]

[Same structure as above]

---

## Failed Tests & Blockers

### [Service Name] - FAILED_TO_TEST
**Reason:** [auth_required, network_error, timeout, api_error, service_down, etc.]
**Error Message:** [exact error]
**Suggestion:** [how to resolve, e.g., "Set OLLAMA_API_URL in .env.dev and ensure Ollama service is running"]

---

## Cross-Service Insights

[Any patterns, dependencies, or interactions discovered across integrations]

---

## Recommendations

- [Any critical issues or setup requirements]
- [Performance or scaling considerations]
- [Dependencies between services]

---

## Test Execution Log

[Link to or excerpt from integrations.debug.log]
```

## Standards & Requirements

### Code Generation

- **Python 3.8+ compatible** — Type hints, f-strings, async/await support if needed
- **Error messages are meaningful** — Not generic "error occurred", but "OLLAMA_API_URL not set" or "Connection refused on localhost:11434"
- **Timeouts are sensible per service** — LLM APIs: 60s, Database: 10s, File processing: 30s, etc.
- **No hardcoded values** — All config from environment
- **Logging is comprehensive** — Every significant action logged

### Testing

- **Pytest conventions** — `test_*.py` files, fixtures, clear test names, assertions with messages
- **Real execution** — Actually call the APIs (not mocked), unless explicitly impossible
- **Diverse inputs per service** — Smart, not generic:
  - LLM APIs: different prompt lengths, temperatures, contexts
  - File processing: different file types/sizes
  - Databases: different query patterns, edge cases
  - Time-series: different time ranges, aggregations
- **Failure mode coverage** — Auth, network, timeout, rate limit, malformed data
- **Response validation** — Verify structure, types, required fields
- **Latency tracking** — Measure every call

### Report Generation

- **No credentials exposed** — Sanitize all env vars, passwords, tokens, API keys in report
- **Full response bodies** — Show actual data (sanitized) so developer understands exact format
- **Timestamps throughout** — When was this run, when was each test executed
- **Actionable findings** — Not just "failed" but "why" and "how to fix"
- **Report all failures and anomalies** — Do not omit unexpected behavior because it seems minor. Every quirk discovered now prevents a production incident later
- **Output goes under the user's name** — The report becomes their reference document. Incomplete or softened findings waste the testing effort

## Special Cases

**Streaming APIs (e.g., LLM chat with stream=true):**
- Test both streamed and non-streamed responses
- Measure latency from first token to completion
- Validate chunk format

**Databases:**
- Test CRUD operations
- Test connection pooling if applicable
- Test transaction handling
- Test query timeouts

**File Processing APIs:**
- Test with multiple file types
- Test error handling for unsupported formats
- Test large file handling

**Rate-Limited APIs:**
- Detect rate limit headers
- Test behavior when rate limited
- Suggest backoff strategies in report

**Webhook/Async APIs:**
- If applicable, test callback handling
- Test idempotency if needed

## Directory Structure (Final)

```
integration_tests/
├── .env.dev                          # Template credentials file
├── integrations.debug.log            # Debug log from test execution
├── ITEMIZED_FUNCTIONS_REPORT.md      # Final summary report
├── run_all_tests.py                  # Master test runner
├── function_ollama.py                # Function wrapper
├── test_ollama.py                    # Standard tests
├── heavy_test_ollama.py              # (if needed) Heavy tests
├── function_github_linguist.py       # Another wrapper
├── test_github_linguist.py           # Standard tests
├── function_postgres.py              # Another wrapper
├── test_postgres.py                  # Standard tests
└── [More function/test pairs...]
```

## Execution

1. User provides architecture files and triggers skill
2. Skill generates all files in `integration_tests/`
3. User fills in `.env.dev` with actual credentials
4. User runs: `python run_all_tests.py`
5. All tests execute, report generated
6. Developer reviews `ITEMIZED_FUNCTIONS_REPORT.md` and `integrations.debug.log`

## Quality Assurance

- **No questions asked** — Read architecture, infer integrations, generate tests
- **LLM confidence in test design** — Trust that generated tests cover the right scenarios
- **Sanitization is rigorous** — Scan all report content for credentials before writing
- **File encoding is UTF-8** — Handle responses with special characters correctly
- **All files importable** — Generated Python is syntactically correct and runs