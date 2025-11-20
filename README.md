# 📘 Rôle Ansible — `helm_deploy`

## 🧩 Objectif

Ce rôle Ansible permet d’automatiser l’installation, la mise à jour et la désinstallation de **charts Helm** sur plusieurs clusters Kubernetes ou OpenShift à partir d’une **base de définition centralisée**.

L’objectif est de disposer d’une **source de vérité unique** pour les charts, tout en laissant la flexibilité de définir pour chaque cluster ses valeurs spécifiques.

---

## 1. Structure du projet

```bash
.
├── README.md
├── deploy-chart.yml              # Playbook pour installer / mettre à jour les charts
├── uninstall-chart.yml           # Playbook pour désinstaller les charts
├── inventories
│   ├── cluster1
│   │   ├── group_vars
│   │   │   └── all.yml          # Variables spécifiques à cluster1 (kubeconfig, overrides)
│   │   └── hosts                # Inventaire Ansible pour cluster1
│   └── cluster2
│       └── group_vars
│           └── all.yml          # Variables spécifiques à cluster2 (kubeconfig, overrides)
├── roles
│   └── helm_deploy
│       ├── defaults
│       │   └── main.yml         # Définition globale des dépôts Helm et des charts
│       ├── tasks
│       │   ├── add_repos.yml    # Ajout / mise à jour des repositories Helm
│       │   ├── build_lists.yml  # Construction des listes d’installation / désinstallation
│       │   ├── install_or_update.yml # Installation / upgrade des charts
│       │   ├── uninstall.yml    # Désinstallation des releases
│       │   └── main.yml         # (optionnel, non utilisé directement ici)
│       ├── templates
│       │   ├── values-grafana.yaml.j2
│       │   ├── values-jenkins-cluster1.yaml.j2
│       │   ├── values-jenkins-cluster2.yaml.j2
│       │   └── values-prometheus.yaml.j2
│       └── vars
│           └── main.yml         # (vide pour l’instant, réservé pour des vars futures)


```## 2. Description des principaux composants

Cette section détaille la structure interne du rôle `helm_deploy` et la logique utilisée pour la gestion des charts Helm.

---

### 2.1 `roles/helm_deploy/defaults/main.yml`

Ce fichier définit la configuration **par défaut**, commune à tous les clusters.

#### Repositories Helm

```yaml
helm_repositories:
  - { name: grafana,    url: "https://grafana.github.io/helm-charts" }
  - { name: jenkins,    url: "https://charts.jenkins.io" }
  - { name: prometheus, url: "https://prometheus-community.github.io/helm-charts" }
```

#### Liste globale des charts

```yaml
helm_charts:
  - name: grafana
    chart: grafana/grafana
    chart_version: "10.1.4"
    namespace: monitoring
    create_namespace: true
    values_template: "values-grafana.yaml.j2"
    wait: true
    atomic: true

  - name: jenkins
    chart: jenkins/jenkins
    chart_version: "5.8.105"
    namespace: cicd
    create_namespace: true
    values_template: "values-jenkins.yaml.j2"
    wait: true
    atomic: true

  - name: prometheus
    chart: prometheus/kube-prometheus-stack
    chart_version: "78.5.0"
    namespace: monitoring
    create_namespace: true
    values_template: "values-prometheus.yaml.j2"
    wait: true
    atomic: true
```

Ceci représente **la base commune** à tous les clusters.  
Les clusters peuvent ensuite overrider certains champs via `helm_chart_overrides`.

---

### 2.2 Inventaires par cluster (`inventories/clusterX`)

#### Exemple `cluster1`

```yaml
# inventories/cluster1/group_vars/all.yml
kubeconfig: "/home/ubuntu/.kube/kubeconfig"

helm_chart_overrides:
  jenkins:
    values_template: "values-jenkins-cluster1.yaml.j2"
    chart_version: "5.8.104"
    namespace: cicd-test
```

#### Exemple `cluster2`

```yaml
# inventories/cluster2/group_vars/all.yml
kubeconfig: "/home/maha/.kube/kubeconfig"

helm_chart_overrides:
  jenkins:
    values_template: "values-jenkins-cluster2.yaml.j2"
    chart_version: "5.8.103"
```

#### Inventaire hosts

```ini
# inventories/cluster1/hosts
[all]
localhost ansible_connection=local
```

---

### 2.3 Tâches du rôle

#### `add_repos.yml`

```yaml
kubernetes.core.helm_repository:
  name: "{{ item.name }}"
  repo_url: "{{ item.url }}"
  state: present
```

---

#### `build_lists.yml`

Logique :

1. Fusionne :
   - charts par défaut (`helm_charts`)
   - overrides (`helm_chart_overrides`)
   → résultat dans `_effective_charts`

2. Construit :
   - `charts_to_install` (install/upgrade)
   - `helm_releases_uninstall` (désinstallation)

Prend en compte :

- `deploy_list` (liste de charts à déployer)
- `uninstall_list` (liste de charts à désinstaller)

---

#### `install_or_update.yml`

Pour chaque chart :

1. Rendu du template Jinja associé (`values-*.yaml.j2`)
   - rendu YAML → dict python
   - stocké dans `_rendered_values[item.name]`

2. Appel Helm :

```yaml
kubernetes.core.helm:
  name: "{{ item.name }}"
  chart_ref: "{{ item.chart }}"
  chart_version: "{{ item.chart_version }}"
  release_namespace: "{{ item.namespace }}"
  kubeconfig: "{{ kubeconfig }}"
  values: "{{ _rendered_values[item.name] }}"
  state: present
```

---

#### `uninstall.yml`

Pour chaque release :

```yaml
kubernetes.core.helm:
  name: "{{ item.name }}"
  release_namespace: "{{ item.namespace }}"
  kubeconfig: "{{ kubeconfig }}"
  state: absent
```

> Remarque : supprime la **release**, pas forcément le namespace.

---

### 2.4 Templates de valeurs (`templates/values-*.yaml.j2`)

#### Exemple Grafana

```yaml
adminUser: admin
adminPassword: grafana

service:
  type: ClusterIP
```

#### Exemple Jenkins cluster1

```yaml
controller:
  admin:
    username: admin
    password: admin

persistence:
  enabled: false

service:
  type: ClusterIP
```

#### Exemple Jenkins cluster2

```yaml
controller:
  admin:
    username: admin_cluster2
    password: pass2

persistence:
  enabled: false

service:
  type: ClusterIP
```

Ces templates permettent d’avoir **des valeurs propres à chaque cluster** pour un même chart.


---

## 3. Playbooks fournis

### 3.1 `deploy-chart.yml`

```yaml
- name: Deploy (install/upgrade to target versions)
  hosts: localhost
  gather_facts: false

  # Optionnel : cible spécifique
  vars:
    deploy_list:
      - grafana
      - jenkins
      - prometheus

  tasks:
    - import_role: { name: helm_deploy, tasks_from: add_repos.yml }
    - import_role: { name: helm_deploy, tasks_from: build_lists.yml }
    - import_role: { name: helm_deploy, tasks_from: install_or_update.yml }
```

- Si `deploy_list` est défini → seuls ces charts sont installés / mis à jour.  
- Si `deploy_list` est vide ou supprimé → tous les charts de `helm_charts` sont pris en compte.

### 3.2 `uninstall-chart.yml`

```yaml
- name: Uninstall (clean removal)
  hosts: localhost
  gather_facts: false

  vars:
    uninstall_list:
      - grafana
      - jenkins
      - prometheus

  tasks:
    - import_role: { name: helm_deploy, tasks_from: build_lists.yml }
    - import_role: { name: helm_deploy, tasks_from: uninstall.yml }
```

- `uninstall_list` indique quels charts désinstaller.  
- Les namespaces ne sont pas automatiquement supprimés.

---

## 4. Comment utiliser le projet

### 4.1 Pré‑requis

- Ansible installé.  
- Collection Ansible `kubernetes.core` installée.  
- `helm` installé côté machine d’exécution.  
- Un kubeconfig valide pour chaque cluster, référencé dans `group_vars/all.yml`.

### 4.2 Déployer les charts sur un cluster

Exemple : déployer sur `cluster1` tous les charts définis dans `deploy_list` :

```bash
ansible-playbook -i inventories/cluster1 deploy-chart.yml
```

Pour déployer sur `cluster2` :

```bash
ansible-playbook -i inventories/cluster2 deploy-chart.yml
```

Pour déployer **un seul chart** sans modifier le playbook, on peut surcharger `deploy_list` en ligne de commande :

```bash
ansible-playbook -i inventories/cluster1 deploy-chart.yml   -e 'deploy_list=["jenkins"]'
```

### 4.3 Désinstaller des charts

Désinstaller les charts listés dans `uninstall_list` pour `cluster1` :

```bash
ansible-playbook -i inventories/cluster1 uninstall-chart.yml
```

Désinstaller uniquement Jenkins :

```bash
ansible-playbook -i inventories/cluster1 uninstall-chart.yml   -e 'uninstall_list=["jenkins"]'
```

---

## 5. Ajouter un nouveau use case

### 5.1 Ajouter un nouveau chart

1. **Déclarer le chart** dans `roles/helm_deploy/defaults/main.yml` :

```yaml
helm_charts:
  - name: myapp
    chart: myrepo/myapp
    chart_version: "1.2.3"
    namespace: myapp-namespace
    create_namespace: true
    values_template: "values-myapp.yaml.j2"
    wait: true
    atomic: true
```

2. **Créer le template de valeurs** :

```bash
roles/helm_deploy/templates/values-myapp.yaml.j2
```

3. **(Optionnel) Overrides par cluster** dans `inventories/clusterX/group_vars/all.yml` :

```yaml
helm_chart_overrides:
  myapp:
    chart_version: "1.2.4"
    values_template: "values-myapp-clusterX.yaml.j2"
    namespace: myapp-namespace-x
```

4. **Déployer** :

```bash
ansible-playbook -i inventories/cluster1 deploy-chart.yml   -e 'deploy_list=["myapp"]'
```

### 5.2 Ajouter un nouveau cluster

1. Créer un nouveau répertoire d’inventaire, par exemple `inventories/cluster3/`.  

2. Ajouter un fichier `hosts` :

```ini
[all]
localhost ansible_connection=local
```

3. Créer `group_vars/all.yml` :

```yaml
kubeconfig: "/chemin/vers/le/kubeconfig/cluster3"

helm_chart_overrides:
  jenkins:
    values_template: "values-jenkins-cluster3.yaml.j2"
    chart_version: "5.8.106"
```

4. (Optionnel) Créer de nouveaux templates dans `roles/helm_deploy/templates/`.  

5. Lancer le playbook :

```bash
ansible-playbook -i inventories/cluster3 deploy-chart.yml
```

### 5.3 Désinstallation sélective par use case

Exemple : pour un use case où tu veux nettoyer seulement les composants de monitoring :

```bash
ansible-playbook -i inventories/cluster1 uninstall-chart.yml   -e 'uninstall_list=["grafana","prometheus"]'
```

Tu peux aussi créer un autre playbook `uninstall-monitoring.yml` qui fixe `uninstall_list` directement, pour documenter ce use case.

---

## 6. Résumé

- `defaults/main.yml` : définit tous les charts et dépôts Helm au niveau global.  
- `group_vars/all.yml` (par cluster) :
  - fournit le `kubeconfig`,  
  - permet de surcharger la configuration des charts via `helm_chart_overrides`.  
- `deploy-chart.yml` :
  - déploie ou met à jour les charts,  
  - filtrés par `deploy_list` si nécessaire.  
- `uninstall-chart.yml` :
  - désinstalle les releases listées dans `uninstall_list`.  

Ce rôle est conçu pour être facilement réutilisable :

- ajouter un chart = 1 entrée dans `helm_charts` + 1 template de valeurs,  
- ajouter un cluster = 1 inventaire + 1 fichier `group_vars/all.yml`,  
- gérer des use cases différents = jouer sur `deploy_list` et `uninstall_list` (dans les playbooks ou via `-e`).
