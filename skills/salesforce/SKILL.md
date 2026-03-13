---
description: Interact with a Salesforce org — authenticate, run SOQL queries, create/update/delete records, execute Apex, manage metadata, upload/download files, and monitor org limits via sf CLI and REST API.
---

# Salesforce Org Interaction Skill

Use this skill when Claude Code needs to interact with a Salesforce org: authenticate, query data, create/update/delete records, execute Apex code, explore metadata, upload/download files, or monitor org health.

**When this skill is first loaded**, display the following message to the user:

```
🔌 Salesforce Skill loaded.

   By default, write and delete operations require your confirmation.
   To unlock all operations (including dangerous ones) without prompts:

   SALESFORCE_SKIP_WARNINGS=true

   Type or paste it anytime during the conversation to activate.
```

If the user sends `SALESFORCE_SKIP_WARNINGS=true` at any point in the conversation, treat it as if the environment variable is set: skip all write/delete confirmations for the rest of the session.

---

## Operation Safety Levels

This skill handles operations at three safety levels. **Always identify the safety level before executing.**

> **Bypass warnings:** If the user has set `SALESFORCE_SKIP_WARNINGS=true` in their environment or in `.claude/settings`, skip all write/delete confirmations and execute directly.

| Level | Operations | Behavior |
|-------|-----------|----------|
| 🟢 **READ** | SOQL queries, describe, org display, limits, debug logs | Execute freely |
| 🟡 **WRITE** | Create, update, upsert records | Warn user before executing. Show what will be written and ask for confirmation. |
| 🔴 **DELETE** | Delete records, bulk delete, data destroy | Show explicit warning with record count and object type. Require user confirmation before proceeding. **Once the user confirms, execute the delete.** |

### Write operation warning template

Before any write operation, display:

```
⚠️ SALESFORCE WRITE OPERATION
   Object:  {ObjectApiName}
   Action:  {CREATE | UPDATE | UPSERT}
   Records: {count} record(s)
   Org:     {username}

   Proceed? (y/n)

   Tip: set SALESFORCE_SKIP_WARNINGS=true to bypass these confirmations.
```

### Delete operation warning template

Before any delete operation, display:

```
🔴 SALESFORCE DELETE — THIS CANNOT BE UNDONE

   Object:  {ObjectApiName}
   Action:  DELETE
   Records: {count} record(s)
   Org:     {username}

   Deleted records go to the Recycle Bin (recoverable
   for up to 15 days, depending on org settings).
   Bulk API hard-deletes bypass the bin.

   Type "DELETE" to confirm, then proceed with the operation.

   Tip: set SALESFORCE_SKIP_WARNINGS=true to bypass these confirmations.
```

---

## 0. Shared HTTP Helper

All Python REST API functions in this skill use this shared helper for consistent error handling. Define it once and reuse throughout:

```python
import json, urllib.error, urllib.parse, urllib.request

def _sf_request(url, headers, data=None, method=None):
    """Make an HTTP request to Salesforce. Returns parsed JSON or None (204)."""
    req = urllib.request.Request(url, data=data, headers=headers)
    if method:
        req.method = method
    try:
        with urllib.request.urlopen(req) as resp:
            if resp.status == 204:
                return None
            return json.loads(resp.read())
    except urllib.error.HTTPError as e:
        error_body = e.read().decode("utf-8", errors="replace")
        try:
            sf_error = json.loads(error_body)
            if isinstance(sf_error, list) and sf_error:
                msg = f"{sf_error[0].get('errorCode', 'UNKNOWN')}: {sf_error[0].get('message', error_body)}"
            else:
                msg = error_body
        except (json.JSONDecodeError, KeyError):
            msg = error_body
        raise RuntimeError(f"Salesforce API error (HTTP {e.code}): {msg}") from e
```

Use `_sf_request` in all functions below instead of calling `urllib.request.urlopen` directly.

---

## 1. Prerequisites

### Install Salesforce CLI

```bash
npm install -g @salesforce/cli
```

Verify installation:

```bash
sf --version
```

**Cowork note:** Global install fails with `EACCES` in Cowork VMs. Use a local prefix:

```bash
mkdir -p $HOME/.npm-global
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
npm install -g @salesforce/cli
```

### Python Libraries (for file processing)

If the task involves downloading and analyzing files (PDF, DOCX, XLSX):

Preferred — use a virtual environment:

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install pdfplumber python-docx openpyxl
```

Fallback — if a venv is impractical (e.g., Cowork):

```bash
pip3 install pdfplumber python-docx openpyxl --break-system-packages
```

---

## 2. Authentication

Always try methods in this order. Use the first one that works.

> **Cowork:** Neither web login nor session ID work reliably in Cowork. Use **Method 2 (Manual OAuth Flow)** — it is the only reliable method. See details below.

### Check for existing connection first

```bash
sf org list 2>&1
```

If the target org is already listed as `Connected`, skip authentication and set it as default if needed:

```bash
sf config set target-org <username> --global
```

### Method 1: Web Login (recommended — local environments)

Opens the browser for standard OAuth login. Most secure — no tokens to handle manually.

```bash
sf org login web --instance-url https://<INSTANCE>.my.salesforce.com
```

Replace `<INSTANCE>` with the org's My Domain (e.g., `mycompany`). The browser opens automatically; the user logs in and grants access. The CLI stores the refresh token securely.

Add `--set-default` to make it the default org:

```bash
sf org login web --instance-url https://<INSTANCE>.my.salesforce.com --set-default
```

### Method 2: Manual OAuth Flow (Cowork and headless environments)

This is the only reliable method in Cowork. It performs a standard OAuth Authorization Code flow manually, producing a long-lived refresh token.

**Step 1 — Generate the authorization URL:**

```python
import urllib.parse

INSTANCE_URL = "https://<INSTANCE>.my.salesforce.com"
CLIENT_ID = "PlatformCLI"
REDIRECT_URI = "http://localhost:1717/OauthRedirect"

auth_url = (
    f"{INSTANCE_URL}/services/oauth2/authorize"
    f"?response_type=code"
    f"&client_id={CLIENT_ID}"
    f"&redirect_uri={urllib.parse.quote(REDIRECT_URI)}"
    f"&prompt=login%20consent"
    f"&scope=refresh_token%20api%20web"
)
print(auth_url)
```

**Step 2 — Ask the user to open the URL in their browser and log in.**

After login, Salesforce redirects to `http://localhost:1717/OauthRedirect?code=...`. Since no server is running on that port, the page will fail to load. Ask the user to **copy the full URL from the browser address bar** and paste it back.

**Step 3 — Exchange the authorization code for tokens:**

```python
import urllib.parse, urllib.request, json

# Extract the code from the redirect URL the user pasted
redirect_url = "<URL_FROM_USER>"
code = urllib.parse.parse_qs(urllib.parse.urlparse(redirect_url).query)["code"][0]

INSTANCE_URL = "https://<INSTANCE>.my.salesforce.com"
CLIENT_ID = "PlatformCLI"
REDIRECT_URI = "http://localhost:1717/OauthRedirect"

data = urllib.parse.urlencode({
    "grant_type": "authorization_code",
    "code": code,
    "client_id": CLIENT_ID,
    "redirect_uri": REDIRECT_URI
}).encode()

req = urllib.request.Request(f"{INSTANCE_URL}/services/oauth2/token", data=data, method="POST")
with urllib.request.urlopen(req) as resp:
    token_data = json.loads(resp.read())

# token_data contains: access_token, refresh_token, instance_url, scope, etc.
instance_url = token_data["instance_url"]
access_token = token_data["access_token"]
refresh_token = token_data["refresh_token"]
```

**Step 4 — Use REST API directly (bypass sf CLI in Cowork).**

The sf CLI has a DNS resolution bug in Cowork VMs (`DomainNotFoundError`). Use Python REST API calls for all operations instead of `sf data query`, `sf org display`, etc. See the Python examples throughout this skill.

### Method 3: Access Token (last resort)

Use only when both web login and Manual OAuth are not possible, and the org does not have IP-based session restrictions.

**Important:** Many Salesforce orgs lock sessions to the originating IP address. If the session ID was obtained from a different IP (e.g., user's browser vs. Cowork VM), authentication will fail with `INVALID_SESSION_ID` or `Bad_OAuth_Token`. In that case, use Method 2.

**Important:** Modern Salesforce orgs use `HttpOnly` cookies for the session ID. The `document.cookie` trick does **not** work in those orgs.

Ask the user for their session ID. They can get it from one of these methods:

1. **Developer Console** — Open Developer Console, execute anonymous Apex: `System.debug(UserInfo.getSessionId());`, copy from the debug log
2. **URL in Classic UI** — Switch to Classic, copy the `sid=` parameter from the URL
3. **Browser cookie (only if not HttpOnly)** — F12 → Application → Cookies → copy `sid` value

Then authenticate:

```bash
export SF_ACCESS_TOKEN="<TOKEN_FROM_USER>"
sf org login access-token \
  --instance-url https://<INSTANCE>.my.salesforce.com \
  --no-prompt \
  --set-default
```

**Note:** Session IDs expire after 2-12 hours depending on org settings. Both web login and Manual OAuth are preferred because they use refresh tokens that last much longer.

### Token Refresh

If the access token expires, refresh it using the stored refresh token:

```python
def sf_refresh_token(refresh_token, instance_url):
    """Refresh an expired access token. Returns (new_access_token, instance_url).
    Note: instance_url may change after org migrations — always use the returned value."""
    data = urllib.parse.urlencode({
        "grant_type": "refresh_token",
        "refresh_token": refresh_token,
        "client_id": "PlatformCLI"
    }).encode()
    headers = {"Content-Type": "application/x-www-form-urlencoded"}
    result = _sf_request(
        f"{instance_url}/services/oauth2/token", headers=headers, data=data
    )
    return result["access_token"], result.get("instance_url", instance_url)
```

Refresh tokens last for months (vs hours for session IDs).

### Verify connection

```bash
sf org display --json
```

Or via REST API (recommended in Cowork):

```python
def sf_verify_connection(instance_url, access_token):
    """Verify the connection by fetching org limits."""
    headers = {"Authorization": f"Bearer {access_token}"}
    limits = _sf_request(
        f"{instance_url}/services/data/v62.0/limits/", headers=headers
    )
    remaining = limits["DailyApiRequests"]["Remaining"]
    print(f"Connected. API calls remaining today: {remaining}")
    return limits
```

**API version note:** This skill uses API `v62.0` (Spring '26). To check the latest version available in your org: `GET /services/data/` — it returns all available versions. Replace `v62.0` throughout if needed.

---

## 3. Running SOQL Queries 🟢

### Via sf CLI

```bash
sf data query --query "SELECT Id, Name FROM Account LIMIT 10" --json
```

### Bulk queries

For large result sets or queries that may time out, add `--bulk` to use Bulk API 2.0:

```bash
sf data query --query "SELECT Id, Name FROM Account" --bulk --wait 10 --json
```

**Note:** The standard REST query endpoint paginates and handles any result size. Use `--bulk` when the query itself is complex/slow, or when you need to export data without pagination overhead.

### Tooling API queries

Query metadata objects using the Tooling API:

```bash
sf data query --query "SELECT Id, Name, Status FROM ApexClass WHERE Status = 'Active'" \
  --use-tooling-api --json
```

### Via REST API (Python)

For programmatic access with pagination support. **Required in Cowork** (sf CLI has DNS issues).

```python
def sf_query(soql, instance_url, access_token):
    """Execute a SOQL query with automatic pagination."""
    api_base = f"{instance_url}/services/data/v62.0"
    url = f"{api_base}/query/?q={urllib.parse.quote(soql)}"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    data = _sf_request(url, headers=headers)
    records = data.get("records", [])

    # Handle pagination
    while data.get("nextRecordsUrl"):
        next_url = f"{instance_url}{data['nextRecordsUrl']}"
        data = _sf_request(next_url, headers=headers)
        records.extend(data.get("records", []))

    return records
```

**Note:** The Tooling API also paginates via `nextRecordsUrl` but uses a different base path (`/services/data/v62.0/tooling/query/`). Use the same pagination pattern.

### Getting credentials from sf CLI (Python)

```python
import json, subprocess

def get_sf_credentials():
    """Get instanceUrl and accessToken from the authenticated sf CLI org."""
    result = subprocess.run(
        ["sf", "org", "display", "--json"],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(f"sf org display failed: {result.stderr.strip()}")
    parsed = json.loads(result.stdout)
    if parsed.get("status") != 0:
        raise RuntimeError(f"sf org display error: {parsed.get('message', result.stderr)}")
    data = parsed["result"]
    return data["instanceUrl"].rstrip("/"), data["accessToken"]
```

---

## 4. Discovering Objects and Fields 🟢

### List all custom objects

```bash
sf data query --query \
  "SELECT QualifiedApiName, Label FROM EntityDefinition WHERE QualifiedApiName LIKE '%__c'" \
  --json
```

**Note:** `EntityDefinition` does NOT support OR/disjunctions in WHERE. Query each condition separately.

### Describe an object's fields

Via sf CLI:

```bash
sf sobject describe --sobject <ObjectApiName> --json
```

Via REST API (required in Cowork):

```python
def sf_describe(sobject, instance_url, access_token):
    """Describe an object's fields via REST API."""
    headers = {"Authorization": f"Bearer {access_token}"}
    return _sf_request(
        f"{instance_url}/services/data/v62.0/sobjects/{sobject}/describe/",
        headers=headers
    )
```

Parse the `fields` array. Each field has: `name`, `type`, `label`.

### List all objects (standard + custom)

```bash
sf sobject list --json
```

### Common field filters for describe output

```python
# Filter fields by relevance
for f in fields:
    if any(kw in f['name'].lower() for kw in ['date', 'amount', 'status', 'name', 'account']):
        print(f"{f['name']} ({f['type']}) - {f['label']}")
```

### Record types for an object

```sql
SELECT Id, Name, DeveloperName, IsActive
FROM RecordType
WHERE SObjectType = '<ObjectApiName>'
AND IsActive = true
```

---

## 5. CRUD Operations

### 5a. Create Record 🟡

**WRITE OPERATION — confirm with user before executing.**

```bash
sf data create record --sobject Account \
  --values "Name='Acme Corp' Industry='Technology' Website='https://acme.example.com'" \
  --json
```

Via REST API (Python):

```python
def sf_create_record(sobject, record_data, instance_url, access_token):
    """Create a single record. Returns the new record ID."""
    url = f"{instance_url}/services/data/v62.0/sobjects/{sobject}/"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    data = json.dumps(record_data).encode("utf-8")
    return _sf_request(url, headers=headers, data=data)
    # Returns {"id": "001...", "success": true}
```

### 5b. Update Record 🟡

**WRITE OPERATION — confirm with user before executing.**

```bash
sf data update record --sobject Account \
  --record-id 001XXXXXXXXXXXX \
  --values "Industry='Finance' Rating='Hot'" \
  --json
```

Via REST API (Python):

```python
def sf_update_record(sobject, record_id, update_data, instance_url, access_token):
    """Update fields on an existing record."""
    url = f"{instance_url}/services/data/v62.0/sobjects/{sobject}/{record_id}"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    data = json.dumps(update_data).encode("utf-8")
    _sf_request(url, headers=headers, data=data, method="PATCH")
    # Returns None (204 No Content) on success
```

### 5c. Delete Record 🔴

**DELETE OPERATION — show warning and ask for confirmation. Once confirmed, execute.**

```bash
sf data delete record --sobject Account \
  --record-id 001XXXXXXXXXXXX \
  --json
```

Via REST API (Python):

```python
def sf_delete_record(sobject, record_id, instance_url, access_token):
    """Delete a single record. Returns None (204) on success."""
    url = f"{instance_url}/services/data/v62.0/sobjects/{sobject}/{record_id}"
    headers = {"Authorization": f"Bearer {access_token}"}
    _sf_request(url, headers=headers, method="DELETE")
```

### 5d. Upsert (Insert or Update) 🟡

**WRITE OPERATION — confirm with user before executing.**

Upsert uses an external ID field to decide whether to insert or update:

```bash
sf data upsert record --sobject Contact \
  --external-id Email \
  --values "Email='jdoe@example.com' FirstName='John' LastName='Doe'" \
  --json
```

---

## 6. Bulk Operations

### Bulk Upsert from CSV 🟡

**BULK WRITE — this affects many records. Show record count from CSV before executing.**

```bash
sf data upsert bulk --sobject Contact \
  --file contacts.csv \
  --external-id Id \
  --wait 10 \
  --json
```

### Bulk Delete from CSV 🔴

**BULK DELETE — show CSV row count and object name. Ask for confirmation, then execute.**

```bash
sf data delete bulk --sobject Contact \
  --file contacts_to_delete.csv \
  --wait 10 \
  --json
```

### Checking bulk job results

After any bulk operation, always check for partial failures:

```bash
sf data bulk results --job-id <JOB_ID> --json
```

The result includes `numberRecordsFailed`. If > 0, retrieve the failed-records CSV to inspect individual errors:

```
GET /services/data/v62.0/jobs/ingest/<JOB_ID>/failedResults/
Authorization: Bearer <ACCESS_TOKEN>
Accept: text/csv
```

### Data Export (Tree format) 🟢

Export records preserving relationships:

```bash
sf data export tree --query "SELECT Id, Name, (SELECT Id, LastName FROM Contacts) FROM Account WHERE Industry = 'Technology'" \
  --output-dir ./export/ \
  --json
```

### Data Import (Tree format) 🟡

**WRITE OPERATION — confirm with user. Show file contents summary.**

```bash
sf data import tree --files ./export/Account.json --json
```

---

## 7. Execute Anonymous Apex 🟡

Run Apex code directly against the org. **Always show the code to the user and confirm before executing.** Apex can perform any DML operation (insert, update, delete, callouts) — treat it with elevated caution.

### Inline execution

```bash
sf apex run --file script.apex --json
```

### From stdin

```bash
echo "System.debug('Hello from Claude Code');" | sf apex run --json
```

### Example: Mass update via Apex 🟡

```apex
// Update all Contacts missing a MailingCountry
List<Contact> contacts = [
    SELECT Id, MailingCountry
    FROM Contact
    WHERE MailingCountry = null
    LIMIT 200
];
for (Contact c : contacts) {
    c.MailingCountry = 'US';
}
update contacts;
System.debug('Updated ' + contacts.size() + ' contacts');
```

**Apex can perform any DML operation. Always review the code with the user before execution.**

### Execute Apex via REST API (Cowork fallback)

```python
def sf_execute_apex(apex_code, instance_url, access_token):
    """Execute anonymous Apex via REST API (POST to avoid URL length limits)."""
    url = f"{instance_url}/services/data/v62.0/tooling/executeAnonymous/"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/x-www-form-urlencoded",
    }
    data = urllib.parse.urlencode({"anonymousBody": apex_code}).encode("utf-8")
    result = _sf_request(url, headers=headers, data=data)
    # executeAnonymous always returns HTTP 200 — check response fields
    if not result.get("compiled"):
        raise RuntimeError(
            f"Apex compile error at line {result.get('line')}: "
            f"{result.get('compileProblem')}"
        )
    if not result.get("success"):
        raise RuntimeError(
            f"Apex runtime exception at line {result.get('line')}: "
            f"{result.get('exceptionMessage')}"
        )
    return result
```

---

## 8. Debug Logs & Monitoring 🟢

### View recent debug logs

```bash
sf apex list log --json
```

### Get a specific log

```bash
sf apex get log --log-id <LOG_ID> --json
```

### Tail logs in real-time

```bash
sf apex tail log --color
```

### Check org API limits

```bash
sf org display --json
```

Via REST API:

```
GET /services/data/v62.0/limits/
Authorization: Bearer <ACCESS_TOKEN>
```

Useful limits to monitor:
- `DailyApiRequests` — total API calls remaining
- `DailyBulkV2QueryJobs` — bulk query jobs
- `SingleEmail` — email sends remaining

---

## 9. Metadata Operations 🟢

### List metadata components

```bash
sf org list metadata-types --json
```

### Retrieve metadata

```bash
sf project retrieve start --metadata "ApexClass:MyClassName" --json
```

### Retrieve by manifest

```bash
sf project retrieve start --manifest manifest/package.xml --json
```

### Deploy metadata 🟡

**DEPLOY modifies org configuration. Confirm target org and components with user.**

```bash
sf project deploy start --metadata "ApexClass:MyClassName" --json
```

Dry-run validation (no changes applied):

```bash
sf project deploy start --metadata "ApexClass:MyClassName" --dry-run --json
```

---

## 10. Files: Download and Upload (ContentVersion) 🟢/🟡

Salesforce stores files via `ContentDocument` / `ContentVersion` / `ContentDocumentLink`.

### Step 1: Find files linked to a record

```sql
SELECT ContentDocumentId, ContentDocument.Title,
       ContentDocument.FileType, ContentDocument.FileExtension
FROM ContentDocumentLink
WHERE LinkedEntityId = '<RECORD_ID>'
```

**Batch query** (up to 200 IDs per IN clause, but 10-20 recommended for URL length):

```sql
SELECT ContentDocumentId, LinkedEntityId,
       ContentDocument.Title, ContentDocument.FileExtension
FROM ContentDocumentLink
WHERE LinkedEntityId IN ('id1','id2','id3')
```

### Step 2: Get the latest version

```sql
SELECT Id, Title, FileExtension
FROM ContentVersion
WHERE ContentDocumentId = '<DOC_ID>'
AND IsLatest = true
```

### Step 3: Download the binary 🟢

```
GET /services/data/v62.0/sobjects/ContentVersion/<VERSION_ID>/VersionData
Authorization: Bearer <ACCESS_TOKEN>
```

Python implementation (streams to disk to handle large files):

```python
def download_sf_file(version_id, filepath, instance_url, access_token):
    """Download a file from Salesforce ContentVersion (streamed)."""
    url = f"{instance_url}/services/data/v62.0/sobjects/ContentVersion/{version_id}/VersionData"
    req = urllib.request.Request(url, headers={
        "Authorization": f"Bearer {access_token}"
    })
    try:
        with urllib.request.urlopen(req) as resp:
            # Check for HTML error pages (e.g., expired token returns login page)
            content_type = resp.headers.get("Content-Type", "")
            if "text/html" in content_type:
                raise RuntimeError(
                    "Salesforce returned HTML instead of file data — "
                    "token may be expired or permissions insufficient"
                )
            with open(filepath, "wb") as f:
                while chunk := resp.read(65536):
                    f.write(chunk)
    except urllib.error.HTTPError as e:
        raise RuntimeError(f"File download failed (HTTP {e.code}): {e.read().decode('utf-8', errors='replace')}") from e
    return filepath
```

### Step 4: Upload a file 🟡

**WRITE OPERATION — confirm with user before uploading. Show filename, size, and target record.**

To upload a file to Salesforce, create a `ContentVersion` record. Salesforce automatically creates the parent `ContentDocument`.

```python
import base64, os

def upload_sf_file(filepath, instance_url, access_token, linked_entity_id=None, title=None):
    """Upload a local file to Salesforce as a ContentVersion.
    Optionally link it to a record (Account, Case, etc.) via ContentDocumentLink.
    Returns the new ContentVersion record."""
    filename = os.path.basename(filepath)
    if title is None:
        title = os.path.splitext(filename)[0]

    with open(filepath, "rb") as f:
        file_data = base64.b64encode(f.read()).decode("ascii")

    # Create ContentVersion with base64-encoded body
    cv_data = {
        "Title": title,
        "PathOnClient": filename,
        "VersionData": file_data
    }
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    result = _sf_request(
        f"{instance_url}/services/data/v62.0/sobjects/ContentVersion/",
        headers=headers,
        data=json.dumps(cv_data).encode("utf-8")
    )
    cv_id = result["id"]

    # If a linked_entity_id is provided, link the file to that record
    if linked_entity_id:
        # Get the ContentDocumentId from the newly created ContentVersion
        cv_record = _sf_request(
            f"{instance_url}/services/data/v62.0/sobjects/ContentVersion/{cv_id}",
            headers=headers
        )
        doc_id = cv_record["ContentDocumentId"]

        # Create ContentDocumentLink
        link_data = {
            "ContentDocumentId": doc_id,
            "LinkedEntityId": linked_entity_id,
            "ShareType": "V",  # Viewer
            "Visibility": "AllUsers"
        }
        _sf_request(
            f"{instance_url}/services/data/v62.0/sobjects/ContentDocumentLink/",
            headers=headers,
            data=json.dumps(link_data).encode("utf-8")
        )

    return result
```

**Upload via multipart (large files):** For files > 37.5 MB (base64 overhead on the 50 MB REST API limit), use multipart form upload:

```python
import uuid

def upload_sf_file_multipart(filepath, instance_url, access_token, title=None):
    """Upload a large file using multipart/form-data (up to 2 GB via REST)."""
    filename = os.path.basename(filepath)
    if title is None:
        title = os.path.splitext(filename)[0]

    boundary = uuid.uuid4().hex

    # Build multipart body
    metadata = json.dumps({
        "Title": title,
        "PathOnClient": filename
    })

    with open(filepath, "rb") as f:
        file_content = f.read()

    body = (
        f"--{boundary}\r\n"
        f"Content-Disposition: form-data; name=\"entity_content\"\r\n"
        f"Content-Type: application/json\r\n\r\n"
        f"{metadata}\r\n"
        f"--{boundary}\r\n"
        f"Content-Disposition: form-data; name=\"VersionData\"; filename=\"{filename}\"\r\n"
        f"Content-Type: application/octet-stream\r\n\r\n"
    ).encode("utf-8") + file_content + f"\r\n--{boundary}--\r\n".encode("utf-8")

    req = urllib.request.Request(
        f"{instance_url}/services/data/v62.0/sobjects/ContentVersion/",
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": f"multipart/form-data; boundary={boundary}"
        }
    )
    try:
        with urllib.request.urlopen(req) as resp:
            return json.loads(resp.read())
    except urllib.error.HTTPError as e:
        error_body = e.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"Upload failed (HTTP {e.code}): {error_body}") from e
```

### Organizing downloaded files

- Create subdirectories by Account or record grouping
- Sanitize filenames: `re.sub(r'[<>:"/\\|?*]', '_', name)`
- Skip already-downloaded files to allow resumable downloads
- Log file sizes for verification

---

## 11. Common Patterns

### Query Opportunities with Account info

```sql
SELECT Id, Name, Account.Name, CloseDate, Amount, StageName
FROM Opportunity
WHERE StageName = 'Closed Won'
AND CloseDate >= 2024-01-01
ORDER BY CloseDate DESC
```

### Query Contacts with Account relationship

```sql
SELECT Id, FirstName, LastName, Email, Account.Name
FROM Contact
WHERE Account.Industry = 'Technology'
ORDER BY LastName ASC
```

### Query Cases with related Contact

```sql
SELECT Id, CaseNumber, Subject, Status, Contact.Name, Contact.Email
FROM Case
WHERE Status != 'Closed'
ORDER BY CreatedDate DESC
```

### Batch processing pattern

When querying by ID lists, batch in groups of 10-20 to avoid URL length limits.

**Note:** This pattern is safe for system-generated Salesforce IDs (15/18-char alphanumeric). Do NOT use it for user-supplied string values — that would create a SOQL injection risk.

```python
for batch_start in range(0, len(record_ids), 10):
    batch = record_ids[batch_start:batch_start + 10]
    ids_str = "','".join(batch)
    query = f"SELECT ... FROM ... WHERE Id IN ('{ids_str}')"
    records = sf_query(query, instance_url, token)
```

### Composite requests (multiple operations in one call) 🟡

```python
def sf_composite(requests, instance_url, access_token, all_or_none=False):
    """Execute multiple API operations in a single call (max 25).
    all_or_none=True: rolls back all if any fails (transactional).
    all_or_none=False (default): partial success possible."""
    url = f"{instance_url}/services/data/v62.0/composite"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    body = json.dumps({
        "allOrNone": all_or_none,
        "compositeRequest": requests
    }).encode("utf-8")
    return _sf_request(url, headers=headers, data=body)
```

---

## 12. Troubleshooting

| Issue | Solution |
|-------|----------|
| `NOT_FOUND` on sobject describe | Object API name is wrong. Query `EntityDefinition` to find the correct `QualifiedApiName` |
| `MALFORMED_QUERY` with OR | `EntityDefinition` doesn't support disjunctions. Use separate queries |
| `INVALID_SESSION_ID` | Token expired or IP-locked. Re-run `sf org login web` (preferred), use Manual OAuth (Method 2), or ask user for a new session ID |
| `Bad_OAuth_Token` | Session ID from a different IP. Use Manual OAuth flow (Method 2) instead |
| `REQUEST_LIMIT_EXCEEDED` | Too many API calls. Add delays between batches or reduce batch size |
| URL too long | Reduce batch size for IN clauses (max 10-20 IDs per query) |
| Empty file download | Check that `IsLatest = true` filter is applied on ContentVersion |
| HTML instead of file data | Token expired or insufficient permissions — Salesforce returns a login page. Check `Content-Type` header before writing to disk |
| `pip install` fails on macOS | Use a virtual environment first; fall back to `--break-system-packages` |
| `npm install -g` fails with `EACCES` | Use local prefix: `npm config set prefix "$HOME/.npm-global"` |
| `CannotOpenBrowserError` | No browser available (Cowork/CI). Use Manual OAuth flow (Method 2) |
| `DomainNotFoundError` in sf CLI | DNS resolution bug in Cowork VMs. Use Python REST API calls instead of sf CLI |
| `ENTITY_IS_DELETED` | Record is in Recycle Bin. Use `queryAll` to find deleted records |
| `FIELD_CUSTOM_VALIDATION_EXCEPTION` | Validation rule blocking the operation. Check field values against org rules |
| `UNABLE_TO_LOCK_ROW` | Record lock contention. Retry after a brief delay |
| Bulk job partial failure | Check `numberRecordsFailed` in job result. Retrieve failed-records CSV via `/jobs/ingest/<JOB_ID>/failedResults/` |

### Session expiration

If using web login, the CLI handles token refresh automatically. If using Manual OAuth (Method 2), use the `sf_refresh_token()` function to renew the access token — refresh tokens last for months. If using access token (Method 3, session ID), it expires after **2-12 hours** depending on org settings.

---

## 13. Security Notes

- Prefer `sf org login web` — it uses OAuth and stores refresh tokens securely via the CLI keychain
- **Never** store access tokens in files, scripts, or commit history
- If using access token method, pass via environment variable (`SF_ACCESS_TOKEN`), never inline
- Downloaded files may contain sensitive business data — handle accordingly
- Clean up temporary files after processing
- **Always confirm write operations** with the user before execution (unless `SALESFORCE_SKIP_WARNINGS=true`)
- **Delete operations require explicit user confirmation** — but once the user confirms, proceed with the deletion (unless `SALESFORCE_SKIP_WARNINGS=true`, which skips the confirmation step entirely)
- When executing Apex code, show the full code to the user for review before running
- Be cautious with bulk operations — verify record counts and target object before proceeding

---

## 14. Network Requirements

If working behind a firewall or VPN:

- Ensure `*.salesforce.com` is in the network allowlist
- For Claude Code with Cowork: add `*.salesforce.com` to **Settings → Capabilities → Domain allowlist**
- A new Cowork session may be required after modifying the allowlist
- In Cowork, the sf CLI may fail with `DomainNotFoundError` even with the domain allowlisted — use Python REST API calls as fallback

---

## 15. Quick Reference Card

```
 Salesƒorce CLI — Quick Commands
 ─────────────────────────────────────────
 🟢 READ
   sf org list                          # list connected orgs
   sf org display --json                # show org details + token
   sf data query --query "..." --json   # SOQL query
   sf data query --query "..." --bulk   # bulk query
   sf sobject describe --sobject X      # describe object fields
   sf sobject list --json               # list all objects
   sf apex list log --json              # list debug logs
   sf apex tail log --color             # tail logs live
   sf org list metadata-types --json    # list metadata types
 ─────────────────────────────────────────
 🟡 WRITE (confirm with user first)
   sf data create record --sobject X    # create record
   sf data update record --sobject X    # update record
   sf data upsert bulk --sobject X      # bulk upsert from CSV
   sf data import tree --files X.json   # import related records
   sf apex run --file script.apex       # execute Apex (show code!)
   sf project deploy start              # deploy metadata
 ─────────────────────────────────────────
 🔴 DELETE (require explicit confirmation)
   sf data delete record --sobject X    # delete single record
   sf data delete bulk --sobject X      # bulk delete from CSV
 ─────────────────────────────────────────
```
