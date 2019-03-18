# ansible-k3s

## But du projet

Faciliter le déploiement de cluster K3s via Scaleway et Ansible

K3s est un projet développer par Rancher permettant de disposer d'un Kubernetes le plus léger possible en terme d'empreinte disque/mémoire, et faire fonctionner le tout sur de l'IoT ou des petites machines types ARM.
[https://k3s.io/](https://k3s.io/)

## Déployer les VMs avec Ansible

[Documentation officielle du module Ansible pour Scaleway](https://docs.ansible.com/ansible/latest/modules/scaleway_image_facts_module.html#scaleway-image-facts-module)

### Prérequis

* Avoir un compte sur Scaleway
* Avoir une clé SSH, avoir la clé publique à la racine du projet, et l'appeler admin.pub
* Installer les package pip sur la machine locale

```bash
pip install jinja2 PyYAML paramiko cryptography packaging
```

* Installer Ansible depuis les sources (>= 2.8devel)
* Installer jq

### Token

Créer un token sur le site de Scaleway pour les accès distants et le stocker dans un fichier scaleway_token

```bash
export SCW_API_KEY='aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
```

Sourcer le fichier

```bash
source scaleway_token
```

### Générer les machines

Utiliser le playbook create_arm_vms.yaml pour :

* récupérer l'ID d'organisation du compte Scaleway
* récupérer un ID d'image compatible debian Stretch
* ajouter si nécessaire la clé SSH de l'admin
* créer autant de machines que nécessaire

```bash
ansible-playbook create_arm_vms_scaleway.yml
```

### Mise en place de l'inventaire dynamique (inventory.yml)

```YAML
plugin: scaleway
regions:
  - par1
tags:
  - k3smaster
  - k3sworker
```

Vérifier le retour de la commande ansible-inventory

```JSON
ansible-inventory --list -i inventory.yml
{
    "_meta": {
        "hostvars": {
            "x.x.x.x": {
                "arch": "arm64",
                "commercial_type": "ARM64-2GB",
                "hostname": "k3smaster1",
[...]
    "k3sworker": {
        "hosts": [
            "x.x.x.x",
            "y.y.y.y",
            "z.z.z.z"
        ]
    }
```

Récupérer les empreintes de nos nouveaux serveurs pour éviter une erreur lors de la première connexion

```bash
ansible-inventory --list -i inventory.yml | jq -r '.k3smaster.hosts | .[]' | xargs ssh-keyscan >> ~/.ssh/known_hosts
ansible-inventory --list -i inventory.yml | jq -r '.k3sworker.hosts | .[]' | xargs ssh-keyscan >> ~/.ssh/known_hosts
```

Ajouter sa clé SSH dans l'agent, puis vérifier qu'on peut se connecter à tous les serveurs via Ansible

```bash
eval `ssh-agent`
ssh-add myprivate.key
```

## Préparation du serveur

Installer les prerequis non présents sur l'image de base (python et sudo)

```bash
ansible-playbook -i inventory.yml -u root install_prerequisites_scaleway.yml
```

Installer k3s

```bash
ansible-playbook -i inventory.yml -u root install_k3s_scaleway.yml
```
