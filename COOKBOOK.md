# Guia Completo para Implantacao de um Cluster CEPH no Exadata da FCI - Mackenzie

## Introducao

O Oracle Exadata, originalmente projetado para hospedar bancos de dados Oracle de alta performance, está localizado na Faculdade de Computação e Informática (FCI) da Universidade Presbiteriana Mackenzie. Atualmente, esse equipamento foi adaptado para funcionar como um rack de uso geral, atendendo as demandas de pesquisa, ensino e extensão.

Para melhorar a gestão e o acesso aos dados compartilhados entre departamentos e grupos de estudo, propomos a implantação de um sistema de arquivos distribuído baseado no CEPH. Esse sistema permitirá que todos os dados estejam acessíveis em rede, com alta disponibilidade e escalabilidade, similar a uma solução NAS.

---

## Capítulo 1 - Conhecendo o CEPH

### O que é o CEPH?

CEPH é uma plataforma de armazenamento distribuído de código aberto que fornece:

* **Object Storage** com compatibilidade com a API S3
* **Block Storage** (semelhante a um disco virtual)
* **File System (CephFS)**, ideal para soluções NAS

Com ele, é possível usar hardware convencional para montar um sistema de armazenamento altamente disponível e escalável.

### Ferramenta Cephadm

Desde a versão Octopus (v15.2.17), o Ceph inclui o `cephadm`, uma ferramenta moderna para instalação, gestão e atualização de clusters. Essa ferramenta substitui abordagens anteriores como `ceph-ansible` e `ceph-deploy`.

O `cephadm` funciona gerenciando os daemons Ceph dentro de contêiners Podman ou Docker, com suporte embutido para atualizações, segurança e escalabilidade.

### Arquitetura do CEPH - Teoria

Um cluster CEPH é composto por vários tipos de daemons, cada um com funções específicas. Os principais são:

#### MON (Monitor)

Responsável por manter o estado do cluster. Os daemons MON mantêm informações de:

* Quais nós estão ativos
* Autenticação e permissões
* Versão do mapa do cluster

Pelo menos **três MONs** são recomendados para alta disponibilidade e quorum.

#### OSD (Object Storage Daemon)

Controla os discos onde os dados realmente são armazenados. Cada disco (sem filesystem) corresponde a um OSD daemon. Os OSDs:

* Armazenam os dados em si
* Replicam e balanceiam os dados
* Informam seu estado ao cluster

A performance e resiliência do cluster dependem fortemente dos OSDs.

#### MDS (Metadata Server) - opcional

Necessário apenas se você for usar o CephFS (sistema de arquivos). Gerencia as metadados (como diretórios, permissões e estrutura de arquivos).

#### RGW (RADOS Gateway) - opcional

Permite acesso ao cluster via protocolo S3 (Object Storage). Ideal para aplicações web/cloud.

Esses componentes se comunicam entre si usando o protocolo interno chamado **CRUSH** (Controlled Replication Under Scalable Hashing), que distribui e replica os dados automaticamente conforme regras de topologia.

---

## Capítulo 2 - Pré-Requisitos

Antes de qualquer instalação, garanta os seguintes pontos em cada nó que participará do cluster:

### 2.1 - Configuração do Armazenamento

* Verifique a presença de dois volumes RAID:

  * Um para o sistema operacional (com filesystem).
  * Um para o CEPH (sem filesystem, apenas o disco bruto).

### 2.2 - Sistema Operacional

* Instale e verifique o funcionamento do Rocky Linux 8 ou 9.

### 2.3 - Acesso à Internet via Proxy

* Configure o proxy HTTP e HTTPS corretamente.
* Consulte o professor responsável para obter endereços e credenciais.

### 2.4 - Requisitos de Firewall
As seguintes portas devem ser liberadas **entre** os nós do cluster

| Porta     | Protocolo | Descrição                                               |
| --------- | --------- | ------------------------------------------------------- |
| 3300      | TCP       | Comunicação entre os daemons Ceph (Ceph MON, MGR)       |
| 6789      | TCP       | Comunicação entre os daemons Ceph MON                   |
| 6800-7300 | TCP       | Comunicação com os daemons OSD (usado dinamicamente)    |
| 22        | TCP       | Comunicação SSH                                         |

As seguintes portas devem ser liberadas para o acesso ao NFS no headnode:
| Porta     | Protocolo | Descrição                                               |
| --------- | --------- | ------------------------------------------------------- |
| 111       | TCP/UDP   | Portmap para NFS                                        |
| 2049      | TCP/UDP   | NFS (usado pelo Ganesha para exportação do CephFS)      |

---

## Capítulo 3 - Configuração Inicial

### 3.1 - SSH Passwordless entre os Nós

Em **todos os nós**:

```bash
ssh-keygen
```

Depois:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@<ip_do_nó>
```

Se a chave não for a padrão, adicione ao `~/.bashrc`:

```bash
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval `ssh-agent -s`
  ssh-add <caminho_para_chave>
fi
```

---

## Capítulo 4 - Instalando o Cephadm

### 4.1 - Download e Instalação

```bash
CEPH_RELEASE=19.2.2
curl --silent --remote-name --location \
  https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x ./cephadm
./cephadm version
```

Se o comando acima retornar a versão, está tudo certo.

### 4.2 - Movendo para local padrão

```bash
mv ./cephadm /usr/sbin/cephadm
```

Recomenda-se instalar o cephadm em **todos os nós**, para facilitar a administração distribuída.

---

## Capítulo 5 - Bootstrap do Cluster

Este comando inicia o cluster com 1 nó como monitor principal (MON):

```bash
cephadm bootstrap \
  --mon-ip=<ip_privado> \
  --cluster-network=10.0.0.0/24 \
  --cluster-name=exadata-ceph
```

**Importante**: Use o IP da rede interna (do Exadata), nunca `localhost`.

---

## Capítulo 6 - Instalando a CLI do Ceph

Para facilitar o gerenciamento via linha de comando:

```bash
cephadm add-repo --release squid
cephadm install ceph-common
```

---

## Capítulo 7 - Adicionando Mais Nós

### 7.1 - Compartilhar Chave SSH

Execute no nó onde o cluster foi iniciado:

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@<ip_do_novo_nó>
```

### 7.2 - Adicionar o Nó ao Cluster

```bash
ceph orch host add <hostname> <ip> _admin
```

Repita o processo para todos os nós.

---

## Capítulo 8 - Provisionando OSDs

O OSD (Object Storage Daemon) é o componente que gerencia o armazenamento bruto. Para adicioná-los:

```bash
ceph orch apply osd --all-available-devices
```

Este comando detecta os discos sem partição nem filesystem e os configura automaticamente.

## Capítulo 9 - Criando um CephFS e Expondo via NFS com Ganesha

O CephFS é o sistema de arquivos nativo do CEPH, que permite criar volumes compartilhados com semântica POSIX, similar a sistemas como NFS ou ext4. Para disponibilizá-lo em uma rede de forma compatível com clientes NFS, é necessário configurar o **NFS Ganesha** sobre o Ceph.

### 9.1 - Criar o Pool e o Filesystem CephFS

Execute os comandos abaixo no nó com a CLI do Ceph instalada:

```bash
ceph fs volume create <nome_do_filesystem>
```

Exemplo:

```bash
ceph fs volume create cephfs-fci
```

Esse comando cria:

* Um filesystem CephFS chamado `cephfs-fci`
* Dois pools padrão: `cephfs-fci_data` e `cephfs-fci_metadata`

Verifique a criação com:

```bash
ceph fs ls
```

### 9.2 - Implantar o Daemon MDS

Se ainda não existir, adicione um daemon **MDS** (Metadata Server), necessário para o funcionamento do CephFS:

```bash
ceph orch apply mds <nome_do_filesystem> --placement='<número de instâncias>'
```

Exemplo:

```bash
ceph orch apply mds cephfs-fci --placement='2'
```

### 9.3 - Instalar e Ativar o NFS Ganesha

Para exportar o CephFS por NFS:

```bash
ceph orch apply nfs <nome-da-instancia-nfs> --port 2049 --placement='<número de nós>'
```

Exemplo:

```bash
ceph orch apply nfs nfs-fci --port 2049 --placement='2'
```

Verifique se o serviço está ativo:

```bash
ceph orch ps | grep nfs
```

### 9.4 - Criar a Exportação NFS

Crie um export para o CephFS com o seguinte comando:

```bash
ceph nfs export create cephfs <nome-da-instancia-nfs> /<export-path> <nome_do_filesystem>
```

Exemplo:

```bash
ceph nfs export create cephfs nfs-fci / fci-cephfs
```

Isso cria um ponto de montagem NFS `/` vinculado ao CephFS `fci-cephfs`.

Você pode listar os exports com:

```bash
ceph nfs export ls nfs-fci
```

### 9.5 - Montando o NFS em Clientes

Nos clientes Linux, monte o volume NFS normalmente:

```bash
sudo mount -t nfs <ip_do_nó_nfs>:/ /mnt/cephfs
```

Adicione no `/etc/fstab` para montagem automática:

```bash
<ip_do_nó_nfs>:/ /mnt/cephfs nfs defaults,_netdev 0 0
```

### Observações

* Os daemons NFS Ganesha rodam em contêineres e são gerenciados pelo `cephadm`.
* Certifique-se que as portas 2049 e 111 estejam liberadas entre os clientes e os nós que rodam o Ganesha.
* Para segurança, configure regras de exportação usando arquivos de export ou políticas via `ceph nfs export`.

## Cápitulo 10 - Limitando o tamanho do File System

É comum que o tamanho do file system criado pelo CephFS tenha um tamanho máximo. Para fazer isso, temos a seguinte abordagem:

- Limitar o file system inteiro (Dar um tamanho máximo para a *pool* utilizada pelo file system)

### 10.1 - Limitando o file system inteiro

Para limitar o tamanho do file system inteiro, é necessário limitar as duas pools utilizadas pelo CephFS:

```
ceph osd pool set-quota max_bytes <nome do fs>_data <tamanho_em_bytes>
ceph osd pool set-quota max_bytes <nome do fs>_metadata <tamanho_em_bytes>
```

Por exemplo:

```
ceph osd pool set-quota max_bytes cephfs-fci_data 107374182400
ceph osd pool set-quota max_bytes cephfs-fci_metadata 107374182400
```

**Observação**: O valor dado ao comando é em **bytes**, então, por exemplo, 100GiB se torna 107374182400


## Conclusão

Com esse guia, você consegue implantar um cluster CEPH funcional sobre o Exadata da FCI. Essa solução vai beneficiar a infraestrutura da faculdade com um armazenamento distribuído, robusto, resiliente e escalável.

Para mais detalhes, consulte sempre a documentação oficial: [https://docs.ceph.com/en/latest](https://docs.ceph.com/en/latest)

---

## Apêndice

* **ceph orch ls**: lista todos os daemons
* **ceph -s**: status do cluster
* **ceph health detail**: detalhes de problemas no cluster
* **ceph orch ps**: status dos containers do cluster
