# CSBS Backup Compliance Automation on Huawei Cloud Stack

An automated, pipeline-enforced compliance check that asserts the Elastic Cloud Server estate remains bound to its Cloud Backup Service protection policy and that its backups are actually completing, together with the reverse-engineering toolkit that established a supported non-browser transport to the backup API.

---

## Table of Contents

- [Overview](#overview)
- [Real-World Business Value](#real-world-business-value)
- [Skills Demonstrated](#skills-demonstrated)
- [Project Folder Structure](#project-folder-structure)
- [Tasks and Implementation Steps](#tasks-and-implementation-steps)
- [Core Implementation Breakdown](#core-implementation-breakdown)
- [IAM Role and Permissions](#iam-role-and-permissions)
- [Project Features (Detailed Breakdown)](#project-features-detailed-breakdown)
- [Design Decisions and Highlights](#design-decisions-and-highlights)
- [Local Testing and Validation](#local-testing-and-validation)
- [Errors Encountered and Resolved](#errors-encountered-and-resolved)
- [Conclusion](#conclusion)

---

## Overview

Backup protection on this Huawei Cloud Stack tenant is expressed through the Cloud Backup Console at `<console-host>/cbs`. A named Service Level Agreement, in this environment `SLA_ECS_Backup`, defines the backup interval, retention period and backup window. An Elastic Cloud Server becomes subject to that schedule by being registered as an associated resource, also described as a protected object, of the Service Level Agreement. Nothing in the platform prevents that association from being removed, and nothing raises an alarm when a scheduled backup silently stops succeeding.

This component closes that gap. `csbs_backup_check.py` authenticates to the Huawei Cloud Stack Identity and Access Management service using an access key and secret key pair, exchanges those credentials for a scoped token, queries the protected objects of the Service Level Agreement through the registered `csbs-vbs` API fusion route, and asserts three conditions: that the instance is still associated, that a backup has completed at least once, and that the most recent backup is not older than a configurable threshold. When any assertion fails the script posts a synthetic alert to Alertmanager and exits with a non-zero status so that the Azure DevOps pipeline stage fails.

The remaining files in this folder are the diagnostic toolkit that made the check possible. Establishing a supported, non-browser transport to the backup API required systematically eliminating every candidate: DNS resolution, the Keystone service catalogue, the full Keystone registry, host-header routing against the shared API gateway virtual IP address, the ManageOne operations layer, and finally the vendor product documentation. Each probe is committed so that the negative results are evidence rather than recollection.

### Scope boundaries

Included in scope:

- Assertion that a named Elastic Cloud Server remains an associated resource of a named Service Level Agreement.
- Assertion of backup recency against a configurable maximum age.
- Assertion of the compliance flag reported by the backup service.
- Alert delivery to Alertmanager and pipeline failure on drift.
- The reproducible transport and endpoint discovery probes.

Intentionally excluded from scope:

- Automatic remediation. The check is deliberately alert-only. See [Design Decisions and Highlights](#design-decisions-and-highlights).
- Creation of the Service Level Agreement or the initial association, both of which were performed through the console. See `ADR 0002`.
- Backup restoration and recovery testing.
- Environments beyond development. The instance identifiers are currently constants scoped to the development environment.

---

## Real-World Business Value

- **Backup coverage becomes a verified control rather than an assumption.** Prior to this work the only evidence that an instance was protected was a console screen that nobody was scheduled to check. Coverage is now asserted automatically on every pipeline run.
- **Silent backup failure is detected.** The check does not merely confirm that the association exists; it confirms that a backup has completed and that the most recent one falls inside the permitted age window. An association that exists but produces no backups is the failure mode most likely to be discovered during a restore attempt, which is the worst possible time to discover it.
- **Audit evidence is generated as a by-product.** Each pipeline run is a timestamped, attributable record that backup protection was verified. This is directly usable as control evidence in a regulated banking environment.
- **The infrastructure pipeline enforces the control.** The stage depends on the Elastic Cloud Server stage, so a change that removes protection cannot pass through the same pipeline that provisions compute.
- **Dead ends are documented, so investigation cost is paid once.** The probe scripts and their recorded results prevent a future engineer from repeating a multi-day endpoint search that has already been concluded.
- **Quantified impact.** The check covers the development Elastic Cloud Server estate on every run of the development infrastructure pipeline, asserting three independent conditions against a Service Level Agreement that provides a daily backup interval and seven day retention. The permitted backup age is two days, so a single missed run is tolerated and a second consecutive miss raises an alert within one pipeline execution.

---

## Skills Demonstrated

- **Protocol reverse engineering.** Determining the real authentication and transport model of an undocumented, console-internal API by inspecting network traffic, replaying requests outside the browser, and isolating the precise reason each replay failed.
- **Request signing implemented from first principles.** A complete SDK-HMAC-SHA256 canonical request, string to sign and signature implementation written against `hashlib` and `hmac`, without an SDK dependency.
- **Systematic negative-result engineering.** Designing probes whose purpose is to eliminate hypotheses conclusively, and distinguishing an HTTP 404 that means "no such route" from an HTTP 401 or 403 that means "the route exists but authentication failed".
- **Defensive API client design.** Tolerating multiple response envelopes and multiple field-name spellings for the same attribute, paginating safely with an explicit bound, and failing loudly on an unparseable response rather than silently reporting success.
- **Observability integration.** Emitting a synthetic alert directly to the Alertmanager v2 API using a label set consistent with the existing rule taxonomy, so that a scripted finding routes through the same notification path as a Prometheus alert.
- **Architectural decision records.** Capturing superseded decisions rather than deleting them, so that the reasoning behind a reversal remains available.
- **Azure Pipelines integration.** A dependent stage that injects secrets from a variable group and gates on a script exit status.
- **Constrained-environment Python.** All code targets the Python 3.6 standard library available on the CentOS 7 bastion, using `urllib` rather than `requests`.

---

## Project Folder Structure

```text
infra-live/
├── 3-vdc-data/
│   └── 0-dev/
│       └── csbs/
│           ├── csbs_backup_check.py            # The production compliance check
│           ├── csbs_api_probe.py               # Transport probe: DNS, AK/SK, IAM token replay
│           ├── csbs_catalog_probe.py           # Prints the IAM service catalogue
│           ├── csbs_endpoint_discovery.py      # Full Keystone service and endpoint registry sweep
│           ├── csbs_gateway_probe.py           # Host-header routing probe against the gateway VIP
│           ├── csbs_console_entry_probe.py     # Locates the console CAS entry point
│           ├── csbs_hcs_internal_probe.py      # Searches OC, SC and internal DNS layers
│           ├── csbs_ecs_storage_probe.py       # Inspects the Same Storage backup prerequisite
│           └── extract_csbs_docs.py            # Extracts the vendor PDFs to searchable text
├── docs/
│   └── adr/
│       ├── 0001-ak-sk-signing-for-csbs-backup-script.md
│       └── 0002-bind-bastion-to-existing-sla-not-new-policy.md
└── pipelines/
    └── 0-dev.yml                               # Contains the csbs_backup_check stage
```

Purpose of each major folder:

- **`3-vdc-data/0-dev/csbs`** holds the compliance check and its diagnostic toolkit. It sits under the data layer and follows the same per-environment layout as the sibling Data Replication Service integration, so that promotion to a further environment is a directory copy with changed constants rather than a restructure.
- **`docs/adr`** holds the architectural decision records. It exists so that reversed decisions remain visible with their reasoning intact.
- **`pipelines`** holds the Azure Pipelines definitions, keeping delivery logic versioned beside the code it executes.

Key files:

| Artefact | Purpose |
| --- | --- |
| `csbs_backup_check.py` | The production compliance check executed by the pipeline |
| `csbs_api_probe.py` | Establishes which transport and authentication scheme works from the bastion |
| `csbs_catalog_probe.py` | Prints the service catalogue embedded in a scoped token |
| `csbs_endpoint_discovery.py` | Queries the unfiltered Keystone registry and sweeps candidate host names |
| `csbs_gateway_probe.py` | Tests documented CSBS paths against the shared gateway VIP by Host header |
| `csbs_console_entry_probe.py` | Walks console application paths for a server-side CAS entry point |
| `csbs_hcs_internal_probe.py` | Reuses the Operation Centre session to search internal layers |
| `csbs_ecs_storage_probe.py` | Verifies the Same Storage prerequisite for instance-level backup |
| `extract_csbs_docs.py` | Converts the vendor PDFs to text and surfaces endpoint and authentication evidence |
| ADR 0001 | Records the superseded authentication decision |
| ADR 0002 | Records the decision to bind to the existing Service Level Agreement |
| Pipeline definition | Contains the `csbs_backup_check` stage |

---

## Tasks and Implementation Steps

1. **Established the actual backup object model.**
   The original plan assumed Cloud Server Backup Service version 1 semantics, in which a script creates a backup policy consisting of a `policies` object and a set of `scheduled_operations`, then automatically binds resources by name filter. Inspection of the live Cloud Backup Console showed that this tenant runs the Cloud Backup and Recovery generation of the service instead, in which the real object is a Service Level Agreement and instances become associated resources of it. There is no `policies` or `scheduled_operations` object model to write against. Artefact: `ADR 0002`.

2. **Chose to bind to the existing Service Level Agreement rather than create a new one.**
   `SLA_ECS_Backup` already existed with a Bronze tier policy providing a daily interval, seven day retention and a backup window between 22:00 and 06:00. Creating a parallel policy would have produced two competing schedules on the same estate. Artefact: `ADR 0002`.

3. **Determined the authentication model of the backup API.**
   Browser traffic showed that `/cbs/op/rest/v1/*` is a console-internal API authenticated by session cookies and custom tokens rather than by access key signing. The initial decision to sign requests with an access key and secret key pair was therefore superseded in favour of replicating the ManageOne single sign-on flow already reverse-engineered for the Operation Centre exporter. Artefact: `ADR 0001`.

4. **Proved that the console session approach could not work from the pipeline agent.**
   Console session cookies proved to be bound to the originating address: an identical request that returned HTTP 200 from a workstation returned HTTP 412 from the bastion. The console also has no server-side `unisess` route, so its login is entirely client-side JavaScript and cannot be replayed. Artefacts: `csbs_api_probe.py` and `csbs_console_entry_probe.py`.

5. **Conducted an exhaustive search for a northbound backup API.**
   The service catalogue embedded in a scoped token is filtered by role, so the unfiltered Keystone registry was queried directly. It returned eighteen services and twenty-five endpoints, none of them a backup service. A DNS sweep of thirteen candidate names across two domain suffixes resolved nothing. A Host-header probe against the shared API gateway virtual IP address, carrying a valid token, returned HTTP 404 for every candidate host and path combination, which is a routing answer rather than an authentication answer. Artefacts: `csbs_catalog_probe.py`, `csbs_endpoint_discovery.py`, `csbs_gateway_probe.py` and `csbs_hcs_internal_probe.py`.

6. **Used the vendor documentation to turn the finding into a precise request to the platform provider.**
   The product documentation confirms that a northbound API exists in the product and is addressed as `csbs.<region0_id>.<external_global_domain_name>`, with both values sourced from a deployment export file held only by the platform administrator. This converted an open-ended investigation into a single, answerable question for the vendor. Artefact: `extract_csbs_docs.py`.

7. **Eliminated the Same Storage capability as the cause of non-compliance.**
   Instance-level backup requires all volumes of an instance to reside on one storage backend. The Elastic Cloud Server API is present in the service catalogue and is therefore reachable with access key signing, so it was queried directly to confirm that the instance already satisfies the prerequisite. Artefact: `csbs_ecs_storage_probe.py`.

8. **Implemented the compliance check against the registered API fusion route.**
   Once the platform provider registered a `csbs-vbs` API fusion service on the gateway, the check was written to authenticate with an access key and secret key pair, exchange them for a scoped token, verify that the fusion service is registered before use, and query the protected objects of the Service Level Agreement. Artefact: `csbs_backup_check.py`.

9. **Restricted the check to assertion and alerting.**
   Automatic re-association was explicitly rejected, because a missing binding may be a deliberate decommissioning and repairing it automatically would convert an intentional change into a hidden one.

10. **Integrated the check as a pipeline stage.**
    A `csbs_backup_check` stage was added to the development infrastructure pipeline with a dependency on the Elastic Cloud Server stage, so that protection is verified after compute is provisioned. Artefact: `0-dev.yml`.

---

## Core Implementation Breakdown

### Execution flow

```text
Azure DevOps pipeline stage  csbs_backup_check   (dependsOn: ecs)
   │  injects HCS_ACCESS_KEY, HCS_SECRET_KEY, CSBS_API_GATEWAY_HOST
   ▼
csbs_backup_check.py
   │
   ├─ get_iam_token()        AK/SK, SDK-HMAC-SHA256 signed  ──► IAM  /v3/auth/tokens
   │                                                             └─► X-Subject-Token
   ├─ discover_service()     asserts the csbs-vbs fusion route is registered
   ├─ check_association()    GET /protected-objects?sla_id=...  (paginated)
   ├─ evaluate()             associated?  ever backed up?  recent enough?  compliant?
   │
   ├─ pass ─► log and exit 0
   └─ fail ─► raise_alert()  POST /api/v2/alerts ──► Alertmanager ──► Teams and email
              die()          exit 1 ──────────────► pipeline stage fails
```

### Authenticating with an access key and secret key pair

The Identity and Access Management gateway expects an SDK-HMAC-SHA256 signature. The implementation builds the canonical request, derives the string to sign, and assembles the `Authorization` header without an SDK dependency.

```python
def _signed_headers(method, host, path, query="", body=b""):
    """Create the SDK-HMAC-SHA256 headers used by the HCS IAM gateway."""
    canonical_path = path if path.endswith("/") else path + "/"
    date_str = datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")
    headers = {"Host": host, "X-Project-Id": PROJECT_ID, "X-Sdk-Date": date_str}
    signed_keys = sorted(key.lower() for key in headers)
    canonical_headers = "".join(
        "{}:{}\n".format(key.lower(), headers[key].strip())
        for key in sorted(headers, key=str.lower)
    )
    canonical_request = "{}\n{}\n{}\n{}\n{}\n{}".format(
        method, canonical_path, query, canonical_headers,
        ";".join(signed_keys), hashlib.sha256(body).hexdigest(),
    )
    string_to_sign = "SDK-HMAC-SHA256\n{}\n{}".format(
        date_str, hashlib.sha256(canonical_request.encode()).hexdigest()
    )
    signature = hmac.new(
        SECRET_KEY.encode(), string_to_sign.encode(), hashlib.sha256
    ).hexdigest()
    headers["Authorization"] = (
        "SDK-HMAC-SHA256 Access={}, SignedHeaders={}, Signature={}".format(
            ACCESS_KEY, ";".join(signed_keys), signature
        )
    )
    return headers
```

The signed request is exchanged for a project-scoped token. The token is returned in a response header rather than in the body, and is cached for the lifetime of the process.

```python
def get_iam_token():
    global IAM_TOKEN
    if IAM_TOKEN:
        return IAM_TOKEN
    path = "/v3/auth/tokens"
    url = "https://{}{}".format(IAM_HOST, path)
    payload = {
        "auth": {
            "identity": {
                "methods": ["hw_ak_sk"],
                "hw_ak_sk": {"access": {"key": ACCESS_KEY}, "secret": {"key": SECRET_KEY}},
            },
            "scope": {"project": {"id": PROJECT_ID}},
        }
    }
    body = json.dumps(payload).encode("utf-8")
    headers = _signed_headers("POST", IAM_HOST, path, body=body)
    headers["Content-Type"] = "application/json"
    status, response_body, response_headers = _json_request("POST", url, headers, payload)
    token = response_headers.get("X-Subject-Token")
    if status != 201 or not token:
        die("IAM token request failed with HTTP {}: {}".format(status, response_body[:500]))
    IAM_TOKEN = token
    return IAM_TOKEN
```

### Verifying the transport before relying on it

The backup API is reached through an API fusion route registered on the gateway by the platform provider. Because that registration is outside this team's control, the script confirms it exists before issuing any business query. A missing route therefore produces a clear message about the route rather than a confusing failure inside the compliance logic.

```python
def discover_service():
    """Verify that Huawei's registered CSBS-VBS API-fusion service is visible."""
    require_gateway_host()
    url = _api_url("", {"service_name": SERVICE_NAME})
    status, body, _ = _json_request(
        "GET", url, {"Accept": "application/json", "X-Auth-Token": get_iam_token()}
    )
    if status < 200 or status >= 300:
        die("CSBS service discovery failed with HTTP {}: {}".format(status, body[:500]))
    if "sourceUrl" not in body and "source_url" not in body:
        die("CSBS service '{}' returned no sourceUrl: {}".format(SERVICE_NAME, body[:500]))
    log.info("CSBS service '%s' is registered", SERVICE_NAME)
```

### Querying protected objects defensively

The response envelope and the field spellings vary across the endpoints of this platform, so both are normalised. Two small helpers absorb that variance rather than scattering it through the logic.

```python
def _first_present(dct, keys, default=None):
    """Return the first present non-empty value from keys in a dict."""
    for key in keys:
        if key in dct and dct.get(key) not in (None, ""):
            return dct.get(key)
    return default


def _extract_items(data):
    """Extract item list across known response envelopes."""
    for key in ("items", "records", "data", "resources"):
        value = data.get(key)
        if isinstance(value, list):
            return value
    return []
```

Pagination is bounded explicitly and terminates on any of three conditions: a match, a reported total page count, or a short page. When no match is found the first ten entries observed are logged, so that a failure is diagnosable from the pipeline log alone without a second run.

```python
def check_association():
    """Return the protected-object entry for RESOURCE_ID, or None."""
    page_size = 100
    page_no = 0
    max_pages = 50
    seen = []

    while page_no < max_pages:
        data = cbs_get(
            "/protected-objects",
            params={"sla_id": SLA_ID, "page_size": page_size, "page_no": page_no},
        )
        items = _extract_items(data)

        for item in items:
            rid = _first_present(item, ["resource_id", "resourceId", "id"], "")
            rname = _first_present(item, ["resource_name", "name", "serverName"], "")
            if len(seen) < 10:
                seen.append("{}:{}".format(rid, rname))
            if rid == RESOURCE_ID or rname == RESOURCE_NAME:
                return item

        total_pages = _first_present(data, ["total_pages", "totalPages"], None)
        if total_pages is not None and page_no + 1 >= int(total_pages):
            break
        if len(items) < page_size:
            break
        page_no += 1

    log.warning(
        "No matching resource found in protected objects; sampled entries: %s",
        ", ".join(seen) if seen else "<none>",
    )
    return None
```

### The compliance assertions

Four conditions are evaluated in order of severity, and each returns a human-readable reason that becomes the alert summary.

```python
def evaluate(entry):
    """Return (ok, reason) for the protected-object entry."""
    if entry is None:
        return False, "{} is no longer an associated resource of {}".format(
            RESOURCE_NAME, SLA_NAME)
    latest_time = _first_present(entry, ["latest_time", "latestTime"])
    if not latest_time:
        return False, "{} has never completed a backup under {}".format(
            RESOURCE_NAME, SLA_NAME)
    try:
        latest_ts = time.mktime(time.strptime(latest_time[:19], "%Y-%m-%dT%H:%M:%S"))
    except ValueError:
        return False, "Could not parse latest_time '{}'".format(latest_time)
    age = time.time() - latest_ts
    if age > MAX_BACKUP_AGE_SECONDS:
        return False, "{}'s last backup was {:.0f}h ago (limit {:.0f}h)".format(
            RESOURCE_NAME, age / 3600, MAX_BACKUP_AGE_SECONDS / 3600)
    sla_compliance = _first_present(entry, ["sla_compliance", "slaCompliance"], True)
    if sla_compliance is False:
        return False, "{} reports sla_compliance=false".format(RESOURCE_NAME)
    return True, "OK"
```

The default maximum backup age is two days, which is twice the daily backup interval of the Bronze policy. This tolerates one missed run before alerting and is overridable through `CSBS_MAX_BACKUP_AGE_SECONDS`.

### Alerting and failing the stage

Both actions are performed, not one or the other. The alert reaches the on-call channel immediately, and the non-zero exit status makes the finding visible in the pipeline history.

```python
def raise_alert(reason):
    body = [
        {
            "labels": {
                "alertname": ALERTNAME,
                "instance": RESOURCE_NAME,
                "sla": SLA_NAME,
                "severity": "warning",
            },
            "annotations": {"summary": reason},
        }
    ]
    req = Request(
        "{}/api/v2/alerts".format(ALERTMANAGER_URL),
        data=json.dumps(body).encode("utf-8"),
        method="POST",
    )
    req.add_header("Content-Type", "application/json")
    try:
        urlopen(req, context=ssl_ctx, timeout=15).read()
        log.info("Posted alert to Alertmanager: %s", reason)
    except Exception as e:
        log.error("Failed to post alert to Alertmanager: %s", e)


def main():
    discover_service()
    entry = check_association()
    ok, reason = evaluate(entry)
    if ok:
        log.info("%s is compliant under %s", RESOURCE_NAME, SLA_NAME)
        return
    raise_alert(reason)
    die(reason)
```

Alert delivery failure is logged but does not mask the finding, because `die(reason)` runs regardless and the stage still fails.

### Pipeline stage

```yaml
  # CSBS backup compliance check (asserts <bastion-host> stays associated
  # to SLA_ECS_Backup; does not create the association -- see
  # infra-live/docs/adr/0002-bind-bastion-to-existing-sla-not-new-policy.md)
  - stage: csbs_backup_check
    displayName: "CSBS Backup Compliance Check"
    dependsOn:
      - ecs
    jobs:
      - job: check_compliance
        pool: <agent-pool-name>
        steps:
          - checkout: self
          - script: python3 3-vdc-data/0-dev/csbs/csbs_backup_check.py
            displayName: "Check <bastion-host> backup compliance"
            workingDirectory: "$(Build.SourcesDirectory)"
            env:
              HCS_ACCESS_KEY: $(HCS_ACCESS_KEY)
              HCS_SECRET_KEY: $(HCS_SECRET_KEY)
              CSBS_API_GATEWAY_HOST: $(api_gateway_float_ip)
              CSBS_API_SERVICE_NAME: csbs-vbs
```

### Configuration reference

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `HCS_ACCESS_KEY` | Yes | None | Access key used for request signing and token exchange |
| `HCS_SECRET_KEY` | Yes | None | Secret key used for request signing and token exchange |
| `CSBS_API_GATEWAY_HOST` | Yes | Empty | The API gateway floating address provided by the platform team |
| `HCS_PROJECT_ID` | No | `<project-id>` | Project scope for the token request |
| `HCS_IAM_HOST` | No | `<iam-host>` | Identity service endpoint |
| `CSBS_API_FUSION_PREFIX` | No | `/v1/apigw/extension/api-fusion/apis` | Gateway path prefix for the fusion route |
| `CSBS_API_SERVICE_NAME` | No | `csbs-vbs` | Registered fusion service name |
| `CSBS_SOURCE_PREFIX` | No | `/console/rest/v1` | Path prefix of the backup API behind the fusion route |
| `CSBS_MAX_BACKUP_AGE_SECONDS` | No | `172800` | Maximum permitted age of the most recent successful backup |
| `ALERTMANAGER_URL` | No | `http://<bastion-ip>:9093` | Alertmanager endpoint for the synthetic alert |

---

## IAM Role and Permissions

### Identity model

The check runs under the same access key and secret key pair used by the other Huawei Cloud Stack integrations on the bastion, scoped to a single project. The token request is project-scoped rather than domain-scoped, so the credential cannot be used to enumerate or act on resources outside the development project.

| Permission | Why it is required |
| --- | --- |
| Identity token issuance for the project | Required to exchange the access key and secret key pair for the scoped token that the gateway accepts |
| Read access to the backup service protected objects | Required to determine whether the instance is associated with the Service Level Agreement and when it was last backed up |
| Read access to the Elastic Cloud Server API | Required only by `csbs_ecs_storage_probe.py` to confirm the Same Storage prerequisite; not required by the compliance check |
| Network access to Alertmanager on port 9093 | Required to post the synthetic alert |

No write scope of any kind is requested or used. The check issues only HTTP GET requests against the backup service. The one HTTP POST it performs is to Alertmanager, which is an internal observability component rather than a cloud control plane.

The `CSBS Administrator` and `CSBS Readonly` roles exist in the tenant role catalogue and are recorded in the role-based access control inventory at `0-vdc-guardrails`. Where a dedicated identity is used for this check, `CSBS Readonly` is the correct assignment, because the check never modifies backup configuration.

### Secret handling

- No credential is committed to the repository. `HCS_ACCESS_KEY` and `HCS_SECRET_KEY` are declared as secret variables in the Azure DevOps variable group `<secrets-variable-group>` and are injected into the step environment at execution time, where they are masked in the pipeline log.
- The script reads credentials exclusively from environment variables. There is no configuration file, no command-line credential argument, and no fallback to a hard-coded default.
- Every probe script states in its docstring that it prints no credentials, and `csbs_console_entry_probe.py` prints cookie names only, never cookie values, so that captured output can be pasted into a ticket safely.
- Error paths truncate response bodies to five hundred characters before logging, which limits the volume of unexpected upstream content that reaches the log.
- Certificate verification is disabled against the internal endpoints, consistent with every other integration in this repository, because the private cloud presents an internally signed certificate chain. This is a documented environment constraint rather than a per-script choice. See [Design Decisions and Highlights](#design-decisions-and-highlights).

```python
ssl_ctx = ssl.create_default_context()
ssl_ctx.check_hostname = False
ssl_ctx.verify_mode = ssl.CERT_NONE
```

---

## Project Features (Detailed Breakdown)

### 1. Association drift detection

**Purpose.** Detect the removal of an instance from its backup Service Level Agreement.

**Implementation.** `check_association()` pages through the protected objects of the Service Level Agreement and matches on either the resource identifier or the resource name, so that a change to one attribute does not produce a false negative.

**Operational considerations.** When no match is found, up to ten observed entries are logged as identifier and name pairs. This distinguishes the case where the instance is genuinely absent from the case where the response envelope changed shape, without requiring a second diagnostic run.

**Artefact.** `csbs_backup_check.py`.

### 2. Backup recency assertion

**Purpose.** Detect the case where the association is intact but backups have stopped succeeding.

**Implementation.** The most recent backup timestamp is parsed from the protected-object entry and compared against `CSBS_MAX_BACKUP_AGE_SECONDS`, which defaults to two days against a daily backup policy.

**Operational considerations.** An entry with no recorded backup timestamp fails immediately with a distinct message, because an instance that has never been backed up is a different and more serious condition than one whose backups have lapsed. An unparseable timestamp also fails rather than being treated as acceptable, so a change to the upstream date format cannot silently disable the control.

### 3. Compliance flag assertion

**Purpose.** Surface the backup service's own compliance verdict.

**Implementation.** The `sla_compliance` field is checked explicitly against `False`. The default when the field is absent is `True`, so an unevaluated instance does not generate a false alarm; the recency assertion already covers the case where nothing is being backed up.

**Operational considerations.** The flag is governed by the evaluation rules of the Service Level Agreement itself, which can report a value inconsistent with a manually verified successful backup. It is therefore treated as one signal among three rather than as the sole verdict.

### 4. Synthetic alert emission

**Purpose.** Route a scripted finding through the same notification path as a Prometheus alert.

**Implementation.** A single alert is posted to the Alertmanager v2 API with `alertname` set to `CsbsBackupComplianceFailed` and labels for the instance and the Service Level Agreement, matching the label conventions of the existing rule set so that the notification renders identically to a rule-generated alert in Microsoft Teams and email.

**Operational considerations.** Posting directly to the Alertmanager API was chosen over invoking `amtool` over SSH, because the latter would require the pipeline agent to hold an SSH credential purely to raise an alert. Alert delivery failure is logged and does not suppress the non-zero exit status.

### 5. Transport and endpoint discovery toolkit

**Purpose.** Establish, and permanently record, which transports into the backup API are viable.

**Implementation.** Seven read-only probes each test one hypothesis:

| Probe | Hypothesis tested |
| --- | --- |
| `csbs_api_probe.py` | Candidate backup endpoints resolve, accept signed requests, or accept a replayed identity token |
| `csbs_catalog_probe.py` | A backup service appears in the catalogue embedded in a scoped token |
| `csbs_endpoint_discovery.py` | A backup service exists in the unfiltered Keystone registry, or resolves by name |
| `csbs_gateway_probe.py` | A backup route exists on the shared gateway under a Host header, despite absent DNS |
| `csbs_console_entry_probe.py` | The console exposes a server-side authentication entry point that can be replayed |
| `csbs_hcs_internal_probe.py` | The operations layers or the internal domain expose a backup resource type |
| `csbs_ecs_storage_probe.py` | The Same Storage prerequisite explains the observed non-compliance |

The gateway probe is explicit about how to read its output, which is the distinction that makes the result conclusive:

```text
Distinguishing outcomes:
  200/403/401 -> the route EXISTS (auth/permission issue only)
  404         -> no such vhost/route on the gateway
```

**Operational considerations.** Every probe is read-only and prints no credentials. They are safe to re-run when the platform provider changes a registration, which converts them from single-use investigation scripts into a permanent regression harness for the transport.

### 6. Vendor documentation extraction

**Purpose.** Determine whether the absent API is a product limitation or a deployment configuration gap.

**Implementation.** `extract_csbs_docs.py` converts the two vendor PDFs, together more than five thousand pages, to plain text and searches for the signals that would prove or disprove a callable northbound API.

**Operational considerations.** The extraction requires PyMuPDF and writes its output outside the repository, so the large derived text files are never committed. The finding it produced, that the endpoint is composed from two deployment-specific values held only by the platform administrator, is what turned an open investigation into a single answerable request.

---

## Design Decisions and Highlights

**The check asserts and alerts; it never repairs.** The alternative was to have the script call the association endpoint to restore a missing binding. That was rejected because a removed association may be a deliberate decommissioning, and automatically restoring it would both undo an intended change and hide the fact that it ever occurred. The trade-off accepted is that a genuine accidental unbinding requires a human action to correct, which is the correct cost for preserving the integrity of the signal.

**Binding to the existing Service Level Agreement was preferred to creating a new one.** Creating a parallel policy would have produced two schedules acting on the same estate with different retention periods, which is a source of both cost and confusion. Reusing `SLA_ECS_Backup` keeps a single authoritative schedule.

**The authentication decision was reversed twice, and both reversals are recorded.** The original choice of access key signing was superseded when the console API proved to be session-authenticated, and the session approach was in turn abandoned when console sessions proved to be address-bound and the console login proved to be entirely client-side. The final implementation returned to access key signing once the platform provider registered a gateway route that accepts a scoped token. `ADR 0001` is marked superseded rather than deleted, so the reasoning behind the reversal remains available to whoever next encounters this API.

**Replaying browser session cookies was rejected outright.** It would have produced a check that worked once and then failed silently on cookie expiry, and it would have required a human credential to be captured and stored. A check that cannot be trusted to fail correctly is worse than no check.

**The transport is verified before the business logic runs.** `discover_service()` exists because the gateway registration is controlled by the platform provider rather than by this team. Asserting it separately means that a deregistration produces a message naming the route, rather than an opaque failure inside the compliance evaluation that would be misread as a backup problem.

**Response parsing is deliberately tolerant, but timestamp parsing is deliberately strict.** Field-name and envelope variance is absorbed, because it is common across this platform's endpoints and carries no risk. An unparseable timestamp, by contrast, fails the check, because treating it as acceptable would allow an upstream format change to silently disable the control. Tolerance is applied where it prevents false alarms and withheld where it would cause false assurance.

**Both alerting and stage failure occur, rather than one or the other.** The alert reaches the on-call channel within seconds and the failed stage creates a durable, auditable record. Choosing only the alert would leave no pipeline evidence; choosing only the failure would delay notification until somebody read the build history.

**Failures exit with a message, never with a partial result.** The `die()` helper is used at every point where a response cannot be trusted, including a non-2xx status, unparseable JSON, and an error object embedded in a 2xx body. A compliance check that reports success because it could not read the answer is the most dangerous possible outcome.

**Only the Python 3.6 standard library is used.** The bastion runs CentOS 7 with dead package repositories, so a dependency on `requests` or a vendor SDK would have introduced an installation problem on the very host the pipeline agent runs on. Using `urllib`, `hmac` and `hashlib` keeps the script deployable anywhere in the estate without preparation.

**Certificate verification is disabled, consistently and deliberately.** The private cloud presents an internally signed certificate chain and every integration in this repository makes the same choice. Hardening this one script in isolation would create an inconsistency without improving the security posture. The correct remedy is to distribute the internal certificate authority bundle across the estate and enable verification everywhere at once, which is an estate-wide change rather than a change to this component.

**Negative results were committed as code.** Seven probe scripts that all returned "no" are more valuable in the repository than a summary paragraph, because they are re-runnable. When the platform provider changes a registration, the same scripts confirm the change rather than requiring the investigation to be reconstructed.

---

## Local Testing and Validation

### Prerequisites

- Python 3.6 or later. The bastion `<bastion-host>` at `<bastion-ip>` satisfies this.
- An access key and secret key pair with read access to the development project.
- Network access from the execution host to the Identity service, the API gateway and Alertmanager.
- PyMuPDF, required only by `extract_csbs_docs.py`.

### Syntax validation

```bash
python3 -m py_compile 3-vdc-data/0-dev/csbs/*.py
```

Expected outcome: no output and a zero exit status.

### Running the compliance check

Credentials are sourced from the existing secrets file on the bastion rather than typed on the command line, so that they do not enter the shell history.

```bash
ssh <bastion-host>
set -a; . /etc/drs-exporter/secrets.env; set +a
export CSBS_API_GATEWAY_HOST='<api_gateway_float_ip>'
python3 3-vdc-data/0-dev/csbs/csbs_backup_check.py
echo "exit status: $?"
```

Expected outcome on a compliant estate:

```text
2026-08-31 09:14:02,118 INFO CSBS service 'csbs-vbs' is registered
2026-08-31 09:14:03,402 INFO <bastion-host> is compliant under SLA_ECS_Backup
exit status: 0
```

Expected outcome when the association has been removed:

```text
2026-08-31 09:16:41,905 INFO CSBS service 'csbs-vbs' is registered
2026-08-31 09:16:42,733 WARNING No matching resource found in protected objects; sampled entries: 9f2c...:ecs-test0527
2026-08-31 09:16:42,810 INFO Posted alert to Alertmanager: <bastion-host> is no longer an associated resource of SLA_ECS_Backup
2026-08-31 09:16:42,811 ERROR <bastion-host> is no longer an associated resource of SLA_ECS_Backup
exit status: 1
```

### Exercising the failure path without breaking backups

Setting the maximum permitted backup age to a value smaller than the real backup age forces the recency assertion to fail. This validates alert delivery and the exit status end to end without altering any backup configuration.

```bash
CSBS_MAX_BACKUP_AGE_SECONDS=1 python3 3-vdc-data/0-dev/csbs/csbs_backup_check.py
echo "exit status: $?"   # expect 1
```

### Confirming the alert reached Alertmanager

```bash
curl -s http://<bastion-ip>:9093/api/v2/alerts \
  | jq -r '.[] | select(.labels.alertname=="CsbsBackupComplianceFailed")
           | "\(.labels.instance) \(.labels.sla) \(.annotations.summary)"'
```

Expected outcome: one line naming the instance, the Service Level Agreement and the failure reason. The alert also appears in the Microsoft Teams channel through the standard Alertmanager routing.

### Re-running the transport probes

```bash
set -a; . /etc/drs-exporter/secrets.env; set +a
python3 3-vdc-data/0-dev/csbs/csbs_catalog_probe.py
python3 3-vdc-data/0-dev/csbs/csbs_endpoint_discovery.py
python3 3-vdc-data/0-dev/csbs/csbs_gateway_probe.py
```

Expected outcome at the time of the recorded investigation: the catalogue lists eighteen services and no backup service; the registry returns eighteen services and twenty-five endpoints with no backup match; the gateway returns HTTP 404 for every candidate host and path combination. A change in any of these results indicates that the platform provider has altered a registration and that the check's configuration should be revisited.

### Extracting the vendor documentation

```bash
pip3 install --user PyMuPDF
python3 3-vdc-data/0-dev/csbs/extract_csbs_docs.py
```

Expected outcome: plain-text dumps written under `/tmp/csbs-docs/` and the matching endpoint and authentication evidence printed to standard output.

---

## Errors Encountered and Resolved

| Symptom | Root cause | Fix | Preventative measure |
| --- | --- | --- | --- |
| The planned script had no policy object to create | The tenant runs the Cloud Backup and Recovery generation of the service, not Cloud Server Backup Service version 1, so there is no `policies` and `scheduled_operations` model; the real object is a Service Level Agreement with associated resources | The scope was changed from creating a policy to asserting an association against the existing `SLA_ECS_Backup` | The object model was verified against the live console before any further code was written, and recorded in `ADR 0002` |
| Access key signed requests to the backup API returned the login page rather than data | The `/cbs/op/rest/v1/*` endpoints are console-internal and authenticated by session cookies and custom tokens, not by access key signature | The authentication approach was changed, and `ADR 0001` was marked superseded with the evidence | Authentication schemes are now confirmed by replaying real traffic before an implementation is committed |
| A request copied verbatim from a working browser session returned HTTP 200 from a workstation and HTTP 412 from the bastion | Console session cookies are bound to the originating source address | The console-session approach was abandoned as unviable for a pipeline agent | Any transport that depends on a captured browser session is rejected on principle for automated checks |
| The console single sign-on flow could not be replicated from a script | Unlike the Operation Centre host, the console exposes no server-side `unisess` route; both `/unisess/v1/auth` and `/unisess/v1/auth/session` return HTTP 404 with `no route found with those values`, because the login is entirely client-side JavaScript | Attempts to replicate the flow were stopped and the finding was recorded | `csbs_console_entry_probe.py` is committed so the conclusion is reproducible rather than remembered |
| No backup service could be found in the service catalogue | The catalogue embedded in a scoped token is filtered by role and scope, so absence there is not proof of absence | The unfiltered Keystone registry was queried directly, returning eighteen services and twenty-five endpoints with no backup entry | `csbs_endpoint_discovery.py` queries the registry rather than the token catalogue |
| Probing the console virtual IP address produced HTTP 200 for every candidate host name, appearing to confirm many routes | `<console-vip>` is a catch-all console listener and answers any Host header, so its responses carry no information | Probing was moved to the API gateway virtual IP address `<gateway-vip>`, where HTTP 404 for every candidate proved the routes genuinely absent | The gateway probe documents how to interpret each status code, so a catch-all responder is not mistaken for a discovery |
| The dedicated service account `svc-csbs-backup-dev` returned `User's status is abnormal` on the backup APIs despite a successful single sign-on | The ManageOne console session bootstrap does not complete for that account type | A working account was used temporarily and the limitation was documented | The account is defined in `policy as code` with a comment explaining why its authentication type differs from the other service accounts |
| The instance reported `sla_compliance: false` even after a manually triggered backup succeeded | The Same Storage prerequisite was suspected, but investigation confirmed the instance already satisfies it with a single volume on one backend; compliance is governed by the evaluation rules of the Service Level Agreement itself | Same Storage was eliminated as a cause and the compliance flag was demoted to one signal among three | The check asserts association and recency independently, so it does not depend solely on a flag whose evaluation semantics are outside this team's control |

---

## Conclusion

This work converts backup protection from an assumption into an enforced, auditable control. A single script, executed by the infrastructure pipeline immediately after compute is provisioned, asserts that the estate remains bound to its backup Service Level Agreement, that backups have actually run, and that the most recent one is recent enough to matter. Failure produces both an alert on the on-call channel and a failed pipeline stage, so the finding is neither missed nor forgotten.

The greater part of the effort, however, was establishing that the check could be built at all. Determining that the backup API existed in the product but was not exposed in this deployment required eliminating six distinct hypotheses across DNS, the token catalogue, the Keystone registry, gateway host routing, the operations layer and the console session model, and then reading the vendor documentation closely enough to identify precisely which two deployment values the platform provider needed to supply. Committing those negative results as re-runnable probes, and recording the reversed authentication decision rather than quietly deleting it, means the next engineer inherits the conclusions instead of the search.

The judgement calls are what the implementation demonstrates most clearly: refusing to automatically repair a missing association because that would hide a deliberate change; refusing to replay a browser session because a check that fails silently is worse than no check; being tolerant of response-shape variance but strict about timestamps, because one prevents false alarms and the other prevents false assurance.

The design anticipates its own extension. Coverage widens from a single instance to the full estate by replacing the `RESOURCE_ID` and `RESOURCE_NAME` constants with an iteration over the discovered inventory, and promotion to a further environment is a directory copy with a changed Service Level Agreement identifier, because the folder layout already mirrors the per-environment structure used elsewhere in the data layer. The deliberate omission of automatic remediation leaves that capability available should a guarded form of it ever be justified, without any part of the current implementation needing to be undone.
