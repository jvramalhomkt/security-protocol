# Referência: Segurança em CI/CD Pipelines

## Diagnóstico Geral

```bash
# Verificar secrets expostos em código
# Ferramentas de detecção:
gitleaks detect --source . --verbose
trufflehog git file://. --only-verified
git log --all --full-history -- "*.env" "*.key" "*.pem"

# Verificar variáveis no histórico Git
git log --all -p | grep -E "password|secret|token|key|api_key" -i | head -30
```

---

## GitHub Actions

### 🔴 CRÍTICO

#### Secrets hardcoded em workflows
```yaml
# ❌ NUNCA fazer
- name: Deploy
  run: |
    export DB_PASS="senha123"
    deploy.sh

# ✅ Usar GitHub Secrets
- name: Deploy
  env:
    DB_PASS: ${{ secrets.DB_PASSWORD }}
  run: deploy.sh
```

#### Permissões excessivas do GITHUB_TOKEN
```yaml
# ❌ Sem restrição (padrão perigoso em repos antigos)
# permite write em tudo

# ✅ Principle of Least Privilege
permissions:
  contents: read        # apenas leitura no código
  packages: write       # escrita apenas no necessário
  pull-requests: write  # comentários em PRs

# Ou negar tudo e conceder granularmente:
permissions: {}  # tudo negado por padrão
```

#### Injeção via pull_request de forks
```yaml
# ❌ Perigoso — código de PR externo pode ler secrets
on:
  pull_request:

# ✅ Usar pull_request_target apenas com cuidado
# Ou usar ambiente de aprovação:
on:
  pull_request_target:
jobs:
  test:
    environment: external-pr  # requer aprovação manual
```

---

### 🟠 ALTO

#### Ações de terceiros sem versão fixada
```yaml
# ❌ Sem fixar versão (supply chain attack)
- uses: actions/checkout@main
- uses: actions/checkout@v4

# ✅ Fixar pelo SHA do commit
- uses: actions/checkout@v4.1.1
# Ou pelo hash:
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
```

#### Workflow sem OIDC (usando chaves de longa duração)
```yaml
# ❌ Chave AWS estática nos secrets
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_KEY }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET }}

# ✅ OIDC — sem credenciais de longa duração
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS via OIDC
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/github-actions-role
      aws-region: us-east-1
```

#### Artefatos e cache sem validação
```yaml
# ✅ Verificar hash de artefatos baixados
- name: Download artifact
  uses: actions/download-artifact@v4
  with:
    name: build-output

- name: Verify checksum
  run: sha256sum -c checksums.txt
```

---

### 🟡 MÉDIO

#### Pipeline sem scan de segurança
```yaml
# ✅ Adicionar ao pipeline:

# 1. Scan de dependências
- name: Dependency audit
  run: npm audit --audit-level=high

# 2. SAST (análise estática)
- name: SAST with Semgrep
  uses: semgrep/semgrep-action@v1
  with:
    config: p/security-audit p/owasp-top-ten

# 3. Scan de secrets
- name: Scan for secrets
  uses: gitleaks/gitleaks-action@v2

# 4. Scan de imagem Docker
- name: Scan Docker image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'

# 5. Análise de licenças
- name: License check
  run: npx license-checker --failOn GPL
```

#### Branch protection ausente
```bash
# Verificar via GitHub API
curl -H "Authorization: Bearer TOKEN" \
  https://api.github.com/repos/ORG/REPO/branches/main/protection

# Configurar via API
curl -X PUT -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  https://api.github.com/repos/ORG/REPO/branches/main/protection \
  -d '{
    "required_status_checks": {"strict": true, "contexts": ["test", "security-scan"]},
    "enforce_admins": true,
    "required_pull_request_reviews": {"required_approving_review_count": 2},
    "restrictions": null,
    "required_linear_history": true
  }'
```

---

## GitLab CI/CD

### Verificações
```yaml
# .gitlab-ci.yml — boas práticas

variables:
  # ❌ Nunca hardcodar
  # DB_PASSWORD: "senha123"
  
  # ✅ Usar CI/CD Variables (mascaradas)
  DB_PASSWORD: $DB_PASSWORD

# Scan de segurança integrado (GitLab Ultimate)
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml

# Ambiente protegido — requer aprovação para produção
deploy-prod:
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # requer aprovação
```

---

## Supply Chain Security

### SLSA Framework
```bash
# Verificar se releases têm proveniência
# https://slsa.dev/

# Gerar SBOM (Software Bill of Materials)
syft IMAGE_NAME -o spdx-json > sbom.json
grype sbom:sbom.json  # verificar CVEs

# Assinar imagens com cosign
cosign sign --key cosign.key myregistry/myapp:latest
cosign verify --key cosign.pub myregistry/myapp:latest
```

### Verificações de Dependências
```bash
# Node.js
npm audit --audit-level=critical
npx better-npm-audit audit

# Python
pip-audit
safety check -r requirements.txt

# Go
govulncheck ./...

# Java
mvn dependency-check:check

# Ruby
bundler-audit check --update
```

---

## Checklist de Segurança CI/CD

```
[ ] Secrets armazenados no vault/secret manager, não no código
[ ] Ações de terceiros fixadas por SHA ou versão imutável
[ ] GITHUB_TOKEN com permissões mínimas
[ ] OIDC ao invés de chaves de longa duração para cloud
[ ] Branch protection habilitado (require PR review, status checks)
[ ] Pipeline inclui: SAST, SCA, secret scan, container scan
[ ] Ambientes de produção com aprovação manual
[ ] SBOM gerado em cada release
[ ] Imagens assinadas (cosign/notary)
[ ] Dependabot ou Renovate para updates automáticos
[ ] Auditoria de acessos ao repositório
[ ] Logs de pipeline retidos por 90+ dias
```

---

## Score por Categoria

| Categoria           | Peso | Verificações                              |
|--------------------|------|-------------------------------------------|
| Secrets/Credenciais | 30%  | Nenhum hardcoded, OIDC, mascarado         |
| Permissões          | 20%  | GITHUB_TOKEN, branch protection           |
| Supply chain        | 20%  | Versões fixas, SBOM, assinaturas          |
| Scans de segurança  | 20%  | SAST, SCA, secrets, containers            |
| Aprovação/Controle  | 10%  | Deploy manual em prod, revisão de código  |
