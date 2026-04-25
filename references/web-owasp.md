# Referência: Aplicações Web — OWASP Top 10

## OWASP Top 10 (2021) — Checklist Completo

---

### A01 — Broken Access Control 🔴 CRÍTICO

**Verificações:**
```bash
# Testar acesso a recursos sem autenticação
curl -I https://dominio.com/admin
curl -I https://dominio.com/api/users
curl -I https://dominio.com/.env
curl -I https://dominio.com/config.php

# Verificar arquivos sensíveis expostos
curl -s https://dominio.com/.git/config
curl -s https://dominio.com/.env
curl -s https://dominio.com/backup.sql
curl -s https://dominio.com/wp-config.php

# Enumerar diretórios
gobuster dir -u https://dominio.com -w /usr/share/wordlists/dirb/common.txt
```

**Remediações:**
- Implementar middleware de autenticação em todas as rotas protegidas
- Usar RBAC (Role-Based Access Control)
- Negar acesso por padrão, permitir explicitamente
- Logs de acesso a funções administrativas

---

### A02 — Cryptographic Failures 🔴 CRÍTICO

**Verificações:**
```bash
# Verificar SSL/TLS
testssl.sh https://dominio.com
nmap --script ssl-enum-ciphers -p 443 dominio.com

# Verificar headers de segurança
curl -I https://dominio.com | grep -E "Strict-Transport|Content-Security|X-Frame"

# Verificar se dados sensíveis trafegam em HTTP
curl -I http://dominio.com  # deve redirecionar para HTTPS
```

**Remediações:**
```nginx
# Nginx — forçar HTTPS e headers de segurança
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
}
```

---

### A03 — Injection 🔴 CRÍTICO

**Verificações:**
```bash
# SQL Injection básico
sqlmap -u "https://dominio.com/page?id=1" --batch --level=3

# XSS básico — injetar em campos de formulário
# Payload: <script>alert(1)</script>
# Payload encoded: %3Cscript%3Ealert(1)%3C/script%3E

# Command injection
curl "https://dominio.com/ping?host=127.0.0.1;id"
```

**Remediações:**
- Usar prepared statements / ORM (nunca concatenar SQL)
- Sanitizar e validar toda entrada do usuário
- Implementar CSP (Content Security Policy)
- WAF (Web Application Firewall): ModSecurity, Cloudflare WAF

---

### A04 — Insecure Design 🟠 ALTO

**Verificações:**
- Rate limiting em endpoints de login/registro?
- Captcha em formulários públicos?
- Limite de tentativas de senha?
- Tokens de reset de senha com expiração?

**Remediações:**
```bash
# Exemplo: Rate limiting com Nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

location /login {
    limit_req zone=login burst=3 nodelay;
}
```

---

### A05 — Security Misconfiguration 🟠 ALTO

**Verificações:**
```bash
# Verificar headers de segurança completos
curl -I https://dominio.com

# Verificar informações vazadas nos headers
curl -I https://dominio.com | grep -E "Server:|X-Powered-By:|X-AspNet"

# Verificar CORS
curl -H "Origin: https://evil.com" -I https://dominio.com/api

# Verificar modo debug ativo
curl https://dominio.com/api/debug
curl https://dominio.com/?debug=true

# Scan completo de misconfigurações
nikto -h https://dominio.com
```

**Remediações:**
```nginx
# Nginx — remover informações do servidor
server_tokens off;

# Apache
ServerTokens Prod
ServerSignature Off
```

---

### A06 — Vulnerable and Outdated Components 🟠 ALTO

**Verificações:**
```bash
# Node.js
npm audit
npm outdated

# Python
pip-audit
safety check

# PHP / Composer
composer audit

# Ruby
bundle audit

# Verificar versões expostas
curl https://dominio.com/package.json
curl https://dominio.com/composer.json
```

**Remediações:**
```bash
npm audit fix
npm update

pip install --upgrade [pacote]
```

---

### A07 — Identification and Authentication Failures 🟠 ALTO

**Verificações:**
- Senhas fracas aceitas? Tentar: `123456`, `password`, `admin`
- Brute force possível? Testar sem rate limiting
- Tokens JWT seguros?

```bash
# Verificar JWT
# Decodificar em jwt.io e verificar:
# - alg: não deve ser "none"
# - exp: deve ter expiração
# - Segredo forte?

# Testar brute force (com autorização)
hydra -l admin -P /usr/share/wordlists/rockyou.txt dominio.com http-post-form "/login:username=^USER^&password=^PASS^:Invalid"
```

---

### A08 — Software and Data Integrity Failures 🟡 MÉDIO

**Verificações:**
- Dependências verificadas por hash/checksum?
- CI/CD com verificação de integridade?
- Atualizações automáticas sem validação de assinatura?

---

### A09 — Security Logging and Monitoring Failures 🟡 MÉDIO

**Verificações:**
```bash
# Verificar se logs existem e são adequados
ls -la /var/log/nginx/
ls -la /var/log/apache2/

# Verificar formato dos logs (deve ter IP, timestamp, user-agent)
tail -20 /var/log/nginx/access.log
tail -20 /var/log/nginx/error.log
```

**Remediações:**
- Logar: tentativas de login, acessos a recursos protegidos, erros 4xx/5xx, mudanças de dados críticos
- Alertas em tempo real (ex: Sentry, Datadog, ELK Stack)
- Retenção mínima de 90 dias

---

### A10 — Server-Side Request Forgery (SSRF) 🟡 MÉDIO

**Verificações:**
```bash
# Testar SSRF em parâmetros que aceitam URLs
curl "https://dominio.com/fetch?url=http://169.254.169.254/latest/meta-data/"
curl "https://dominio.com/proxy?target=http://localhost:6379"
```

---

## Headers de Segurança — Referência Completa

```
✅ Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
✅ Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-xxx'
✅ X-Content-Type-Options: nosniff
✅ X-Frame-Options: SAMEORIGIN
✅ Referrer-Policy: strict-origin-when-cross-origin
✅ Permissions-Policy: geolocation=(), microphone=(), camera=()
❌ Server: (deve ser removido ou genérico)
❌ X-Powered-By: (deve ser removido)
```

## Ferramentas Recomendadas

```bash
# Análise passiva
curl -I https://dominio.com
whatweb https://dominio.com

# Scanners
nikto -h https://dominio.com
nuclei -u https://dominio.com
wapiti -u https://dominio.com

# Headers
https://securityheaders.com
https://observatory.mozilla.org

# SSL
testssl.sh https://dominio.com
https://www.ssllabs.com/ssltest/
```
