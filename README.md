# Lab AD / Entra ID Hybride — SOC & Réponse à Incident

**Projet de portfolio personnel — Saad Agoumi, étudiant ingénieur à IMT Atlantique, cybersécurité & Machine Learning.**

---

## En bref

Ce projet construit un lab **Active Directory hybride** (on-premise + Microsoft Entra ID), volontairement vulnérable, pour générer des scénarios d'incident réalistes contre l'annuaire — puis, pour chacun d'eux, appliquer le cycle complet **détection → analyse/triage → confinement → éradication → récupération → retour d'expérience** (NIST SP 800-61), avec des preuves concrètes à chaque étape.

L'objectif n'est pas de collecter des writeups offensifs isolés : la technique exécutée n'est jamais le sujet du writeup, seulement son déclencheur. Chaque cas suit le même triptyque **détection instrumentée (SIEM) → triage → réponse à incident**, avec une preuve concrète (alerte SIEM réelle, Event ID, action de confinement réellement exécutée) à l'appui. C'est ce qui distingue ce projet d'un simple lab de pentest — il est pensé pour un profil SOC Analyst / Incident Responder, pas pour un profil offensif.

Chaque writeup suit ce même schéma en trois temps : **détection instrumentée → triage → réponse**. Le triage reproduit le raisonnement d'un analyste SOC L1 découvrant l'alerte sans connaître le contexte (cadre 5W, verdict, justification d'escalade) ; la réponse reproduit ce que ferait un incident responder une fois l'alerte confirmée.

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
│   └── recon.md
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
Cartographie complète du lab : découverte réseau (`nmap`), énumération SMB/LDAP (`NetExec`, `ldapsearch`), collecte **BloodHound CE**, requêtes **Cypher** personnalisées, et audit automatisé (**AD Miner**). Finding majeur : un compte de service Microsoft Entra Connect dispose de droits **DCSync complets** sur les deux domaines de la forêt. La phase inclut aussi une analyse de ce que cette reconnaissance laisse (ou non) comme trace côté SIEM, avec un triage détaillé des alertes remontées (cadre 5W, verdict, justification d'escalade et recommandations).
→ Writeup complet : [`attack-writeups/recon.md`](attack-writeups/recon.md)

### 🔶 Module 4 — Étude de cas d'incident : abus ADCS (ESC1 à ESC16) (en cours)
Prérequis identifié au module 3 : le lab ne dispose pas encore d'un rôle **ADCS** — son installation, avec des templates de certificats volontairement vulnérables, est la première étape de ce module, avant l'exploitation (Certipy, comme déclencheur), la détection, le triage, puis les phases de confinement/éradication/récupération/retour d'expérience.

### Modules suivants (prévus)
- **Module 5** — Étude de cas d'incident : chaîne Kerberos/délégation (Kerberoasting, shadow credentials, RBCD, abus d'ACL) de l'accès initial à Domain Admin.
- **Module 6** — Étude de cas d'incident : BadSuccessor / dMSA (CVE-2025-53779), patch inclus.
- **Module 7** — Étude de cas d'incident : extraction de secrets (DCSync) & persistence.
- **Module 8** — Étude de cas d'incident : pivot d'identité hybride Entra ID (on-prem ↔ cloud).
- **Agent copilote** — agent IA connecté à BloodHound, pour le triage et l'évaluation de sévérité d'un compte compromis.
- **Reporting final** — synthèse de tous les incidents avec statut de détection par technique.

---

## Stack technique (à ce jour)

GOAD, Vagrant, Ansible, VMware Workstation, Wazuh, Sysmon (Olaf Hartong), NetExec, BloodHound CE, Neo4j, AD Miner, Impacket, Microsoft Entra ID, Microsoft Entra Connect.
