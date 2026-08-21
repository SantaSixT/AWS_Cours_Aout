# Rendu Final - Travaux Pratiques DevSecOps 4A-11 (AWS, Jenkins, Ansible)

**Auteur** : Antoine Rohrbasser (SQY MCS 27.1)
**Environnement** : Instance EC2 Native (t3.micro, Amazon Linux 2023), Terraform, PowerShell. *Note : En raison de l'absence de WSL2, l'architecture Dockerisée initiale a été transformée en architecture native 100% EC2 automatisée.*

---

## 1. Explications Rapides (Architecture & Choix Techniques)

L'objectif était de provisionner une infrastructure AWS (via Terraform) et d'y exécuter une chaîne CI/CD (Jenkins, Ansible, AWS CLI) de manière sécurisée et reproductible.
- **Provisionnement** : Terraform gère exclusivement le compute (`t3.micro`), la paire de clés SSH (`ED25519`), et le Security Group. L'accès SSH est restreint à l'IP publique de l'administrateur.
- **Sécurité** : L'instance utilise IMDSv2 requis, les disques EBS sont chiffrés. Aucun conteneur Docker n'est utilisé afin d'isoler l'environnement au niveau système.
- **Exécution TP** : Un script d'orchestration (`run-labs.sh`) installe Java 21, Jenkins, Ansible, et configure les rôles via `JCasC`. Les nœuds Jenkins (agent) et Ansible (cible web) sont simulés de manière sécurisée par des utilisateurs Linux cloisonnés (`jenkins-agent`, `automation`) communiquant via SSH en localhost.

---

## 2. Résultats des TPs (Preuves d'exécution)

### TP 2 : Connecter Jenkins à AWS (aws sts)
Jenkins exécute l'AWS CLI avec les variables d'environnement injectées dynamiquement via `systemd` (aucun secret stocké en clair dans l'UI Jenkins).
**Résultat `aws sts get-caller-identity`** :
```json
{
    "UserId": "AIDAZBZP7DWOEQ5AXXXXX",
    "Account": "622333992348",
    "Arn": "arn:aws:iam::622333992348:user/user28"
}
```

### TP 5 : Imposer une configuration avec Ansible
Vérification de la connectivité et application du playbook d'installation sur le nœud `web_lab`.
**Résultat `ansible web_lab -m ping`** :
```json
web01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

**Résultat `ansible-playbook baseline.yml`** :
```text
PLAY [Baseline de laboratoire] *************************************************

TASK [Gathering Facts] *********************************************************
ok: [web01]

TASK [Installer curl] **********************************************************
changed: [web01]

TASK [Creer le repertoire de preuve] *******************************************
changed: [web01]

PLAY RECAP *********************************************************************
web01 : ok=3 changed=2 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

### TP 6 : AWS CLI et gestion réseau
Création d'un Security Group dynamique et vérification des règles minimales (Aucun accès libre `0.0.0.0/0`).
**Résultat `aws ec2 create-security-group`** :
```json
{
    "GroupId": "sg-09b3b85d345f1b123"
}
```

---

## 3. Scripts d'Automatisation

### Script d'exécution globale (`scripts/run-labs.sh`)
Ce script s'exécute sur l'instance EC2 pour automatiser l'installation des dépendances, la création du Swap, le démarrage de Jenkins (via JCasC) et la création des agents locaux.

```bash
#!/bin/bash
set -e

echo "=== Démarrage de l'exécution automatisée des TP (Native) ==="
set -a; source /home/ec2-user/.env; set +a

# 1. Swap creation (pour t3.micro)
if [ ! -f /swapfile ]; then
  fallocate -l 2G /swapfile
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
  echo "/swapfile swap swap defaults 0 0" >> /etc/fstab
fi

# 2. AWS CLI v2
if ! command -v aws &> /dev/null; then
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" -s
  unzip -q awscliv2.zip && sudo ./aws/install
fi

# 3. TP 1: Installation Jenkins & Java
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo -q
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
dnf install -y java-21-amazon-corretto jenkins

# Injection des variables d'environnement dans Jenkins (TP 2)
mkdir -p /etc/systemd/system/jenkins.service.d/
cat <<EOF > /etc/systemd/system/jenkins.service.d/override.conf
[Service]
Environment="CASC_JENKINS_CONFIG=/var/lib/jenkins/casc_configs"
Environment="JAVA_OPTS=-Djenkins.install.runSetupWizard=false"
Environment="JENKINS_ADMIN_ID=${JENKINS_ADMIN_ID:-admin}"
Environment="JENKINS_ADMIN_PASSWORD=${JENKINS_ADMIN_PASSWORD:-admin}"
Environment="AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}"
Environment="AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}"
Environment="AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION:-eu-west-3}"
EOF
systemctl daemon-reload

# 4. Configuration JCasC et Plugins
mkdir -p /var/lib/jenkins/casc_configs
cp -r /home/ec2-user/casc/* /var/lib/jenkins/casc_configs/ 2>/dev/null || true
chown -R jenkins:jenkins /var/lib/jenkins/casc_configs
systemctl enable --now jenkins

# 5. TP 4: Nœud Jenkins (Agent Local)
useradd -m -d /home/jenkins/agent jenkins-agent
mkdir -p /home/jenkins/agent/.ssh
ssh-keygen -t ed25519 -f /home/jenkins/agent/.ssh/id_ed25519 -N "" -q || true
chown -R jenkins-agent:jenkins-agent /home/jenkins/agent/.ssh

# 6. TP 5: Utilisateur Ansible cible (web_lab)
useradd -m automation
mkdir -p /home/automation/.ssh
cat /home/jenkins/agent/.ssh/id_ed25519.pub > /home/automation/.ssh/authorized_keys
chown -R automation:automation /home/automation/.ssh
chmod 600 /home/automation/.ssh/authorized_keys
dnf install -y ansible
```

---

## 4. Fichiers de Configuration

### Terraform Security Group (`terraform/security.tf`)
Contrôle strict des accès pour l'administration.
```hcl
resource "aws_security_group" "lab_sg" {
  name        = "${var.prefix}-sg"
  description = "Security Group pour le Lab Jenkins"
  vpc_id      = var.vpc_id
  tags = {
    Name        = "${var.prefix}-sg"
    Owner       = "user28"
    Environment = "Lab"
  }
}

resource "aws_vpc_security_group_ingress_rule" "ssh" {
  security_group_id = aws_security_group.lab_sg.id
  description       = "Acces SSH restreint a IP administrateur"
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
  cidr_ipv4         = var.admin_cidr
}
```

### Jenkins JCasC (`casc/jenkins.yaml`)
Configuration de Jenkins as Code (Création d'utilisateurs, matrice de sécurité et définition de l'agent).
```yaml
jenkins:
  systemMessage: "Bienvenue sur Jenkins - Lab 4A-11 (JCasC)\n"
  mode: NORMAL
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "${JENKINS_ADMIN_ID}"
          password: "${JENKINS_ADMIN_PASSWORD}"
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:${JENKINS_ADMIN_ID}"
        - "Overall/Read:authenticated"
  nodes:
    - permanent:
        labelString: "aws-lab"
        name: "agent-01"
        numExecutors: 1
        remoteFS: "/home/jenkins/agent"
```

---

## 5. Playbook Ansible

### `playbooks/baseline.yml`
Playbook idempotent (TP5) pour installer `curl` et créer un répertoire de preuve. Ce playbook est exécuté sur l'inventaire `web_lab` en connexion locale via SSH.

```yaml
---
- name: Baseline de laboratoire
  hosts: web_lab
  become: true
  tasks:
    - name: Installer curl
      ansible.builtin.package:
        name: curl
        state: present
    
    - name: Creer le repertoire de preuve
      ansible.builtin.file:
        path: /etc/training
        state: directory
        mode: '0755'
```
