# S12_reference.md - Шпаргалка (1 страница)

## Команды (локально через Docker)

```bash
# Hadolint → JSON
docker run --rm -i hadolint/hadolint   hadolint -f json - < Dockerfile > EVIDENCE/S12/hadolint.json || true

# Checkov → JSON (Terraform/K8s)
docker run --rm -v $PWD:/src bridgecrew/checkov:latest   -d /src/iac -o json > EVIDENCE/S12/checkov.json || true

# Trivy → JSON (image)
docker build -t s09s12-app:local .
docker run --rm -v $PWD:/work aquasec/trivy:latest   image --format json --output /work/EVIDENCE/S12/trivy.json --ignore-unfixed s09s12-app:local || true
```

## Дерево ожидаемых артефактов

```text
EVIDENCE/
└── S12/
    ├── hadolint.json
    ├── checkov.json
    └── trivy.json
# (опц.) attestation.json / cosign.*
```

## Ссылка в DS.md (пример)

```text
S12 - IaC & Container Security: Hadolint/Checkov/Trivy, артефакты в EVIDENCE/S12. 
Actions: <URL успешного job>. Харднинг: <две+ меры кратко>. Остаток: <FP/accept/план>.
```
