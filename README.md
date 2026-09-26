# Lab AD / Entra ID Hybride — Attaque & Détection

**Projet de portfolio personnel — Saad Agoumi, étudiant ingénieur à IMT Atlantique, cybersécurité & Machine Learning.**

---

## En bref

Ce projet construit un lab **Active Directory hybride** (on-premise + Microsoft Entra ID), volontairement vulnérable, pour reproduire des techniques d'attaque modernes contre l'annuaire — puis, pour chacune d'elles, construire et observer réellement sa **détection** dans un SIEM (Wazuh + Sysmon), plutôt que de se contenter d'en décrire la théorie.

L'objectif n'est pas de collecter des writeups offensifs isolés : chaque technique suit le même triptyque **attaque → correctif → détection instrumentée**, avec une preuve concrète (alerte SIEM réelle, Event ID, règle MITRE ATT&CK) à l'appui. C'est ce qui distingue ce projet d'un simple lab de pentest — il est pensé pour un profil qui comprend l'attaque de l'intérieur au service de la défense.

**Statut : projet en cours de construction.** Ce README est mis à jour au fil de l'avancement — voir la section [Avancement](#avancement) ci-dessous pour l'état exact à ce jour.

---

## Architecture

```
                        ┌─────────────────────────────┐
                        │  Poste attaquant (WSL2)      │
                        └──────────────┬───────────────┘
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 │                                            │
        ┌────────▼─────────┐   Microsoft Entra Connect ┌──────▼──────────────┐
        │  AD on-prem       │◄─────── sync ────────────►│   Entra ID           │
        │  (GOAD-Light)     │                            │  (tenant gratuit,    │
        │  DC01, DC02...    │                            │   essai Azure)       │
        │  + 1 DC Win 2025  │                            │  Conditional Access, │
        │  (non patché)     │                            │  hybrid join         │
        └─────────┬─────────┘                            └──────────────────────┘
                  │ Sysmon + agent Wazuh
        ┌─────────▼─────────┐
        │  SIEM (Wazuh)     │
        │  règles + alertes │
        │  par technique    │
        └───────────────────┘
```

Le lab tourne entièrement en local (VMware Workstation Pro), le SIEM sur une VM Ubuntu dédiée, le poste attaquant sur WSL2 (choix assumé plutôt qu'une VM Kali, pour des raisons de ressources CPU détaillées dans `infra/README.md`).

---

## Structure du repo

```
.
├── infra/                  # Construction du lab (GOAD-Light, DC Windows Server 2025, Entra ID)
│   └── README.md
├── siem/                   # Déploiement Wazuh + Sysmon, sur les 4 machines du lab
│   └── setup.md
├── attack-writeups/        # Un writeup détaillé par module d'attaque
│   └── 03-recon.md
└── agent/                  # Agent IA copilote sur BloodHound (à venir)
```

---

## Avancement

### ✅ Module 1 — Infrastructure du lab
Déploiement de **GOAD-Light** (forêt `sevenkingdoms.local` / `north.sevenkingdoms.local`, 2 domaines) via Vagrant/Ansible, avec ajout d'un DC **Windows Server 2025 volontairement non patché** (build RTM 26100.1742) pour reproduire BadSuccessor/dMSA (CVE-2025-53779) au module 6. Tenant **Microsoft Entra ID** créé (essai Azure gratuit) et synchronisé via Microsoft Entra Connect.
→ Détails et procédure de reconstruction : [`infra/README.md`](infra/README.md)

### ✅ Module 2 — Instrumentation SIEM
Déploiement de **Wazuh** (indexer + manager + dashboard, all-in-one) sur une VM Ubuntu dédiée, puis agent Wazuh + **Sysmon** (config **Olaf Hartong**, alignée MITRE ATT&CK) sur les 4 machines du lab (DC01, DC02, SRV02, DC03). Chaîne de bout en bout validée : événement Sysmon → agent → manager → alerte dans le dashboard.
→ Détails et procédure de reconstruction : [`siem/setup.md`](siem/setup.md)

### ✅ Module 3 — Reconnaissance moderne
Cartographie complète du lab : découverte réseau (`nmap`), énumération SMB/LDAP (`NetExec`, `ldapsearch`), collecte **BloodHound CE**, requêtes **Cypher** personnalisées, et audit automatisé (**AD Miner**). Finding majeur : un compte de service Microsoft Entra Connect dispose de droits **DCSync complets** sur les deux domaines de la forêt. La phase inclut aussi une analyse de ce que cette reconnaissance laisse (ou non) comme trace côté SIEM.
→ Writeup complet : [`attack-writeups/recon.md`](attack-writeups/recon.md)

### 🔶 Module 4 — ADCS, de ESC1 à ESC16 (en cours)
Prérequis identifié au module 3 : le lab ne dispose pas encore d'un rôle **ADCS** — son installation, avec des templates de certificats volontairement vulnérables, est la première étape de ce module, avant l'exploitation (Certipy) et la construction des règles de détection correspondantes.

### Modules suivants (prévus)
- **Module 5** — Kerberos & délégation avancée (Kerberoasting, shadow credentials, RBCD, abus d'ACL).
- **Module 6** — BadSuccessor / dMSA (CVE-2025-53779) : attaque, démonstration du correctif, détection.
- **Module 7** — Extraction de secrets & persistence (DCSync, LAPS, NTLM relay).
- **Module 8** — Identité hybride Entra ID (vol de PRT, pivot on-prem ↔ cloud, Cloud Kerberos Trust).
- **Agent copilote** — agent IA connecté à BloodHound, pour la priorisation d'attaque et le triage d'incident.
- **Reporting final** — synthèse de toutes les techniques avec statut de détection par technique.

---

## Stack technique (à ce jour)

GOAD, Vagrant, Ansible, VMware Workstation, Wazuh, Sysmon (Olaf Hartong), NetExec, BloodHound CE, Neo4j, AD Miner, Impacket, Microsoft Entra ID, Microsoft Entra Connect.
