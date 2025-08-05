Com base nas imagens que você forneceu, aqui está a documentação com os comandos e suas respectivas descrições:

### Instalação e Configuração Inicial

**Instalar SSH:**

```bash
sudo apt-get install ssh
```

**Verificar status do SSH:**

```bash
sudo systemctl status ssh
```

**Mudar o hostname:**

```bash
sudo hostnamectl set-hostname ud101
sudo hostname -f
```

**Configurar o arquivo /etc/hosts:**

```bash
sudo nano /etc/hosts
# Adicione as linhas:
# 192.168.1.8 clockwork.local clockwork
# 192.168.1.8 dc.clockwork.local dc
```

**Verificar conectividade (ping):**

```bash
ping -c2 clockwork.local
```

**Instalar e configurar NTPDATE:**

```bash
sudo apt-get install ntpdate
sudo ntpdate -q clockwork.local
sudo ntpdate clockwork.local
```

**Instalar pacotes necessários (Samba, Kerberos, Winbind):**

```bash
sudo apt-get install samba krb5-config krb5-user winbind libpam-winbind libnss-winbind libpam-krb5
```

**Verificar autenticação Kerberos:**

```bash
kinit administrator@CLOCKWORK.LOCAL
klist
```

**Fazer backup e criar novo arquivo de configuração do Samba:**

```bash
mv /etc/samba/smb.conf /etc/samba/smb.conf.initial
nano /etc/samba/smb.conf
```

-----

### Configuração do Samba e Autenticação

**Configurar o arquivo `/etc/samba/smb.conf` (conteúdo parcial):**

```ini
[global]
    workgroup = CLOCKWORK
    realm = CLOCKWORK.LOCAL
    netbios name = ud101
    security = ADS
    dns forwarder = 192.168.1.8
    idmap config * : backend = tdb
    idmap config * : range = 50000-1000000
    template homedir = /home/%D/%U
    template shell = /bin/bash
    winbind use default domain = true
    winbind offline logon = false
    winbind nss info = rfc2307
    winbind enum users = yes
    winbind enum groups = yes
    vfs objects = acl_xattr
    map acl inherit = Yes
    store dos attributes = Yes
```

**Reiniciar os serviços do Samba:**

```bash
sudo systemctl restart smbd nmbd
```

**Parar serviços desnecessários:**

```bash
sudo systemctl stop samba-ad-dc
```

**Habilitar serviços do Samba para iniciar com o sistema:**

```bash
sudo systemctl enable smbd nmbd
```

**Unir o Ubuntu ao domínio SAMBA AD:**

```bash
sudo net ads join -U administrator
```

**Listar computadores no SAMBA AD:**

```bash
sudo samba-tool computer list
```

-----

### Configurar a Autenticação de Contas AD

**Editar o arquivo `/etc/nsswitch.conf`:**

```bash
sudo nano /etc/nsswitch.conf
# Altere as linhas para:
# passwd: compat winbind
# group: compat winbind
# shadow: compat winbind
# hosts: files dns
```

**Reiniciar o serviço Winbind:**

```bash
sudo systemctl restart winbind
```

**Listar usuários e grupos do domínio:**

```bash
wbinfo -u
wbinfo -g
```

**Verificar o módulo Winbind com o comando `getent`:**

```bash
sudo getent passwd | grep administrator
sudo getent group|grep 'domain admins'
```

-----

### Configuração Adicional de Autenticação e Permissões

**Configurar `pam-auth-update` para autenticação com contas de domínio:**

```bash
sudo pam-auth-update
```

**Editar o arquivo `/etc/pam.d/common-account` para criar diretórios home automaticamente:**

```bash
nano /etc/pam.d/common-account
# Adicionar a linha no final do arquivo:
# session required pam_mkhomedir.so skel=/etc/skel/ umask=0022
```

**Autenticar-se com a conta Samba4 AD:**

```bash
su administrator
```

**Adicionar conta de domínio com privilégios de root:**

```bash
sudo usermod -aG sudo administrator
```

**Autenticar-se com GUI (comando de exemplo):**

```bash
administrator@clockwork.local
```
