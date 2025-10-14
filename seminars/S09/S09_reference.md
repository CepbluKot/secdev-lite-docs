# S09_reference.md - Шпаргалка (1 страница)

## Команды (локально через Docker)

```bash
# SBOM (CycloneDX)
docker run --rm -v $PWD:/work -w /work anchore/syft:latest packages dir:. -o cyclonedx-json > EVIDENCE/S09/sbom.json

# SCA (Grype)
docker run --rm -v $PWD:/work -w /work anchore/grype:latest sbom:/work/EVIDENCE/S09/sbom.json -o json > EVIDENCE/S09/sca_report.json

# Сводка (если есть jq; в CI формируется автоматически)
echo "# SCA summary" > EVIDENCE/S09/sca_summary.md
jq '[.matches[].vulnerability.severity] | group_by(.) | map({(.[0]): length}) | add' EVIDENCE/S09/sca_report.json >> EVIDENCE/S09/sca_summary.md || true
```

## Дерево ожидаемых артефактов

```text
EVIDENCE/
└── S09/
    ├── sbom.json
    ├── sca_report.json
    └── sca_summary.md
```

## Ссылка в DS.md (пример)

```text
S09 - SBOM & SCA: SBOM и SCA выполнены, артефакты в EVIDENCE/S09. Actions: <URL успешного job>.
```
