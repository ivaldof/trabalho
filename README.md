# Guia Completo: Configuração Cliente Ubuntu em Domínio Samba Active Directory

## Informações do Ambiente

**Cenário de Rede:**
- Controlador de Domínio (Ubuntu Server): 192.168.10.2
- Cliente Ubuntu: 192.168.10.4/24
- Domínio: SAMBA.LOCAL

## Parte 1: Configuração do Servidor Samba AD (Já Configurado)

### Verificação do Servidor
Antes de configurar o cliente, verifique se o servidor está funcionando:

```bash
# No servidor (192.168.10.2)
sudo systemctl status samba-ad-dc
host -t A samba.local
host -t SRV _kerberos._udp.samba.local
```

## Parte 2: Configuração do Cliente Ubuntu (192.168.10.4)

### 2.1 Configuração de Rede do Cliente

```bash
# Verificar interfaces de rede
ip -br a

# Configurar netplan
sudo nano /etc/netplan/01-netcfg.yaml
```

**Conteúdo do arquivo netplan:**
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
```

### 2.2 Configuração do Hostname do Cliente

```bash
# Definir hostname
sudo hostnamectl set-hostname cliente-ubuntu

# Configurar /etc/hosts
sudo nano /etc/hosts
```

**Conteúdo do /etc/hosts:**
```
127.0.0.1    localhost
127.0.1.1    cliente-ubuntu.samba.local cliente-ubuntu
192.168.10.2  dc.samba.local dc
192.168.10.2  samba.local samba

::1      ip6-localhost ip6-loopback
fe00::0  ip6-localnet
ff00::0  ip6-mcastprefix
ff02::1  ip6-allnodes
ff02::2  ip6-allrouters
```

### 2.3 Configuração DNS do Cliente

```bash
# Desativar systemd-resolved
sudo systemctl disable --now systemd-resolved
sudo unlink /etc/resolv.conf

# Configurar DNS manualmente
sudo nano /etc/resolv.conf
```

**Conteúdo do /etc/resolv.conf:**
```
nameserver 192.168.10.2
nameserver 8.8.8.8
search samba.local
```

```bash
# Proteger o arquivo
sudo chattr +i /etc/resolv.conf
```

### 2.4 Instalação de Pacotes no Cliente

```bash
sudo apt-get update
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin packagekit krb5-user
```

**Durante a instalação do Kerberos:**
- Realm: SAMBA.LOCAL
- Servidor Kerberos: dc.samba.local
- Servidor administrativo: dc.samba.local

### 2.5 Descoberta e Ingresso no Domínio

```bash
# Descobrir o domínio
realm discover samba.local

# Ingressar no domínio
sudo realm join --user=administrator samba.local
```

### 2.6 Configuração do SSSD

```bash
# Verificar configuração do SSSD
sudo nano /etc/sssd/sssd.conf
```

**Configuração recomendada:**
```ini
[sssd]
domains = samba.local
config_file_version = 2
services = nss, pam

[domain/samba.local]
ad_domain = samba.local
krb5_realm = SAMBA.LOCAL
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = False
fallback_homedir = /home/%u
access_provider = ad
```

```bash
# Reiniciar serviços
sudo systemctl restart sssd
sudo systemctl enable sssd
```

### 2.7 Configuração do PAM para Login

```bash
# Configurar criação automática de diretório home
sudo pam-auth-update --enable mkhomedir
```

## Parte 3: Criação de Usuários e Grupos no Servidor

### 3.1 Criação de Grupos

```bash
# No servidor (192.168.10.2)
sudo samba-tool group add vendas
sudo samba-tool group add administracao
```

### 3.2 Criação de Usuários

```bash
# Criar 4 usuários
sudo samba-tool user create joao --given-name="João" --surname="Silva"
sudo samba-tool user create maria --given-name="Maria" --surname="Santos"
sudo samba-tool user create pedro --given-name="Pedro" --surname="Costa"
sudo samba-tool user create ana --given-name="Ana" --surname="Oliveira"
```

### 3.3 Associação de Usuários aos Grupos

```bash
# Adicionar usuários aos grupos
sudo samba-tool group addmembers vendas joao,maria
sudo samba-tool group addmembers administracao pedro,ana

# João pertence a ambos os grupos
sudo samba-tool group addmembers administracao joao

# Verificar membros dos grupos
sudo samba-tool group listmembers vendas
sudo samba-tool group listmembers administracao
```

## Parte 4: Teste de Login no Cliente

### 4.1 Verificação de Conectividade

```bash
# No cliente
ping dc.samba.local
kinit administrator@SAMBA.LOCAL
klist
```

### 4.2 Login com Usuário do Domínio

```bash
# Testar autenticação
getent passwd joao
id joao

# Login via interface gráfica ou SSH
ssh joao@localhost
```

## Parte 5: Configuração de Compartilhamentos

### 5.1 Criação de Diretórios no Servidor

```bash
# No servidor
sudo mkdir -p /samba/compartilhado
sudo mkdir -p /samba/vendas
sudo mkdir -p /samba/administracao

# Definir permissões
sudo chown -R root:"domain users" /samba/compartilhado
sudo chown -R root:vendas /samba/vendas
sudo chown -R root:administracao /samba/administracao

sudo chmod -R 2775 /samba/
```

### 5.2 Configuração do Samba para Compartilhamentos

```bash
# Editar configuração do Samba
sudo nano /etc/samba/smb.conf
```

**Adicionar ao final do arquivo:**
```ini
[compartilhado]
    path = /samba/compartilhado
    browseable = yes
    read only = no
    valid users = @"domain users"
    create mask = 0664
    directory mask = 2775

[vendas]
    path = /samba/vendas
    browseable = yes
    read only = no
    valid users = @vendas
    create mask = 0664
    directory mask = 2775

[administracao]
    path = /samba/administracao
    browseable = yes
    read only = no
    valid users = @administracao
    create mask = 0664
    directory mask = 2775
```

```bash
# Reiniciar serviço
sudo systemctl restart samba-ad-dc
```

## Parte 6: Testes de Acesso aos Compartilhamentos

### 6.1 Teste via Linha de Comando

```bash
# No cliente, testar com diferentes usuários
smbclient //192.168.10.2/compartilhado -U joao
smbclient //192.168.10.2/vendas -U maria
smbclient //192.168.10.2/administracao -U pedro
```

### 6.2 Montagem de Compartilhamentos

```bash
# Instalar utilitários
sudo apt install cifs-utils

# Criar pontos de montagem
sudo mkdir -p /mnt/compartilhado
sudo mkdir -p /mnt/vendas
sudo mkdir -p /mnt/administracao

# Montar compartilhamentos
sudo mount -t cifs //192.168.10.2/compartilhado /mnt/compartilhado -o username=joao,domain=samba.local
sudo mount -t cifs //192.168.10.2/vendas /mnt/vendas -o username=maria,domain=samba.local
sudo mount -t cifs //192.168.10.2/administracao /mnt/administracao -o username=pedro,domain=samba.local
```

### 6.3 Teste de Permissões

```bash
# Testar criação de arquivos
echo "Teste João" | sudo tee /mnt/compartilhado/arquivo_joao.txt
echo "Teste Maria" | sudo tee /mnt/vendas/arquivo_maria.txt
echo "Teste Pedro" | sudo tee /mnt/administracao/arquivo_pedro.txt

# Listar arquivos
ls -la /mnt/compartilhado/
ls -la /mnt/vendas/
ls -la /mnt/administracao/
```

## Comandos de Verificação e Troubleshooting

### Verificações no Servidor

```bash
# Status dos serviços
sudo systemctl status samba-ad-dc
sudo systemctl status chronyd

# Verificar usuários e grupos
sudo samba-tool user list
sudo samba-tool group list

# Testar configuração
testparm
```

### Verificações no Cliente

```bash
# Status do SSSD
sudo systemctl status sssd

# Verificar resolução DNS
nslookup dc.samba.local
dig samba.local

# Testar Kerberos
kinit joao@SAMBA.LOCAL
klist

# Verificar informações do domínio
realm list
```

### Logs Importantes

```bash
# Logs do SSSD no cliente
sudo tail -f /var/log/sssd/sssd.log
sudo tail -f /var/log/sssd/sssd_samba.local.log

# Logs do Samba no servidor
sudo tail -f /var/log/samba/log.smbd
sudo tail -f /var/log/samba/log.samba-ad-dc
```

## Resumo das Atividades Obrigatórias

✅ **1. Ingresso do cliente Ubuntu no domínio:** Configurado via `realm join`

✅ **2. Criação de usuários e grupos:** 4 usuários (joao, maria, pedro, ana) e 2 grupos (vendas, administracao)

✅ **3. Associação de usuários aos grupos:** João pertence a ambos os grupos

✅ **4. Login no cliente com usuário do domínio:** Testado via SSH e interface gráfica

✅ **5. Compartilhamentos com permissões por grupo:** 3 compartilhamentos configurados

✅ **6. Teste de acesso aos compartilhamentos:** Verificado com diferentes usuários

## Observações Finais

- Certifique-se de que os firewalls estejam configurados adequadamente
- Mantenha sincronização de tempo entre servidor e cliente
- Documente as senhas criadas para os usuários
- Realize backups das configurações importantes
- Teste regularmente a conectividade e autenticação
