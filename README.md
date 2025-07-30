Ótimo! Como a configuração do servidor Samba AD já está feita e funcionando, agora vamos ao **passo a passo completo para configurar o Ubuntu Desktop como cliente do domínio Samba AD**, conforme seu cenário:

---

## 🧷 **CLIENTE UBUNTU DESKTOP – Ingressar no Domínio Samba**

### ✅ 1. **Atualizar e Instalar os Pacotes Necessários**

```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli samba-common-bin oddjob oddjob-mkhomedir packagekit krb5-user libnss-sss libpam-sss
```

> Durante a instalação, informe:

* **Realm**: `SAMBA.LOCAL`
* **Servidor KDC**: `dc.samba.local`
* **Administração**: `dc.samba.local`

---

### ✅ 2. **Configurar o DNS do Cliente Ubuntu**

Edite o `/etc/resolv.conf`:

```bash
sudo nano /etc/resolv.conf
```

```txt
nameserver 192.168.10.2
search samba.local
```

Proteja o arquivo:

```bash
sudo chattr +i /etc/resolv.conf
```

---

### ✅ 3. **Testar Resolução DNS**

```bash
ping dc.samba.local
host -t SRV _kerberos._udp.samba.local
```

---

### ✅ 4. **Ingressar no Domínio Samba**

```bash
sudo realm join --user=administrator samba.local
```

> Será solicitada a senha do `administrator` do domínio.
> Se aparecer erro de "realm not found", verifique DNS e hora do sistema.

---

### ✅ 5. **Verificar se Ingressou com Sucesso**

```bash
realm list
```

Você deve ver algo assim:

```txt
realm-name: SAMBA.LOCAL
domain-name: samba.local
configured: kerberos-member
```

---

### ✅ 6. **Permitir Login de Usuários do Domínio**

Edite:

```bash
sudo nano /etc/pam.d/common-session
```

Adicione no final:

```txt
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

Isso cria automaticamente o diretório home dos usuários do domínio no primeiro login.

---

### ✅ 7. **Testar Login com Usuário do Domínio**

Saia da sessão ou vá para “Trocar usuário” e entre com:

**Usuário**: `samba.local\nome_do_usuario`
**Senha**: (senha definida no Samba)

---

## 👥 **SERVIDOR – Criação de Usuários e Grupos**

### ✅ 8. **Criar Usuários no Samba**

```bash
sudo samba-tool user create joao
sudo samba-tool user create maria
sudo samba-tool user create pedro
sudo samba-tool user create ana
```

### ✅ 9. **Criar Grupos e Adicionar Membros**

```bash
sudo samba-tool group add professores
sudo samba-tool group add alunos

sudo samba-tool group addmembers professores joao ana
sudo samba-tool group addmembers alunos maria ana
```

> A usuária **ana** está nos dois grupos.

---

## 📁 **COMPARTILHAMENTO DE PASTAS NO SERVIDOR**

### ✅ 10. **Criar Diretórios e Definir Permissões**

```bash
sudo mkdir -p /samba/professores
sudo mkdir -p /samba/alunos

sudo groupadd samba-professores
sudo groupadd samba-alunos

sudo chown :samba-professores /samba/professores
sudo chown :samba-alunos /samba/alunos

sudo chmod 2770 /samba/professores
sudo chmod 2770 /samba/alunos
```

### ✅ 11. **Mapear Grupos do Samba para o Linux**

```bash
sudo net groupmap add ntgroup="professores" unixgroup=samba-professores type=domain
sudo net groupmap add ntgroup="alunos" unixgroup=samba-alunos type=domain
```

### ✅ 12. **Configurar o `smb.conf`**

Edite:

```bash
sudo nano /etc/samba/smb.conf
```

Adicione ao final:

```ini
[professores]
    path = /samba/professores
    valid users = @"SAMBA\professores"
    read only = no
    force group = samba-professores
    create mask = 0660
    directory mask = 0770

[alunos]
    path = /samba/alunos
    valid users = @"SAMBA\alunos"
    read only = no
    force group = samba-alunos
    create mask = 0660
    directory mask = 0770
```

### ✅ 13. **Reiniciar o Samba**

```bash
sudo systemctl restart samba-ad-dc
```

---

## 🧪 **TESTES FINAIS**

* Tente acessar via Nautilus do Ubuntu Desktop:

  ```bash
  smb://samba.local/professores
  ```

  Faça login com `samba.local\ana` ou `samba.local\joao`.

* Acesse `/samba/alunos` com `maria` e verifique permissões diferentes.

---

Se quiser, posso gerar um **PDF de documentação** para esse passo a passo também. Deseja isso?

sudo net groupmap modify ntgroup="professores" unixgroup=samba-professores

sudo net groupmap modify ntgroup="alunos" unixgroup=samba-alunos
