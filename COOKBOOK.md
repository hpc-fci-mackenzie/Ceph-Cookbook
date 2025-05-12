# CEPH Cookbook
## O Exadata

O Oracle Exadata é um Rack de 20 nós localizado na Faculdade De Computação e Informática (FCI) da Universidade Presbiteriana Mackenzie. Ele foi adpatado do seu uso original, que é hospedar bancos de dados oracle, para um rack general-purpose utilizado pela FCI para estudo e apoio. \
A FCI possuí vários departamentos e grupos de estudo que se benificiariam de um sistema de arquivos distribuido e exposto pela rede (algo como um NAS) para que arquivos sejam disponibilizados automaticamente. Para tal, foi proposto implantar um cluster CEPH em certos nós do exadata para prover esse sistema de arquivos.

## Documentação do [CEPH](https://docs.ceph.com/en/reef/)

O CEPH é uma camada de software que possibilita provisionar clusters de armazenento distribuido sobre hardwares convencionais, dando acesso a file systems NAS, Object Storage usando a API S3 e block storage em um ambiente escalavel e de alta disponibilidade. \
A partir da versão Ceph Octopus (v15.2.17), foi introduzido a ferramenta cephadm, que foi construída especificamente para a instalação, manutenção e upgrade de clusters ceph. De acordo com Sebastian Wagner, ex membro do time de liderança do Ceph e desenvolvedor lead do projeto do cephadm, a ferramenta é a ferramenta preferida para implantar clusters Ceph que não são em Kubernetes. Com o lançamento do Ceph Octopus e do cephadm, as antigas formas de implantar o Ceph como ceph-ansible ou ceph-deploy foram descontinuadas.

------------------------------------------------------------------------------------------------------------------------------------

## Pré-instalação

Antes de realizar qualquer instalação do Ceph, é necessário verificar e, caso precise, corrigir os seguintes pontos em cada nó que será utilizado pelo Ceph:
1. Verificar se a RAID está corretamente configurada e funcionando. (**Devido a necessidade do CEPH de um dispositivo de armazenamento sem sistema de arquivo, é necessário criar dois volumes RAIDs, um para o SO e um para o CEPH**)
3. Verificar se a instalação do sistema operacional está funcionando (Rocky Linux 8 ou 9)
4. Verificar se o proxy http e https estão corretamente configurados (Verificar com o professor o endereço e as credencials para a proxy)
   
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

Para iniciar o Cluster CEPH, devemos utilizar o bootstrap do Cephadmdn. \
Este comando inicia o cluster com 1 nó e coloca o primeiro [**mon**](https://docs.ceph.com/en/latest/cephadm/services/mon/) deamon neste nó. \
Para fazer o bootstrap do cluster no nó que está executando o cephadm, **execute o seguinte comando como root**:
Os argumentos devem ser preenchidos da seguinte forma:
1. ip-privado-do-nó -> O IP privado na rede do exadata do nó que está rodando o comando. **É importante não utilizar localhost ou 127.0.0.1 neste parametro**
2. CIDR da rede dos OSDs -> O segmento de rede que será utilizado pelos OSDs. Atualmente está sendo utilizado 10.0.0.0/24
3. nome do cluster -> Nome do cluster. Atualmente é exadata-ceph
```
cephadm bootstrap --mon-ip= <ip-privado-do-nó> --cluster-network=<CIDR da rede dos OSDs>
```
Este comando irá demorar um tempo para executar.

### 5ª Etapa - Instalando a CLI do Ceph no node

Para facilitar o uso do CEPH, podemos instalar a CLI do ceph e outros pacotes para administracao e uso do cluster diretamente no no. Para fazer isso, **execute os seguintes comandos como root**:
```
cephadm add-repo --release quincy
cephadm install ceph-common
```
### 6ª Etapa - Adicionando os nós restantes no Cluster

Agora, para adicionar os nós restantes no cluster, deve se copiar as chaves SSH do ceph para os nós
```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@<novo nó>
```
Então, execute o seguinte comando para adicionar um host novo e adicionar o label `_admin` para a utilizacao do cephadm:
```
ceph orch host add <hostname do no> <ip do no> _admin 
```
Faca isso para todo no que sera adicionado ao cluster.

### 7ª Etapa - Adicionando OSDs para os dispostivos de armazenamento
