# S10_reference.md - Шпаргалка (1 страница)

## Команды (локально через Docker)

```bash
# Semgrep → SARIF
docker run --rm -v $PWD:/src returntocorp/semgrep:latest   semgrep ci --config p/ci --sarif --output /src/EVIDENCE/S10/semgrep.sarif --metrics=off || true

# Gitleaks → JSON
docker run --rm -v $PWD:/repo zricethezav/gitleaks:latest   detect --source=/repo --report-format=json --report-path=/repo/EVIDENCE/S10/gitleaks.json || true
```

## Дерево ожидаемых артефактов

```text
EVIDENCE/
└── S10/
    ├── semgrep.sarif
    └── gitleaks.json
```

## Ссылка в DS.md (пример)

```text
S10 - SAST & Secrets: Semgrep (SARIF), Gitleaks (JSON), артефакты в EVIDENCE/S10. Actions: <URL успешного job>. FP/исключения: <кратко>.
```
