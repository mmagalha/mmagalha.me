+++
authors = [
    "mmagalha",
]
title = "Ansible K8S Onpremisses - Parte 1"
date = "2020-06-08T22:03:42-03:00"
description = "Neste artigo vemos como criar um cluster K8S de forma automatizada usando Ansible"
tags = [
    "Ansible",
    "proxmox",
    "k8s",
]
categories = [
    "Ansible",
    "automação",
]
series = ["How to"]
aliases = ["ansible-K8S"]
images = [
    "ansible.jpg",
]
+++
Neste artigo vemos como criar um cluster K8S de forma automatizada usando Ansible.
<!--more-->

## Criando um cluster Kubernetes em infra on-premisses com Ansible

Faz muito tempo que não escrevo nada e tenho me sentido na obrigação de devolver um pouco do que tenho aprendido em todos esses anos, sendo assim nada melhor do que falar de um assunto que, além de estar em alta, eu tenho me dedicado no momento.

Sem mais delongas…

Meu playbook está separado em 3 partes:

* Criação das VMs no Proxmox usando Ansible ( o que será tratado nesse artigo );
* Preparação das VMs para subir o cluster K8S usando Ansible
* Configuração do cluster K8S também com Ansible
* Configuração de alguns extras no cluster K8S (eu sei que eu falei que seriam 3 partes, SURPRESA … )

Na última década tenho me dedicado quase que exclusivamente ao VMware e de muitas formas acabei negligenciando outros hipervisos, mas como tenho me dedicado ao estudo de práticas de DevOps decidi que iria mudar meu “MINDSET” (não tenho a mínima ideia do que isso significa, mas vi o @Gomex usando e deve ser algo muito foda) e nas idas e vinda me deparei com o Proxmox uma distro baseada no meu querido Debian e que entrega uma séria de facilidades como VMs, Containers LXC, CEPH entre outros talvez um dia eu faça um artigo sobre ele.

Mas a estrela da noite é o Ansible, que segundo a Red Hat é uma ferramenta para automação que possui a capacidade de gerenciar configurações, realizar deploy de aplicações e até infraestrutura por não possuir agentes seu uso é muito simples além de usar em seus playbooks yaml e onde há yaml há vida. Para quem deseja saber um pouco mais sobre o Ansible aconselho os vídeos do @Badtux no canal do LunuxTips no YouTube aqui e aqui.

Bla, bla, bla e até agora nada de útil …

Então abaixo está a estrutura simplificada dos diretórios e arquivos que criei

no momento iremos nos concentrar no bloco install-vms e seus arquivos
``` 
├── install-vms
├── tasks
| └── main.yml
└── vars
└── main.yml
```


no primeiro arquivo o main.yml temos as tarefas de criação e customização das máquinas virtuais no ambiente Proxmox, esa rule usa o módulo do proxmox para clonar as VMs e iniciá-las ( https://docs.ansible.com/ansible/latest/modules/proxmox_kvm_module.html ), porém para setar as configurações do cloud-init foi necessário passar o comando diretamente, no final o pulo do gato um pequeno delay para que as outras rules aguardem as VMs subir.

# tasks file for install-vm
```

- name: Clone VMs
  proxmox_kvm:
    api_user    : "{{ pve_user }}"
    api_password: "{{ pve_pass }}"
    api_host    : "{{ pve_host }}"
    clone       : ubnttpl   # The VM source
    name        : k8s0{{ item }}  # The target VM name
    newid       : 100{{ item }}
    node        : pve01
    storage     : Local-01
    format      : qcow2
    timeout     : 500
    state       : started
  loop: "{{ vm_id }}"

- name: Set VM IP
  command:
    qm set 100{{ item }} --ipconfig0 ip=172.32.0.22{{ item }}/24,gw=172.32.0.254
  loop: "{{ vm_id }}"
  ignore_errors: False
  
- name: start VMs
  proxmox_kvm:
    api_user    : "{{ pve_user }}"
    api_password: "{{ pve_pass }}"
    api_host    : "{{ pve_host }}"
    vmid        : 100{{ item }}
    state       : started
  loop: "{{ vm_id }}"

- name: Wait 600 seconds for target connection to become reachable/usable
  wait_for_connection:
        delay: 600
```

no Arquivo acima temos algumas variáveis que são de extrema importância e que foram setadas no arquivo main.yml dentro da pasta vars

* pve_host: define o host de controle Proxmox, em resumo o IP da API
* pve_user: Usuário com permissões para criar VMs no Proxmox
* pve_pass: A senha do “cara ai de cima” nesse caso a senha não está declarada no arquivo e sim será passada através de uma variável de ambiente
* vm_id: Uma lista de “IDs” para as VMs que servirão de índice para criar e nomear as VMs do nosso cluster
* vm_name: “nessa variável criei um prefixo que será usado para nomear as VMs. 


# vars file for install-vms
```
pve_host: pve01
pve_user: root@pam
pve_pass: "{{ lookup('env','PVE_PASS') }}"
vm_id:
- 1
- 2
- 3
- 4
vm_name: "k8s0"

```

O arquivo principal do playbook está abaixo e realiza as operações de criar as VMs e configurá-las;
```
- hosts: pve
  user: root
  gather_facts: no
  pre_tasks:
  - name: 'atualizando repo'
    raw: 'apt-get update'
  - name: 'instalando o python'
    raw: 'apt-get install -y python3 python3-proxmoxer'
  roles:
  - { role: install-vms, tags: ["install_pve_vms"]}

- hosts: k8s
  become: yes
  user: mmagalha
  gather_facts: no
  pre_tasks:
  - name: 'atualizando repo'
    raw: 'apt-get update'
  - name: 'instalando o python'
    raw: 'apt-get install -y python3'
  roles:
  - { role: install-k8s, tags: ["install_k8s_role"]}

- hosts: k8s_master
  become: yes
  user: mmagalha
  roles:
  - { role: create-cluster, tags: ["create_cluster_role"]}

- hosts: k8s_workers
  become: yes
  user: mmagalha
  roles:
  - { role: join-workers, tags: ["join_workers_role"]} 
```

Neste pequeno texto tratamos da automação do processo de criação das VMs do nosso cluster no Proxmox, não vou me alongar explicando o resto do código, pois o Jeferson já mostrou isso tudo com maestria e vc pode encontrar aqui e o código completo você pode encontrar na minha conta do github aqui

Então …

Sabe a tal quarta parte a instalação dos extras, vai ficar para um proximo post e que me forçará a escrever um outro artigo lá vamos instalar:

* CNI ( weavenet )
* MetalLB ( um fantástico load balance para infras on-premisses )
* ISTIO ( nosso queridinho )