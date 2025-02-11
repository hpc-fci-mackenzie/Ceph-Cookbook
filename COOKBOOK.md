# CookBook
## Documentação do [CEPH](https://docs.ceph.com/en/reef/)

Primeiro é necessário definir a melhor forma de instalar o CEPH: Cephadm, Rook, Ceph-Ansible, ceph-deploy(não recebe mais suporte), ceph-salt, via juju, via puppet, OpenNebula HCI clusters ou manualmente.
> Vale lembrar que nessa ocasião estamos implantando o CEPH em um cluster com 20 nós, então nesse caso iremos explorar a opção mais adequada.

------------------------------------------------------------------------------------------------------------------------------------

## Cephadm
[cephadm]([https://docs.ceph.com/en/reef/install/index_manual/#install-manual](https://docs.ceph.com/en/quincy/cephadm/index.html))

A partir da versão Ceph Octopus (v15.2.17), foi introduzido a ferramenta cephadm, que foi construída especificamente para a instalação, manutenção e upgrade de clusters ceph. De acordo com Sebastian Wagner, ex membro do time de liderança do Ceph e desenvolvedor lead do projeto do cephadm, a ferramenta é a ferramenta preferida para implantar clusters Ceph que não são em Kubernetes. Com o lançamento do Ceph Octopus e do cephadm, as antigas formas de implantar o Ceph como ceph-ansible ou ceph-deploy foram descontinuadas.

### 1ª Etapa - Instalação do Cephadm em um dos hosts




### 2ª Etapa - Deploy


### 3ª Etapa - Eventuais Upgrades
