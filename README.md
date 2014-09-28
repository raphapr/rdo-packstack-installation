![](http://latincloud.lccv.ufal.br/img/logos/lccv.png)

OpenStack
=========

Sistemas de Computação em Nuvem vem sendo adotados por várias instituições como forma de disponibilizar diversos tipos de serviços, como armazenamento de dados, hospedagem de servidores, entre outros. Mantendo o foco em IaaS ( Infrastructure as a Service ), o Openstack surge como uma solução importante por oferecer escalabilidade e disponibilidade de serviço, fornecendo um infra-estrutura altamente flexível e de rápido provisionamento.

O Openstack é uma colaboração global de desenvolvedores e tecnólogos de computação em nuvem produzindo uma plataforma de computação em nuvem ubíqua e de código aberto para nuvens públicas ou privadas.Na prática o openstack é um conjunto de softwares direcionados à configuração e gerência de um ambiente de computação em nuvem, aproveitando tecnologias já existentes em ambientes Linux. 

Índice
=========

* [1. Infraestrutura e Configuração de Rede](#1.-infraestrutura-e-configuração-de-rede)
* [2. Pré-configuração Packstack](#2.-pré-configuração-packstack)
  * [2.2 Configurando os hostnames](#2.2-configurando-os-hostnames)
    * [2.2.2 Verificando conectividade](#2.2.2-verificando-conectividade)
  * [2.3 Configurando o SSH](#2.3-configurando-o-ssh)
* [3. Instalando e executando o Packstack](#3.-instalando-e-executando-o-packstack)
  * [3.1 Repositórios](#3.1-repositórios)
  * [3.2 Executando o instalador packstack](#3.2-executando-o-instalador-packstack)
* [4. Pós-configuração de instalação](#4.-pós-configuração-de-instalação)
  * [4.1 Configurando a rede externa](#4.1-configurando-a-rede-externa)
  * [4.2 Criando a rede externa](#4.2-criando-a-rede-externa)
  * [4.3 Criando a tenant network (tenant network)](#4.3-criando-a-tenant-network-(tenant-network))
* [5. Instanciando uma máquina virtual](#5.-instanciando-uma-máquina-virtual)
* [6. Notas importantes](#6.-notas-importantes)
* [Referências](#referências)

# 1. Infraestrutura e Configuração de Rede

Como mencionado anteriormente trabalhamos em ambiente GNU/Linux, mais especificamente temos como base a distribuição centOS e OpenStack versão IceHouse. No entanto este guia  também serve para Distribuições GNU/Linux Fedora e Red Hat.

Utilizamos três RU's SUN FIRE X4170, cada máquina conta com:

* 2 processadores Intel(R) Xeon(R) CPU X5570
* 24GB de memória RAM
* 128GB de HD
* 2 interface infiniband ( rede de interconexão de alto desempenho )
* 4 interfaces Gigabit Ethernet

O processo de instalação foi realizado no CentOS 6.5, utilizando a ferramenta [packstack](https://wiki.openstack.org/wiki/Packstack) de instalação. Nós consideramos uma arquitetura multi-node com o módulo OpenStack Neutron que requer três tipos de nós:

* **controller**: máquina que hospeda os serviços de gerência (Keystone, Glance, Nova, Horizon...)
* **network**: máquina que hospeda os serviços de rede  e é responsável por fornecer a rede virtual responsável por conectar as máquinas virtuais na rede externa(neutron)
* **compute**: máquina que hospeda as máquinas virtuais (hypervisor)


![](https://raw.githubusercontent.com/raphapr/rdo-packstack-installation/master/network.jpg)


Para uma instalação OpenStack Multi-node você precisará de três interfaces de rede:

* **Rede de gerenciamento** (eth0): Rede utilizada para gerência, não acessível pela rede externa.
* **Rede de tráfego entre VMs** (eth1): Rede utilizada como rede interna para o tráfego entre máquinas virtuais no OpenStack.
* **Rede externa** (eth2): Esta rede está conectada apenas no network node para fornecer acesso externo às Máquinas virtuais.


# 2. Pré-configuração Packstack

Antes de instalar o OpenStack através do packstack, primeiramente preparamos a rede das máquinas.

## 2.1 Configurando a rede 

### Controller node

No nó controller, configuramos uma interface para a rede gerência (eth0) da seguinte maneira:

Modifique o arquivo  "/etc/sysconfig/network-scripts/ifcfg-eth0" como abaixo:

<pre>
# Internal Network
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.11
</pre>

### Network node

No nó network, inicialmente configuramos duas interfaces de rede:

/etc/sysconfig/network-scripts/ifcfg-eth0

<pre>
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.21
NETMASK=255.255.255.0
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth1

<pre>
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.83.21
NETMASK=255.255.255.0
</pre>

### Compute node

No nó compute, configuramos duas interfaces de rede:

/etc/sysconfig/network-scripts/ifcfg-eth0
<pre>
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.31
NETMASK=255.255.255.0
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth1
<pre>
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.83.31
NETMASK=255.255.255.0
</pre>

## 2.2 Configurando os hostnames

Esta etapa deve ser feita para todos os nós.

Após configurar as interfaces, reinicie o serviço de rede:
<pre>
# service network restart
</pre>
Configure o nome da máquina:

Modifique a linha que começa com '''HOSTNAME=''' no arquivo '''/etc/sysconfig/network''' como abaixo:
<pre>
HOSTNAME=controller
</pre>

Modifique o arquivo '''/etc/hosts''' como abaixo:
<pre>
127.0.0.1       localhost
192.168.0.11    controller
192.168.0.21    network
192.168.0.31    compute01
</pre>

### 2.2.2 Verificando conectividade

Para todos os nós, dê o ping de um site na internet:

<pre>
# ping -c 4 openstack.org
PING openstack.org (174.143.194.225) 56(84) bytes of data.
64 bytes from 174.143.194.225: icmp_seq=1 ttl=54 time=18.3 ms
64 bytes from 174.143.194.225: icmp_seq=2 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=3 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=4 ttl=54 time=17.4 ms

--- openstack.org ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3022ms
rtt min/avg/max/mdev = 17.489/17.715/18.346/0.364 ms
</pre>

Dê o ping do controller para o network e compute, e vice-versa.


**Atenção: A partir daqui todas as configurações serão feitas no controller.**

## 2.3 Configurando o SSH

Configure o acesso SSH sem senha através de certificados RSA do controller para os demais nós:

<pre>
# ssh-keygen -t rsa
# ssh-copy-id controller
# ssh-copy-id network
# ssh-copy-id compute01
</pre>

# 3. Instalando e executando o Packstack

## 3.1 Repositórios

Atualize os pacotes atuais do sistema:
<pre>
# sudo yum update -y
</pre>

Configure os repositórios RDO:
<pre>
# sudo yum install -y https://rdo.fedorapeople.org/rdo-release.rpm
</pre>

Instale o instalador Packstack:
<pre>
# sudo yum install -y openstack-packstack
</pre>

## 3.2 Executando o instalador packstack

Gere um _answers file_ contendo todas as configurações necessárias para a instalação do ambiente OpenStack:

<pre>
# packstack --gen-answer-file=packstack-answers-LCCV.txt
</pre>

Para essa instalação, nosso arquivo _answer files_ difere do padrão pelas linhas abaixo:

<pre>
CONFIG_CINDER_INSTALL=n
CONFIG_NOVA_COMPUTE_HOSTS=192.168.82.31
CONFIG_NEUTRON_SERVER_HOST=192.168.82.21
CONFIG_NEUTRON_LBAAS_HOSTS=192.168.82.11
CONFIG_NEUTRON_OVS_TENANT_NETWORK_TYPE=gre
CONFIG_NEUTRON_OVS_TUNNEL_RANGES=1000:3000
CONFIG_NEUTRON_OVS_TUNNEL_IF=eth1
CONFIG_NEUTRON_ML2_FLAT_NETWORKS=*
</pre>

+ [Answer file completo](https://github.com/raphapr/rdo-packstack-installation/blob/master/packstack-answers-LCCV.txt "Answers file")
+ [Tabela com informações das chaves de configuração](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/3/html/Getting_Started_Guide/ch04s03s02.html "Tabela com informações das chaves de configuração")

Uma vez configurado seu answers file, inicie a instalação com o comando:

<pre>
# packstack --answer-file=packstack-answers-LCCV.txt
</pre>

# 4. Pós-configuração de instalação

## 4.1 Configurando a rede externa

Assumindo que a rede externa é a 192.168.84.0/24, e a interface de rede utilizada ela é a **eth2**, edite os seguintes arquivo de configuração do **network node**:

/etc/sysconfig/network-scripts/ifcfg-br-ex

<pre>
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.84.2    # Seu IP na rede externa, pode ser qualquer um.
NETMASK=255.255.255.0  # seu netmask
GATEWAY=192.168.84.1   # seu gateway
DNS1=192.168.84.25     # seu nameserver
ONBOOT=yes
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth2

<pre>
DEVICE=eth2
HWADDR=52:54:00:92:05:AE # endereço MAC
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
</pre>

Reinicie o network:

<pre>
# service network restart
</pre>

## 4.2 Criando a rede externa

No **controller node**, carregue as credenciais de administrador:

Lembrando que todas 

<pre>
# . /root/keystonerc_admin
</pre>

Insira os seguintes comandos:

<pre>
# neutron net-create ext-net --shared --router:external=True
</pre>

<pre>
# neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=192.168.84.3,end=192.168.84.254 \
  --disable-dhcp --gateway 192.68.84.1 192.168.84.0/24
</pre>

## 4.3 Criando a tenant network (tenant network)

A tenant network é a rede interna de acesso das instancias, sua arquitetura isola a rede das outras. Crie com os seguintes comandos:

<pre>
# neutron net-create demo-net 
# neutron net-create demo-net
# neutron subnet-create demo-net --name demo-subnet --dns-nameserver 8.8.8.8 --gateway 10.0.0.1 10.0.0.0/24
</pre>

**Crie um roteador para a rede interna e o anexe na rede externa e redes internas à ela**

<pre>
# neutron router-create demo-router
</pre>

Anexe o roteador na sub-rede criada:

<pre>
# neutron router-interface-add demo-router demo-subnet
</pre>

Anexe o roteador na rede externa configurando-a como o gateway:

<pre>
# neutron router-gateway-set demo-router ext-net
</pre>

Certifique-se de que as redes foram criadas:

<pre>
# neutron net-list
# neutron subnet-list
</pre>

# 5. Instanciando uma máquina virtual

Registre no Nova sua chave pública RSA:

<pre>
# nova keypair-add --pub-key ~/.ssh/id_rsa.pub demo-key
</pre>

Registre a imagem CirrOS de testes nos repositórios do Glance:

<pre>
# glance image-create \
  --copy-from http://download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img \
  --is-public true \
  --container-format bare \
  --disk-format qcow2 \
  --name cirros
</pre>

Libere as portas para ICMP (ping) e SSH:

<pre>
# neutron security-group-rule-create --protocol icmp default
# neutron security-group-rule-create --protocol tcp \
  --port-range-min 22 --port-range-max 22 default
</pre>

Finalmente crie a máquina virtual:

<pre>
# nova boot --flavor m1.tiny --image cirros --nic net-id=8asc64-d8da-44b6-91cd-4acc87ee3500 --security-group default --key-name demo-key vm_teste
</pre>

Veja o **net-id** de sua rede através do comando **neutron net-list** 

Libere um endereço de IP flutuante da rede externa:

<pre>
# nova floating-ip-create external
+-------------+-------------+----------+----------+
| Ip          | Instance Id | Fixed Ip | Pool      |
+-------------+-------------+----------+----------+
| 192.168.84.3 | None        | None     | external |
+-------------+-------------+----------+----------+
</pre>

Atribua para a nova instância:

<pre>
# nova add-floating-ip vm_teste 192.168.84.3
</pre>

Verifique a conectividade com ping/ssh:

<pre>
# ping 192.168.84.3
# ssh cirros@192.168.84.3
</pre>

Senha padrão do CirroS: cubswin:)

# 6. Integrando com LDAP

## 6.1 Configurando Keystone

Modifique o arquivo de configuração do Keystone:

/etc/keystone/keystone.conf

<pre>
[identity]
#driver = keystone.identity.backends.sql.Identity
driver = keystone.identity.backends.ldap.Identity
</pre>

Defina a localização do servidor LDAP:

<pre>
[ldap]
url = ldap://localhost
user = dc=Manager,dc=example,dc=org
password = samplepassword
suffix = dc=example,dc=org
use_dumb_member = False
allow_subtree_delete = False
dumb_member=cn=dumb,dc=nonexistent
</pre>

Crie organizational units (OU) no diretório LDAP e defina sua localização correspondente no arquivo 'keystone.conf':

<pre>
user_tree_dn = ou=Users,dc=example,dc=org
user_objectclass = inetOrgPerson
user_id_attribute=uid   # usamos uid para fazer a busca no ldap
user_name_attribute=uid #
user_mail_attribute=mail
user_pass_attribute=userPassword
user_attribute_ignore=default_project_id,tenants,enabled
tenant_tree_dn = ou=Groups,dc=example,dc=org
tenant_objectclass = groupOfNames
tenant_id_attribute=cn
tenant_member_attribute=member
tenant_desc_attribute=description
tenant_attribute_ignore=enabled
role_tree_dn = ou=Roles,dc=example,dc=org
role_objectclass = organizationalRole
</pre>

É recomendável que estejam habilitadas as funções apenas de leitura no LDAP:

<pre>
user_allow_create = False
user_allow_update = False
user_allow_delete = False
 
tenant_allow_create = False
tenant_allow_update = False
tenant_allow_delete = False
 
role_allow_create = False
role_allow_update = False
role_allow_delete = False
</pre>

## 6.2 Reabiliatando os Serviços

Ao realizar a integração com o LDAP o keystone não mais usará o back-end nativo, ou seja todos os usuários e serviços deixarão de funcionar, requerendo que seja feito o cadastro de cada serviço no LDAP.

Recrie usuário, projeto e regra admin:

<pre>
$ export OS_SERVICE_TOKEN=ADMIN_TOKEN
$ export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0

$ keystone user-create --name=admin --pass=ADMIN_PASS --email=ADMIN_EMAIL
$ keystone role-create --name=admin

$ keystone tenant-create --name=admin --description="Admin Tenant"
$ keystone user-role-add --user=admin --tenant=admin --role=admin

$ keystone role-create --name=_member_
$ keystone user-role-add --user=admin --role=_member_ --tenant=admin

$ keystone tenant-create --name=service --description="Service Tenant"
</pre>

Crie um serviço para Serviço de Identidade (keystone):

<pre>
$ keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
</pre>

Associe um endpoint ao Serviço de Identidade:

<pre>
$ keystone endpoint-create --service-id=$(keystone service-list | awk '/ identity / {print $2}') --publicurl=http://controller:5000/v2.0 --internalurl=http://controller:5000/v2.0 --adminurl=http://controller:35357/v2.0
</pre>


Este processo de criar um serviço e associá-lo a um endpoint será feito para cada Serviço do OpenStack:


<pre>
$ keystone service-list

+----------------------------------+------------+----------------+------------------------------+
|                id                |    name    |      type      |         description          |
+----------------------------------+------------+----------------+------------------------------+
| bb064511680e189nfa9ee0772cb45bec | ceilometer |    metering    |          Telemetry           |
| 60f87d388f142983ddadf467a1dcc78a |   glance   |     image      |   OpenStack Image Service    |
| 44a11ae78add4983b0c8d70713a1ac4f |    heat    | orchestration  |        Orchestration         |
| 82f6f98738734f54b5d89857bc04cc4d |  heat-cfn  | cloudformation | Orchestration CloudFormation |
| 42js4c7f10e9d398ab5573ab330cf6ee |  keystone  |    identity    |  OpenStack Identity Service  |
| 852e2uu2298u26ccbc958baac67216c8 |  neutron   |    network     |     OpenStack Networking     |
| 27f8902f9s124b50be356e656c6870d7 |    nova    |    compute     |      OpenStack Compute       |
| 97719281s5454bf983cvba4031a359aa |  nova_ec2  |      ec2       |   EC2 Compatibility Layer    |
| 93f8ed2826944a1982c7532982chfj22 |   novav3   |   computev3    | Openstack Compute Service v3 |
+----------------------------------+------------+----------------+------------------------------+

</pre>

### Glance

Crie um usuário:

<pre>
$ keystone user-create --name=glance --pass=GLANCE_PASS --email=glance@example.com
$ keystone user-role-add --user=glance --tenant=service --role=admin
</pre>

Crie um serviço:

<pre>
$ keystone service-create --name=glance --type=image --description="OpenStack Image Service"
</pre>

Associe um endpoint:

<pre>
$ keystone endpoint-create --service-id=$(keystone service-list | awk '/ image / {print $2}') --publicurl=http://controller:9292 --internalurl=http://controller:9292 --adminurl=http://controller:9292
</pre>

### Nova

Crie um usuário:

<pre>
$ keystone user-create --name=nova --pass=NOVA_PASS --email=nova@example.com
$ keystone user-role-add --user=nova --tenant=service --role=admin
</pre>

Crie um serviço:

<pre>
$ keystone service-create --name=nova --type=compute --description="OpenStack Compute"
</pre>

Associe um endpoint:

<pre>
$ keystone endpoint-create --service-id=$(keystone service-list | awk '/ compute / {print $2}') --publicurl=http://controller:8774/v2/%\(tenant_id\)s --internalurl=http://controller:8774/v2/%\(tenant_id\)s --adminurl=http://controller:8774/v2/%\(tenant_id\)s
</pre>

### Neutron

Crie um usuário:

<pre>
$ keystone user-create --name neutron --pass NEUTRON_PASS --email neutron@example.com
$ keystone user-role-add --user neutron --tenant service --role admin
</pre>

Crie um serviço:

<pre>
$ keystone service-create --name neutron --type network --description "OpenStack Networking"
</pre>

Associe um endpoint:

<pre>
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ network / {print $2}') --publicurl http://controller:9696 --adminurl http://controller:9696 --internalurl http://controller:9696
</pre>

### Nova_ec2

Crie um serviço:

<pre>
$ keystone service-create --name nova_ec2 --type ec2 --description "EC2 Compatibility Layer"

</pre>

Associe um endpoint:

<pre>
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ ec2 / {print $2}') --publicurl http://192.168.82.11:8773/services/Cloud --adminurl http://192.168.82.11:8773/services/Admin --internalurl http://192.168.82.11:8773/services/Cloud
</pre>

### Novav3

Crie um serviço:

<pre>
$ keystone service-create --name novav3 --type computev3 --description "Openstack Compute Service v3"
</pre>

Associe um endpoint:

<pre>
$ keystone endpoint-create --service-id $(keystone service-list | awk '/ v3 / {print $2}') --publicurl http://192.168.82.11:8774/v3 --adminurl http://192.168.82.11:8774/v3 --internalurl http://192.168.82.11:8774/v3
</pre>

### Heat

Crie um usuário:

<pre>
$ keystone user-create --name=heat --pass=HEAT_PASS --email=heat@example.com
$ keystone user-role-add --user=heat --tenant=service --role=admin
</pre>

Crie um serviço:
<pre>
$ keystone service-create --name=heat --type=orchestration --description="Orchestration"
</pre>

Associe um endpoint:

<pre>
keystone endpoint-create --service-id=$(keystone service-list | awk '/ orchestration / {print $2}') --publicurl=http://controller:8004/v1/%\(tenant_id\)s --internalurl=http://controller:8004/v1/%\(tenant_id\)s --adminurl=http://controller:8004/v1/%\(tenant_id\)s
</pre>

### Heat-cnf

Crie um serviço:

<pre>
$ keystone service-create --name=heat-cfn --type=cloudformation --description="Orchestration CloudFormation"
</pre>

Associe um endpoint:

<pre>
$ keystone endpoint-create --service-id=$(keystone service-list | awk '/ cloudformation / {print $2}') --publicurl=http://controller:8000/v1 --internalurl=http://controller:8000/v1 --adminurl=http://controller:8000/v1
</pre>

### Ceilometer

Crie um usuário:

<pre>
$ keystone user-create --name=ceilometer --pass=CEILOMETER_PASS --email=ceilometer@example.com
$ keystone user-role-add --user=ceilometer --tenant=service --role=admin
</pre>

Crie um serviço:

<pre>
$ keystone service-create --name=ceilometer --type=metering --description="Telemetry"
</pre>

Associe um endpoint:

<pre>
$ keystone endpoint-create --service-id=$(keystone service-list | awk '/ metering / {print $2}') --publicurl=http://controller:8777 --internalurl=http://controller:8777 --adminurl=http://controller:8777
</pre>

### Reinicie os Serviços

<pre>
service openstack-glance-api restart
service openstack-glance-registry restart
service openstack-nova-api start
service openstack-nova-cert start
service openstack-nova-consoleauth restart
service openstack-nova-scheduler restart
service openstack-nova-conductor restart
service openstack-nova-novncproxy restart
service openstack-heat-api restart
service openstack-heat-api-cfn restart
service openstack-heat-engine restart
service openstack-ceilometer-api restart
service openstack-ceilometer-notification restart
service openstack-ceilometer-central restart
service openstack-ceilometer-collector restart
service openstack-ceilometer-alarm-evaluator restart
service openstack-ceilometer-alarm-notifier restart
</pre>


# 7. Notas importantes

## A conexão com a internet nas máquinas virtuais está muito lenta?

No **Network node**, desative o *Generic Receive Offload* (GRO) da interface externa (eth2) através do ethtool:

<pre>
ethtool -K eth2 gro off
</pre>

Adicione o comando no arquivo /etc/rc.local para que a alteração permaneça após o boot.

## Deletando uma rede no Neutron

Caso você esteja com problemas em deletar uma rede no neutron, a sequência de comandos a seguir pode ser útil:

<pre>
# neutron router-gateway-clear <router-id>
# neutron router-interface-delete <router-id> <tenant-network-id>
# neutron router-delete <router-id>
# neutron net-delete <tenant-network-id>
</pre>

# Referências

* http://oddbit.com/rdo-hangout-multinode-packstack-slides/
* http://docs.openstack.org/icehouse/install-guide/install/yum/content/
* http://www.monkey-code.com/blog/2013/10/04/building-a-multi-node-openstack-cloud/
* https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/
