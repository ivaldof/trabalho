# Guia: Configuração Cliente Ubuntu para Domínio Samba AD

## Informações do Ambiente

**Cenário de Rede:**
- Controlador de Domínio (Ubuntu Server): 192.168.10.2 ✅ (já configurado)
- Cliente Ubuntu: 192.168.10.4/24 ⚠️ (vamos configurar)
- Domínio: SAMBA.LOCAL

## FOCO: Configuração do Cliente Ubuntu (192.168.10.4)

### Pré-requisito: Servidor já deve estar funcionando
O servidor Samba AD em 192.168.10.2 já deve estar configurado e rodando.

## Passo 1: Configuração de Rede do Cliente Ubuntu

**IMPORTANTE:** Abra o terminal no seu cliente Ubuntu e execute os comandos abaixo:

```bash
# 1. Verificar interfaces de rede disponíveis
ip -br a
```

Você verá algo como:
```
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s3           UP             192.168.1.100/24
enp0s8           DOWN
```

```bash
# 2. Configurar a rede para o domínio
sudo nano /etc/netplan/01-netcfg.yaml
```

**Cole este conteúdo no arquivo (substitua enp0s8 pela sua interface):**
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
# 3. Aplicar a configuração
sudo netplan apply

# 4. Verificar se o IP foi configurado
ip -br a
```

Agora você deve ver: `enp0s8 UP 192.168.10.4/24`

## Passo 2: Configuração do Nome da Máquina

```bash
# 1. Definir o nome da máquina cliente
sudo hostnamectl set-hostname cliente-ubuntu

# 2. Verificar o nome
hostname
```

```bash
# 3. Configurar o arquivo hosts para reconhecer o domínio
sudo nano /etc/hosts
```

**Substitua TODO o conteúdo do arquivo por:**
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

## Passo 3: Configuração DNS (Muito Importante!)

```bash
# 1. Parar e desabilitar o systemd-resolved
sudo systemctl disable --now systemd-resolved

# 2. Remover o link simbólico
sudo unlink /etc/resolv.conf
```

```bash
# 3. Criar arquivo DNS manual
sudo nano /etc/resolv.conf
```

**Cole este conteúdo:**
```
nameserver 192.168.10.2
nameserver 8.8.8.8
search samba.local
```

```bash
# 4. Proteger o arquivo para não ser alterado
sudo chattr +i /etc/resolv.conf

# 5. Testar se consegue resolver o domínio
ping -c2 samba.local
```

**Se o ping funcionar, continue. Se não funcionar, verifique as configurações anteriores.**

## Passo 4: Instalação dos Pacotes Necessários

```bash
# 1. Atualizar repositórios
sudo apt-get update

# 2. Instalar pacotes para ingressar no domínio
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin packagekit krb5-user
```

**Durante a instalação, você será perguntado sobre Kerberos:**
- **Realm padrão:** `SAMBA.LOCAL` (digite exatamente assim e aperte Enter)
- **Servidor Kerberos:** `dc.samba.local` (digite exatamente assim e aperte Enter)  
- **Servidor administrativo:** `dc.samba.local` (digite exatamente assim e aperte Enter)

**IMPORTANTE:** Digite exatamente como mostrado acima!

## Passo 5: Ingressar no Domínio (A Parte Principal!)

```bash
# 1. Descobrir o domínio
realm discover samba.local
```

**Você deve ver algo como:**
```
samba.local
  type: kerberos
  realm-name: SAMBA.LOCAL
  domain-name: samba.local
  configured: no
  server-software: active-directory
  client-software: sssd
```

```bash
# 2. Ingressar no domínio
sudo realm join --user=administrator samba.local
```

**Será pedida a senha do administrator do servidor Samba.**

```bash
# 3. Verificar se ingressou corretamente
realm list
```

**Deve mostrar que está configurado (configured: kerberos-member)**

## Passo 6: Configuração Final do Sistema

```bash
# 1. Permitir criação automática de diretório home para usuários do domínio
sudo pam-auth-update --enable mkhomedir

# 2. Reiniciar o serviço SSSD
sudo systemctl restart sssd
sudo systemctl enable sssd

# 3. Verificar se SSSD está funcionando
sudo systemctl status sssd
```

## Passo 7: Teste Básico do Cliente

```bash
# 1. Testar se consegue obter informações de um usuário do domínio
# (assumindo que o usuário 'administrator' existe no servidor)
getent passwd administrator

# 2. Testar autenticação Kerberos
kinit administrator@SAMBA.LOCAL
# Digite a senha do administrator

# 3. Verificar ticket Kerberos
klist
```

**Se tudo funcionou até aqui, seu cliente está no domínio!**

---

# AGORA NO SERVIDOR: Criar Usuários e Grupos

**ATENÇÃO:** Os próximos comandos devem ser executados no SERVIDOR (192.168.10.2), não no cliente!

## Passo 8: Criação de Grupos no Servidor

```bash
# SSH para o servidor ou acesse diretamente
ssh usuario@192.168.10.2

# Criar 2 grupos
sudo samba-tool group add vendas
sudo samba-tool group add administracao

# Verificar grupos criados
sudo samba-tool group list | grep -E "(vendas|administracao)"
```

## Passo 9: Criação de Usuários no Servidor

```bash
# Criar 4 usuários
sudo samba-tool user create joao --given-name="João" --surname="Silva"
sudo samba-tool user create maria --given-name="Maria" --surname="Santos"  
sudo samba-tool user create pedro --given-name="Pedro" --surname="Costa"
sudo samba-tool user create ana --given-name="Ana" --surname="Oliveira"

# Verificar usuários criados
sudo samba-tool user list | grep -E "(joao|maria|pedro|ana)"
```

**Anote as senhas que você criar para cada usuário!**

## Passo 10: Associar Usuários aos Grupos

```bash
# Adicionar usuários aos grupos
sudo samba-tool group addmembers vendas joao,maria
sudo samba-tool group addmembers administracao pedro,ana

# João vai pertencer aos DOIS grupos (conforme solicitado)
sudo samba-tool group addmembers administracao joao

# Verificar associações
sudo samba-tool group listmembers vendas
sudo samba-tool group listmembers administracao
```

---

# DE VOLTA AO CLIENTE: Teste de Login

## Passo 11: Testar Login com Usuário do Domínio

**VOLTE para o cliente Ubuntu (192.168.10.4):**

```bash
# 1. Verificar se o cliente consegue "ver" os usuários do domínio
getent passwd joao
getent passwd maria
getent passwd pedro
getent passwd ana

# 2. Testar informações de grupo
id joao
```

**Resultado esperado para `id joao`:**
```
uid=xxxxx(joao) gid=xxxxx(domain users) groups=xxxxx(domain users),xxxxx(vendas),xxxxx(administracao)
```

```bash
# 3. Fazer login via SSH com usuário do domínio
ssh joao@localhost
# Digite a senha do usuário joao
```

**Ou faça logout da interface gráfica e faça login com:**
- Usuário: `joao`
- Senha: (a senha que você definiu)

## Passo 12: Criar Compartilhamentos no Servidor

**VOLTE para o servidor (192.168.10.2):**

```bash
# 1. Criar diretórios para compartilhamento
sudo mkdir -p /samba/publico
sudo mkdir -p /samba/vendas-apenas
sudo mkdir -p /samba/admin-apenas

# 2. Definir permissões básicas
sudo chown -R root:"domain users" /samba/publico
sudo chown -R root:vendas /samba/vendas-apenas  
sudo chown -R root:administracao /samba/admin-apenas

sudo chmod -R 2775 /samba/
```

```bash
# 3. Editar configuração do Samba para adicionar compartilhamentos
sudo nano /etc/samba/smb.conf
```

**Adicione estas seções NO FINAL do arquivo:**

```ini
[publico]
    path = /samba/publico
    browseable = yes
    read only = no
    valid users = @"domain users"
    create mask = 0664
    directory mask = 2775
    comment = Compartilhamento para todos os usuarios

[vendas-apenas]  
    path = /samba/vendas-apenas
    browseable = yes
    read only = no
    valid users = @vendas
    create mask = 0664
    directory mask = 2775
    comment = Apenas para grupo vendas

[admin-apenas]
    path = /samba/admin-apenas
    browseable = yes 
    read only = no
    valid users = @administracao
    create mask = 0664
    directory mask = 2775
    comment = Apenas para grupo administracao
```

```bash
# 4. Reiniciar o Samba
sudo systemctl restart samba-ad-dc

# 5. Verificar se a configuração está correta
testparm
```

## Passo 13: Teste de Acesso aos Compartilhamentos

**DE VOLTA ao cliente (192.168.10.4):**

```bash
# 1. Instalar utilitários para acessar compartilhamentos
sudo apt install cifs-utils

# 2. Testar acesso com usuário João (tem acesso a todos)
smbclient //192.168.10.2/publico -U joao
# Digite a senha, depois digite: ls, quit

smbclient //192.168.10.2/vendas-apenas -U joao  
# Digite a senha, depois digite: ls, quit

smbclient //192.168.10.2/admin-apenas -U joao
# Digite a senha, depois digite: ls, quit

# 3. Testar com usuário Maria (só vendas e público)
smbclient //192.168.10.2/publico -U maria
smbclient //192.168.10.2/vendas-apenas -U maria
smbclient //192.168.10.2/admin-apenas -U maria  # Deve DAR ERRO!

# 4. Testar com usuário Pedro (só admin e público)  
smbclient //192.168.10.2/admin-apenas -U pedro
smbclient //192.168.10.2/vendas-apenas -U pedro  # Deve DAR ERRO!
```

---

# RESUMO FINAL - Checklist das Atividades

## ✅ Atividades Obrigatórias Concluídas:

**1. ✅ Ingressar o cliente Ubuntu ao domínio Samba Active Directory**
- Cliente configurado na rede 192.168.10.4/24
- DNS apontando para o servidor (192.168.10.2)
- Ingressado via `realm join`

**2. ✅ Criar quatro usuários e dois grupos no servidor Samba**
- Usuários: joao, maria, pedro, ana
- Grupos: vendas, administracao

**3. ✅ Associar os usuários aos grupos de forma organizada**
- joao: vendas + administracao (pertence a 2 grupos)
- maria: vendas
- pedro: administracao  
- ana: administracao

**4. ✅ Realizar login no cliente Ubuntu utilizando usuário do domínio**
- Testado via SSH: `ssh joao@localhost`
- Pode fazer login na interface gráfica também

**5. ✅ Criar diretórios compartilhados com permissões distintas**
- `/samba/publico` - todos os usuários do domínio
- `/samba/vendas-apenas` - apenas grupo vendas
- `/samba/admin-apenas` - apenas grupo administracao

**6. ✅ Testar acesso aos compartilhamentos com diferentes usuários**
- João: acessa todos (está nos 2 grupos)
- Maria: acessa público e vendas (só grupo vendas)
- Pedro: acessa público e admin (só grupo administracao)

## 🔧 Comandos de Verificação Rápida

### No Cliente:
```bash
# Verificar se está no domínio
realm list

# Verificar usuários do domínio
getent passwd joao maria pedro ana

# Testar login
ssh joao@localhost
```

### No Servidor:
```bash
# Verificar usuários e grupos
sudo samba-tool user list | grep -E "(joao|maria|pedro|ana)"
sudo samba-tool group list | grep -E "(vendas|administracao)"

# Verificar membros dos grupos
sudo samba-tool group listmembers vendas
sudo samba-tool group listmembers administracao

# Status do serviço
sudo systemctl status samba-ad-dc
```

## 🚨 Problemas Comuns e Soluções

**Problema: "realm discover" não encontra o domínio**
```bash
# Verificar DNS
ping samba.local
nslookup dc.samba.local
```

**Problema: Login não funciona**
```bash
# Reiniciar SSSD no cliente
sudo systemctl restart sssd
sudo systemctl status sssd
```

**Problema: Compartilhamento não acessível**
```bash
# No servidor, verificar configuração
testparm
sudo systemctl restart samba-ad-dc
```

## 📝 Entrega do Trabalho

Documente os seguintes pontos:
1. Screenshots do cliente no domínio (`realm list`)
2. Lista de usuários criados (`sudo samba-tool user list`)
3. Membros dos grupos (`sudo samba-tool group listmembers`)
4. Teste de login com usuário do domínio
5. Acesso aos compartilhamentos com diferentes usuários
6. Configuração de rede do cliente (`ip -br a`)
