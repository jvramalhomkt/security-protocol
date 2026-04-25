# Referência: Compliance, Score e Alertas Rápidos

## Calculadora de Score de Segurança

Use esta tabela para calcular o score final após auditoria:

```
SCORE BASE: 100

Deduções por severidade:
  🔴 CRÍTICO:  -25 pontos cada (máx: -75)
  🟠 ALTO:     -10 pontos cada (máx: -40)
  🟡 MÉDIO:    -5  pontos cada (máx: -20)
  🟢 BAIXO:    -1  ponto  cada (máx: -5)

Bônus por boas práticas:
  ✅ Auditoria recente (< 3 meses):     +5
  ✅ MFA habilitado em todos os acessos: +5
  ✅ Backups testados e funcionando:     +5
  ✅ Monitoramento/alertas ativos:       +5

CLASSIFICAÇÃO FINAL:
  90-100: 🟢 EXCELENTE — Postura de segurança madura
  75-89:  🟡 BOM — Pequenas melhorias recomendadas
  50-74:  🟠 ATENÇÃO — Vulnerabilidades significativas
  25-49:  🔴 ALTO RISCO — Ação imediata necessária
  0-24:   💀 CRÍTICO — Sistema comprometível agora
```

---

## Alertas de Comprometimento Imediato

Se qualquer um dos itens abaixo for encontrado, **pare tudo e investigue primeiro**:

```bash
# 1. Processos suspeitos
ps aux | grep -E "nc |ncat |netcat |/tmp/|\.sh " | grep -v grep
find /tmp /var/tmp -type f -newer /tmp -executable 2>/dev/null

# 2. Conexões de rede suspeitas
ss -tulnp | grep -vE "ssh|http|https|mysql|postgres|redis"
netstat -anp | grep ESTABLISHED | grep -v "127.0.0.1"

# 3. Crontabs suspeitos
for user in $(cut -f1 -d: /etc/passwd); do crontab -u $user -l 2>/dev/null | grep -v "^#" | grep -v "^$"; done
cat /etc/cron* /var/spool/cron/crontabs/* 2>/dev/null

# 4. Alterações recentes em binários do sistema
find /bin /sbin /usr/bin /usr/sbin -newer /etc/passwd -type f 2>/dev/null

# 5. SSH authorized_keys suspeitos
find / -name "authorized_keys" 2>/dev/null | xargs cat
```

---

## Compliance por Framework

### LGPD (Lei Geral de Proteção de Dados — Brasil)
```
Artigo 46 — Medidas técnicas de segurança obrigatórias:
[ ] Criptografia de dados pessoais em repouso
[ ] Criptografia em trânsito (TLS)
[ ] Controle de acesso aos dados pessoais
[ ] Logs de acesso a dados pessoais
[ ] Política de retenção e eliminação de dados
[ ] Anonimização quando possível
[ ] Avaliação de impacto (DPIA) para dados sensíveis
[ ] DPO (Encarregado) designado
[ ] Canal para direitos dos titulares
[ ] Plano de resposta a incidentes (notificar ANPD em 72h)
```

### OWASP ASVS (Application Security Verification Standard)
```
Nível 1 — Básico (mínimo para qualquer app):
[ ] Autenticação e gerenciamento de sessão
[ ] Controle de acesso
[ ] Validação de entrada
[ ] Criptografia
[ ] Tratamento de erros (sem vazamento de stack trace)
[ ] Proteção de dados sensíveis

Nível 2 — Padrão (apps com dados sensíveis):
  + Autenticação multi-fator
  + Gerenciamento de secrets
  + Logging e monitoramento

Nível 3 — Avançado (apps críticos / financeiros):
  + Análise de ameaças formal
  + Revisão de código obrigatória
  + Testes de penetração regulares
```

### CIS Controls (Top 6 prioritários)
```
CIS Control 1: Inventário de ativos
CIS Control 2: Inventário de software
CIS Control 3: Proteção de dados
CIS Control 4: Configuração segura
CIS Control 5: Gerenciamento de contas
CIS Control 6: Gerenciamento de vulnerabilidades
```

---

## Ferramentas por Categoria — Referência Rápida

```
RECONHECIMENTO:
  nmap, masscan, amass, subfinder, shodan

ANÁLISE WEB:
  nikto, gobuster, wfuzz, burpsuite, zap (OWASP ZAP)
  nuclei, wapiti, whatweb

SSL:
  testssl.sh, sslyze, sslscan

ANÁLISE DE CÓDIGO:
  semgrep, bandit (Python), eslint-security, brakeman (Ruby)
  sonarqube, checkmarx, veracode

SECRETS:
  gitleaks, trufflehog, detect-secrets, git-secrets

CONTAINERS:
  trivy, clair, anchore, grype, syft, docker scout
  kube-bench, kube-hunter, polaris

BANCO DE DADOS:
  sqlmap, pgaudit, mysql audit plugin

SISTEMA:
  lynis, rkhunter, chkrootkit, ossec, wazuh

MONITORAMENTO:
  fail2ban, ossec/wazuh, crowdsec, auditd
  elk stack, grafana+loki, datadog, sentry

PENTEST FRAMEWORKS:
  metasploit, burpsuite pro, cobalt strike (licenciado)
  kali linux, parrot os
```

---

## Template de Relatório Executivo (para não-técnicos)

```markdown
# Relatório de Segurança — Resumo Executivo

**Sistema:** [nome]
**Data:** [data]
**Score:** [X]/100 — [Classificação]

## Situação Atual
[1-2 parágrafos explicando o estado em linguagem simples]

## Principais Riscos
1. [Risco em linguagem de negócio, ex: "Dados de clientes podem ser acessados por pessoas não autorizadas"]
2. [...]

## O Que Pode Acontecer Se Não Agirmos
- [Consequência concreta: multa ANPD, vazamento, indisponibilidade]

## Investimento Necessário
| Ação             | Esforço    | Custo  | Prioridade  |
|-----------------|-----------|--------|-------------|
| [ação 1]        | 2 horas   | R$ 0   | IMEDIATA    |
| [ação 2]        | 1 dia     | R$ 500 | Esta semana |

## Próxima Auditoria Recomendada
[data — tipicamente 3-6 meses após remediação]
```

---

## Plano de Resposta a Incidentes

```
FASE 1 — DETECÇÃO (0-1h)
  1. Identificar natureza do incidente
  2. Preservar evidências (não reiniciar sistemas!)
  3. Acionar equipe de resposta

FASE 2 — CONTENÇÃO (1-4h)
  1. Isolar sistemas comprometidos
  2. Revogar credenciais suspeitas
  3. Bloquear IPs maliciosos
  4. Backup do estado atual (forense)

FASE 3 — ERRADICAÇÃO (4-24h)
  1. Identificar vetor de ataque
  2. Remover malware/backdoors
  3. Corrigir vulnerabilidade explorada

FASE 4 — RECUPERAÇÃO (24-72h)
  1. Restaurar de backup limpo
  2. Monitoramento intensivo
  3. Validar integridade dos dados

FASE 5 — PÓS-INCIDENTE
  1. Relatório de RCA (Root Cause Analysis)
  2. Notificar ANPD se dados pessoais afetados (72h)
  3. Atualizar playbooks de segurança
  4. Lições aprendidas
```
