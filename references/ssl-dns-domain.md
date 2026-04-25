# Referência: SSL / DNS / Domínio

## SSL/TLS

### Diagnóstico Completo
```bash
# Verificar certificado
openssl s_client -connect dominio.com:443 -servername dominio.com 2>/dev/null | openssl x509 -noout -dates -subject -issuer

# Verificar cadeia de certificados
openssl s_client -connect dominio.com:443 -showcerts 2>/dev/null

# Verificar protocolos aceitos
nmap --script ssl-enum-ciphers -p 443 dominio.com

# Ferramenta completa
testssl.sh dominio.com

# Online
# https://www.ssllabs.com/ssltest/
```

### Problemas Comuns e Correções

**Certificado expirado / próximo do vencimento:**
```bash
# Verificar dias restantes
echo | openssl s_client -connect dominio.com:443 2>/dev/null | openssl x509 -noout -enddate

# Renovar Let's Encrypt
certbot renew
certbot renew --force-renewal

# Configurar renovação automática
crontab -e
# Adicionar: 0 12 * * * /usr/bin/certbot renew --quiet
```

**Protocolos inseguros (TLS 1.0, 1.1, SSL 2/3):**
```nginx
# Nginx — apenas TLS 1.2 e 1.3
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
```

**HSTS não configurado:**
```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

**OCSP Stapling:**
```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

---

## DNS

### Diagnóstico Completo
```bash
# Verificar registros principais
dig dominio.com A
dig dominio.com AAAA
dig dominio.com MX
dig dominio.com TXT
dig dominio.com NS
dig dominio.com SOA
dig dominio.com CAA

# Verificar SPF
dig dominio.com TXT | grep "v=spf"

# Verificar DKIM
dig default._domainkey.dominio.com TXT

# Verificar DMARC
dig _dmarc.dominio.com TXT

# Verificar DNSSEC
dig dominio.com DNSKEY
dig +dnssec dominio.com

# Verificar zone transfer (vulnerabilidade)
dig @ns1.dominio.com dominio.com AXFR
```

### Problemas Comuns e Correções

**SPF ausente ou mal configurado:**
```
# SPF mínimo recomendado
v=spf1 include:_spf.google.com ~all

# SPF restritivo (rejeita outros servidores)
v=spf1 ip4:SEU_IP include:sendgrid.net -all
```

**DMARC ausente:**
```
# Registro DMARC básico (monitoramento)
_dmarc.dominio.com TXT "v=DMARC1; p=none; rua=mailto:dmarc@dominio.com"

# DMARC enforcement (quarantine)
_dmarc.dominio.com TXT "v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc@dominio.com"

# DMARC enforcement total (rejeitar)
_dmarc.dominio.com TXT "v=DMARC1; p=reject; pct=100; rua=mailto:dmarc@dominio.com"
```

**CAA ausente (autoriza quem pode emitir cert):**
```
# Permitir apenas Let's Encrypt
dominio.com CAA 0 issue "letsencrypt.org"
dominio.com CAA 0 issuewild "letsencrypt.org"
dominio.com CAA 0 iodef "mailto:security@dominio.com"
```

**Zone Transfer habilitado:**
```bash
# Verificar nos servidores autoritativos
# Se retornar registros = VULNERABILIDADE CRÍTICA

# Correção no BIND
zone "dominio.com" {
    type master;
    allow-transfer { none; };  # ou IPs específicos de slaves
};
```

---

## Domínio / Registrador

### Verificações
```bash
# WHOIS — verificar privacidade, datas, nameservers
whois dominio.com

# Verificar expiração
whois dominio.com | grep -i "expir"

# Verificar subdomínios expostos
amass enum -d dominio.com
subfinder -d dominio.com
```

### Checklist de Segurança do Registrador
- [ ] 2FA habilitado na conta do registrador
- [ ] Bloqueio de transferência ativado
- [ ] WHOIS Privacy Protection ativo
- [ ] Email de contato monitorado
- [ ] Domínio com renovação automática
- [ ] Alerta de expiração configurado (90/30/7 dias)

### Subdomain Takeover
```bash
# Verificar subdomínios apontando para serviços inexistentes
# (ex: CNAME para S3/Heroku/GitHub pages que não existe mais)
subfinder -d dominio.com | while read sub; do
    host $sub 2>/dev/null | grep -E "CNAME|has address"
done

# Verificar com ferramentas específicas
subjack -w subdomains.txt -t 100 -timeout 30 -o results.txt
```

---

## Score por Categoria

| Categoria     | Peso | Verificações                              |
|--------------|------|-------------------------------------------|
| TLS/SSL      | 30%  | Versão, ciphers, expiração, HSTS          |
| SPF/DKIM/DMARC| 25% | Configuração correta, enforcement          |
| DNSSEC       | 15%  | Habilitado e válido                        |
| CAA          | 10%  | Restrição de CA                            |
| Registrador  | 10%  | 2FA, lock, privacidade                    |
| Subdomínios  | 10%  | Takeover, exposição desnecessária          |
