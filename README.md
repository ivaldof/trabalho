# Guia Detalhado: Configuração Samba AD - Servidor e Cliente

## 🖥️ CENÁRIO DO TRABALHO

**Você tem:**
- **SERVIDOR** Ubuntu Server (192.168.10.2) - Samba AD já configurado ✅
- **CLIENTE** Ubuntu Desktop (192.168.10.4) - Precisa configurar ❌

**Objetivo:** Fazer o cliente ingressar no domínio e criar usuários/compartilhamentos

---

# 🔧 PARTE 1: CONFIGURAÇÕES NO SERVIDOR (192.168.10.2)

> **ONDE:** Acesse o servidor Ubuntu Server em 192.168.10.2
> **COMO:** Via SSH `ssh ivaldo@192.168.10.2` ou diretamente no servidor

## Passo S1: Criar Grupos no Servidor

```bash
# Entrar no servidor
ssh ivaldo@192.168.10.2

# Criar os 2 grupos solicitados
sudo samba-tool group add vendas
sudo samba-tool group add administracao

# Verificar se foram criados
sudo samba-tool group list | grep -E "(vendas|administracao)"
```

**✅ Resultado esperado:**
```
vendas
administracao
```

## Passo S2: Criar Usuários no Servidor

```bash
# Criar os 4 usuários solicitados
sudo samba-tool user create joao --given-name="João" --surname="Silva"
# Vai pedir senha - ANOTE A SENHA!

sudo samba-tool user create maria --given-name="Maria" --surname="Santos"
# Vai pedir senha - ANOTE A SENHA!

sudo samba-tool user create pedro --given-name="Pedro" --surname="Costa"  
# Vai pedir senha - ANOTE A SENHA!

sudo samba-tool user create ana --given-name="Ana" --surname="Oliveira"
# Vai pedir senha - ANOTE A SENHA!

# Verificar se foram criados
sudo samba-tool user list | grep -E "(joao|maria|pedro|ana)"
```

**📝 IMPORTANTE:** Anote as senhas que você criar para cada usuário!

## Passo S3: Associar Usuários aos Grupos

```bash
# Grupo VENDAS: João e Maria
sudo samba-tool group addmembers vendas joao,maria

# Grupo ADMINISTRAÇÃO: Pedro e Ana  
sudo samba-tool group addmembers administracao pedro,ana

# João também vai para ADMINISTRAÇÃO (ele fica nos 2 grupos)
sudo samba-tool group addmembers administracao joao

# VERIFICAR as associações
echo "=== MEMBROS DO GRUPO VENDAS ==="
sudo samba-tool group listmembers vendas

echo "=== MEMBROS DO GRUPO ADMINISTRAÇÃO ==="
sudo samba-tool group listmembers administracao
```

**✅ Resultado esperado:**
- **Vendas:** joao, maria
- **Administração:** pedro, ana, joao

## Passo S4: Criar Diretórios para Compartilhamento

```bash
# Criar diretórios
sudo mkdir -p /samba/publico
sudo mkdir -p /samba/vendas-apenas
sudo mkdir -p /samba/admin-apenas

# Definir proprietários dos diretórios
sudo chown -R root:"domain users" /samba/publico
sudo chown -R root:vendas /samba/vendas-apenas
sudo chown -R root:administracao /samba/admin-apenas

# Definir permissões
sudo chmod -R 2775 /samba/

# Verificar se foram criados
ls -la /samba/
```

## Passo S5: Configurar Compartilhamentos no Samba

```bash
# Fazer backup da configuração atual
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup

# Editar arquivo de configuração
sudo nano /etc/samba/smb.conf
```

**📝 ADICIONE estas seções NO FINAL do arquivo:**

```ini
[publico]
    path = /samba/publico
    browseable = yes
    read only = no
    valid users = @"domain users"
    create mask = 0664
    directory mask = 2775
    comment = Acesso para todos os usuarios do dominio

[vendas-apenas]
    path = /samba/vendas-apenas
    browseable = yes
    read only = no
    valid users = @vendas
    create mask = 0664
    directory mask = 2775
    comment = Acesso apenas para o grupo vendas

[admin-apenas]
    path = /samba/admin-apenas
    browseable = yes
    read only = no
    valid users = @administracao
    create mask = 0664
    directory mask = 2775
    comment = Acesso apenas para o grupo administracao
```

```bash
# Salvar e sair (Ctrl+X, Y, Enter)

# Testar configuração
testparm

# Reiniciar Samba
sudo systemctl restart samba-ad-dc

# Verificar se está rodando
sudo systemctl status samba-ad-dc
```

**✅ O servidor está configurado!**

---

# 💻 PARTE 2: CONFIGURAÇÕES NO CLIENTE (192.168.10.4)

> **ONDE:** Acesse o cliente Ubuntu Desktop em 192.168.10.4
> **COMO:** Diretamente na máquina cliente

## Passo C1: Configurar Rede do Cliente

```bash
# Verificar interfaces de rede
ip -br a
```

**📝 Você verá algo como:**
```
lo               UNKNOWN        127.0.0.1/8
enp0s3           UP             192.168.1.100/24  
enp0s8           DOWN           (esta é a que vamos configurar)
```

```bash
# Configurar rede
sudo nano /etc/netplan/01-netcfg.yaml
```

**🔄 SUBSTITUA todo o conteúdo por:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses: [192.168.10.4/24]
      nameservers:
        addresses: [192.168.10.2, 8.8.8.8]
        search: [samba.local]
```

```bash
# Aplicar configuração
sudo netplan apply

# Verificar se funcionou  
ip -br a | grep 192.168.10.4
```

**✅ Deve mostrar:** `enp0s8 UP 192.168.10.4/24`

## Passo C2: Configurar Nome da Máquina Cliente

```bash
# Definir hostname
sudo hostnamectl set-hostname cliente-ubuntu

# Verificar
hostname
```

```bash
# Configurar arquivo hosts
sudo nano /etc/hosts
```

**🔄 SUBSTITUA todo o conteúdo por:**
```
127.0.0.1    localhost
127.0.1.1    cliente-ubuntu  
192.168.10.2  dc.samba.local dc samba.local

::1      ip6-localhost ip6-loopback
fe00::0  ip6-localnet
ff00::0  ip6-mcastprefix
ff02::1  ip6-allnodes
ff02::2  ip6-allrouters
```

## Passo C3: Configurar DNS do Cliente

```bash
# Parar serviço systemd-resolved
sudo systemctl disable --now systemd-resolved

# Remover link simbólico
sudo unlink /etc/resolv.conf

# Criar arquivo DNS manual
sudo nano /etc/resolv.conf
```

**📝 CONTEÚDO do arquivo:**
```
nameserver 192.168.10.2
nameserver 8.8.8.8
search samba.local
```

```bash
# Proteger arquivo
sudo chattr +i /etc/resolv.conf

# TESTAR conectividade com servidor
ping -c3 192.168.10.2
ping -c3 samba.local
```

**✅ Se o ping funcionar, continue!**

## Passo C4: Instalar Pacotes para Domínio

```bash
# Atualizar repositórios
sudo apt-get update

# Instalar pacotes necessários
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin packagekit krb5-user
```

**🔔 DURANTE A INSTALAÇÃO, responda:**
- **Realm padrão Kerberos:** `SAMBA.LOCAL`
- **Servidor Kerberos:** `dc.samba.local`
- **Servidor administrativo:** `dc.samba.local`

## Passo C5: Ingressar no Domínio

```bash
# Descobrir o domínio
realm discover samba.local
```

**✅ Deve mostrar algo como:**
```
samba.local
  type: kerberos
  realm-name: SAMBA.LOCAL
  domain-name: samba.local
  configured: no
  server-software: active-directory
```

```bash
# Ingressar no domínio
sudo realm join --user=administrator samba.local
```

**🔑 Digite a senha do administrator do servidor**

```bash
# Verificar se ingressou
realm list
```

**✅ Deve mostrar:** `configured: kerberos-member`

## Passo C6: Configurar Sistema para Usuários do Domínio

```bash
# Permitir criação automática de home
sudo pam-auth-update --enable mkhomedir

# Reiniciar SSSD
sudo systemctl restart sssd
sudo systemctl enable sssd

# Verificar status
sudo systemctl status sssd
```

## Passo C7: Testar Usuários do Domínio

```bash
# Testar se consegue ver usuários do servidor
getent passwd joao
getent passwd maria  
getent passwd pedro
getent passwd ana

# Testar informações de grupo do João (deve estar nos 2 grupos)
id joao
```

**✅ Resultado esperado para `id joao`:**
```
uid=xxxxx(joao) gid=xxxxx(domain users) groups=xxxxx(domain users),xxxxx(vendas),xxxxx(administracao)
```

## Passo C8: Testar Login com Usuário do Domínio

```bash
# Testar login via SSH
ssh joao@localhost
# Digite a senha do joao

# Verificar diretório home foi criado
ls /home/
```

**🖥️ TAMBÉM pode fazer logout da interface gráfica e login com:**
- **Usuário:** `joao`
- **Senha:** (senha que você definiu)

---

# 📁 PARTE 3: TESTAR COMPARTILHAMENTOS

> **ONDE:** No cliente Ubuntu Desktop (192.168.10.4)

## Passo T1: Instalar Utilitários de Rede

```bash
# Instalar smbclient
sudo apt install cifs-utils smbclient
```

## Passo T2: Testar Acesso aos Compartilhamentos

```bash
# TESTE 1: João (deve acessar TODOS os compartilhamentos)
echo "=== TESTE COM JOÃO (todos os compartilhamentos) ==="
smbclient //192.168.10.2/publico -U joao
# Digite senha, depois: ls, quit

smbclient //192.168.10.2/vendas-apenas -U joao
# Digite senha, depois: ls, quit

smbclient //192.168.10.2/admin-apenas -U joao  
# Digite senha, depois: ls, quit

echo "✅ João deve conseguir acessar todos!"
```

```bash
# TESTE 2: Maria (só público e vendas)
echo "=== TESTE COM MARIA (público e vendas apenas) ==="
smbclient //192.168.10.2/publico -U maria
# Digite senha, depois: ls, quit

smbclient //192.168.10.2/vendas-apenas -U maria
# Digite senha, depois: ls, quit

smbclient //192.168.10.2/admin-apenas -U maria
# Deve DAR ERRO de acesso negado ❌

echo "✅ Maria deve acessar público e vendas, mas NÃO admin!"
```

```bash
# TESTE 3: Pedro (só público e admin)
echo "=== TESTE COM PEDRO (público e admin apenas) ==="
smbclient //192.168.10.2/publico -U pedro
# Digite senha, depois: ls, quit

smbclient //192.168.10.2/admin-apenas -U pedro
# Digite senha, depois: ls, quit

smbclient //192.168.10.2/vendas-apenas -U pedro  
# Deve DAR ERRO de acesso negado ❌

echo "✅ Pedro deve acessar público e admin, mas NÃO vendas!"
```

---

# ✅ CHECKLIST FINAL - TRABALHO COMPLETO

## No Servidor (192.168.10.2):
- [x] ✅ Criados 2 grupos: `vendas` e `administracao`
- [x] ✅ Criados 4 usuários: `joao`, `maria`, `pedro`, `ana` 
- [x] ✅ João pertence a 2 grupos (vendas + administracao)
- [x] ✅ Criados 3 compartilhamentos com permissões diferentes
- [x] ✅ Configuração do Samba funcionando

## No Cliente (192.168.10.4):
- [x] ✅ Rede configurada (192.168.10.4/24)
- [x] ✅ DNS apontando para servidor (192.168.10.2)
- [x] ✅ Cliente ingressado no domínio SAMBA.LOCAL
- [x] ✅ Login funcionando com usuários do domínio
- [x] ✅ Acesso aos compartilhamentos testado

## Atividades Obrigatórias:
1. ✅ **Ingressar cliente no domínio:** Feito via `realm join`
2. ✅ **Criar 4 usuários e 2 grupos:** joao, maria, pedro, ana + vendas, administracao  
3. ✅ **Usuário em 2 grupos:** João está em vendas E administracao
4. ✅ **Login no cliente:** Testado com usuários do domínio
5. ✅ **Compartilhamentos com permissões:** 3 compartilhamentos diferentes
6. ✅ **Testar acessos:** Validado que permissões funcionam

---

# 🚨 COMANDOS DE VERIFICAÇÃO RÁPIDA

## Verificar no Servidor:
```bash
# Usuários e grupos
sudo samba-tool user list | grep -E "(joao|maria|pedro|ana)"
sudo samba-tool group listmembers vendas
sudo samba-tool group listmembers administracao

# Serviço funcionando
sudo systemctl status samba-ad-dc
```

## Verificar no Cliente:
```bash
# Domínio
realm list

# Usuários do domínio
getent passwd joao maria pedro ana

# Grupos do João
id joao
```

**🎉 TRABALHO COMPLETO!** Todas as atividades obrigatórias foram executadas!
