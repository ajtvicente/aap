# Red Hat Ansible Automation Platform 2.5

![AAP-02](https://www.redhat.com/rhdc/managed-files/ansible-hero-img-ohs1.png)


## Sumário
- [Red Hat Ansible Automation Platform 2.5](#red-hat-ansible-automation-platform-25)
    - [Sumário](#sumário)
- [Arquitetura](#arquitetura)
- [PostgreSQL](#postgresql)
    - [Habilite *hstore* para PostgreSQL](#habilite-hstore-para-postgresql)
    - [Tuning Kernel Linux](#tuning-kernel-linux)
    - [Tuning PostgreSQL](#tuning-postgresql)
- [Deploy Ansible](#deploy-ansible)


# Arquitetura

Foram disponibilizados inicialmente seis servidores para o deploy do Red Hat Ansible Automation Platform 2.5 (CaC), conforme listado abaixo:

* VRBPCLAP0552
* VRBPCLAP0566
* VRBPCLAP0567
* VRBPCLAP0568
* VRBPCLAP0569
* VRBPCLAP0570

Pontos de atenção:

* Na topologia de referência de crescimento (growth) para implantação em RPM tem como requisitos 6 servidores:

VM-Count | Type
---------|-----------------------
1        | Platform Gateway
1        | Automation Controller
1        | Automation Hub
1        | Execution Node
1        | PostgreSQL
1        | Redis

![AAP-02](https://access.redhat.com/webassets/avalon/d/Red_Hat_Ansible_Automation_Platform-2.5-Tested_deployment_models-en-US/images/a6bab844df620cd5ae45fac1ec8f2b85/rpm-a-env-a.png)

* Na topologia de referência de crescimento (growth) para implantação em Container tem como requisitos 7 servidores:

VM-Count | Type
---------|-----------------------
1        | Load Balancer
1        | Platform Gateway
1        | Automation Controller
1        | Automation Hub
1        | Execution Node
1        | PostgreSQL
1        | Redis

![AAP-03](https://access.redhat.com/webassets/avalon/d/Red_Hat_Ansible_Automation_Platform-2.5-Tested_deployment_models-en-US/images/02b948e9d139f9a478e80ba4164dceae/cont-a-env-a.png)


Desta forma, com a disponibilização de um ambiente com 6 servidores, recomendamos a implantação inicial em *RPM*, e posterior migração para *Container*, em uma topologia de crescimento (growth), onde os serviços do cluster encontram-se em servidores apartados (isolados), possibilitando o deploy de réplicas destes serviços.

Para o Bradesco, sugerimos duas propostas de arquitetura, onde o diferencial entre elas é o deploy do Private Automation Hub ou alta disponibilidade no Automation Controller, conforme abaixo:

![AAP-02](https://github.com/ajtvicente/Red_Hat_Ansible_Automation_Platform/blob/ajtvicente/app/files/CaC-Bradesco.png)

* Optional One:

VM-Count | Type
---------|-----------------------
1        | Platform Gateway
1        | Automation Controller
1        | Automation Hub
1        | Execution Node
1        | PostgreSQL
1        | Redis

* Optional Two:

VM-Count | Type
---------|-----------------------
1        | Platform Gateway
1        | Automation Controller
1        | Automation Controller
1        | Execution Node
1        | PostgreSQL
1        | Redis


# PostgreSQL

Instale o PostgreSQL, seguindo a matrix de compatibilidade para o Red Hat Enterprise Linux e Red Hat Ansible Automation Platform:

Database         | Operating System                | Ansible
-----------------|---------------------------------|----------------------------------------
PostgreSQL 13    | Red Hat Enterprise Linux 8 or 9 | Red Hat Ansible Automation Platform 2.4
PostgreSQL 15    | Red Hat Enterprise Linux 9      | Red Hat Ansible Automation Platform 2.5

Instale o banco de dados:

```
# dnf install -y postgresql:15/server
```

Inicie, ative e verifique o status do banco de dados PostgreSQL:

```
# postgresql-setup --initdb 
# systemctl start postgresql.service 
# systemctl enable postgresql.service 
# systemctl status postgresql.service
```

Verifique a versão do Banco de Dados:

```
# postgres -V
```

Atribua uma senha parra o usuário *postgres*:

```
# passwd postgres
Changing password for user postgres.
New password:
BAD PASSWORD: The password fails the dictionary check - it is too simplistic/systematic
Retype new password:
passwd: all authentication tokens updated successfully.
```

Atribua uma senha de administração para o banco de dados PostgreSQL:

```
# su - postgres
$ psql -c "ALTER USER postgres WITH PASSWORD '<newpassword>';"
ALTER ROLE
$ exit
```

Crie os usuários e databases, e atribua o owner as databases:

```
# sudo -i -u postgres createuser -P awx
Enter password for new role:
Enter it again:
# sudo -i -u postgres createdb -O awx awx

# sudo -i -u postgres createuser -P automationhub
Enter password for new role:
Enter it again:
# sudo -i -u postgres createdb -O automationhub automationhub

# sudo -i -u postgres createuser -P automationedacontroller
Enter password for new role:
Enter it again:
# sudo -i -u postgres createdb -O automationedacontroller automationedacontroller

# sudo -i -u postgres createuser -P automationgateway
Enter password for new role:
Enter it again:
# sudo -i -u postgres createdb -O automationgateway automationgateway
```

Edite *pg_hba.conf* com as permissões de rede necessárias para que os hosts possam acessar o banco de dados PostgreSQL:

```
# vim /var/lib/pgsql/data/pg_hba.conf
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
host    all             all             <ip>/32                 md5
host    all             all             <ip>/32                 md5
host    all             all             <ip>/32                 md5
host    all             all             <ip>/32                 md5
# IPv6 local connections:
host   all             all              <ip>/32                 md5
```

Edite *postgresql.conf* com o IP e Port para acesso ao banco de dados PostgreSQL:

```
# vim /var/lib/pgsql/data/postgresql.conf
listen_addresses = '<ip>'               # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = 5432                             # (change requires restart)
```

Reinicie e verifique o status do serviço do banco de dados PostgreSQL:

```
# systemctl restart postgresql.service
# systemctl status postgresql.service
```

Aplique as regras de firewall necessárias para o acesso ao banco de dados PostgreSQL:

```
# firewall-cmd --zone=public --permanent --add-port=5432/tcp --permanent
# firewall-cmd --add-service=postgresql --permanent
# firewall-cmd --reload
# firewall-cmd --list-services
```

Teste o acesso ao banco de dados PostgreSQL via comando:

```
# psql -U postgres -h <ip_or_hostname> -p 5432 template1
Password for user postgres: 
```

## Habilite *hstore* para PostgreSQL

Habilite a extensão *hstore* para o banco de dados PostgreSQL do Automation Hub.

A partir do Red Hat Ansible Automation Platform 2.4, o script de migração do banco de dados usa campos *hstore* para armazenar informações, portanto, a extensão *hstore* para o banco de dados PostgreSQL do Automation Hub deve ser habilitada:

* Esse processo é automático ao usar o instalador do Red Hat Ansible Automation Platform em um servidor PostgreSQL gerenciado.
* Se o banco de dados PostgreSQL for externo, você deverá habilitar a extensão *hstore* para o banco de dados PostgreSQL antes da instalação do Automation Hub.
* Se o *hstore* não for habilitado antes da instalação do Automation Hub, uma falha será gerada durante a migração do banco de dados.

Instale o pacote necessário para habilitar o *hstore*:

```
# dnf install -y postgresql-contrib
```

Habilite o *hstore* na base de dados do Automation Hub:

```
# su - postgres
$ psql -d automationhub -c "CREATE EXTENSION hstore;"
CREATE EXTENSION

$ exit
```

Verifique se o *hstore* está habilitado:

```
# su - postgres
$ psql -d automationhub -c "SELECT * FROM pg_available_extensions WHERE name='hstore'"
  name  | default_version | installed_version |                     comment                      
--------+-----------------+-------------------+--------------------------------------------------
 hstore | 1.7             | 1.7               | data type for storing sets of (key, value) pairs
(1 row)

$ exit
```

## Tuning Kernel Linux

Os valores abaixo devem ser usados como referência para a aplicação de tuning de banco de dados PostgreSQL, e devem ser ajustados conforme sizing de cada ambiente.

* Operating System Limits

Altere os limites do sistema operacional no Red Hat Enterprise Linux 8 e 9:

```
# vi /etc/security/limits.conf
*        soft    nofile    500000
*        hard    nofile    500000
root     soft    nofile    500000
root     hard    nofile    500000
postgres soft    nofile    500000
postgres hard    nofile    500000
*        soft    nproc     unlimited
*        hard    nproc     unlimited
root     soft    nproc     unlimited
root     hard    nproc     unlimited
postgres soft    nproc     unlimited
postgres hard    nproc     unlimited
*        soft    stack     unlimited
*        hard    stack     unlimited
root     soft    stack     unlimited
root     hard    stack     unlimited
postgres soft    stack     unlimited
postgres hard    stack     unlimited
```

Reinicie seu sistema operacional para alterar os limites:

```
# reboot
```


## Tuning PostgreSQL

Altere as configurações conforme recomendado pelo [PGTune](https://pgtune.leopard.in.ua/):

No exemplo abaixo, temos um banco de dados PostgreSQL com os seguiintes parâmetros:

Type                  |  Value
----------------------|--------
DB version            | 15
OS Type               | Linux
DB Type               | Online Transaction Processing
Total Memory (RAM)    | 16
Number of CPUs        | 4
Number of Connections | 8000
Data Storage          | SSD


Edite *postgresql.conf* com os parêmtros de tuning para o banco de dados PostgreSQL:

```
# vim /var/lib/pgsql/data/postgresql.conf

<Tuning>
```

Reinicie e verifique o status do serviço do banco de dados PostgreSQL:

```
# systemctl restart postgresql.service
# systemctl status postgresql.service
```

Verifique as configurações do PostgreSQL:

```
# su - postgres
$ psql
postgres=# SELECT name, context, unit, setting FROM pg_settings WHERE name IN ('max_connections', 'shared_buffers', 'effective_cache_size', 'listen_addresses', 'wal_level', 'work_mem');
         name         |  context   | unit |   setting    
----------------------+------------+------+--------------
 effective_cache_size | user       | 8kB  | 1572864
 listen_addresses     | postmaster |      | 10.156.28.22
 max_connections      | postmaster |      | 8000
 shared_buffers       | postmaster | 8kB  | 524288
 wal_level            | postmaster |      | replica
 work_mem             | user       | kB   | 262
(6 rows)


postgres=# SELECT name, setting, unit FROM pg_settings LIMIT 7;
            name            |  setting   | unit 
----------------------------+------------+------
 allow_in_place_tablespaces | off        | 
 allow_system_table_mods    | off        | 
 application_name           | psql       | 
 archive_cleanup_command    |            | 
 archive_command            | (disabled) | 
 archive_mode               | off        | 
 archive_timeout            | 0          | s
(7 rows)
```

Reinicie seu sistema operacional para alterar os limites:

```
# reboot
```

# Deploy Ansible

Gere a chave SSH em todos os servidores que compõem o cluster:

```
# ssh-keygen
```

Copie a chave SSH, comente no servidor que vai vai fazer a instalação:

```
# cat ~/.ssh/id_rsa.pub | ssh root@VRBPCLAP0552 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# cat ~/.ssh/id_rsa.pub | ssh root@VRBPCLAP0566 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# cat ~/.ssh/id_rsa.pub | ssh root@VRBPCLAP0567 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# cat ~/.ssh/id_rsa.pub | ssh root@VRBPCLAP0568 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# cat ~/.ssh/id_rsa.pub | ssh root@VRBPCLAP0569 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
# cat ~/.ssh/id_rsa.pub | ssh root@VRBPCLAP0570 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Teste o acesso SSH:

```
# ssh root@VRBPCLAP0552
# ssh root@VRBPCLAP0566
# ssh root@VRBPCLAP0567
# ssh root@VRBPCLAP0568
# ssh root@VRBPCLAP0569
# ssh root@VRBPCLAP0570
```

Habilite o repositório do Ansible em todos os servidores:

```
# subscription-manager repos --enable=ansible-automation-platform-2.5-for-rhel-9-x86_64-rpms
```

Descompacte o instalador:

```
# cd /opt/
# tar xvzf ansible-automation-platform-setup-bundle-<version>.tar.gz
# cd ansible-automation-platform-setup-bundle-<version>
```

Edite o arquivo de inventory:

```yml
# This is the AAP enterprise installer inventory file
# Please consult the docs if you're unsure what to add
# For all optional variables please consult the Red Hat documentation:
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/rpm_installation

# This section is for your AAP Gateway host(s)
# -----------------------------------------------------
[automationgateway]
VRBPCLAP0552
# gateway1.example.org
# gateway2.example.org

# This section is for your AAP Controller host(s)
# -----------------------------------------------------
[automationcontroller]
VRBPCLAP0566 node_type=control
# controller1.example.org
# controller2.example.org

[automationcontroller:vars]
peers=execution_nodes

# This section is for your AAP Execution host(s)
# -----------------------------------------------------
[execution_nodes]
VRBPCLAP0567 node_type=execution
# hop1.example.org node_type='hop'
# exec1.example.org
# exec2.example.org

# This section is for your AAP Automation Hub host(s)
# -----------------------------------------------------
[automationhub]
VRBPCLAP0568
# hub1.example.org
# hub2.example.org

# This section is for your AAP EDA Controller host(s)
# -----------------------------------------------------
[automationedacontroller]
# eda1.example.org
# eda2.example.org

[redis]
VRBPCLAP0569
# gateway1.example.org
# gateway2.example.org
# hub1.example.org
# hub2.example.org
# eda1.example.org
# eda2.example.org

[all:vars]
redis_mode=standalone

# Common variables
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/rpm_installation/appendix-inventory-files-vars#ref-general-inventory-variables
# -----------------------------------------------------

# AAP Gateway
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/rpm_installation/appendix-inventory-files-vars#ref-gateway-variables
# -----------------------------------------------------
automationgateway_admin_password=<set your own>
automationgateway_pg_host=<set your own>
automationgateway_pg_database=<set your own>
automationgateway_pg_username=<set your own>
automationgateway_pg_password=<set your own>

# AAP Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/rpm_installation/appendix-inventory-files-vars#ref-controller-variables
# -----------------------------------------------------
admin_password=<set your own>
pg_host=<set your own>
pg_database=<set your own>
pg_username=<set your own>
pg_password=<set your own>

# AAP Automation Hub
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/rpm_installation/appendix-inventory-files-vars#ref-hub-variables
# -----------------------------------------------------
automationhub_admin_password=<set your own>
automationhub_pg_host=<set your own>
automationhub_pg_database=<set your own>
automationhub_pg_username=<set your own>
automationhub_pg_password=<set your own>

# AAP EDA Controller
# https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/rpm_installation/appendix-inventory-files-vars#event-driven-ansible-controller
# -----------------------------------------------------
automationedacontroller_admin_password=<set your own>
automationedacontroller_pg_host=<set your own>
automationedacontroller_pg_database=<set your own>
automationedacontroller_pg_username=<set your own>
automationedacontroller_pg_password=<set your own>
```

Modelo de implantação do Redis (6-VMs):

![AAP-02](https://access.redhat.com/webassets/avalon/d/Red_Hat_Ansible_Automation_Platform-2.5-Planning_your_installation-en-US/images/cd74e4ccc48246637c57db5612871963/gw-clustered-redis.png)
