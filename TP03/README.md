# TP03 - Baptiste PELLARIN
02/02/2022
----

Pour plus de rapidité j'ai modifié `ansible.cfg` pour qu'il prenne en compte tous les fichiers dans le dossier `inventories/`

```ini
[defaults]
inventory =~/CPE/devOPS/TP03/inventories,/etc/ansible/hosts
```

## 3-1 : Documentation de l'inventaire et des commandes de base

Vérification si les hôtes sont joignables
```bash
$ ansible all -m ping
```

Récupération de la distrib du serveur
```bash
$ ansible all -m setup -a "filter=ansible_distribution*"
```
Suppression de apache
```bash
$ ansible all -m yum -a "name=httpd state=absent" --become
```

`setup.yml`
```yaml
all:
  vars:
    ansible_user: centos # Utilisateur (autorisé à sudo)
    ansible_ssh_private_key_file: ~/CPE/devOPS/TP03/ansible_priv_key # Localisation de la clé privée
  children:
    prod:
      hosts: baptiste.pellarin.takima.cloud # FQDN du serveur   
```

## Roles

(J'ai utilisé cette doc)[https://devopssec.fr/article/roles-ansible]

J'ai supprimé les dossiers `tests`/`defaults`/`handlers`/`vars` du rôle.

```yaml
- name: Installation docker (via role)
  hosts: all
  gather_facts: false
  become: yes

  roles:
    - docker # On utilise le role docker
```

# Déploiement de l'application

## 3-3 : docker_container playbook

```yaml
- name: Création du réseau d'interconnexion
  docker_network:
    name: tp03

- name: Copy environement db
  ansible.builtin.copy:
    src: ./roles/simple_api/files/db.env
    dest: db.env
    mode: '0644'

- name: Création du conteneur de la base de données
  docker_container:
    pull: true
    name: psql_tp01
    image: swano/tp-devops-cpe:database
    env_file: db.env # On change les paramètres de création de la base de données

- name: Copy environement api
  ansible.builtin.copy:
    src: ./roles/simple_api/files/api.env
    dest: api.env
    mode: '0644'

- name: Création du conteneur de l'application
  docker_container:
    name: spring
    image: swano/tp-devops-cpe:simple-api
    pull: true
    links:
      - "psql_tp01:psql_tp01" # Liaison avec la BDD
    env_file: api.env  # On change les paramètres de connexion

- name: Création du conteneur pour le proxy 
  docker_container:
    name: reverse_proxy
    image: swano/tp-devops-cpe:http_server
    pull: true
    ports:
      - "80:80"  # On expose le port 80 sur la machine hôte
    links:
      - "spring:spring" # Liaison avec spring

- name: Ajout des conteneurs au réseau précédement créé
  docker_network:
    name: tp03
    connected:
      - spring
      - reverse_proxy
      - psql_tp01
    appends: yes
```

## Frontend



Ajout du déploiement du frontend dans la playbook

```yaml
...

- name: Création du conteneur de frontend
  docker_container:
    name: frontend
    pull: true
    image: swano/tp-devops-cpe:frontend

...
```

## Autodeploiement

```yaml
name: CI devops 2022 CPE auto-deployment
on: 
  repository_dispatch:
    types: [on-demand-deployment]
  push:
    branches: 
      - main
    pull_request:

jobs:
  playbook_ansible:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Playbook deployment
        uses: dawidd6/action-ansible-playbook@v2
        with:
          playbook: setup_app.yml
          key: ${{ secrets.PRIVATE_KEY }}
          options: |
            --inventory inventories/setup.yml
````
