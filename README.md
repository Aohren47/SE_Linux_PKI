# 🔐 Durcissement d'un serveur Linux pour une PKI open source

Projet de déploiement d'une **infrastructure à clé publique (PKI)** open source sur un serveur **Rocky Linux 9** durci selon les bonnes pratiques de sécurité (CIS, ANSSI).

> Outils principaux : **Rocky Linux 9**, **step-ca** (Smallstep), **SELinux**, **AIDE**, **auditd**, **Lynis**, **OpenSCAP**.

---

## 📋 Contexte et objectifs

L'objectif était de mettre en place une PKU open source pour gérer l'émission de certificats, tout en **durcissant** la plateforme Linux hôte à un niveau avancé. Le projet répond aux **4 besoins de sécurité** suivants :

| Besoin | Comment j'y réponds |
|---|---|
| **Confidentialité** des clés | Chiffrement LUKS + permissions strictes + SELinux + AC racine hors ligne |
| **Intégrité** du système | Contrôle d'intégrité avec AIDE |
| **Disponibilité** du service | Service systemd avec redémarrage automatique |
| **Journalisation** | auditd (traçabilité des accès aux clés) + journald persistant |

---

## 🏗️ Architecture de la PKI

J'ai mis en place une hiérarchie à deux niveaux :

```
        AC RACINE (hors ligne)          <- clé privée isolée, signe uniquement l'intermédiaire
              │
              ▼
   AC INTERMÉDIAIRE (en ligne)          <- service step-ca, émet les certificats
              │
       ┌──────┴───────┐
       ▼              ▼
  Utilisateurs    Serveurs / services internes
```

- **AC racine** : générée puis **isolée hors ligne** (clé exportée hors du serveur, puis supprimée avec `shred`). Elle n'est ramenée que pour signer une nouvelle intermédiaire.
- **AC intermédiaire** : le démon `step-ca`, qui émet les certificats du quotidien.
- **Révocation** : choix assumé de la **révocation passive** (certificats à courte durée de vie), adaptée à une PKI interne, plutôt qu'une CRL/OCSP qui constitue un point de défaillance central.

---

## 🛠️ Stack technique

- **OS** : Rocky Linux 9 (choisi pour son support natif de **SELinux** et la maturité du contenu CIS)
- **PKI** : step-ca (binaire Go léger, gère nativement la hiérarchie racine/intermédiaire)
- **Virtualisation** : VMware Workstation
- **Chiffrement disque** : LUKS (`aes-xts-plain64`)

---

## 📦 Déploiement de la PKI

```bash
# Dépôt Smallstep + installation
sudo dnf install -y step-cli step-ca

# Initialisation (génère AC racine + intermédiaire)
step ca init

# Isolation de l'AC racine hors ligne
# (export de la clé racine vers un support externe, puis :)
shred -u ~/.step/secrets/root_ca_key

# Émission d'un certificat de test
step ca certificate "test.entreprise.local" test.crt test.key
step certificate inspect test.crt --short
```

---

## 🔒 Durcissement du système

### 1. Service systemd + utilisateur non-privilégié
step-ca tourne sous un utilisateur système dédié `step` (sans shell de connexion), via une unité systemd **durcie** (sandboxing : `ProtectSystem=strict`, `NoNewPrivileges`, `ProtectHome`, etc.). La clé intermédiaire et le mot de passe sont en permissions `600`.

### 2. Sécurité du système de fichiers
Options `nodev,nosuid,noexec` appliquées sur `/tmp`, `/var/tmp`, `/var/log`, `/home`, `/dev/shm` (via montages *bind*, faute de partitionnement séparé à l'installation).

### 3. Chiffrement des clés
Disque entièrement chiffré avec **LUKS** dès l'installation → les clés de l'AC sont protégées au repos.

### 4. Durcissement SSH
- Authentification par **clé uniquement** (mots de passe désactivés)
- Connexion **root interdite**
- `MaxAuthTries`, timeout d'inactivité, `X11Forwarding no`, `AllowUsers`
- Bannière d'avertissement légal

### 5. Réduction de la surface d'attaque
Installation minimale → seuls **2 ports exposés** sur le réseau : SSH (22) et la PKI (9000). `chronyd` reste en local.

### 6. Journalisation et intégrité
- **journald** en mode persistant
- **auditd** avec règles dédiées : tout accès aux clés de l'AC est tracé
- **AIDE** : base de référence d'intégrité + vérification quotidienne automatique (cron)

### 7. SELinux
Mode **enforcing**. Les clés étiquetées avec le type dédié `cert_t` (réservé au matériel cryptographique). Aucun refus SELinux (AVC) après configuration.

### 8. Durcissement noyau
Réglages `sysctl` (anti-spoofing, rejet des redirections ICMP, `tcp_syncookies`, `kptr_restrict`, etc.), politique de mots de passe, `umask` restrictif, comptabilité des processus, scanner anti-rootkit (rkhunter).

---

## 📊 Résultats des audits

### Lynis (hardening index)

| Étape | Score |
|---|---|
| Installation par défaut | 66 / 100 |
| Après durcissement de base | 79 / 100 |
| **Après optimisation** | **85 / 100** |

➡️ Amélioration de **+29 %** par rapport à la baseline.

### OpenSCAP (conformité CIS)

Audit selon le référentiel **CIS Benchmark Level 1 Server** :
**85,99 % de conformité.**

---

## 📁 Structure du dépôt

```
.
├── README.md
├── docs/
│   └── rapport-complet.pdf          # rapport détaillé avec captures
├── configs/
│   ├── step-ca.service              # unité systemd durcie
│   ├── 01-hardening.conf            # durcissement SSH
│   ├── 99-hardening.conf            # durcissement sysctl
│   ├── pki.rules                    # règles auditd
│   └── fstab-extrait                # montages sécurisés
└── audits/
    ├── lynis-avant.txt              # score 66
    ├── lynis-apres.txt              # score 85
    ├── oscap-report.html            # rapport CIS
    ├── audit-cis.json
    └── audit-cis.csv
```

---

## ✅ Bilan

Ce projet m'a permis de mettre en œuvre une **défense en profondeur** complète : la clé privée de l'AC est protégée par plusieurs couches indépendantes (chiffrement LUKS, permissions Unix, confinement SELinux, traçabilité auditd, isolation hors ligne de la racine). L'efficacité du durcissement est validée objectivement par la progression des scores d'audit.

---

*Projet réalisé dans le cadre de ma formation — Sujet n°2 : Durcissement OSv PKI.*
