---
question: 2
title: "Backup-Script-Of-Ledger"
framework: "R-T-F (Role-Task-Format)"
provider: "OpenAI"
model: "GPT 6.0 Luna"
effort: "medium"
status: "final"
---

## Cenário

O time percebeu que o PostgreSQL do Ledger, rodando em EC2 há anos, nunca
teve uma rotina de backup. É preciso criar um cron job diário para cobrir
essa lacuna (ver contexto geral em
[`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
Role: You are a Staff SRE, with a background on databasa, aws and Linux. You always follow the best practices and security frameworks.
Task: We have a PostgreSQL running on EC2 that has never been backup. The objetive is create a script that will do a dump with pg_dump, compact it with gzip, upload this file into the S3 bucket using aws s3 cp, keep it for 30 days of retention in S3, remove the one's older than 30 days and register in the /var/log/ledger-backup.log with the timestamp and have a good exit code in case of failures that help the team understands where to start looking and nothing generic. Enviroment information:Host: ledger-db.internal.hvt.io
Porta: 5432
Banco: ledger_prod
Usuário de backup: backup_user
Senha: variável de ambiente PGPASSWORD, populada pelo AWS Secrets Manager via IAM role da instância
Região AWS: us-east-1
SO da instância: Ubuntu 22.04 LTS
Diretório de trabalho com 80 GB livres: /var/backups/ledger
Tamanho médio atual do dump compactado: ~12 GB
Format: bash script that will be run as a cron job
```

## Modelo

**Modelo usado:** GPT 6.0 Luna (OpenAI), effort medium

**Por que esse modelo:** segundo provider do desafio (diversificação exigida
pelo enunciado); bom histórico em geração de scripts de infraestrutura com
tratamento de erro explícito.

## Output

Resposta do modelo (resumo da execução + script gerado):

```
Vou deixar um script pronto para cron, com configuração do bucket via variável
obrigatória, log detalhado por etapa e códigos de saída distintos. Para
retenção, vou limitar a exclusão ao prefixo configurado no S3; em produção,
uma regra de Lifecycle no bucket é ainda mais robusta, mas também incluirei a
limpeza solicitada no próprio script.

Criei o script executável ledger-backup.sh. Ele gera um dump SQL comprimido,
verifica o arquivo, envia para o S3 e remove backups com mais de 30 dias
somente no prefixo configurado e com o padrão de nome esperado.

Configure S3_BUCKET e PGPASSWORD no ambiente do cron; o script não busca o
segredo no Secrets Manager. Execute-o como root para permitir escrita em
/var/log/ledger-backup.log e uso de /var/backups/ledger. O usuário também
precisa ter pg_dump, gzip, aws, jq e flock instalados.

Códigos de saída: 10 configuração; 11 dependências; 12 log, lock ou espaço;
20 dump; 21 compressão/integridade; 30 upload; 31 listagem para retenção;
32 exclusão no S3.

O role da instância precisa de acesso para enviar, listar e excluir objetos
no bucket/prefixo. Também recomendo configurar criptografia padrão e uma
regra de Lifecycle no bucket como proteção adicional. O script foi criado,
mas não executado neste ambiente.
```

Script gerado (`ledger-backup.sh`):

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

## Justificativa

**Role** ("Staff SRE... background on database, aws and Linux... follow the
best practices and security frameworks") leva o modelo a usar `set -uo
pipefail`, `flock` contra execuções concorrentes, `umask 077` e checagem de
integridade do gzip — decisões de um script de produção, não um exemplo
didático. **Task** detalha cada etapa pedida (pg_dump → gzip → `aws s3 cp` →
retenção de 30 dias → log com timestamp → exit codes específicos e não
genéricos) e todas aparecem implementadas 1:1 no script, inclusive os exit
codes distintos por categoria de falha (10–32), exatamente como solicitado.
**Format** ("bash script that will be run as a cron job") resultou em um
script autônomo sem interação, pronto para agendamento, em vez de uma
explicação em prosa.

## Notas

O script usa `aws s3 cp` só para o upload, mas a retenção/exclusão foi
implementada via `aws s3api list-objects-v2` + `delete-object` em vez de
`aws s3 cp` sozinho — mais granular, mas é uma pequena liberdade do modelo
não pedida explicitamente. Também não busca a senha no Secrets Manager
diretamente: espera `PGPASSWORD` já populada no ambiente do cron, o que é
compatível com o enunciado (que já descreve isso como responsabilidade da
IAM role/wrapper), mas vale confirmar esse ponto com a Lorraine antes do
deploy.
