# 🚀 Rôle Ansible — `helm_deploy`

## 🎯 Objectif

Ce rôle Ansible permet d’automatiser :
- l’installation,  
- la mise à jour,  
- et la désinstallation  

de **charts Helm** sur plusieurs clusters Kubernetes ou OpenShift à partir d’une **base de définition centralisée**.

L’objectif est de disposer d’une **source de vérité unique** pour la gestion des charts Helm, tout en permettant à chaque cluster d’avoir ses propres valeurs personnalisées (templates `values.yaml` différents selon l’environnement).

---

## 📂 Structure du projet

```text
helm-deploy/
├── inventories/
│   ├── cluster1/
│   │   ├── hosts
│   │   └── group_vars/all.yml
│   └── cluster2/
│       ├── hosts
│       └── group_vars/all.yml
├── roles/
│   └── helm_deploy/
│       ├── defaults/main.yml
│       ├── tasks/
│       │   ├── add_repos.yml
│       │   ├── build_lists.yml
│       │   ├── install_or_update.yml
│       │   └── uninstall.yml
│       └── templates/
│           ├── values-grafana.yaml.j2
│           ├── values-jenkins-cluster1.yaml.j2
│           ├── values-jenkins-cluster2.yaml.j2
│           └── values-prometheus.yaml.j2
├── deploy-chart.yml
└── uninstall-chart.yml
```
## ⚙️ Principe de fonctionnement

1. **`roles/helm_deploy/defaults/main.yml`**  
   ➜ Définit la **base commune** de tous les charts : nom, version, namespace, dépôt Helm et template par défaut.  
   C’est la **source de vérité centrale** du rôle.

2. **`inventories/clusterX/group_vars/all.yml`**  
   ➜ Contient les informations spécifiques à un cluster :  
   - le chemin de son `kubeconfig`  
   - les **valeurs ou templates spécifiques** à certains charts (via `helm_chart_overrides`).

3. **Playbooks (`deploy-chart.yml` / `uninstall-chart.yml`)**  
   ➜ Déterminent **quels charts installer ou désinstaller** sur un cluster donné.  
   Ces fichiers sont communs à tous les clusters.

---
## 🧩 Exemple de configuration

### 🧱 `roles/helm_deploy/defaults/main.yml`

```yaml
helm_repositories:
  - { name: grafana, url: "https://grafana.github.io/helm-charts" }
  - { name: jenkins, url: "https://charts.jenkins.io" }
  - { name: prometheus, url: "https://prometheus-community.github.io/helm-charts" }

helm_charts:
  - name: grafana
    chart: grafana/grafana
    chart_version: "7.3.10"
    namespace: monitoring
    create_namespace: true
    values_template: "values-grafana.yaml.j2"

  - name: jenkins
    chart: jenkins/jenkins
    chart_version: "5.4.0"
    namespace: cicd
    create_namespace: true
    values_template: "values-jenkins.yaml.j2"

  - name: prometheus
    chart: prometheus-community/prometheus
    chart_version: "25.12.0"
    namespace: monitoring
    create_namespace: true
    values_template: "values-prometheus.yaml.j2"

```
### 📘 En résumé :

- Les **inventaires** identifient le cluster.  
- Les **playbooks** définissent quoi faire (installer ou désinstaller).  
- Le **rôle** exécute les actions Helm nécessaires.

