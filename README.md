Com certeza! Para facilitar o entendimento, vamos detalhar as configurações necessárias tanto para o servidor Ubuntu (que será o Controlador de Domínio Samba Active Directory) quanto para o cliente Ubuntu, explicando cada passo.

Configuração do Servidor Ubuntu (Controlador de Domínio Samba Active Directory)
O servidor Ubuntu, com o IP 

192.168.10.2/24, será a peça central da nossa rede, atuando como um Controlador de Domínio Samba.


Objetivo: Configurar o Ubuntu Server para ser o Controlador de Domínio Samba Active Directory para o domínio SAMBA.LOCAL.

Passos:

Configuração de Acesso Remoto (SSH)

Para que serve? O SSH (Secure Shell) permite que você acesse e gerencie seu servidor remotamente a partir de outro computador, como o seu cliente Windows. Isso evita a necessidade de um monitor e teclado conectados diretamente ao servidor.

Como fazer:

Primeiro, atualize a lista de pacotes disponíveis:

Bash

sudo apt-get update
Em seguida, instale o servidor OpenSSH:

Bash

sudo apt-get install openssh-server -y
O -y no final aceita automaticamente todas as perguntas de confirmação.

Configuração de Rede do Servidor

Para que serve? É crucial que o servidor tenha um endereço IP fixo e consiga se comunicar com a internet e com os clientes na rede local. No nosso caso, o servidor terá o IP 

192.168.10.2/24.

Como fazer:

Identificar interfaces de rede:

Bash

ip -br a
Este comando mostrará todas as interfaces de rede do seu servidor (por exemplo, 

enp0s3, enp0s8, enp0s9).  Você precisa identificar qual interface será usada para a rede local (

192.168.10.0/24) e qual, se houver, será para a internet.

Acessar o diretório de configuração do Netplan:

Bash

cd /etc/netplan
Editar o arquivo de configuração do Netplan:

Bash

sudo nano 01-netcfg.yaml # ou o nome do seu arquivo .yaml, pode variar
Dentro do arquivo, adicione ou modifique as seguintes linhas. Adapte os nomes das interfaces (enp0s3, enp0s8, enp0s9) conforme o que você identificou no comando ip -br a:

YAML

network:
  version: 2
  ethernets:
    enp0s3: # Exemplo: Interface para a internet, obtém IP via DHCP
      dhcp4: true
    enp0s8: # Exemplo: Interface para a rede local, IP fixo
      [cite_start]addresses: [192.168.10.2/24] # IP do servidor 
    enp0s9: # Exemplo: Outra interface para a internet (se houver), via DHCP
      dhcp4: true
      nameservers:
        [cite_start]addresses: [8.8.8.8] # DNS público para resolução de nomes na internet 
Explicação:


dhcp4: true: A interface obterá automaticamente um endereço IP via DHCP (comum para conexão com a internet ).

addresses: [192.168.10.2/24]: Define o endereço IP estático para a interface da rede local. O /24 indica a máscara de sub-rede (255.255.255.0).

nameservers: addresses: [8.8.8.8]: Define o servidor DNS primário (aqui, o DNS público do Google) para que o servidor possa resolver nomes de domínio na internet.

Aplicar as configurações de rede:

Bash

sudo netplan apply
Verificar se as configurações foram aplicadas corretamente:

Bash

ip -br a
Confirme se o enp0s8 (ou a interface que você configurou) agora tem o IP 192.168.10.2/24.

Acesso Remoto via SSH (do Cliente Windows)

Para que serve? Agora que o SSH está instalado e a rede configurada, você pode acessar o servidor do seu PC Windows.

Como fazer:

No PowerShell do Windows (ou em um terminal Linux/macOS), digite:

Bash

ssh usuario@192.168.10.2
Substitua usuario pelo nome de usuário que você usa no servidor Ubuntu. Você será solicitado a digitar a senha.

Configuração do Hostname (Nome do Servidor)

Para que serve? Definir um hostname claro (dc para Domain Controller) e configurar o arquivo hosts ajuda o servidor a se identificar corretamente na rede e a resolver nomes internamente, o que é fundamental para o Samba.

Como fazer:

Verificar o hostname atual:

Bash

hostname
Alterar o hostname para dc (Domain Controller):

Bash

sudo hostnamectl set-hostname dc
Configurar o arquivo hosts:

Bash

sudo nano /etc/hosts
Adicione as seguintes linhas ao arquivo. Elas mapeiam os nomes de domínio para o endereço IP do seu servidor:

127.0.0.1    localhost
127.0.1.1    samba
192.168.10.2  dc.samba.local dc
192.168.10.2  samba.local samba
Explicação:

192.168.10.2 dc.samba.local dc: Associa o IP do servidor ao nome de domínio completo (dc.samba.local) e ao nome curto (dc). Isso é essencial para o funcionamento do Samba AD.

192.168.10.2 samba.local samba: Faz o mesmo para o nome do domínio principal (samba.local) e seu nome curto (samba).

Verificar as alterações:

Bash

hostname -f
ping -c2 dc.samba.local
O comando hostname -f deve retornar dc.samba.local. O ping deve ter sucesso.

Configuração do DNS no Servidor

Para que serve? O Samba Active Directory integra seu próprio servidor DNS. Precisamos garantir que o sistema operacional use o DNS do Samba para resolver nomes de domínio internos (samba.local) e, se necessário, encaminhe requisições para a internet.

Como fazer:

Desativar o systemd-resolved: Este serviço padrão do Ubuntu pode interferir com o DNS do Samba.

Bash

sudo systemctl disable --now systemd-resolved
sudo unlink /etc/resolv.conf
O unlink remove o link simbólico para o arquivo resolv.conf gerenciado pelo systemd-resolved.

Configurar manualmente o arquivo /etc/resolv.conf:

Bash

sudo nano /etc/resolv.conf
Adicione as seguintes linhas:

nameserver    192.168.10.2 # Aponta para o próprio servidor DNS (Samba)
nameserver    8.8.8.8      # DNS público para resolver nomes da internet 
search        samba.local  # Define o domínio de busca padrão
Proteger o arquivo resolv.conf: Isso evita que outros serviços (como o systemd-resolved, se for reativado por engano) sobrescrevam suas configurações.

Bash

sudo chattr +i /etc/resolv.conf
Instalação do Samba (Active Directory Domain Controller)

Para que serve? Estes pacotes são a base para transformar seu Ubuntu em um Controlador de Domínio Active Directory, permitindo a autenticação de usuários e o gerenciamento de recursos.

Como fazer:

Atualizar a lista de pacotes novamente:

Bash

sudo apt-get update
Instalar os pacotes necessários:

Bash

sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules smbclient winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user dnsutils chrony net-tools
Configuração durante a instalação: Durante o processo, você pode ser solicitado a configurar o Kerberos. Use as seguintes informações:

Realm: SAMBA.LOCAL

Servidor Kerberos: dc.samba.local

Servidor administrativo: dc.samba.local

Desativar serviços Samba desnecessários (NMBd e SMBd): Quando o Samba atua como AD DC, ele usa um serviço diferente (samba-ad-dc). Os serviços smbd, nmbd e winbind podem causar conflitos.

Bash

sudo systemctl disable --now smbd nmbd winbind
Habilitar e iniciar o serviço samba-ad-dc:

Bash

sudo systemctl unmask samba-ad-dc # Remove qualquer máscara que possa impedir o serviço
sudo systemctl enable samba-ad-dc # Habilita o serviço para iniciar com o sistema
Provisionamento do Active Directory

Para que serve? Este é o passo que realmente transforma o Samba em um Controlador de Domínio, criando a estrutura do AD, o banco de dados de usuários, grupos, etc.

Como fazer:

Fazer backup da configuração padrão do Samba: O smb.conf padrão não é usado para um AD DC, então o movemos para um backup.

Bash

sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
Provisionar o domínio:

Bash

sudo samba-tool domain provision
Durante o provisionamento, você será guiado por algumas perguntas. Responda da seguinte forma:


Realm [SAMBA.LOCAL]: Pressione Enter (já preenchido com o nome do domínio )

Domain [SAMBA]: Pressione Enter (já preenchido)

Server Role (dc, member, standalone) [dc]: Pressione Enter (para Domain Controller)

DNS backend (SAMBA_INTERNAL, BIND9_DLZ, NONE) [SAMBA_INTERNAL]: Pressione Enter (o Samba gerenciará o DNS internamente)

DNS forwarder (IP address or hostname): 8.8.8.8 (para o Samba encaminhar requisições DNS não-locais para a internet)

Administrator password: (crie uma senha forte para o administrador do domínio)

Retype password: (repita a senha)

Configuração do Kerberos no Servidor

Para que serve? O Kerberos é o protocolo de autenticação usado no Active Directory. Precisamos garantir que o sistema use o arquivo de configuração do Kerberos gerado pelo Samba.

Como fazer:

Fazer backup do arquivo krb5.conf existente:

Bash

sudo mv /etc/krb5.conf /etc/krb5.conf.orig
Copiar o arquivo krb5.conf gerado pelo Samba:

Bash

sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
Iniciar o serviço samba-ad-dc:

Bash

sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
Verifique se o serviço está active (running).

Configuração de Sincronização de Tempo (NTP)

Para que serve? A sincronização de tempo é crítica para o Kerberos e o Active Directory. Se houver uma diferença de tempo significativa entre o servidor e os clientes, a autenticação falhará. O Chrony é um serviço NTP que pode ser integrado ao Samba.

Como fazer:

Configurar permissões para o Chrony:

Bash

sudo apt-get update
sudo chown root:_chrony /var/lib/samba/ntp_signd/
sudo chmod 750 /var/lib/samba/ntp_signd/
Configurar o Chrony:

Bash

sudo nano /etc/chrony/chrony.conf
Adicione as seguintes linhas ao final do arquivo:

bindcmdaddress 192.168.10.2  # Vincula o comando de controle do Chrony ao IP do servidor
allow 192.168.10.0/24      # Permite que a rede local sincronize o tempo com este servidor
ntpsigndsocket /var/lib/samba/ntp_signd # Integra o Chrony com o Samba
Reiniciar e verificar o serviço Chrony:

Bash

sudo systemctl restart chronyd
sudo systemctl status chronyd
Confirme que o serviço está active (running).

Verificação e Testes Finais no Servidor

Para que serve? Confirmar que o Samba AD está funcionando corretamente e que a resolução de nomes e autenticação Kerberos estão ok.

Como fazer:

Verificar resolução de hosts DNS:

Bash

host -t A samba.local      # Deve retornar o IP do servidor
host -t A dc.samba.local   # Deve retornar o IP do servidor
Verificar registros SRV do Kerberos e LDAP:

Bash

host -t SRV _kerberos._udp.samba.local # Deve retornar o hostname do servidor
host -t SRV _ldap._tcp.samba.local     # Deve retornar o hostname do servidor
Verificar recursos Samba:

Bash

smbclient -L samba.local -N # Deve listar os compartilhamentos padrão do Samba AD
Teste de autenticação Kerberos:

Bash

kinit administrator@SAMBA.LOCAL # Será solicitada a senha do administrador do domínio
klist                           # Deve mostrar um ticket Kerberos válido
Este passo é crucial para confirmar que a autenticação Kerberos está funcionando.

Teste de acesso SMB:

Bash

sudo smbclient //localhost/netlogon -U 'administrator' # Acessa um compartilhamento padrão do AD
Você deve conseguir se conectar.

Verificar a configuração do Samba:

Bash

testparm
sudo samba-tool domain level show
testparm verifica a sintaxe do smb.conf. samba-tool domain level show mostra o nível funcional do domínio.

Gerenciamento de Usuários e Grupos no Servidor Samba


Para que serve? É aqui que você cria os usuários e grupos solicitados.

Como fazer:

Alterar senha do administrador (boa prática após o provisionamento):

Bash

sudo samba-tool user setpassword administrator

Criar quatro usuários: 

Bash

sudo samba-tool user create usuario1 --description="Usuário Teste 1"
sudo samba-tool user create usuario2 --description="Usuário Teste 2"
sudo samba-tool user create usuario3 --description="Usuário Teste 3"
sudo samba-tool user create usuario4 --description="Usuário Teste 4"
Para cada usuário, será solicitada uma senha. Crie senhas para cada um.


Criar dois grupos: 

Bash

sudo samba-tool group add grupoA
sudo samba-tool group add grupoB

Associar usuários aos grupos: 

Associe usuario1, usuario2 e usuario3 ao grupoA:

Bash

sudo samba-tool group addmembers grupoA usuario1,usuario2,usuario3
Associe 

usuario3 e usuario4 ao grupoB (note que usuario3 pertencerá a ambos os grupos ):

Bash

sudo samba-tool group addmembers grupoB usuario3,usuario4
Listar para verificar:

Bash

sudo samba-tool user list
sudo samba-tool group list
sudo samba-tool group listmembers grupoA
sudo samba-tool group listmembers grupoB
Criação e Configuração de Diretório Compartilhado


Para que serve? Para que os usuários do domínio possam acessar arquivos e pastas na rede com permissões específicas, conforme solicitado.

Como fazer:

Criar um diretório para o compartilhamento:

Bash

sudo mkdir -p /srv/samba/compartilhado
Definir permissões de sistema de arquivos: Permite que os membros do domínio acessem o diretório.

Bash

sudo chown -R root:"Domain Users" /srv/samba/compartilhado
sudo chmod -R 2770 /srv/samba/compartilhado
chown -R root:"Domain Users": Define o proprietário como root e o grupo como "Domain Users" (grupo padrão do Samba AD que inclui todos os usuários do domínio).

chmod -R 2770:

2: Define o "setgid bit", o que significa que novos arquivos e diretórios criados dentro de /srv/samba/compartilhado herdarão o grupo "Domain Users".

770: Permissões de leitura, escrita e execução para o proprietário (root) e para o grupo (Domain Users). Nenhuma permissão para "outros".

Editar o arquivo smb.conf para criar o compartilhamento:

Bash

sudo nano /etc/samba/smb.conf
Adicione o seguinte bloco ao final do arquivo:

Ini, TOML

[compartilhado]
    path = /srv/samba/compartilhado
    read only = no
    browseable = yes
    valid users = @grupoA @grupoB  # Grupos permitidos a acessar o compartilhamento
    write list = @grupoA           # Apenas membros de 'grupoA' podem escrever
    force group = "Domain Users"   # Garante que arquivos criados pertençam ao grupo
    create mask = 0660             # Permissões para novos arquivos
    directory mask = 0770          # Permissões para novos diretórios
Explicação:

path: O caminho do diretório a ser compartilhado.

read only = no: Permite operações de escrita.

browseable = yes: O compartilhamento será visível na rede.

valid users = @grupoA @grupoB: Somente usuários que são membros do grupoA ou grupoB podem acessar este compartilhamento.

write list = @grupoA: Define que apenas os membros de grupoA terão permissão de escrita. Os membros de grupoB (que não estão em grupoA) terão apenas permissão de leitura.

Reiniciar o serviço Samba para aplicar as mudanças:

Bash

sudo systemctl restart samba-ad-dc
Configuração do Cliente Ubuntu
O cliente Ubuntu, com o IP 

192.168.10.4/24, será ingressado no domínio Samba Active Directory.


Objetivo: Ingressar o cliente Ubuntu no domínio SAMBA.LOCAL e testar o acesso aos compartilhamentos.


Passos:

Configuração de Rede do Cliente

Para que serve? O cliente precisa ter um endereço IP na mesma rede do servidor e, crucialmente, deve usar o servidor Samba AD como seu servidor DNS primário para conseguir resolver o nome do domínio.

Como fazer:

Acessar o diretório de configuração do Netplan:

Bash

cd /etc/netplan
Editar o arquivo de configuração do Netplan:

Bash

sudo nano 01-netcfg.yaml # ou o nome do seu arquivo .yaml
Adicione ou modifique as seguintes linhas (adapte o nome da interface, por exemplo, enp0s3):

YAML

network:
  version: 2
  ethernets:
    enp0s3: # Adapte o nome da interface de rede do seu cliente
      [cite_start]addresses: [192.168.10.4/24] # IP do cliente Ubuntu 
      nameservers:
        [cite_start]addresses: [192.168.10.2, 8.8.8.8] # O primeiro DNS deve ser o IP do seu DC [cite: 23]
        [cite_start]search: [samba.local] # O domínio de busca 
Aplicar as configurações de rede:

Bash

sudo netplan apply
Verificar a conectividade:

Bash

ping 192.168.10.2    # Deve pingar o servidor
ping dc.samba.local  # Deve resolver o nome do servidor
ping samba.local     # Deve resolver o nome do domínio
Instalar Pacotes Necessários para Ingressar no Domínio

Para que serve? Estes pacotes (realmd, sssd, adcli, krb5-user, etc.) permitem que o Ubuntu se integre a um domínio Active Directory, gerenciando usuários, grupos e autenticação.

Como fazer:

Bash

sudo apt update
sudo apt install -y realmd sssd sssd-tools samba-common samba-common-bin adcli krb5-user packagekit-tools
Configuração do Kerberos no Cliente

Para que serve? O cliente também precisa saber onde encontrar o servidor Kerberos para autenticação no domínio.

Como fazer:

Durante a instalação dos pacotes acima, você provavelmente será solicitado a configurar o Kerberos. Use:

Realm: SAMBA.LOCAL

Servidor Kerberos: dc.samba.local

Se não for perguntado ou se precisar corrigir, edite manualmente o arquivo /etc/krb5.conf:

Bash

sudo nano /etc/krb5.conf
Adicione ou modifique para:

Ini, TOML

[libdefaults]
        default_realm = SAMBA.LOCAL
        dns_lookup_realm = true
        dns_lookup_kdc = true

[realms]
        SAMBA.LOCAL = {
                kdc = dc.samba.local
                admin_server = dc.samba.local
        }

[domain_realm]
        .samba.local = SAMBA.LOCAL
        samba.local = SAMBA.LOCAL
Ingressar o Cliente Ubuntu no Domínio Samba Active Directory

Para que serve? Este é o passo que realmente conecta o cliente ao domínio, permitindo que ele reconheça os usuários e grupos do AD.

Como fazer:

Descobrir o domínio (opcional, apenas para verificar):

Bash

sudo realm discover SAMBA.LOCAL
Deve retornar informações sobre o seu domínio.

Ingressar no domínio:

Bash

sudo realm join -U administrator SAMBA.LOCAL
Será solicitada a senha do usuário administrator do domínio Samba que você configurou no servidor.

Verificação: Você pode verificar se o cliente foi ingressado com:

Bash

realm list
Configuração do SSSD (System Security Services Daemon)

Para que serve? O SSSD é um serviço que gerencia a autenticação e as informações de usuários de fontes externas (como o Active Directory) no sistema Linux. O comando realm join configura o SSSD automaticamente.

Como fazer (verificação e reinício):

Verifique o arquivo de configuração do SSSD:

Bash

sudo cat /etc/sssd/sssd.conf
Ele deve conter informações sobre o domínio samba.local.

Reinicie o serviço SSSD para garantir que as configurações sejam aplicadas:

Bash

sudo systemctl restart sssd
Teste se os usuários do domínio são reconhecidos:

Bash

id usuario1@samba.local
id usuario3@samba.local
Isso deve mostrar as informações de UID, GID e grupos do usuário do domínio.

Realizar Login no Cliente Ubuntu com Usuário do Domínio


Para que serve? Testar a funcionalidade principal: autenticar usuários do domínio no cliente Ubuntu.

Como fazer:

No terminal (para testar a autenticação):

Bash

su - usuario1@samba.local
Será solicitada a senha do usuario1. Se o login for bem-sucedido, você estará logado como o usuário do domínio no terminal. Digite exit para sair.

No ambiente gráfico (interface de usuário):

Reinicie o cliente Ubuntu.

Na tela de login, você poderá ter a opção de "Not listed?" ou "Outro usuário". Clique nela.

Digite o nome de usuário no formato usuario1@samba.local ou, dependendo da configuração do SSSD, apenas usuario1.

Digite a senha do usuário.

Se tudo estiver correto, você fará login no ambiente gráfico com o usuário do domínio. O diretório home do usuário será criado automaticamente (por exemplo, /home/usuario1@samba.local).

Testar o Acesso ao Compartilhamento a partir do Cliente Ubuntu


Para que serve? Verificar se os usuários do domínio podem acessar o compartilhamento criado no servidor, com as permissões corretas.

Como fazer:

Certifique-se de estar logado como um dos usuários do domínio (ex: usuario1@samba.local).

Criar um ponto de montagem para o compartilhamento:

Bash

mkdir ~/compartilhamento_samba
Montar o compartilhamento do Samba:

Bash

sudo mount -t cifs //dc.samba.local/compartilhado ~/compartilhamento_samba -o username=usuario1,domain=SAMBA.LOCAL,uid=$(id -u),gid=$(id -g),nounix,iocharset=utf8
//dc.samba.local/compartilhado: Caminho UNC do compartilhamento no servidor.

~/compartilhamento_samba: Onde o compartilhamento será montado no cliente.

username=usuario1: O nome de usuário do domínio para autenticação.

domain=SAMBA.LOCAL: O domínio ao qual o usuário pertence.

uid=$(id -u),gid=$(id -g): Garante que os arquivos e diretórios montados pertençam ao usuário logado e seu grupo primário, facilitando o acesso.

nounix: Desabilita extensões Unix para o Samba (às vezes causa problemas).

iocharset=utf8: Garante a codificação correta de caracteres.

Você será solicitado a digitar a senha do usuario1.

Testar o acesso e as permissões:

Com usuario1 (membro de grupoA e grupoB - tem permissão de escrita):

Bash

touch ~/compartilhamento_samba/teste_usuario1.txt
echo "Hello from user1" > ~/compartilhamento_samba/arquivo1.txt
ls -l ~/compartilhamento_samba/
Você deve conseguir criar arquivos e diretórios.

Desmonte o compartilhamento:

Bash

sudo umount ~/compartilhamento_samba
Repita o teste com usuario4 (membro de grupoB - tem apenas permissão de leitura, conforme sua configuração no smb.conf):

Logue-se como usuario4@samba.local no cliente Ubuntu.

Monte o compartilhamento usando as credenciais de usuario4.

Tente criar um arquivo:

Bash

touch ~/compartilhamento_samba/teste_usuario4.txt
Isso deve falhar com uma mensagem de "Permissão negada", confirmando que as permissões distintas estão funcionando.

Você ainda deve conseguir ler arquivos existentes no compartilhamento.

Este guia detalhado cobre todos os passos para configurar o servidor e o cliente, permitindo que você compreenda e execute o cenário proposto no PDF.
