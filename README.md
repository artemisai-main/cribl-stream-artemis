# Artemis AI-Native SIEM (Cribl Pack)

----

## About this Pack

This Pack ships an out-of-the-box bidirectional integration between **Cribl Stream** and the **Artemis AI-Native SIEM**:

* **Cribl Stream → Artemis** — a pre-configured Amazon S3 destination (`out_artemis_s3`) writes events into the Artemis-managed ingestion bucket using cross-account `sts:AssumeRole` + `ExternalId`. No long-lived AWS credentials live in your Worker Group.
* **Artemis → Cribl Stream** — a pre-configured Cribl HTTP source (`in_artemis_cases`) accepts Artemis case NDJSON authenticated by a Bearer token, and runs each case through the `artemis_case_fanout` pipeline so you can route them into the SOC tooling you already operate (Slack, ServiceNow, Splunk, Datadog, …).
* **Reference pipelines** for **Okta**, **AWS CloudTrail**, and **Syslog** show the minimum hygiene required to send those data types to Artemis. Artemis itself does the heavy normalization once data lands in S3 — the Pack's pipelines stay deliberately small.

The Pack mirrors the on-disk shape Artemis already supports through its in-product **Cribl Stream connector** (`cribl_stream` sourcetype). After installing the Pack and pasting in the four values Artemis hands you at integration setup, customer onboarding becomes a one-screen task.

## Deployment

### What you need from Artemis

When you provision the Cribl Stream connector inside Artemis, the portal generates four values that map directly to this Pack's variables:

| Artemis value                   | Pack variable (`default/vars.yml`) |
|---------------------------------|------------------------------------|
| AWS account ID                  | `artemis_aws_account_id`           |
| Cross-account IAM role ARN      | `artemis_iam_role_arn`             |
| `ExternalId`                    | `artemis_external_id`              |
| S3 bucket                       | `artemis_s3_bucket`                |
| AWS region                      | `artemis_aws_region`               |
| S3 key prefix (per integration) | `artemis_key_prefix`               |
| Case webhook bearer token       | `artemis_case_webhook_token`       |

### 1. Install the Pack

* Cribl Stream UI → **Packs** → **Add Pack** → upload the `.crbl` file (or install from the [Cribl Packs Dispensary](https://packs.cribl.io)).
* Open the Pack and go to **Knowledge** → **Variables**. Paste each of the values above into the matching variable.

### 2. Configure outbound (Stream → Artemis S3)

The S3 destination `out_artemis_s3` ships with literal placeholder strings that the customer overwrites with the values Artemis supplied. Cribl's S3 destination schema does not evaluate `C.vars.*` expressions in the `bucket`, `region`, `keyPrefix`, `assumeRoleArn`, or `externalId` fields at runtime — they must be set as literal strings on the destination panel itself. The `default/vars.yml` entries are kept for documentation and reference; they do not drive the destination.

Open `out_artemis_s3` and replace each placeholder:

* `Bucket`: paste `artemis_s3_bucket` (replaces `REPLACE_WITH_ARTEMIS_S3_BUCKET`).
* `Region`: leave at `us-east-1` unless Artemis instructed otherwise.
* `Key Prefix`: paste `artemis_key_prefix` (replaces `REPLACE_WITH_ARTEMIS_KEY_PREFIX`).
* `AssumeRole ARN`: paste `artemis_iam_role_arn` (replaces `arn:aws:iam::YOUR_ARTEMIS_AWS_ACCOUNT_ID:role/...`).
* `External ID`: paste `artemis_external_id` (replaces `REPLACE_WITH_ARTEMIS_EXTERNAL_ID`).

Other settings already preconfigured: `format = json`, `compress = gzip`, server-side encryption enabled (`AES256`), partitioned by `YYYY/MM/DD/HH`.

Save and **Commit & Deploy**.

The default Routes wire **Okta**, **CloudTrail**, and **Syslog** through their respective pipelines and out to `out_artemis_s3`. Adjust the route filters or add new routes for additional sourcetypes — Artemis accepts any sourcetype string, but only the ones listed in the Artemis portal will be auto-normalized.

### 3. Configure inbound (Artemis → Cribl HTTP source)

* Source: `in_artemis_cases` (`type: cribl_http`, port `10081`, path `/services/collector/event`, Bearer token from `artemis_case_webhook_token`). Expose this on a routable address and put the URL into the Artemis "Cribl Stream — Notifications" connector (`cribl_notifications`).
* Each POST is a single Artemis case as JSON. The `artemis_case_fanout` pipeline parses, promotes useful fields (`case_id`, `severity`, `status`, `link`, …), buckets severity into `low / medium / high`, and stamps `__artemis_target` so you can write your own Routes that select which downstream destination(s) get a copy.

#### Configure Case Fanout

The Pack does **not** ship Slack / ServiceNow / Splunk / Datadog destinations — those need credentials that belong to your Worker Group, not in a published Pack. Add your existing destinations and create routes like:

```yaml
- name: cases_to_slack
  pipeline: passthru
  filter: __artemis_target.includes('slack')
  output: slack_webhook
- name: cases_to_servicenow
  pipeline: passthru
  filter: __artemis_target.includes('servicenow')
  output: servicenow_webhook
- name: cases_to_splunk
  pipeline: passthru
  filter: __artemis_target.includes('splunk')
  output: splunk_hec
- name: cases_to_datadog
  pipeline: passthru
  filter: __artemis_target.includes('datadog')
  output: datadog_logs
```

### 4. Commit & deploy

Commit the changes in the Pack and deploy your Worker Group. Within a minute Artemis should report the integration as healthy on the **Connectors** page.

#### Variables

| Variable                       | Description                                                              |
|--------------------------------|--------------------------------------------------------------------------|
| `artemis_aws_account_id`       | Artemis-side AWS account that owns the cross-account ingestion role.     |
| `artemis_iam_role_arn`         | Cross-account IAM role the Worker assumes to write to S3.                |
| `artemis_external_id`          | STS ExternalId. Treat as a shared secret.                                |
| `artemis_s3_bucket`            | Artemis-managed S3 bucket. Format: `<org>-all-ingested-security-logs`.   |
| `artemis_aws_region`           | Region of the S3 bucket.                                                 |
| `artemis_key_prefix`           | S3 prefix that scopes writes to your integration + index.                |
| `artemis_case_webhook_url`     | URL of the Cribl HTTP Source. Set in Artemis as the case webhook target. |
| `artemis_case_webhook_token`   | Bearer token Artemis presents on case POSTs.                             |

## Upgrades

Upgrading a Pack while keeping the same Pack ID can have side effects — see [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading). Keep your variable overrides backed up before upgrading.

## Release Notes

### Version 0.9.0 (beta)

* Initial publication. Bidirectional Stream ↔ Artemis integration.
* Pre-configured S3 destination with cross-account `AssumeRole` + `ExternalId`.
* Pre-configured Cribl HTTP source with bearer-token auth for case ingest.
* Reference hygiene pipelines for Okta, AWS CloudTrail, and Syslog.
* `artemis_case_fanout` pipeline with severity bucketing and routing hints.
* 20+ synthetic sample events per data type (no PII, no expiration dates).

## Contributing to the Pack

Issues, PRs, and ideas welcome. The Pack source lives inside the Artemis monorepo under `assets/cribl_pack/artemis/`. Reach the maintainers in the [Cribl Community Slack](https://cribl-community.slack.com) `#packs` channel or by filing an issue at <https://github.com/artemisai-main/artemis>.

## License

This Pack is released under the [Apache 2.0 License](./LICENSE).
