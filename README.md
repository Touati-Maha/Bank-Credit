# 📘 Rôle Ansible — `helm_deploy`

## 🧩 Objectif

Ce rôle Ansible permet d’automatiser l’installation, la mise à jour et la désinstallation de **charts Helm** sur plusieurs clusters Kubernetes ou OpenShift à partir d’une **base de définition centralisée**.

L’objectif est de disposer d’une **source de vérité unique** pour les charts, tout en laissant la flexibilité de définir pour chaque cluster ses valeurs spécifiques.

---

## 📁 Structure du projet

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

---

## ⚙️ Principe de fonctionnement

1. **`defaults/main.yml`**  
   ➜ Définit la **base commune** (tous les charts, versions, namespaces, dépôts Helm, templates par défaut).  
   C’est la **source de vérité centrale** du rôle.

2. **`group_vars/all.yml`**  
   ➜ Définit pour chaque cluster :
   - le `kubeconfig`
   - les **valeurs spécifiques** (override) pour certains charts.  

3. **`deploy-chart.yml`** et **`uninstall-chart.yml`**  
   ➜ Définissent **quels charts installer ou supprimer** pour tous les clusters.

---

## 🚀 Scénarios d’utilisation

### 🔹 1. Déploiement / Mise à jour des charts

📄 **`deploy-chart.yml`**
```yaml
- name: Deploy charts (install/upgrade)
  hosts: localhost
  gather_facts: false

  vars:
    deploy_list:
      - grafana
      - jenkins

  tasks:
    - name: Add Helm repos
      import_role:
        name: roles/helm_deploy
        tasks_from: add_repos.yml

    - name: Build lists
      import_role:
        name: roles/helm_deploy
        tasks_from: build_lists.yml

    - name: Install / Upgrade selected charts
      import_role:
        name: roles/helm_deploy
        tasks_from: install_or_update.yml
```

#### ▶️ Commandes d’exécution

```bash
# Pour cluster1
ansible-playbook -i inventories/cluster1 deploy-chart.yml

# Pour cluster2
ansible-playbook -i inventories/cluster2 deploy-chart.yml
```

#### 🧠 Ce que fait le rôle

1. Ajoute les dépôts Helm définis.  
2. Construit la liste finale des charts à déployer (fusion des `defaults` et des `overrides`).  
3. Pour chaque chart :  
   - s’il n’existe pas → installation  
   - s’il existe avec une autre version → mise à jour  
   - s’il est déjà à jour → aucune action  

---

### 🔹 2. Désinstallation propre

📄 **`uninstall-chart.yml`**
```yaml
- name: Uninstall charts (clean removal)
  hosts: localhost
  gather_facts: false

  vars:
    uninstall_list:
      - prometheus

  tasks:
    - name: Build lists
      import_role:
        name: roles/helm_deploy
        tasks_from: build_lists.yml

    - name: Uninstall selected charts
      import_role:
        name: roles/helm_deploy
        tasks_from: uninstall.yml
```

#### ▶️ Commande

```bash
ansible-playbook -i inventories/cluster1 uninstall-chart.yml
```

---

## 📦 Variables importantes

| Variable | Définie où | Description |
|-----------|-------------|-------------|
| `helm_repositories` | `defaults/main.yml` | Liste des dépôts Helm à ajouter |
| `helm_charts` | `defaults/main.yml` | Définition complète des charts : nom, version, namespace, template, etc. |
| `deploy_list` | Playbook | Liste des charts à installer / mettre à jour |
| `uninstall_list` | Playbook | Liste des charts à désinstaller |
| `helm_chart_overrides` | `group_vars/all.yml` | Surcharge pour un cluster (valeurs ou templates différents) |
| `kubeconfig` | `group_vars/all.yml` | Fichier kubeconfig du cluster cible |

---

## 🧱 Exemple complet

### ✅ Inventaires

#### `inventories/cluster1/group_vars/all.yml`
```yaml
kubeconfig: "/home/maha/.kube/cluster1-config"

helm_chart_overrides:
  jenkins:
    values_template: "values-jenkins-cluster1.yaml.j2"
```

#### `inventories/cluster2/group_vars/all.yml`
```yaml
kubeconfig: "/home/maha/.kube/cluster2-config"

helm_chart_overrides:
  jenkins:
    values_template: "values-jenkins-cluster2.yaml.j2"
```

---

## 🧹 Résultat attendu

| Cluster | Chart | Action | Version |
|----------|--------|---------|----------|
| cluster1 | Grafana | install/upgrade | 7.3.10 |
| cluster1 | Jenkins | install (override cluster1) | 5.4.0 |
| cluster1 | Prometheus | uninstall | — |
| cluster2 | Grafana | install/upgrade | 7.3.10 |
| cluster2 | Jenkins | install (override cluster2) | 5.4.0 |

---

## ✅ Résumé rapide

| Action | Commande | Fichier concerné |
|---------|-----------|-----------------|
| Installer / Mettre à jour | `ansible-playbook -i inventories/clusterX deploy-chart.yml` | `deploy-chart.yml` |
| Désinstaller | `ansible-playbook -i inventories/clusterX uninstall-chart.yml` | `uninstall-chart.yml` |
| Ajouter un nouveau chart | Modifier `roles/helm_deploy/defaults/main.yml` | — |
| Surcharger un chart pour un cluster | Modifier `inventories/clusterX/group_vars/all.yml` | — |
| Mettre à jour la version d’un chart | Modifier `chart_version` dans `defaults/main.yml` | — |

---
