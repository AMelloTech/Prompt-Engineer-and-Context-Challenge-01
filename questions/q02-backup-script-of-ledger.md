---
question: 2
title: "Backup-Script-Of-Ledger"
framework: "R-T-F (Role-Task-Format)"
provider: "OpenAI"
model: "GPT 6.0 Luna"
effort: "medium"
status: "final"
---

## Scenario

The team realized that Ledger's PostgreSQL, running on EC2 for years, never
had a backup routine. A daily cron job needs to be created to close that
gap (see the general context in
[`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
Role: You are a Staff SRE, with a background on databasa, aws and Linux. You always follow the best practices and security frameworks.
Task: We have a PostgreSQL running on EC2 that has never been backup. The objetive is create a script that will do a dump with pg_dump, compact it with gzip, upload this file into the S3 bucket using aws s3 cp, keep it for 30 days of retention in S3, remove the one's older than 30 days and register in the /var/log/ledger-backup.log with the timestamp and have a good exit code in case of failures that help the team understands where to start looking and nothing generic. Enviroment information:Host: ledger-db.internal.hvt.io
Port: 5432
Database: ledger_prod
Backup user: backup_user
Password: PGPASSWORD environment variable, populated by AWS Secrets Manager via the instance's IAM role
AWS Region: us-east-1
Instance OS: Ubuntu 22.04 LTS
Working directory with 80 GB free: /var/backups/ledger
Current average compressed dump size: ~12 GB
Format: bash script that will be run as a cron job
```

## Model

**Model used:** GPT 6.0 Luna (OpenAI), medium effort

**Why this model:** second provider for the challenge (diversification
required by the assignment); good track record generating infrastructure
scripts with explicit error handling.

## Output

Model's response (execution summary + generated script):

```
I'll leave a cron-ready script, with the bucket configured via a required
variable, detailed per-step logging, and distinct exit codes. For retention,
I'll scope the deletion to the configured S3 prefix; in production, an S3
Lifecycle rule would be even more robust, but I'll also include the
requested cleanup in the script itself.

I created the executable script ledger-backup.sh. It produces a compressed
SQL dump, verifies the file, uploads it to S3, and removes backups older
than 30 days, scoped only to the configured prefix and the expected naming
pattern.

Set S3_BUCKET and PGPASSWORD in the cron environment; the script does not
fetch the secret from Secrets Manager itself. Run it as root so it can write
to /var/log/ledger-backup.log and use /var/backups/ledger. The user also
needs pg_dump, gzip, aws, jq, and flock installed.

Exit codes: 10 configuration; 11 dependencies; 12 log, lock, or disk space;
20 dump; 21 compression/integrity; 30 upload; 31 retention listing;
32 S3 deletion.

The instance role needs permission to put, list, and delete objects in the
bucket/prefix. I'd also recommend enabling default encryption and an S3
Lifecycle rule on the bucket as an additional safeguard. The script was
created but not executed in this environment.
```

Generated script (`ledger-backup.sh`):

```bash
#!/usr/bin/env bash
# PostgreSQL logical backup to S3. Intended to run from root's cron.
set -uo pipefail
umask 077

readonly LOG_FILE="/var/log/ledger-backup.log"
readonly WORK_DIR="/var/backups/ledger"
readonly DB_HOST="ledger-db.internal.hvt.io"
readonly DB_PORT="5432"
readonly DB_NAME="ledger_prod"
readonly DB_USER="backup_user"
readonly AWS_REGION="us-east-1"
readonly S3_PREFIX_DEFAULT="ledger/postgresql/"
readonly LOCK_FILE="/run/lock/ledger-backup.lock"

# Exit codes: 10 configuration; 11 dependencies; 12 log/lock/workspace;
# 20 dump failure; 21 gzip/integrity failure; 30 upload failure;
# 31 S3 listing/retention failure; 32 S3 deletion failure.
log() {
    local level="$1"; shift
    printf '%s [%s] %s\n' "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" "$level" "$*" >> "$LOG_FILE"
}

die() {
    local code="$1"; shift
    log ERROR "$* (exit_code=$code)"
    exit "$code"
}

if ! { : >> "$LOG_FILE"; } 2>/dev/null; then
    printf '%s [ERROR] Cannot write log file %s; check cron user and permissions. (exit_code=12)\n' \
        "$(date -u '+%Y-%m-%dT%H:%M:%SZ')" "$LOG_FILE" >&2
    exit 12
fi

log INFO "Backup job started."

[[ -n "${S3_BUCKET:-}" ]] || die 10 "S3_BUCKET is not set; configure the destination bucket in the cron environment"
S3_PREFIX="${S3_PREFIX:-$S3_PREFIX_DEFAULT}"
[[ "$S3_PREFIX" != /* ]] || die 10 "S3_PREFIX must be a relative S3 key prefix"
[[ "$S3_PREFIX" == */ ]] || S3_PREFIX="${S3_PREFIX}/"
[[ -n "${PGPASSWORD:-}" ]] || die 10 "PGPASSWORD is not set; provide it securely to the cron environment"
export PGPASSWORD

for command in pg_dump gzip aws jq flock df; do
    command -v "$command" >/dev/null 2>&1 || die 11 "Required command not found: $command"
done

mkdir -p "$WORK_DIR" 2>/dev/null || die 12 "Cannot create/access work directory $WORK_DIR"
[[ -w "$WORK_DIR" ]] || die 12 "Work directory is not writable: $WORK_DIR"
exec 9>"$LOCK_FILE" || die 12 "Cannot open lock file $LOCK_FILE"
flock -n 9 || die 12 "Another backup job is already running (lock $LOCK_FILE)"

# Leave headroom for the expected ~12 GiB compressed artifact and filesystem overhead.
available_kb="$(df -Pk "$WORK_DIR" | awk 'NR==2 {print $4}')"
[[ "$available_kb" =~ ^[0-9]+$ ]] || die 12 "Could not determine free space for $WORK_DIR"
(( available_kb >= 15728640 )) || die 12 "Insufficient free space: ${available_kb} KiB available; at least 15728640 KiB required"

timestamp="$(date -u '+%Y%m%dT%H%M%SZ')"
filename="${DB_NAME}_${timestamp}.sql.gz"
local_file="${WORK_DIR}/${filename}"
tmp_file="${WORK_DIR}/.${filename}.partial"
s3_key="${S3_PREFIX}${filename}"
cleanup() { rm -f -- "$tmp_file"; }
trap cleanup EXIT

log INFO "Starting pg_dump for database $DB_NAME on $DB_HOST:$DB_PORT."
pg_dump --no-password --host="$DB_HOST" --port="$DB_PORT" --username="$DB_USER" \
    --dbname="$DB_NAME" --format=plain --no-owner --no-acl 2>>"$LOG_FILE" | gzip -c > "$tmp_file"
pipeline_status=("${PIPESTATUS[@]}")
if (( pipeline_status[0] != 0 )); then
    die 20 "pg_dump failed with status ${pipeline_status[0]}; inspect preceding PostgreSQL diagnostics in $LOG_FILE"
fi
if (( pipeline_status[1] != 0 )); then
    die 21 "gzip failed with status ${pipeline_status[1]} while compressing the dump"
fi
[[ -s "$tmp_file" ]] || die 21 "Compressed dump is empty"
gzip -t "$tmp_file" 2>>"$LOG_FILE" || die 21 "gzip integrity check failed for the completed dump"
mv -- "$tmp_file" "$local_file" || die 12 "Cannot finalize dump file in $WORK_DIR"
log INFO "Dump completed and gzip integrity verified: $local_file ($(du -h "$local_file" | awk '{print $1}'))."

log INFO "Uploading s3://$S3_BUCKET/$s3_key."
if ! aws s3 cp "$local_file" "s3://$S3_BUCKET/$s3_key" --region "$AWS_REGION" --only-show-errors >>"$LOG_FILE" 2>&1; then
    die 30 "S3 upload failed for s3://$S3_BUCKET/$s3_key; verify instance role permissions, network, and bucket policy"
fi
log INFO "Upload completed: s3://$S3_BUCKET/$s3_key."

cutoff="$(date -u -d '30 days ago' '+%Y-%m-%dT%H:%M:%SZ')" || die 31 "Could not calculate the 30-day retention cutoff"
log INFO "Finding objects older than $cutoff under s3://$S3_BUCKET/$S3_PREFIX."
listing_file="${WORK_DIR}/.${DB_NAME}_${timestamp}.s3-list.json"
trap 'cleanup; rm -f -- "$listing_file"' EXIT
if ! aws s3api list-objects-v2 --bucket "$S3_BUCKET" --prefix "$S3_PREFIX" \
    --region "$AWS_REGION" --output json > "$listing_file" 2>>"$LOG_FILE"; then
    die 31 "S3 object listing failed; verify s3:ListBucket permission and bucket access"
fi
if ! jq -e 'type == "object" and ((.Contents // []) | type == "array")' \
    "$listing_file" >/dev/null 2>&1; then
    die 31 "S3 returned an invalid object listing; retention cleanup was not attempted"
fi

expired_count=0
while IFS= read -r key; do
    [[ -n "$key" ]] || continue
    log INFO "Deleting expired S3 object: s3://$S3_BUCKET/$key."
    if ! aws s3api delete-object --bucket "$S3_BUCKET" --key "$key" \
        --region "$AWS_REGION" >>"$LOG_FILE" 2>&1; then
        die 32 "Could not delete expired object s3://$S3_BUCKET/$key; verify s3:DeleteObject permission"
    fi
    ((expired_count += 1))
done < <(jq -r --arg cutoff "$cutoff" --arg prefix "$S3_PREFIX" \
    '.Contents[]? | select(.LastModified < $cutoff) | .Key | select(startswith($prefix) and test("ledger_prod_[0-9]{8}T[0-9]{6}Z[.]sql[.]gz$"))' \
    "$listing_file")

rm -f -- "$local_file" || die 12 "S3 upload succeeded, but local dump cleanup failed for $local_file"
log INFO "Backup job completed successfully. Expired objects deleted: $expired_count. Local dump removed."
exit 0
```

## Rationale

**Role** ("Staff SRE... background on database, aws and Linux... follow the
best practices and security frameworks") drives the model to use `set -uo
pipefail`, `flock` against concurrent runs, `umask 077`, and a gzip
integrity check — decisions of a production script, not a teaching example.
**Task** details every requested step (pg_dump → gzip → `aws s3 cp` →
30-day retention → timestamped logging → specific, non-generic exit codes),
and all of them appear implemented 1:1 in the script, including the
distinct exit codes per failure category (10–32), exactly as requested.
**Format** ("bash script that will be run as a cron job") resulted in a
self-contained, non-interactive script ready for scheduling, instead of a
prose explanation.

