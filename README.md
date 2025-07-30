# Guia Completo: Instalação Samba AD + Configuração Cliente

## 📋 CENÁRIO DO TRABALHO

**Você vai configurar:**
- **SERVIDOR** Ubuntu Server (192.168.10.2) - Instalar Samba AD do zero ❌
- **CLIENTE** Ubuntu Desktop (192.168.10.4) - Ingressar no domínio ❌  
- **Domínio:** SAMBA.LOCAL

---

# 🖥️ PARTE 1: INSTALAÇÃO E CONFIGURAÇÃO DO SERVIDOR SAMBA AD

> **ONDE:** Servidor Ubuntu Server em 192.168.10.2
> **USUÁRIO:** ivaldo

## Passo S1: Configuração Inicial do Servidor

```bash
# Acessar o servidor (se estiver via SSH)
ssh ivaldo@192.168.10.2

# OU trabalhe diretamente no servidor
```

### Configurar Rede do Servidor

```bash
# Verificar interfaces
ip -br a

# Configurar rede estática
sudo nano /etc/netplan/01-netcfg.yaml
```

**📝 Conteúdo do arquivo:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses: [192.168.10.2/24]
      nameservers:
        addresses: [8.8.8.8]
```

```bash
# Aplicar configuração
sudo netplan apply

# Verificar
ip -br a | grep 192.168.10.2
```

### Configurar Hostname do Servidor

```bash
# Definir hostname
sudo hostnamectl set-hostname dc

# Verificar
hostname
```

```bash
# Configurar /etc/hosts
sudo nano /etc/hosts
```

**📝 Conteúdo:**
```
127.0.0.1    localhost
127.0.1.1    dc
192.168.10.2  dc.samba.local dc
192.168.10.2  samba.local samba

::1      ip6-localhost ip6-loopback
fe00::0  ip6-localnet
ff00::0  ip6-mcastprefix
ff02::1  ip6-allnodes
ff02::2  ip6-allrouters
```

```bash
# Testar resolução
hostname -f
ping -c2 dc.samba.local
```

## Passo S2: Configuração DNS do Servidor

```bash
# Desativar systemd-resolved
sudo systemctl disable --now systemd-resolved
sudo unlink /etc/resolv.conf

# Criar resolv.conf manual
sudo nano /etc/resolv.conf
```

**📝 Conteúdo:**
```
nameserver 192.168.10.2
nameserver 8.8.8.8
search samba.local
```

```bash
# Proteger arquivo
sudo chattr +i /etc/resolv.conf
```

## Passo S3: Instalação dos Pacotes Samba

```bash
# Atualizar sistema
sudo apt-get update

# Instalar todos os pacotes necessários
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules smbclient winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user dnsutils chrony net-tools
```

**🔔 Durante a instalação do Kerberos, configure:**
- **Realm:** `SAMBA.LOCAL`
- **Servidor Kerberos:** `dc.samba.local`
- **Servidor administrativo:** `dc.samba.local`

```bash
# Desativar serviços que não precisamos
sudo systemctl disable --now smbd nmbd winbind

# Habilitar samba-ad-dc
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
```

## Passo S4: Configuração do Active Directory

```bash
# Fazer backup da configuração padrão
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig

# Provisionar o domínio
sudo samba-tool domain provision
```

**🔔 Durante o provisionamento, configure:**
- **Realm:** `SAMBA.LOCAL` (pressione Enter)
- **Domain:** `SAMBA` (pressione Enter)  
- **Server Role:** `dc` (pressione Enter)
- **DNS backend:** `SAMBA_INTERNAL` (pressione Enter)
- **DNS forwarder:** `8.8.8.8`
- **Administrator password:** (crie uma senha FORTE e ANOTE!)
- **Retype password:** (repita a senha)

```bash
# Configurar Kerberos
sudo mv /etc/krb5.conf /etc/krb5.conf.orig
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Iniciar serviço
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```

## Passo S5: Configuração NTP (Sincronização de Tempo)

```bash
# Configurar permissões NTP
sudo chown root:_chrony /var/lib/samba/ntp_signd/
sudo chmod 750 /var/lib/samba/ntp_signd/

# Configurar Chrony
sudo nano /etc/chrony/chrony.conf
```

**📝 Adicionar NO FINAL do arquivo:**
```
bindcmdaddress 192.168.10.2
allow 192.168.10.0/24
ntpsigndsocket /var/lib/samba/ntp_signd
```

```bash
# Reiniciar Chrony
sudo systemctl restart chronyd
sudo systemctl status chronyd
```

## Passo S6: Verificação e Testes do Servidor

```bash
# Testar resolução DNS
host -t A samba.local
host -t A dc.samba.local

# Testar registros Kerberos  
host -t SRV _kerberos._udp.samba.local
host -t SRV _ldap._tcp.samba.local

# Testar recursos Samba
smbclient -L samba.local -N

# Testar autenticação Kerberos
kinit administrator@SAMBA.LOCAL
# Digite a senha do administrator
klist

# Testar acesso SMB
smbclient //localhost/netlogon -U 'administrator'

# Verificar configuração
testparm
sudo samba-tool domain level show
```

**✅ Se todos os testes passaram, o servidor está funcionando!**

## Passo S7: Criar Usuários e Grupos

```bash
# Criar grupos
sudo samba-tool group add vendas
sudo samba-tool group add administracao

# Criar usuários
sudo samba-tool user create joao --given-name="João" --surname="Silva"
sudo samba-tool user create maria --given-name="Maria" --surname="Santos"
sudo samba-tool user create pedro --given-name="Pedro" --surname="Costa"  
sudo samba-tool user create ana --given-name="Ana" --surname="Oliveira"

# Associar usuários aos grupos
sudo samba-tool group addmembers vendas joao,maria
sudo samba-tool group addmembers administracao pedro,ana
sudo samba-tool group addmembers administracao joao

# Verificar
sudo samba-tool group listmembers vendas
sudo samba-tool group listmembers administracao
```

## Passo S8: Criar Compartilhamentos

```bash
# Criar diretórios
sudo mkdir -p /samba/publico
sudo mkdir -p /samba/vendas-apenas
sudo mkdir -p /samba/admin-apenas

# Definir proprietários
sudo chown -R root:"domain users" /samba/publico
sudo chown -R root:vendas /samba/vendas-apenas
sudo chown -R root:administracao /samba/admin-apenas

# Definir permissões
sudo chmod -R 2775 /samba/

# Configurar compartilhamentos
sudo nano /etc/samba/smb.conf
```

**📝 Adicionar NO FINAL do arquivo:**
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
# Testar configuração e reiniciar
testparm
sudo systemctl restart samba-ad-dc
```

**🎉 SERVIDOR SAMBA AD CONFIGURADO COMPLETAMENTE!**

---

# 💻 PARTE 2: CONFIGURAÇÃO DO CLIENTE UBUNTU

> **ONDE:** Cliente Ubuntu Desktop em 192.168.10.4
> **USUÁRIO:** ivaldo

## Passo C1: Configurar Rede do Cliente

```bash
# Verificar interfaces
ip -br a

# Configurar rede
sudo nano /etc/netplan/01-netcfg.yaml
```

**📝 Conteúdo:**
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
# Aplicar e verificar
sudo netplan apply
ip -br a | grep 192.168.10.4
```

## Passo C2: Configurar Hostname do Cliente

```bash
# Definir hostname
sudo hostnamectl set-hostname cliente-ubuntu

# Configurar hosts
sudo nano /etc/hosts
```

**📝 Conteúdo:**
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
# Desativar systemd-resolved
sudo systemctl disable --now systemd-resolved
sudo unlink /etc/resolv.conf

# Criar resolv.conf
sudo nano /etc/resolv.conf
```

**📝 Conteúdo:**
```
nameserver 192.168.10.2
nameserver 8.8.8.8
search samba.local
```

```bash
# Proteger e testar
sudo chattr +i /etc/resolv.conf
ping -c3 samba.local
```

## Passo C4: Instalar Pacotes para Domínio

```bash
# Atualizar
sudo apt-get update

# Instalar pacotes
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin packagekit krb5-user
```

**🔔 Durante instalação Kerberos:**
- **Realm:** `SAMBA.LOCAL`
- **Servidor Kerberos:** `dc.samba.local`
- **Servidor administrativo:** `dc.samba.local`

## Passo C5: Ingressar no Domínio

```bash
# Descobrir domínio
realm discover samba.local

# Ingressar no domínio
sudo realm join --user=administrator samba.local
# Digite a senha do administrator

# Verificar
realm list
```

## Passo C6: Configurar Sistema

```bash
# Habilitar criação de home
sudo pam-auth-update --enable mkhomedir

# Reiniciar SSSD
sudo systemctl restart sssd
sudo systemctl enable sssd
```

## Passo C7: Testar Cliente

```bash
# Testar usuários do domínio
getent passwd joao maria pedro ana

# Testar grupos do João
id joao

# Testar login
ssh joao@localhost
```

---

# 📁 PARTE 3: TESTAR COMPARTILHAMENTOS

```bash
# Instalar utilitários
sudo apt install cifs-utils smbclient

# Testar João (todos os compartilhamentos)
smbclient //192.168.10.2/publico -U joao
smbclient //192.168.10.2/vendas-apenas -U joao  
smbclient //192.168.10.2/admin-apenas -U joao

# Testar Maria (só público e vendas)
smbclient //192.168.10.2/publico -U maria
smbclient //192.168.10.2/vendas-apenas -U maria
smbclient //192.168.10.2/admin-apenas -U maria  # Deve dar erro

# Testar Pedro (só público e admin)
smbclient //192.168.10.2/publico -U pedro
smbclient //192.168.10.2/admin-apenas -U pedro
smbclient //192.168.10.2/vendas-apenas -U pedro  # Deve dar erro
```

---

# ✅ TRABALHO COMPLETO!

## Atividades Obrigatórias:
1. ✅ **Servidor Samba AD instalado e configurado**
2. ✅ **Cliente Ubuntu ingressado no domínio**  
3. ✅ **4 usuários e 2 grupos criados**
4. ✅ **João pertence a 2 grupos**
5. ✅ **Login no cliente funcionando**
6. ✅ **3 compartilhamentos com permissões diferentes**
7. ✅ **Testes de acesso validados**

## 📝 Senhas para Anotar:
- **Administrator:** (senha do domínio)
- **joao:** (senha do usuário)
- **maria:** (senha do usuário) 
- **pedro:** (senha do usuário)
- **ana:** (senha do usuário)

**🎯 Agora você tem o guia completo desde a instalação até os testes finais!**

O erro que está aparecendo várias vezes é:

```
mv: não foi possível obter estado de '/etc/krb5.conf': Arquivo ou diretório inexistente
```

Isso significa que o arquivo `/etc/krb5.conf` **não existe no seu sistema** no momento.

Esse arquivo é essencial para o funcionamento do Kerberos, especialmente em configurações de domínio Samba AD, como a que você está montando.

---

### ✅ Como resolver:

Você precisa **criar o arquivo `/etc/krb5.conf`** com o conteúdo correto. Como o seu domínio é `samba.local` e seu servidor (DC) está com o IP `192.168.10.2`, aqui está um exemplo básico de configuração que deve funcionar:

#### 🛠️ Passo 1 – Criar o arquivo `/etc/krb5.conf`

Execute:

```bash
sudo nano /etc/krb5.conf
```

E cole o seguinte conteúdo:

```ini
[libdefaults]
    default_realm = SAMBA.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    default_tgs_enctypes = arcfour-hmac-md5
    default_tkt_enctypes = arcfour-hmac-md5
    permitted_enctypes = arcfour-hmac-md5

[realms]
    SAMBA.LOCAL = {
        kdc = dc.samba.local
        admin_server = dc.samba.local
    }

[domain_realm]
    .samba.local = SAMBA.LOCAL
    samba.local = SAMBA.LOCAL
```

---

### ✅ Passo 2 – Testar se o Kerberos está funcionando

Depois de salvar o arquivo, teste com:

```bash
kinit Administrator
```

Ele deve pedir a senha do domínio Samba AD.

Se tudo estiver correto, use:

```bash
klist
```

E verá o ticket do Kerberos ativo.

---

O erro mostrado na imagem indica que o cliente Ubuntu não está conseguindo se comunicar com o servidor Kerberos (KDC) do domínio `SAMBA.LOCAL`. Isso geralmente acontece por causa de problemas na resolução de nome ou na configuração incorreta do arquivo `/etc/krb5.conf`.

---

### ✅ Problemas detectados:

1. **Arquivo `/etc/krb5.conf` ausente**:

   * A mensagem mostra:

     ```bash
     mv: não foi possível obter estado de '/etc/krb5.conf': Arquivo ou diretório inexistente
     ```
   * Isso indica que o arquivo de configuração do Kerberos **não existe**, então o `kinit` está tentando se comunicar com um KDC que ele não sabe onde está.

2. **Erro do `kinit`**:

   * A mensagem:

     ```bash
     kinit: Cannot contact any KDC for realm 'SAMBA.LOCAL' while getting initial credentials
     ```
   * Significa que **não foi possível resolver ou alcançar o servidor Kerberos** para o realm/domínio `SAMBA.LOCAL`.

---

### ✅ Solução passo a passo:

#### 1. Criar ou configurar corretamente o arquivo `/etc/krb5.conf`

Crie o arquivo com este conteúdo básico (ajuste se o hostname ou IP do seu DC for diferente):

```ini
[libdefaults]
    default_realm = SAMBA.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = false

[realms]
    SAMBA.LOCAL = {
        kdc = 192.168.10.2
        admin_server = 192.168.10.2
    }

[domain_realm]
    .samba.local = SAMBA.LOCAL
    samba.local = SAMBA.LOCAL
```

> Substitua `192.168.10.2` pelo IP do seu servidor Samba AD, se for diferente.

Salve o arquivo como `/etc/krb5.conf`.

---

#### 2. Testar conectividade e DNS

Verifique se o nome do domínio é resolvido corretamente:

```bash
ping dc.samba.local
```

Ou, se estiver usando somente IP:

```bash
ping 192.168.10.2
```

Você também pode adicionar o domínio ao `/etc/hosts` para garantir que resolva:

```bash
192.168.10.2    dc.samba.local dc
```

---

#### 3. Testar novamente o `kinit`

Depois do ajuste do arquivo e conectividade, teste novamente:

```bash
kinit Administrator
```

Se pedir senha e não exibir erro, está funcionando.

---

Se quiser, posso gerar o conteúdo completo do `/etc/krb5.conf` baseado no seu cenário com Samba AD. Deseja isso?

