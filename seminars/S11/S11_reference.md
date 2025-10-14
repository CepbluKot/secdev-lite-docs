# S11_reference.md - Шпаргалка (1 страница)

## Команды (локально через Docker)

```bash
# Baseline (рекомендуется для CI)
docker run --rm --network host -v $PWD:/zap/wrk owasp/zap2docker-stable   zap-baseline.py -t http://localhost:8080 -r zap_baseline.html -J zap_baseline.json -d
mv zap_baseline.* EVIDENCE/S11/

# Full (только локально, по желанию)
docker run --rm --network host -v $PWD:/zap/wrk owasp/zap2docker-stable   zap-full-scan.py -t http://localhost:8080 -r zap_full.html -J zap_full.json -d
mv zap_full.* EVIDENCE/S11/
```

## Дерево ожидаемых артефактов

```text
EVIDENCE/
└── S11/
    ├── zap_baseline.html
    └── zap_baseline.json
    # (опц.)
    ├── zap_full.html
    └── zap_full.json
```

## Ссылка в DS.md (пример)

```text
S11 - DAST (ZAP baseline): отчёты в EVIDENCE/S11. Actions: <URL успешного job>. Триаж: <fix/accept/FP кратко>.
```
