# Referência: VPS / Servidor Linux

## Checklist de Auditoria

### 🔴 CRÍTICO — Verificar primeiro

#### Acesso SSH
```bash
# Verificar configuração atual do SSH
cat /etc/ssh/sshd_config | grep -E "PermitRootLogin|PasswordAuthentication|Port|PubkeyAuthentication|AllowUsers"

# Verificar tentativas de brute force
grep "Failed password" /var/log/auth.log | tail -20
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head -10
```

**Remediações:**
```bash
# Desabilitar login root via SSH
sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# Desabilitar autenticação por senha (usar apenas chaves)
sed -i 's/^PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config

# Mudar porta padrão (segurança por obscuridade, mas útil)
sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config

# Reiniciar SSH (CUIDADO: mantenha sessão ativa)
systemctl restart sshd
```

#### Firewall
```bash
# Verificar status do firewall
ufw status verbose
iptables -L -n -v

# Verificar portas abertas
ss -tulnp
netstat -tulnp
nmap -sV localhost
```

**Remediações:**
```bash
# Configurar UFW (Ubuntu/Debian)
ufw default deny incoming
ufw default allow outgoing
ufw allow 2222/tcp   # SSH (porta customizada)
ufw allow 80/tcp     # HTTP
ufw allow 443/tcp    # HTTPS
ufw enable

# Bloquear IPs com muitas tentativas (fail2ban)
apt install fail2ban -y
systemctl enable fail2ban
systemctl start fail2ban
```

#### Atualizações do Sistema
```bash
# Verificar pacotes desatualizados
apt list --upgradable 2>/dev/null
yum check-update

# Verificar última atualização
stat /var/cache/apt/pkgcache.bin | grep Modify

# Verificar CVEs conhecidos nos pacotes instalados
apt install debsecan -y && debsecan
```

**Remediações:**
```bash
# Atualizar tudo
apt update && apt upgrade -y && apt autoremove -y

# Habilitar atualizações automáticas de segurança
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades
```

---

### 🟠 ALTO

#### Usuários e Permissões
```bash
# Listar usuários com shell
grep -E "/bin/bash|/bin/sh|/bin/zsh" /etc/passwd

# Verificar usuários com senha vazia
awk -F: '($2 == "") {print $1}' /etc/shadow

# Verificar usuários sudo
getent group sudo
cat /etc/sudoers

# Verificar SUID/SGID suspeitos
find / -perm /4000 -o -perm /2000 2>/dev/null | grep -v proc

# Verificar arquivos world-writable
find / -writable -type f 2>/dev/null | grep -v proc | grep -v sys
```

**Remediações:**
```bash
# Remover usuário desnecessário
userdel -r username

# Restringir sudo a usuário específico
echo "username ALL=(ALL:ALL) ALL" >> /etc/sudoers.d/username

# Corrigir permissão perigosa
chmod 644 /arquivo/problematico
chown root:root /arquivo/problematico
```

#### Serviços em Execução
```bash
# Listar todos os serviços ativos
systemctl list-units --type=service --state=active

# Verificar serviços na inicialização
systemctl list-unit-files --type=service | grep enabled

# Verificar processos suspeitos
ps aux --sort=-%cpu | head -20
ps aux --sort=-%mem | head -20
```

**Remediações:**
```bash
# Desabilitar serviço desnecessário
systemctl disable --now nome_servico
```

#### Logs e Auditoria
```bash
# Verificar se auditd está ativo
systemctl status auditd

# Verificar logs de autenticação
tail -100 /var/log/auth.log
last -20  # últimos logins

# Verificar alterações recentes em arquivos críticos
find /etc -newer /etc/passwd -type f 2>/dev/null
find / -newer /tmp -name "*.sh" 2>/dev/null
```

---

### 🟡 MÉDIO

#### Kernel e Parâmetros do Sistema
```bash
# Verificar configurações de segurança do kernel
sysctl -a | grep -E "ip_forward|rp_filter|syn_cookies|icmp_echo_ignore"

# Verificar versão do kernel
uname -r
```

**Remediações:**
```bash
# Aplicar hardening do kernel via sysctl
cat >> /etc/sysctl.conf << EOF
# Proteção contra IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Desabilitar IP forwarding (se não for roteador)
net.ipv4.ip_forward = 0

# Proteção SYN flood
net.ipv4.tcp_syncookies = 1

# Ignorar broadcasts ICMP
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Proteção contra ataques Smurf
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Desabilitar source routing
net.ipv4.conf.all.accept_source_route = 0
EOF

sysctl -p
```

#### Criptografia e Autenticação
```bash
# Verificar algoritmos SSH permitidos
sshd -T | grep -E "ciphers|macs|kexalgorithms"

# Verificar força das senhas
cat /etc/pam.d/common-password
```

**Remediações:**
```bash
# Hardening do SSH — adicionar ao sshd_config
cat >> /etc/ssh/sshd_config << EOF
# Criptografia forte apenas
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512

# Limites de conexão
MaxAuthTries 3
MaxSessions 5
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
EOF
```

---

### 🟢 INFORMATIVO / BOAS PRÁTICAS

#### Monitoramento
```bash
# Instalar ferramentas de monitoramento
apt install -y htop iotop nethogs ncdu

# Verificar uso de disco
df -h
du -sh /* 2>/dev/null | sort -rh | head -10

# Verificar arquivos grandes inesperados
find / -size +100M -type f 2>/dev/null | grep -v proc
```

#### Ferramentas Recomendadas
```bash
# Lynis — auditoria completa do sistema
apt install lynis -y
lynis audit system

# rkhunter — rootkit hunter
apt install rkhunter -y
rkhunter --check

# chkrootkit
apt install chkrootkit -y
chkrootkit

# AIDE — monitoramento de integridade de arquivos
apt install aide -y
aide --init
cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

---

## Score por Categoria

| Categoria        | Peso | Verificações                          |
|-----------------|------|---------------------------------------|
| Acesso SSH       | 25%  | Root login, auth por chave, porta     |
| Firewall         | 20%  | UFW/iptables, portas expostas         |
| Atualizações     | 20%  | Pacotes, kernel, CVEs                 |
| Usuários/Perms   | 15%  | Sudo, SUID, world-writable            |
| Serviços         | 10%  | Serviços desnecessários               |
| Kernel           | 5%   | Parâmetros sysctl                     |
| Logs/Auditoria   | 5%   | auditd, fail2ban                      |
