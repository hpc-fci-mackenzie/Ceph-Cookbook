# CookBook
## Documentação do [CEPH](https://docs.ceph.com/en/reef/)

O CEPH é uma camada de software que possibilita provisionar clusters de armazenento distribuido sobre hardwares convencionais, dando acesso a file systems, Object Storage usando a API S3 e block storage em um ambiente escalavel e de alta disponibilidade. \
A partir da versão Ceph Octopus (v15.2.17), foi introduzido a ferramenta cephadm, que foi construída especificamente para a instalação, manutenção e upgrade de clusters ceph. De acordo com Sebastian Wagner, ex membro do time de liderança do Ceph e desenvolvedor lead do projeto do cephadm, a ferramenta é a ferramenta preferida para implantar clusters Ceph que não são em Kubernetes. Com o lançamento do Ceph Octopus e do cephadm, as antigas formas de implantar o Ceph como ceph-ansible ou ceph-deploy foram descontinuadas.

------------------------------------------------------------------------------------------------------------------------------------

## Pré-instalação

Antes de realizar qualquer instalação do Ceph, é necessário verificar e, caso precise, corrigir os seguintes pontos em cada nó que será utilizado pelo Ceph:
1. Verificar se a RAID está corretamente configurada e funcionando (Raid 5 ou 6, utilizando todos os HDs do nó)
2. Verificar se a instalação do sistema operacional está funcionando (Rocky Linux 8 ou 9)
3. Verificar se o proxy http e https estão corretamente configurados (Verificar com o professor o endereço e as credencials para a proxy)
   
## Instalação

### 1ª Etapa - Configurar ssh passwordless entre os nós para o usuário Root

Em cada um dos nós que será utilizado no cluster, **como o usuário root**, gere um par de chaves que sera utilizado pelo ceph.
```
ssh-keygen
```
Então, utilize o comando `ssh-copy-id` para copiar a chave criada para cada nó no cluster, incluindo ele mesmo.
```
ssh-copy-id -i <caminho para a chave> root@<ip_do_nó>
```
*Caso a chave criada não seja a padrão do comando `ssh-keygen`, ela deverá ser adicionada ao agente de ssh toda vez que o máquina é reiniciada, para fazer isso automaticamente, adicione o seguinte trecho ao arquivo `~/.bashrc`*
```
if [ -z "$SSH_AUTH_SOCK" ] ; then
 eval `ssh-agent -s`
 ssh-add <caminho para a chave>
fi
```

### 2ª Etapa - Instalação das dependências do Cephadm

O Cephadm provisiona os daemons do cluster ceph utilizando containers podman, então não é necessário instalar os pacotes do Ceph manualmente. Porém, ainda existem algumas dependências a serem instaladas em cada nó que será instalado o Cephadm. Elas são:
- `Podman` ([Verifique a compatibilidade da versão do Podman e Ceph](https://docs.ceph.com/en/quincy/cephadm/compatibility/#cephadm-compatibility-with-podman))
- `Python >= 3.6`
- `Chrony` ou outra ferramenta de sincronização de tempo
- `Systemd`

A dependência do `systemd` pode ser ignorada pois o Rocky Linux utilizado no EXADATA usa-o por padrão.  \
Para instalar as depedências necessárias, execute o seguinte comando **como usuário root**:
```
dnf install -y podman chrony python
```

### 2ª Etapa - Instalação do Cephadm

Para instalar o Cephadm, utilizaremos o curl para baixar o seu binário. \
Atualmente, a versão do CEPH mais recente e ativa é a **Squid (19.2.2)**, portanto, utilizaremos ela para implantar o cluster
```
CEPH_RELEASE=19.2.2
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
```
Para verificar que o Cephadm foi instalado corretamente, podemos deixar o binário executavel e verificar a versão do Cephadm utilizando o seguinte comando:
```
# no diretorio que baixou o Cephadm
chmod +x ./cephadm
./cephadm version
```
Se a versão é mostrada, o binário foi baixado corretamente. \
Para simplificar a execução do Cephadm, vamos mover o binário para a pasta `/usr/sbin/`, eliminando a necessidade de passar o caminho inteiro quando precisamos executar. \
**Execute o seguinte comando como usuário root**
```
# no diretorio que baixou o cephadm
mv ./cephadm /usr/sbin/cephadm
```
Para facilitar a administração do Cluster CEPH, é interessante instalar o cephadm em cada nó que será utilizado pelo CEPH. Isto não é estritamente necessário, pois o CEPH será instalado em um nó independente se possui o Cephadm ou não, mas o Cephadm expoe a cli do CEPH, sendo possível administrar o cluster por qualquer nó.

### 4ª Etapa - Bootstrap do Cluster CEPH  

Para iniciar o Cluster CEPH, devemos utilizar o bootstrap do Cephadmdn
