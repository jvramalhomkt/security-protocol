# Referência: Segurança de Bancos de Dados

## PostgreSQL

### Diagnóstico Completo
```sql
-- Verificar versão
SELECT version();

-- Listar usuários e roles
SELECT usename, usesuper, usecreatedb, usecreaterole, passwd
FROM pg_user;

-- Verificar usuários com senha padrão ou vazia
SELECT usename FROM pg_shadow WHERE passwd IS NULL OR passwd = 'md5' || md5('');

-- Verificar permissões excessivas
SELECT grantee, privilege_type, table_name
FROM information_schema.role_table_grants
WHERE grantee NOT IN ('postgres', 'pg_catalog')
ORDER BY grantee;

-- Verificar conexões externas permitidas
SHOW listen_addresses;
SHOW port;

-- Verificar logs habilitados
SHOW log_connections;
SHOW log_failed_connections;
SHOW log_statement;
```

```bash
# Verificar pg_hba.conf (controle de acesso)
cat /etc/postgresql/*/main/pg_hba.conf

# Verificar se postgres está escutando externamente
ss -tlnp | grep 5432
netstat -tlnp | grep 5432
```

### 🔴 CRÍTICO

#### Acesso externo irrestrito
```bash
# pg_hba.conf perigoso:
# host all all 0.0.0.0/0 md5   ← QUALQUER IP pode conectar

# Correto — restringir por IP:
# host all all 127.0.0.1/32 md5
# host myapp myapp_user 10.0.0.0/8 scram-sha-256
```

#### Usuário postgres com senha padrão
```sql
-- Trocar senha imediatamente
ALTER USER postgres WITH PASSWORD 'senha_muito_forte_e_aleatoria';
```

#### Permissões PUBLIC excessivas
```sql
-- Revogar permissões padrão do schema public
REVOKE ALL ON DATABASE mydb FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM PUBLIC;

-- Criar role específica para a aplicação
CREATE ROLE app_role;
GRANT CONNECT ON DATABASE mydb TO app_role;
GRANT USAGE ON SCHEMA public TO app_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_role;
```

### 🟠 ALTO

#### SSL não habilitado
```bash
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_min_protocol_version = 'TLSv1.2'

# pg_hba.conf — exigir SSL
hostssl all all 0.0.0.0/0 scram-sha-256
```

#### Logs insuficientes
```bash
# postgresql.conf — habilitar auditoria
log_connections = on
log_disconnections = on
log_failed_connections = on
log_statement = 'ddl'        # ou 'mod' para INSERT/UPDATE/DELETE
log_duration = on
log_min_duration_statement = 1000  # queries > 1s
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

---

## MySQL / MariaDB

### Diagnóstico Completo
```sql
-- Verificar usuários
SELECT user, host, authentication_string FROM mysql.user;

-- Verificar usuários sem senha
SELECT user, host FROM mysql.user WHERE authentication_string = '' OR authentication_string IS NULL;

-- Verificar permissões globais
SELECT user, host, Super_priv, File_priv, Grant_priv FROM mysql.user WHERE Super_priv='Y' OR File_priv='Y';

-- Verificar banco de dados test
SHOW DATABASES LIKE 'test';

-- Verificar SSL
SHOW VARIABLES LIKE '%ssl%';
```

```bash
# Executar mysql_secure_installation (remediação automática)
mysql_secure_installation

# Verificar porta e binding
grep -E "bind-address|port" /etc/mysql/mysql.conf.d/mysqld.cnf
```

### 🔴 CRÍTICO

```bash
# Hardening imediato
mysql -u root -p << EOF
-- Remover usuários anônimos
DELETE FROM mysql.user WHERE User='';

-- Remover acesso remoto ao root
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');

-- Remover banco test
DROP DATABASE IF EXISTS test;
DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';

FLUSH PRIVILEGES;
EOF
```

```ini
# my.cnf — hardening
[mysqld]
bind-address = 127.0.0.1  # apenas local
local-infile = 0           # desabilitar LOAD DATA LOCAL
skip-symbolic-links = 1
ssl-ca = /etc/mysql/ca.pem
ssl-cert = /etc/mysql/server-cert.pem
ssl-key = /etc/mysql/server-key.pem
```

---

## MongoDB

### Diagnóstico Completo
```bash
# Verificar autenticação habilitada
mongosh --eval "db.adminCommand({getCmdLineOpts: 1})" | grep auth

# Verificar se está escutando externamente sem auth
ss -tlnp | grep 27017

# Testar acesso sem credenciais (CRÍTICO se funcionar)
mongosh --host localhost --eval "db.adminCommand('listDatabases')"
```

### 🔴 CRÍTICO

```yaml
# mongod.conf — habilitar autenticação
security:
  authorization: enabled

net:
  bindIp: 127.0.0.1  # nunca 0.0.0.0 sem auth
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/mongoCA.pem
```

```javascript
// Criar usuário admin
use admin
db.createUser({
  user: "adminUser",
  pwd: passwordPrompt(),
  roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
})

// Criar usuário da aplicação com permissões mínimas
use myapp
db.createUser({
  user: "appUser",
  pwd: passwordPrompt(),
  roles: [ { role: "readWrite", db: "myapp" } ]
})
```

---

## Redis

### Diagnóstico Completo
```bash
# Verificar se está acessível sem senha
redis-cli ping
redis-cli -h localhost INFO server

# Verificar binding
grep -E "bind|requirepass|protected-mode" /etc/redis/redis.conf
```

### 🔴 CRÍTICO

```bash
# redis.conf — hardening
bind 127.0.0.1          # nunca 0.0.0.0
requirepass SenhaFortíssima123!@#
protected-mode yes
rename-command CONFIG ""        # desabilitar comando CONFIG
rename-command FLUSHALL ""      # desabilitar FLUSHALL
rename-command FLUSHDB ""       # desabilitar FLUSHDB
rename-command DEBUG ""
```

---

## Boas Práticas Gerais

### Secrets de Conexão
```bash
# ❌ Nunca no código
DATABASE_URL=postgres://admin:senha123@db:5432/prod

# ✅ Variáveis de ambiente ou secrets manager
# .env (nunca commitado)
# Docker Secrets
# Vault (HashiCorp)
# AWS Secrets Manager
# Doppler
```

### Backups
```bash
# PostgreSQL — backup criptografado
pg_dump mydb | gpg --symmetric --cipher-algo AES256 > backup_$(date +%Y%m%d).sql.gpg

# MySQL
mysqldump --single-transaction mydb | gzip > backup_$(date +%Y%m%d).sql.gz

# Testar restore regularmente!
```

### Monitoramento de Acesso Suspeito
```bash
# Instalar pgaudit (PostgreSQL)
CREATE EXTENSION pgaudit;
SET pgaudit.log = 'write, ddl';
```

---

## Score por Categoria

| Categoria        | Peso | Verificações                              |
|-----------------|------|-------------------------------------------|
| Autenticação     | 30%  | Senhas fortes, sem anônimos               |
| Acesso de rede   | 25%  | Bind local, firewall                      |
| Permissões       | 20%  | Least privilege por usuário/app           |
| Criptografia     | 15%  | SSL/TLS em trânsito, crypt at rest        |
| Auditoria/Logs   | 10%  | Conexões, queries DDL/DML                 |
