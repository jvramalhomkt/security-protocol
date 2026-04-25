# Referência: APIs REST

## Checklist OWASP API Security Top 10

---

### API1 — Broken Object Level Authorization 🔴 CRÍTICO

```bash
# Testar BOLA — acessar recursos de outros usuários
# Autenticar como user1, depois tentar acessar recurso do user2
curl -H "Authorization: Bearer TOKEN_USER1" https://api.dominio.com/users/USER2_ID/data
curl -H "Authorization: Bearer TOKEN_USER1" https://api.dominio.com/orders/ORDER_ID_DE_OUTRO_USER

# Testar com IDs sequenciais
for i in {1..10}; do
  curl -s -H "Authorization: Bearer SEU_TOKEN" https://api.dominio.com/resource/$i | head -1
done
```

**Remediação:** Validar no backend se o recurso solicitado pertence ao usuário autenticado.

---

### API2 — Broken Authentication 🔴 CRÍTICO

```bash
# Verificar JWT
# 1. Decodificar (sem validar): jwt.io ou
echo "SEU_JWT" | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# 2. Testar algoritmo "none"
# Header: {"alg":"none","typ":"JWT"}
# Montar token sem assinatura

# 3. Testar chaves fracas
python3 -c "
import jwt
token = 'SEU_TOKEN_AQUI'
for key in ['secret', 'password', '123456', 'jwt_secret']:
    try:
        decoded = jwt.decode(token, key, algorithms=['HS256'])
        print(f'CHAVE ENCONTRADA: {key}')
        print(decoded)
    except:
        pass
"

# Verificar expiração
curl -H "Authorization: Bearer TOKEN_EXPIRADO" https://api.dominio.com/resource
```

---

### API3 — Broken Object Property Level Authorization 🔴 CRÍTICO

```bash
# Mass Assignment — tentar enviar campos não permitidos
curl -X PATCH https://api.dominio.com/users/ME \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"name": "Novo Nome", "role": "admin", "is_verified": true}'

# Excessive Data Exposure — verificar se API retorna mais do que deveria
curl https://api.dominio.com/users/ME | python3 -m json.tool
```

---

### API4 — Unrestricted Resource Consumption 🟠 ALTO

```bash
# Testar rate limiting
for i in {1..20}; do
  curl -s -o /dev/null -w "%{http_code} " https://api.dominio.com/search?q=test
done

# Testar com payload grande
curl -X POST https://api.dominio.com/upload \
  -H "Content-Type: application/json" \
  -d "$(python3 -c "print('{\"data\": \"' + 'A'*1000000 + '\"}')")"
```

**Remediação:**
```javascript
// Express.js — rate limiting
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutos
  max: 100,
  message: 'Too many requests'
});

app.use('/api/', limiter);
```

---

### API5 — Broken Function Level Authorization 🟠 ALTO

```bash
# Acessar endpoints administrativos sem permissão
curl -H "Authorization: Bearer TOKEN_USER" https://api.dominio.com/admin/users
curl -H "Authorization: Bearer TOKEN_USER" https://api.dominio.com/admin/stats
curl -X DELETE -H "Authorization: Bearer TOKEN_USER" https://api.dominio.com/users/OUTRO_ID

# Testar métodos HTTP não esperados
curl -X DELETE https://api.dominio.com/public-resource
curl -X PUT https://api.dominio.com/public-resource
```

---

### API6 — Unrestricted Access to Sensitive Business Flows 🟠 ALTO

```bash
# Testar bypass de fluxos de negócio
# Ex: comprar produto com preço 0
curl -X POST https://api.dominio.com/checkout \
  -d '{"product_id": 1, "price": 0}'

# Pular etapa de verificação
curl -X POST https://api.dominio.com/payment/confirm \
  -d '{"order_id": 123, "verified": true}'
```

---

### API7 — Server-Side Request Forgery (SSRF) 🟠 ALTO

```bash
# Testar em endpoints que aceitam URLs
curl -X POST https://api.dominio.com/fetch \
  -d '{"url": "http://169.254.169.254/latest/meta-data/"}' # AWS metadata

curl -X POST https://api.dominio.com/webhook \
  -d '{"callback_url": "http://localhost:6379"}' # Redis interno
```

---

### API8 — Security Misconfiguration 🟡 MÉDIO

```bash
# Verificar CORS
curl -H "Origin: https://evil.com" -I https://api.dominio.com

# Verificar headers de segurança na API
curl -I https://api.dominio.com/health

# Verificar endpoints de debug
curl https://api.dominio.com/actuator
curl https://api.dominio.com/actuator/env
curl https://api.dominio.com/.well-known/
curl https://api.dominio.com/api-docs
curl https://api.dominio.com/swagger
```

**Remediação CORS:**
```javascript
// Express.js
const cors = require('cors');

app.use(cors({
  origin: ['https://meusite.com', 'https://app.meusite.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true
}));
```

---

### API9 — Improper Inventory Management 🟡 MÉDIO

```bash
# Descobrir versões antigas da API
curl https://api.dominio.com/v1/users
curl https://api.dominio.com/v2/users
curl https://api.dominio.com/v3/users
curl https://api.dominio.com/beta/users

# Documentação exposta
curl https://api.dominio.com/swagger.json
curl https://api.dominio.com/openapi.json
```

---

### API10 — Unsafe Consumption of APIs 🟡 MÉDIO

- APIs de terceiros são validadas?
- Dados de APIs externas são sanitizados antes de usar?
- Certificados SSL de APIs externas são verificados?

---

## Headers Recomendados para APIs

```
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000
Cache-Control: no-store
Content-Security-Policy: default-src 'none'
```

## Autenticação — Boas Práticas

```bash
# JWT — configurações recomendadas
{
  "alg": "RS256",        # Assimétrico é mais seguro
  "exp": now + 3600,     # Expiração curta (1h para acesso, 7d para refresh)
  "iat": now,
  "jti": uuid(),         # ID único por token (permite revogação)
  "sub": user_id
}
```
