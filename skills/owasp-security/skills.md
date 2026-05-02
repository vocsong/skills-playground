# owasp-security

A read-only, statically-analyzed OWASP security scanner for code repositories. Detects the most common OWASP vulnerabilities across Web, API, LLM/Agentic, and Mobile domains using pattern/grep-based checks. **Runs all domain scans in parallel via subagents for maximum speed.** No external tools required. Produces a chat-only report — no file writes, no changes.

## Who / When

- Developers who want a fast security pulse before committing code
- Triggered by prompts like: "scan for security issues", "owasp check", "security review", "vulnerability scan"

## Workflow

### Phase 0: Auto-detection (sequential — must complete before spawning subagents)

Determine which domains apply by scanning the file tree. Use Glob and Grep to check for domain signals. Skip all dependency/generated directories.

| Signal | Domain enabled |
|--------|---------------|
| `*.html`, `*.jsx`, `*.tsx`, `*.vue`, `*.svelte`, framework files (Django, Rails, Laravel, Express, Next.js, Flask, Spring, ASP.NET, Phoenix) | Web App Top 10 |
| REST routes (`*.controller.*`, `*routes*`, `*router*`), GraphQL schemas (`*.graphql`, `*.gql`), OpenAPI/Swagger specs (`openapi.*`, `swagger.*`), `gRPC` proto files | API Security Top 10 |
| LLM SDK imports (`openai`, `langchain`, `llamaindex`, `autogen`, `crewai`, `langgraph`, `anthropic`, `cohere`, `transformers`, `vllm`, `tiktoken`, `semantic-kernel`, `guidance`), prompt template files (`*.prompt`, `*.jinja2`), MCP config, agent framework files | LLM + Agentic Top 10 |
| Android files (`*.kt`, `*.java` under `android/` or `app/`), iOS files (`*.swift`, `*.m`), mobile configs (`AndroidManifest.xml`, `Info.plist`, `build.gradle*`, `Podfile`, `*.xcodeproj`) | Mobile Top 10 |

If no signals match, default to Web App scan (every repo has web-adjacent risks). Report which domains were detected and which will be scanned.

### Phase 1-4: Parallel domain scans (launch ALL simultaneously)

**Critical: Launch one subagent per detected domain, all in parallel in a single message.** Each subagent receives its domain's entire grep rule set as its task prompt. The subagent searches the repository using Grep for every rule and returns findings with file:line references.

Subagent task descriptions and configurations:

- **Subagent: `owasp-web`** — If Web App detected. Scan all 10 A01-A10 categories using the grep rules in the "Web App Rule Set" section below. Include the full rule set in the prompt so the subagent has all rules locally.
- **Subagent: `owasp-api`** — If API detected. Scan all 10 API1-API10 categories using the grep rules in the "API Rule Set" section below.
- **Subagent: `owasp-llm`** — If LLM/Agentic detected. Scan LLM01-LLM10 plus Agentic-specific rules from the "LLM + Agentic Rule Set" section below.
- **Subagent: `owasp-mobile`** — If Mobile detected. Scan all 10 M1-M10 categories using the grep rules in the "Mobile Rule Set" section below.

Each subagent must return a structured summary of findings organized by risk category, with file:line references and severity for each finding.

### Phase 5: Aggregate and report (sequential — after all subagents complete)

Combine findings from all subagents into the output format specified in the "Output format" section. Do NOT write a report file.

---

## Subagent architecture: how to distribute work

When multiple subagent types are needed, launch them all at once. Example for a repo with all 4 domains detected:

```
Message 1 (single message, multiple Task tool calls in parallel):
  → Task(description="OWASP Web scan", prompt="[full Web App rule set + repo path]", subagent_type="general")
  → Task(description="OWASP API scan", prompt="[full API rule set + repo path]", subagent_type="general")  
  → Task(description="OWASP LLM scan", prompt="[full LLM rule set + repo path]", subagent_type="general")
  → Task(description="OWASP Mobile scan", prompt="[full Mobile rule set + repo path]", subagent_type="general")
```

Each subagent prompt must contain:
1. The repo path to scan
2. The full grep rule set for its domain (copy-pasted from this skills.md)
3. Instructions to skip `node_modules/`, `vendor/`, `.venv/`, `dist/`, `build/`, `target/`, `__pycache__/`, `.git/` directories
4. Instructions to return findings as: `[SEVERITY] [RISK_ID] file:line — description`
5. Instructions to return "✅ No issues found" for any risk category with zero matches

## Web App Rule Set: OWASP Web App Top 10:2025

> **Subagent prompt prefix:** Search the repository for the following web application security patterns using Grep. For each match, report the file path, line number, and the matching line content. Classify by risk category (A01-A10). Skip `node_modules/`, `vendor/`, `.venv/`, `dist/`, `build/`, `target/`, `__pycache__/`, `.git/`, `Pods/`, `.gradle/`.

### A01:2025 - Broken Access Control

Look for routes/endpoints that lack authorization checks — these are framework-specific.

**Generic patterns to grep:**

```
# Unprotected handlers lacking auth decorators/checks
r"@app\.(route|get|post|put|delete)\((.*)\)\s*\n\s*def (?!.*admin_required|login_required|require_auth|check_permissions|@jwt_required|@permission_required)"

# Direct object references: id/userId/accountId in params without ownership checks
r"(req\.params\.id|req\.query\.\w*[iI]d|request\.get\('id'\)|params\[:id\]|\bparams\.id\b)"

# Missing CORS restrictions (wildcard origin)
r"Access-Control-Allow-Origin:\s*\*"
r"cors\(\)\s*$"
r"cors_origin=\s*['\"](?:http://\*|null|\*)"

# Missing CSRF protection on state-changing endpoints
r"(POST|PUT|DELETE|PATCH).*(csrf_exempt|skip_csrf)"
r"@csrf_exempt"
r"csrf\(\): false"

# Path traversal vulnerability patterns
r"(open|readFile|read_file|send_file|sendFile)\((?:.*\+|.*f['\"]).*(?:\.\./|[.]{2}\\|filePath|path)"
r"(require|include|import)\(.*\.\.[\\/]"

# Insecure direct object references in ORM/framework
r"(findById|find_one|findByPk|get_by_id)\((?:req\.|request\.|params\.|body\.)"
r"Model\.find\(.*req\.(?:params|query|body)"
```

### A02:2025 - Security Misconfiguration

```
# Debug mode enabled in production
r"(DEBUG|DEBUG_MODE|debug)\s*=\s*True"
r"APP_DEBUG\s*=\s*true"
r"ENV\s*=\s*['\"](?:development|dev)"

# Default credentials or weak passwords in config
r"password\s*[=:]\s*['\"]\s*(?:admin|password|123456|root|changeme|secret)['\"]"
r"DATABASE_URL\s*=\s*.*://(?:root|admin|postgres):(?:root|admin|password|123456)?@"

# Missing security headers
r"(?:X-Frame-Options\s*:\s*none|X-Content-Type-Options\s*:\s*nosniff)"
r"(?:Strict-Transport-Security|HSTS).*max-age\s*=\s*0"

# Verbose error pages / stack traces in production
r"DEBUG_PROPAGATE_EXCEPTIONS\s*=\s*True"
r"APP_ENV\s*=\s*['\"]production['\"]\s*.*\n.*debug\s*=\s*true"

# Unnecessary HTTP methods allowed
r"methods\s*=\s*\[(?=.*['\"]OPTIONS['\"])(?=.*['\"]TRACE['\"])(?=.*['\"]DELETE['\"])\"
r"Allow:\s*(?:OPTIONS|TRACE|DELETE|PUT|PATCH)\s*,?\s*(?:OPTIONS|TRACE|DELETE|PUT|PATCH)"

# Exposed config/dot files
r"(\.env|config\.yml|config\.json|\.aws/credentials|\.npmrc|\.dockercfg)"

# Directory listing enabled
r"autoindex\s+on"
r"Options\s+.*Indexes"
r"directory_listing\s*:\s*true"

# Cloud storage with public access
r"(aws_s3|google_storage|cloudfiles).*(?:acl\s*[=:]\s*['\"]public|public_read|allUsers|allAuthenticatedUsers)"

# Insecure cookie flags
r"set_cookie.*(?:secure\s*=\s*False|httponly\s*=\s*False|samesite\s*=\s*['\"]None['\"])"
r"cookie\.(?:secure|httpOnly)\s*=\s*false"
```

### A03:2025 - Software Supply Chain Failures

```
# Unpinned dependencies (loose version ranges)
# Check package.json, requirements.txt, pyproject.toml, Cargo.toml, go.mod
r"['\"]\s*[\^~>]\s*\d+"
r"['\"](?:>=|<=|~>)"
r"['\"]latest['\"]"
r"['\"]\*['\"]"

# Git dependencies without commit hash
r"(github\.com|gitlab\.com|bitbucket\.org).*(?:\#)?main\b"

# Deprecated/outdated packages (name-based heuristics)
r"(?:moment|lodash\.template|left-pad|event-stream|ua-parser-js)"

# Typosquatting indicators (look for near-miss popular packages)
# (flag manually — list known popular package names)

# No lockfile present (check for missing)
# package-lock.json, yarn.lock, Pipfile.lock, Cargo.lock, go.sum, Gemfile.lock, poetry.lock

# Inline scripts/external resources in HTML without integrity hashes
r"<script\s[^>]*src\s*=\s*['\"]https?:[^>]*>(?!.*integrity)"
r"<link\s[^>]*href\s*=\s*['\"]https?:[^>]*rel\s*=\s*['\"]stylesheet[^>]*>(?!.*integrity)"
```

### A04:2025 - Cryptographic Failures

```
# Weak hashing algorithms
r"\b(?:MD5|md5|SHA-?1)\b\s*\("
r"(hashlib\.md5|hashlib\.sha1|crypto\.createHash\('md5'|crypto\.createHash\('sha1'))"

# Weak ciphers and modes
r"\b(?:DES|RC4|RC2|Blowfish|ECB\b|CBC\b)\b"
r"(?:des\b|rc4\b|rc2\b)"
r"(?:AES\.MODE_ECB|Cipher\.getInstance\(\"DES\"|\"AES/ECB)"

# Hardcoded encryption keys / secrets / private keys
r"(?:SECRET_KEY|ENCRYPTION_KEY|PRIVATE_KEY|API_KEY|AUTH_TOKEN|JWT_SECRET|access_key_id)\s*[=:]\s*['\"][\w]{8,}['\"]"
r"(?:-----BEGIN (?:RSA |EC )?PRIVATE KEY-----|-----BEGIN OPENSSH PRIVATE KEY-----)"

# HTTP instead of HTTPS for sensitive endpoints
r"https?://\w+\.(?:\.\w+)*/(?:api|auth|login|signup|payment|checkout|admin)"
r"fetch\(['\"]http://(?!localhost|127\.0\.0\.1)"
r"(?:axios|requests)\.(?:get|post|put|patch)\(['\"]http://(?!localhost)"

# Predictable random values for security
r"(?:Math\.random|rand\(|random\.randint|Random\(\))\s*\(.*(?:token|key|password|secret|session)"
r"(?:UUID\.randomUUID|uuid4)\s*\(\s*\)\s*.*(?!.*secure|.*crypto)"

# Plaintext passwords in code
r"(?:password|passwd|pwd)\s*[=:]\s*['\"][^'\"]{1,40}['\"]\s*[;\n\)]"

# Insufficient key sizes
r"(?:RSA\.generate|rsa\.generate)\(\s*(?:512|1024)\s*\)"
r"(?:DSA\.generate|dsa\.generate)\(\s*(?:512|1024)\s*\)"
```

### A05:2025 - Injection

**SQL Injection:**

```
# Raw SQL with string concatenation / interpolation
r"(?:execute|exec|query|raw|rawQuery)\(\s*(?:['\"]|['\"]\s*[\+\s]|f['\"]|`).*(?:SELECT|INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|EXEC)\b"
r"(?:\$\{.*\}|#{.*}|%s|%d)\s*.*(?:SELECT|INSERT|UPDATE|DELETE|DROP)"

# ORM raw methods with user input
r"\.raw\(.*(?:req\.|request\.|params\.|body\.|ctx\.)"
r"cursor\.execute\(.*(?:req\.|request\.|params)\."

# Dynamic table/column/group-by names
r"(?:order_by|group_by|ORDER BY|GROUP BY)\s*[=:.]\s*(?:req\.|request\.|params)"
```

**Cross-Site Scripting (XSS):**

```
# Unsanitized output rendering
r"innerHTML\s*="
r"dangerouslySetInnerHTML"
r"v-html\s*="
r"\{\{\{\s*\w+"
r"(?:unescape|unescapeHTML|mark_safe|safe\s*=\s*['\"]html['\"])\("

# document.write / eval of user input
r"(?:document\.write|eval|Function\(|setTimeout|setInterval)\(.*(?:req\.|params\.|location\.|\.search\b|\.hash\b|\.pathname)"
r"new Function\(.*req\."

# Missing DOMPurify or sanitize-html on user content
# (flag if innerHTML is used without sanitize import nearby)
```

**Command Injection:**

```
# Shell command execution with user input
r"(exec|spawn|execSync|spawnSync|subprocess\.(?:call|run|Popen)|os\.(?:system|popen)|shell_exec|passthru|system)\(.*(?:req\.|request\.|params\.|body\.)"
r"(eval|exec)\(.*(?:req\.|request\.|params\.|body\.|ctx\.)"
r"child_process\.exec\(.*(?:req\.|body\.|query\.|params\.)"
r"(?:os\.popen|commands\.getoutput|commands\.getstatusoutput)\(.*req\."
r"ProcessBuilder.*(?:req\.|request\.)"
r"Runtime\.getRuntime\(\)\.exec\(.*(?:req\.|request\.)"
```

**Template Injection:**

```
# Common template engine injection patterns
r"(?:render_template_string|\.render\b|\.compile\b)\(.*(?:req\.|request\.|params\.|body\.)"
r"(?:Jinja2|jinja2)\("
r"(?:nunjucks\.renderString|nunjucks\.render)\(.*(?:req\.|body\.|query\.)"
r"\.env\.from_string\(.*req\."
```

**NoSQL Injection:**

```
# MongoDB raw operators from user input
r"(?:find|update|aggregate)\(\s*\{(?=[^{]*?\$(?:gt|gte|lt|lte|ne|in|nin|eq|regex|where|or|and)).*\}"
r"\.find\(\s*(?:req\.|request\.|body\.|params\.)"
```

**LDAP/XML/YAML Injection:**

```
# LDAP
r"ldap(?:\.search|\.Search)\(.*(?:req\.|request\.)"
# XML/YAML
r"(?:xml\.etree|\.parseXML|parse_xml|yaml\.load\b|yaml\.unsafe_load)\(.*(?:req\.|request\.|body\.)"
r"yaml\.load\(.*(?:req\.|body)"
```

### A06:2025 - Insecure Design

Not statically detectable in code. Flag if the repo contains no threat model document, no `SECURITY.md`, or no architecture review — suggest manual review.

### A07:2025 - Authentication Failures

```
# Hardcoded credentials
r"(?:username|user|email|login)\s*[=:]\s*['\"][^'\"]{2,50}['\"]\s*[,;)]\s*\n\s*(?:password|passwd|pwd)\s*[=:]\s*['\"]"

# Weak password requirements
r"min_length\s*[=:]\s*[0-7](?!\d)"    # password min length < 8
r"(?:PASSWORD_MIN_LENGTH|password_min_length)\s*=\s*[0-7]"

# JWT with algorithm 'none' or no signature verification
r'"alg"\s*:\s*"none"'
r"(?:verify_signature|decode_without_verify|verify\s*=\s*False|jwt\.decode\b.*,\s*options\s*=\s*\{\s*['\"]verify_signature['\"]\s*:\s*False\s*\})"

# Session fixation vulnerability
r"session_regenerate_id|session\.regenerate"
# Missing session regeneration after login

# Token/credential in URL
r"(?:token|api_key|api-key|access_token|auth)=[^&\s'\"]{8,}"
```

### A08:2025 - Software and Data Integrity Failures

```
# Insecure deserialization
r"(?:pickle\.loads|pickle\.load|unpickle|marshal\.loads|cPickle\.loads)\(.*(?:req\.|request\.|body\.)"
r"(?:jsonpickle\.decode|yaml\.load\b)\(.*(?:req\.|request\.|body\.)"
r"(?:unserialize|ObjectInputStream|readObject)\(.*(?:req\.|request\.)"
r"(?:JSON\.parse|JSON\.parseObject)\(.*(?:req\.|request\.|body)"

# Missing integrity verification on downloads
r"(?:download|fetch|get)\(.*\.(?:exe|deb|rpm|apk|dmg|pkg|msi|bin|zip|tar\.gz).*(?!.*(?:sha256|md5|checksum|integrity|verify))"

# Auto-update without signature verification
r"(?:auto.?update|update.?check|check.?for.?update).*<(?!.*(?:sign|verify|integrity|checksum|sha))"

# CDN scripts without SRI (Subresource Integrity)
r"<script\s.*src\s*=\s*['\"]https?://(?!cdnjs|unpkg|jsdelivr)[^'\"]+\.js['\"].*>(?![^<]*integrity)"
```

### A09:2025 - Security Logging and Alerting Failures

```
# Exception swallowing
r"except\s*(?:Exception|BaseException|Throwable|Exception\s+as\s+e|error):\s*\n\s*(?:pass|continue)\b"
r"catch\s*\(\s*(?:Exception|Throwable)\s*[^)]*\)\s*\{\s*\}"

# Logging of sensitive data (credentials, tokens, PII)
r"(?:log|logger|console\.log|System\.out|print)\((?:.*(?:password|token|secret|credit.?card|ssn|sin|api.?key).*)\)"

# Generic catch returning raw errors to user
r"except.*:\s*\n\s*return.*(?:str\(|repr\(|\.message|\.stack)"
r"response\.(?:json|text|body)\s*=\s*(?:e|err|error|exc)"
```

### A10:2025 - Mishandling of Exceptional Conditions

```
# Empty catch blocks
r"catch\s*\([^)]*\)\s*\{\s*\}"
r"except\s*:\s*\n\s*pass\s*\n"

# Catching Exception (too broad)
r"(?:except\s+Exception|except\s+BaseException|catch\s*\(\s*Exception|catch\s*\(\s*Throwable)"

# Returning detailed system info in errors
r"(?:error|err|exc)\.(?:stack|stacktrace|message|__traceback__)"
r"traceback\.(?:print_exc|format_exc|print_exception)"
r"response\.status\s*=\s*500.*\n.*(?:exc_info|traceback|stack)"

# Returning exception message directly
r"res\.(?:send|json|write)\(.*(?:e\.message|err\.message|error\.message|\.toString)"
```

---

## API Rule Set: OWASP API Security Top 10:2023

> **Subagent prompt prefix:** Search the repository for the following API security patterns using Grep. For each match, report the file path, line number, and the matching line content. Classify by risk category (API1-API10). Skip `node_modules/`, `vendor/`, `.venv/`, `dist/`, `build/`, `target/`, `__pycache__/`, `.git/`.

### API1:2023 - Broken Object Level Authorization

```
# Endpoint with ID param but no auth middleware
r"(?:@app\.|router\.)(?:get|post|put|delete)\(['\"].*[/:](?:\:id|:userId|:accountId|<int:id>|<uuid>).*['\"]\s*\n\s*def\s+(?!.*login_required|jwt_required|token_required|admin_required|permission)"

# GraphQL field resolvers lacking ownership checks
# Look for resolver functions that return user data without auth context

# REST endpoints returning objects by ID without ownership check
r"(?:find|get|fetch).*(?:ById|_by_pk|ByPk).*\((?:req\.|request\.|args\.).*\bid\b"
```

### API2:2023 - Broken Authentication

```
# Missing authentication on API routes
r"(?:router|Blueprint|Flask|express)\.(?:get|post|put|delete|patch)\(['\"][a-z].*(?!.*auth|middleware|guard|protect)"
r"@app\.route\(['\"].*(?!.*\n\s*@(?:jwt_required|login_required|auth_required|token_required))"

# Weak API key patterns
r"api.?key\s*[=:]\s*['\"][\w-]{4,32}['\"]"
r"API_KEY\s*=\s*['\"][\w]{4,}['\"]"

# Token in URL or query string
r"(?:access_token|auth_token|bearer_token|jwt)\s*=\s*[^&\s]+"

# No token expiry
r"JWT_ACCESS_TOKEN_EXPIRES\s*=\s*False"
r"expiresIn\s*:\s*(?!\d)"
```

### API3:2023 - Broken Object Property Level Authorization

```
# Mass assignment (unfiltered request body assigned directly to model)
r"(?:\.save\b|\.create\b|\.update\b|\.insert\b)\(\s*(?:req\.body|request\.body|request\.data|params)"
r"(?:Model|model)\.(?:create|update|save)\(.*(?:req\.data|body|params\.to_hash|request\.POST)"
r"\.assign\(.*(?:req\.body|request\.body)\)"

# No field allowlist on create/update
# (flag when .create or .update receives body without .only, .permit, .pick, whitelist, or safe_params)
```

### API4:2023 - Unrestricted Resource Consumption

```
# Missing rate limiting
# (Check for absence of: rate_limit, limiter, throttle, express-rate-limit, flask-limiter)

# Missing pagination on list endpoints
r"(?:Model|\.model|\.db)\.(?:find|filter|all|list|getAll|findAll)\(\s*\)"
r"SELECT\s+\*\s+FROM\s+\w+(?:\s*;|\s*$)(?!.*LIMIT)"

# Infinite loop / recursion risk in API processing
r"while\s*True\s*:"
r"while\s*\(\s*true\s*\)"
r"for\s*\(;;\)"

# Large file upload without size limits
r"(?:upload|multer|file_upload)\((?!.*(?:limits|fileSize|maxFileSize|MAX_CONTENT_LENGTH))"
```

### API5:2023 - Broken Function Level Authorization

```
# Admin endpoints without role check
r"(?:/admin|/management|/sys|/dashboard)['\"].*(?!.*\n.*(?:is_admin|role|isAdmin|admin_required))"

# GraphQL mutations without auth check
r"(?:mutation\s+(?:delete|remove|update|set|grant|assign|revoke).*(?!.*auth|role))"
```

### API6:2023 - Unrestricted Access to Sensitive Business Flows

Not statically detectable. Flag endpoints dealing with purchases, votes, reviews, comments, likes without rate-limiting.

### API7:2023 - Server-Side Request Forgery (SSRF)

```
# User-controlled URL in HTTP client calls
r"(?:fetch|axios|requests|urllib|http\.(?:get|request)|httpx|httpGet)\((?:req\.|request\.|body\.|params\.).*(?:url|uri|href|link|endpoint|target|dest|host|redirect)"
r"(?:open|urlopen|http\.client|HttpClient)\((?:req\.|request\.|body\.).*url"

# Image/resource download from user-supplied URL
r"(?:download|fetch).*image.*(?:req\.|body\.)"
r"(?:requests\.get|axios\.get|fetch)\((?:req\.|body\.).*img"

# Webhook forwarding without URL validation
r"(?:webhook|callback|custom_url|redirect_uri|return_url).*(?:req\.|request\.)"

# File_get_contents / include with remote URL from user
r"(?:file_get_contents|include|require|curl_exec|curl_setopt).*\$_(?:GET|POST|REQUEST|COOKIE)"

# URL parsing then request without allowlist validation
r"(?:parse_url|urlparse|URL\().*(?:req\.|body\.).*\n.*\n.*(?:request|fetch|get)"
```

### API8:2023 - Security Misconfiguration

```
# Verbose error details in API responses
r"(?:res\.(?:json|send)|return\s*\{).*(?:error\.stack|err\.stack|error\.message|e\.message|traceback)"

# CORS misconfiguration on APIs
r"Access-Control-Allow-Origin:\s*\*"
r"cors\(\)"

# Missing Content-Type validation
# (flag endpoints not checking Content-Type header)

# Unnecessary HTTP methods on API
r"app\.(?:all|use)\(.*api"
```

### API9:2023 - Improper Inventory Management

```
# Debug/test endpoints exposed
r"(?:/debug|/test|/staging|/beta|/sandbox|/demo)['\"]"
r"(?:/graphiql|/playground|/explorer)['\"]"

# Deprecated API version endpoints
r"(?:/v1|/v2).*(?:deprecated|legacy|old)"

# Unsecured Swagger/OpenAPI docs
r"(?:/docs|/swagger|/api-docs|/openapi\.json|/redoc)"
```

### API10:2023 - Unsafe Consumption of APIs

```
# No validation/parsing of third-party API responses
# (Flag when fetch/axios calls to external APIs lack validate, parse, or Zod/Yup schema)

# Trusting external API data without sanitization
r"(?:response|resp)\.(?:json|data|body).*(?:innerHTML|dangerouslySetInnerHTML|eval|exec|Function)"
```

---

## LLM + Agentic Rule Set: OWASP Top 10 for LLM Applications:2025 + Agentic 2026

> **Subagent prompt prefix:** Search the repository for the following LLM/Agentic security patterns using Grep. For each match, report the file path, line number, and the matching line content. Classify by risk category (LLM01-LLM10 and Agentic-specific). Skip `node_modules/`, `vendor/`, `.venv/`, `dist/`, `build/`, `target/`, `__pycache__/`, `.git/`.

### LLM01:2025 - Prompt Injection

```
# Direct user input passed to LLM without sanitization
r"(?:openai|anthropic|cohere|llm|model)\.(?:chat\.completions\.create|completion|generate|invoke|predict|run)\((?!.*(?:guard|filter|validate|sanitize))"
r"(?:llm\.call|chain\.(?:call|run|invoke|predict))(?!.*guard)"

# User input embedded in prompt templates without escaping
r"prompt\s*=\s*f['\"]\s*\{.*(?:user_input|message|query|question|chat_input)"
r"(?:system|user|assistant).*role.*content.*f['\"]"

# Missing prompt injection guard/monitor
# (flag if langchain/llamaindex is used but no guardrails, guardrails-ai, or prompt injection detection)

# LangChain/LLamaIndex without output parsing/validation
r"(?:LLMChain|ConversationChain|AgentExecutor|SimpleSequentialChain|ReActAgent)\("

# System prompt from user-controlled source
r"(?:system_prompt|sys_prompt)\s*=\s*(?:req\.|request\.|body\.|params\.)"
```

### LLM02:2025 - Sensitive Information Disclosure

```
# API keys hardcoded near LLM code
r"(OPENAI_API_KEY|ANTHROPIC_API_KEY|COHERE_API_KEY|GOOGLE_API_KEY|AZURE_OPENAI_KEY|HUGGINGFACE_TOKEN|REPLICATE_API_TOKEN)\s*=\s*['\"]"

# Sensitive data in training/fine-tuning data
# (flag if training data scripts read from PII-heavy sources)

# Sensitive data included in prompt/context
r"(?:prompt|context|messages).*(?:password|secret|token|ssn|credit.?card|dob|date.?of.?birth|address)"
r"f['\"].*(?:\.password|\.secret|\.token|ssn)"

# Vector DB / RAG without access control
r"(?:pinecone|weaviate|qdrant|chroma|chromadb|milvus|pgvector).*index"
# (flag if no auth configured on vector DB)
```

### LLM03:2025 - Supply Chain

```
# Model loaded from URL without integrity/version
r"(?:load_model|from_pretrained|AutoModel|pipeline)\(['\"]https?:"
r"(?:hub\.load|huggingface_hub|snapshot_download)\(.*['\"].*(?:\#main|\#latest)"

# Unpinned model versions
r"model_name\s*=\s*['\"][^'\"]+/[^'\"]+['\"]\s*$"  # missing @revision
r"(?:from_pretrained|load_model)\(['\"][^'\"]+['\"](?:\)|\s*$)"  # no revision

# Clone/download model repos without verification
r"(?:git clone|git_clone|Repo\.clone_from)\(.*(?:huggingface|model|llm|weights)"

# Custom model from untrusted source
r"(?:torch\.load|load_state_dict)\(.*url"
r"(?:pickle\.load|joblib\.load)\(.*(?:url|download|http)"
```

### LLM04:2025 - Data and Model Poisoning

```
# Training/fine-tuning from user-supplied data
r"(?:fine.?tune|fine_tune|train|fit|TrainingArguments|Trainer)\(.*(?:req\.|request\.|body\.|upload|user)"

# RLHF/custom datasets from untrusted sources
r"(?:load_dataset|Dataset\.from|\.read_csv|\.read_json)\(.*(?:http|req\.|request\.)"
r"(?:dataset|training_data|csv_url)\s*=\s*(?:req\.|request\.|body\.)"

# LoRA adapters from external URLs
r"(?:load_adapter|PeftModel\.from_pretrained|LoraModel)\(['\"]https?:"
```

### LLM05:2025 - Improper Output Handling

```
# LLM output rendered directly in HTML/DOM without sanitization
r"(?:\.content|completion|response|\.text|\.answer|output).*innerHTML"
r"(?:\.content|\.text|\.answer).*dangerouslySetInnerHTML"

# LLM output passed to eval/exec/system without validation
r"(?:eval|exec|os\.system|subprocess)\(.*(?:response|completion|output|\.content)"
r"new Function\(.*(?:response|completion|\.content)"

# LLM output as SQL query
r"(?:execute|query|raw)\(.*(?:response|completion|output|\.content)"
r"(?:db\.run|connection\.execute)\(.*\.(?:content|text|answer)"

# LLM output in shell commands
r"(?:exec|spawn|popen)\(.*(?:llm|model|response|output|completion)"
```

### LLM06:2025 - Excessive Agency

```
# LLM with tool-calling / function-calling without restrictions
r"(?:function_call|tool_calls|FunctionTool|Tool\()"
r"(?:ChatOpenAI|ChatAnthropic).*bind_tools"

# Agents with write/delete/payment capabilities
r"(?:Agent|agent|AgentExecutor|Crew|crewai)\(.*(?:write_file|delete|payment|transfer|execute|shell|bash|command)"
r"(?:tools\s*=\s*\[.*(?:ShellTool|PythonREPLTool|WriteFileTool|Terminal|Execute))"

# LangChain agents with dangerous tools
r"(?:create_openai_tools_agent|create_react_agent|ZeroShotAgent|StructuredChatAgent)"
r"(?:crew\s*=\s*Crew\(.*agent"
r"(?:AutoGPT|BabyAGI|AgentGPT)"

# No human approval step before action
# (flag if agent executes tool calls without requiring human confirmation)
r"(?:agent\.run|agent_executor\.invoke|agent\.(?:call|execute)(?!.*approval|confirm|human|review)"

# MCP (Model Context Protocol) server without auth
r"(?:mcp|MCP|modelcontextprotocol)"
r"(?:@mcp\.(?:tool|resource|prompt)\().*(?!.*auth)"
```

### LLM07:2025 - System Prompt Leakage

```
# Hardcoded system prompts in source
r"system_prompt\s*=\s*['\"]"
r"role['\"]\s*:\s*['\"]system['\"].*content"
r"sys_prompt|SYSTEM_PROMPT|SYSTEM_MESSAGE|INSTRUCTIONS|system_message"

# Prompt templates without anti-extraction measures
# (flag system prompts that don't include anti-leak instructions)
```

### LLM08:2025 - Vector and Embedding Weaknesses

```
# RAG pipeline without access control per document
r"(?:vector_store|VectorStore|retriever|Retriever)\(.*(?:similarity_search|search|query)(?!.*(?:filter|metadata|acl|permission))"

# Embedding model from untrusted source
r"embeddings\s*=\s*(?:HuggingFaceEmbeddings|OpenAIEmbeddings)\(.*(?:url|endpoint)"
r"embed_model\s*=\s*['\"]https?:"

# Vector DB connections without auth
r"(?:chromadb\.HttpClient|pinecone\.init|weaviate\.Client|QdrantClient)\((?!.*api_key)"
r"(?:PINECONE_API_KEY|WEAVIATE_API_KEY|QDRANT_API_KEY)\s*=\s*['\"]"

# Unbounded vector search
r"top_k\s*=\s*\d{3,}"
r"(?:similarity_search|query)\((?!.*(?:k\s*=|top_k\s*=|limit\s*=))"
```

### LLM09:2025 - Misinformation

Not statically detectable. Flag if app uses LLM output for critical/medical/legal/financial decisions without human verification.

### LLM10:2025 - Unbounded Consumption

```
# Missing max_tokens on LLM calls
r"(?:completion|chat\.completions\.create|generate|invoke)\((?!.*max_tokens)"

# No rate limiting on LLM endpoints
# (flag LLM API calls without rate limiting middleware)

# Unbounded loop in agent reasoning
r"(?:while True|while\s*\(true\)|for\s*\(;;\)).*(?:agent|llm|model|chain|reasoning|think)"

# Parallel LLM calls without concurrency limits
r"(?:Promise\.all|asyncio\.gather|concurrent\.futures|ThreadPoolExecutor|parallel).*(?:llm|model|completion|agent)"
```

### Agentic-Specific: OWASP Top 10 for Agentic Applications 2026

```
# Goal manipulation / agent prompt injection
r"(?:agent_instructions|agent_prompt|goal_prompt|agent_config)\s*=.*(?:req\.|request\.|body)"
r"(?:crew\.kickoff|crew\.run|Crew\().*(?!.*guardrail|validate|sanitize)"

# Excessive tool permissions on MCP servers
r"(?:mcp_server|MCP_Server|mcp\.server|@mcp\.(?:tool|resource)).*(?:write|delete|execute|admin|sudo|root)"
r"(?:server\s*=\s*FastMCP|mcp\s*=\s*FastMCP)\("

# Agent memory/state tampering
r"(?:agent\.memory|ConversationBufferMemory|BufferMemory).*(?:req\.|request\.|user_input)"

# Cross-agent communication without boundary
r"(?:agent.*send_to_agent|inter_agent|agent.*message.*agent|delegate_to)"

# Unrestricted agent goals/tasks
r"(?:goal\s*=|objective\s*=|task\s*=)\s*['\"].*(?:any|everything|unlimited|all|unrestricted)"

# MCP server accepting arbitrary tool definitions
r"(?:mcp_server|MCP_Server)\.(?:tool|resource)\(.*(?:req\.|request\.|user).*name"
```

---

## Mobile Rule Set: OWASP Mobile Top 10:2024

> **Subagent prompt prefix:** Search the repository (focusing on `.kt`, `.java`, `.swift`, `.m`, `AndroidManifest.xml`, `Info.plist`, `build.gradle`, `Podfile` files) for the following mobile security patterns using Grep. For each match, report the file path, line number, and the matching line content. Classify by risk category (M1-M10). Skip `node_modules/`, `vendor/`, `.venv/`, `dist/`, `build/`, `target/`, `__pycache__/`, `.git/`, `Pods/`, `.gradle/`.

### M1:2024 - Improper Credential Usage

```
# Hardcoded API keys in mobile code
r"(?:apiKey|api_key|API_KEY|client_secret|CLIENT_SECRET)\s*=\s*['\"][\w\-]{8,}['\"]"
r"(?:accessKey|access_key|ACCESS_KEY)\s*=\s*['\"][\w\-]{8,}['\"]"

# Strings.xml / Info.plist containing secrets
# grep for XML/Plist with key-like entries
```

### M2:2024 - Inadequate Supply Chain Security

```
# Outdated SDK versions in build.gradle / Podfile
# Unpinned third-party library versions
```

### M3:2024 - Insecure Authentication/Authorization

```
# Biometric auth without fallback PIN
r"BiometricPrompt.*(?:DEVICE_CREDENTIAL|setDeviceCredentialAllowed\(false)"
r"\.authenticate\(.*cryptoObject.*(?!.*setNegativeButton)"

# Weak biometric implementation
r"BIOMETRIC_WEAK"
r"BiometricManager\.canAuthenticate\(BiometricManager\.BIOMETRIC_STRONG\)"

# Bypassable root/jailbreak detection
r"(?:isRooted|isRootedDevice|isJailbroken|checkRoot)\s*=\s*false"
r"(?:isDeviceRooted|isEmulator)\s*=\s*false"
r"return\s+false\s*;?\s*//.*(?:root|jailbreak|bypass|skip)"
```

### M4:2024 - Insufficient Input/Output Validation

```
# URL scheme handling without validation
r"(?:onNewIntent|handleOpenURL|onReceive).*data\.(?:getStringExtra|getData)"
r"(?:webView\.loadUrl|WKWebView.*load).*(?:getIntent|intent\.|url)"

# WebView without input sanitization
r"webView\.loadUrl\(.*(?:\+|String\.format|getIntent)"
r"setJavaScriptEnabled\(true\)"
r"setAllowFileAccess\(true\)"
r"setAllowUniversalAccessFromFileURLs\(true\)"

# eval of user-controlled data
r"evaluateJavascript\(.*(?:getIntent|intent\.|extras)"
```

### M5:2024 - Insecure Communication

```
# cleartext HTTP traffic
r"(?:usesCleartextTraffic|android:usesCleartextTraffic)\s*=\s*['\"]true['\"]"
r"(?:NSAppTransportSecurity|NSAllowsArbitraryLoads).*true"

# Certificate pinning disabled or missing
r"(?:sslPinning|SSLPinning|pinning).*false"
r"setHostnameVerifier\(.*ALLOW_ALL_HOSTNAME_VERIFIER"
r"trustAllCertificates\s*=\s*true"
r"allowUntrustedCertificates\s*=\s*true"

# HTTP endpoints in mobile code
r"\"http://(?!localhost|127\.0\.0\.1)[^\"\']*\""
r"fetch\(['\"]http://(?!localhost|127\.0\.0\.1)"
```

### M6:2024 - Inadequate Privacy Controls

```
# PII logged
r"(?:Log\.(?:d|e|i|v|w)|NSLog|println|console\.log|print)\(.*(?:email|phone|address|name|ssn|location|lat|lon|lng)"

# Clipboard exposure
r"clipboardManager\.setPrimaryClip\(.*(?:password|token|secret|key)"
r"UIPasteboard\.general\.string\s*="

# Screen capture allowed on sensitive screens
r"FLAG_SECURE\s*=\s*false"
r"setFlags\(.*FLAG_SECURE.*(?!WindowManager\.LayoutParams)"  # check if flag set to 0 or removed
```

### M7:2024 - Insufficient Binary Protections

```
# No obfuscation (ProGuard/R8 disabled)
r"minifyEnabled\s+false"
r"proguardFiles.*getDefaultProguardFile.*(?!\S)"

# Debuggable flag true
r"android:debuggable\s*=\s*['\"]true['\"]"
r"debuggable\s+true"

# Backup flag true (allows data extraction)
r"android:allowBackup\s*=\s*['\"]true['\"]"
r"allowBackup\s+true"

# No root/jailbreak detection
# (flag if no root detection library found: RootBeer, SafetyNet, IOSSecuritySuite)
```

### M8:2024 - Security Misconfiguration

```
# Debug mode enabled
r"(?:BuildConfig\.DEBUG|debug\s*=\s*true|DEBUG\s*=\s*true)"
r"WebView\.setWebContentsDebuggingEnabled\(true\)"

# Exported activities/services/receivers
r"android:exported\s*=\s*['\"]true['\"]"

# File permissions too permissive
r"(?:MODE_WORLD_READABLE|MODE_WORLD_WRITEABLE)"
r"openFileOutput\(.*Context\.MODE_WORLD"

# Firebase/Crashlytics debug
r"FirebaseCrashlytics.*setCrashlyticsCollectionEnabled\(true\)"
```

### M9:2024 - Insecure Data Storage

```
# SharedPreferences without encryption
r"getSharedPreferences\(.*(?!EncryptedSharedPreferences|MODE_PRIVATE)"
r"getPreferences\((?!.*MODE_PRIVATE)"

# SQLite without encryption
r"(?:SQLiteDatabase|openOrCreateDatabase|Room\.databaseBuilder)\((?!.*(?:sqlcipher|encrypt|crypto))"

# Internal storage without encryption
r"openFileOutput\(.*(?!.*(?:encrypt|EncryptedFile|cipher))"

# Keychain without proper access control
r"KeychainWrapper|keychain\s*=\s*.*Keychain\(.*(?!.*accessGroup)"
r"kSecAttrAccessible\s*:\s*kSecAttrAccessibleAlways"

# UserDefaults for sensitive data
r"UserDefaults\.standard\.(?:set|setValue)\(.*(?:password|token|secret|key|credential)"
```

### M10:2024 - Insufficient Cryptography

```
# Custom/custom-baked crypto in mobile code
r"(?:Cipher|Crypto|crypto|crypt)\.(?:getInstance|generateKey|init)"
r"(?:AES|DES|RC4).*(?:ECB|CBC)"  # weak modes

# Hardcoded IV / salt
r"(?:iv|IV|salt|Salt)\s*=\s*[\['\"]"

# Weak PRNG for crypto
r"(?:Random\(\)|Math\.random).*(?:key|encrypt|token|cipher|iv)"
r"SecureRandom\(\)\.setSeed\(.*(?:System\.(?:currentTimeMillis|nanoTime)|timestamp)"

# Static/constant encryption keys
r"(?:SecretKey|secretKey|secret_key)\s*=\s*.*(?:getBytes|String|['\"])"
r"SecretKeySpec\(.*['\"]"
```

---

## Output format

After all subagents complete, consolidate findings into a single chat response. No report file is written.

```
## OWASP Security Scan — <repo-name>

### Detected domains
- Web App ✅ (Framework: Next.js / Django / Express / ... — N source files)
- API ✅ (Found: REST endpoints / GraphQL schema / OpenAPI spec — N endpoints)
- LLM / Agentic ❌ (No LLM imports or agent patterns found)
- Mobile ❌ (No mobile code detected)

### Scan summary
| Domain | Risk | Status |
|--------|------|--------|
| Web App | A01 Broken Access Control | 3 findings (1❌ 2⚠️) |
| Web App | A02 Security Misconfiguration | 5 findings (2❌ 1⚠️ 2💡) |
| ... | ... | ... |
| API | API7 SSRF | 2 findings (1❌ 1⚠️) |
| ... | ... | ... |

---

### Web App: A01 Broken Access Control
- ❌ `src/routes/users.ts:42` — GET /users/:id with no auth middleware; direct object reference
- ⚠️ `app/controllers/api_controller.rb:15` — before_action :authenticate missing on :show
- 💡 `config/cors.py:3` — CORS origin set to `*`; consider restricting

### Web App: A02 Security Misconfiguration
- ❌ `config/settings.py:8` — DEBUG=True in production config
...

### API: API7 Server-Side Request Forgery
- ❌ `src/services/webhook.ts:28` — fetch(url) where url is from req.body without allowlist
...
```

### Severity markers

- `❌` Critical — high-confidence, likely exploitable (hardcoded secrets, SQL injection sinks, missing auth on sensitive routes)
- `⚠️` Warning — probable issue, needs context verification (debug modes, insecure cookies, CORS wildcard, outdated crypto)
- `💡` Info / suggestion — best practice gap (missing CSRF tokens, missing rate limits, unpinned versions)
- `✅` No issues found
- `👁️` Manual review — not statically detectable (Insecure Design, Misinformation, business logic flaws)

---

## Edge cases

- **Empty repo**: Report "nothing to scan" and stop.
- **Only docs/config**: Run auto-detection; if no source code domains detected, report "no application code found to scan."
- **Monorepo**: Each package/workspace may be a different domain. Subagents should scan their relevant subtrees.
- **Large repo (>5000 files)**: Subagents should sample strategically rather than exhaustively grepping every file. Sample at least 50% of source files per directory. Note sampling was used.
- **Domain overlap**: Files may trigger multiple domain rule sets (e.g., API routes in a web app). This is fine — let each subagent report independently; the final report naturally deduplicates across sections.
- **Auto-detection misses a domain**: Default to Web App scan. Offer to re-run with manually specified domains.
- **No `.gitignore`**: Still skip dependency dirs using hardcoded exclusion list (`node_modules/`, `vendor/`, `.venv/`, `dist/`, `build/`, `target/`, `__pycache__/`, `.git/`, `Pods/`, `.gradle/`).
- **Subagent times out**: Report partial findings for that domain with a note. Other domains continue unaffected.
- **Binary files in repo**: Subagents should skip binary files and note count of skipped files.

## Anti-patterns (do NOT do)

- Do NOT run `npm install`, `pip install`, or any dependency installation.
- Do NOT execute lint, test, build, or SAST tools.
- Do NOT modify any files.
- Do NOT write a report file unless explicitly asked.
- Do NOT commit anything.
- Do NOT fetch external vulnerability databases (NVD, OSV, etc.) — scan only what's in the repo.
- Do NOT flag "potential" issues without a file:line reference.
- Do NOT scan `node_modules/`, `vendor/`, `.venv/`, `dist/`, `build/`, `target/`, `__pycache__/`, `.git/`, `Pods/`, `.gradle/`.
- Do NOT run domain scans sequentially. Launch all subagents in parallel in a single message.
