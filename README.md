# Lab AD / Entra ID Hybride — Attaque & Détection 2026

> **Version 2026, adaptée SOC/détection.** Document de cadrage et guide d'exécution étape par étape.
> Auteur : Saad Agoumi — IMT Atlantique — cible : stage de fin d'études (avril 2027) puis CDI en **SOC / analyste sécurité** (une ouverture pentest/red team reste possible, mais n'est plus l'objectif principal).
> Repo GitHub : `ad-entraid-hybride-attaque-detection-2026`

---

## 0. Règles du jeu (à lire une fois, à respecter toujours)

- **Tout se fait sur TON lab, dans TON tenant, avec TES machines.** Aucune technique n'est jamais lancée contre un système que tu ne possèdes pas ou pour lequel tu n'as pas une autorisation écrite. C'est la ligne rouge du métier, et c'est aussi ce qu'un recruteur veut voir : de la maturité, pas juste de la technique.
- Le tenant Entra ID et les ressources Azure utilisent des offres **gratuites** (free trial Azure). Zéro besoin de payer, zéro besoin d'accéder à quoi que ce soit de propriétaire.
- Chaque technique offensive que tu documentes, tu documentes **aussi la détection et la remédiation** — et depuis cette version, tu ne te contentes plus de décrire la détection : tu la **construis et l'observes réellement** dans un SIEM. C'est ce qui distingue un projet SOC crédible d'un simple recueil de writeups offensifs.

---

## 1. Ce qui a changé et pourquoi

### 1.1 BadSuccessor / dMSA est désormais PATCHÉ — et c'est une bonne nouvelle pour toi

- Microsoft a assigné le **CVE-2025-53779** et publié un correctif **moins d'une semaine après la présentation DEF CON 2025** (août 2025).
- Le patch impose une **validation KDC par lien mutuel** entre le dMSA et le compte cible : l'attribut `msDS-ManagedAccountPrecededByLink` peut toujours être écrit, mais le KDC n'émet plus de ticket sauf si le couplage ressemble à une vraie migration.
- **La voie d'escalade directe est donc fermée sur un DC à jour.**

**Ce que ça change concrètement pour ton projet :**
- Tu montes **volontairement un DC Windows Server 2025 non patché** dans ton lab pour reproduire l'attaque d'origine (c'est légitime et courant en lab).
- Tu ne la présentes **pas** comme une 0-day. Tu la présentes comme un **cycle complet** : (a) exploitation de la technique, (b) explication du patch (validation du lien mutuel côté KDC), (c) détection (Event ID 5137 sur création de dMSA, requêtes sur les attributs sensibles) — **avec une alerte SIEM réelle qui se déclenche**, pas juste une phrase qui dit qu'elle se déclencherait. Ce triptyque « attaque → correctif → détection instrumentée » est **exactement** ce qu'un recruteur SOC veut voir : tu ne connais pas juste l'attaque, tu sais à quoi elle ressemble côté défense.

### 1.2 Profondeur > largeur : confirmé, on garde le cap

Le marché récompense la capacité **démontrable** à exploiter un AD moderne et à le défendre, pas une liste de 10 sujets effleurés. La structure « pilier lourd + agent en soutien » reste la bonne. On résiste à la tentation d'ajouter cloud pur, CI/CD, etc. — ces sujets diluent le signal principal.

### 1.3 Le timing dicte l'ordre de construction

Ton PFE démarre **le 1er avril 2027**. Les candidatures ouvrent en général **entre septembre 2026 et janvier 2027** (au fil de l'eau). Tu dois donc pouvoir montrer un livrable **présentable dès décembre 2026**, même partiel. D'où la règle : **on front-load le cœur AD + détection** pour qu'un recruteur croisé en novembre-décembre ait déjà quelque chose de solide à regarder. L'agent et le bonus LLM viennent ensuite.

### 1.4 Ce que le marché confirme

- **AD / Entra ID hybride reste la surface n°1 en pentest interne ET la source n°1 de scénarios de détection en SOC** : les attaques AD (Kerberoasting, Pass-the-Hash, mouvements latéraux) sont les techniques les plus fréquemment détectées en environnement SOC entreprise. Comprendre l'attaque de l'intérieur — ce que tu fais dans ce projet — est précisément ce qui distingue un analyste SOC junior qui coche des cases d'un analyste qui comprend pourquoi une alerte est critique.
- **NetExec est bien le standard 2025-2026** qui a remplacé CrackMapExec (abandonné).
- **ADCS va bien de ESC1 à ESC16** aujourd'hui (la recherche a étendu la liste en 2025).
- **Le pentest augmenté par l'IA et le pentest des applications IA/LLM sont des demandes marché réelles mais encore étroites pour un profil junior** — quelques cabinets et grands comptes commencent à en parler (Orange Cyberdefense, Wavestone, Vaadata), mais ce n'est pas encore un volume de recrutement comparable au SOC ou à l'AD classique. D'où son traitement en bonus optionnel plutôt qu'en pilier central dans cette version.

---

## 2. Le projet : cœur + agent + bonus optionnel

| Bloc | Nom | Poids | Ce que ça prouve à un recruteur |
|---|---|---|---|
| **Cœur** | Exploitation AD / Entra ID moderne + instrumentation détection (SIEM) | **Lourd (≈65 %)** | « Je sais attaquer un SI d'entreprise moderne ET je sais reconnaître/documenter l'attaque côté défense — pas juste réciter un lab INE. » |
| **Agent** | Copilote IA sur BloodHound — priorisation attaque *et* triage défensif | Moyen (≈20 %) | « J'ai automatisé la partie répétitive du travail — que ce soit pour prioriser une attaque ou évaluer l'impact d'un compte compromis en triage SOC. » |
| **Bonus (optionnel)** | Pentest de mon propre agent IA (OWASP LLM Top 10) | Léger, **seulement si le temps le permet, en tout dernier** | « Je sais aussi tester un système IA avec une méthodologie reconnue » — un plus différenciant, pas un pilier sur lequel compter. |

**L'astuce narrative qui rend ce projet fort :** le **même artefact** (l'agent) sert deux publics différents — tu l'as *construit* pour accélérer une analyse, et il fonctionne aussi bien pour un pentester qui priorise une attaque que pour un analyste SOC qui évalue la sévérité d'un compte compromis. Un recruteur voit d'un coup un projet cohérent, quel que soit le poste visé.

**Ordre de construction :** cœur AD (avec détection intégrée dès le module 1) → agent → bonus optionnel si le temps le permet. Voir la roadmap en §8.

---

## 3. Architecture du lab

```
                        ┌─────────────────────────────┐
                        │   Poste attaquant (Kali)     │
                        │   + Agent IA copilote        │
                        └──────────────┬───────────────┘
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 │                                            │
        ┌────────▼─────────┐   Azure AD Connect    ┌──────────▼──────────┐
        │  AD on-prem       │◄─────── sync ────────►│   Entra ID           │
        │  (GOAD-Light)     │                       │  (tenant gratuit,    │
        │  DC01, DC02...    │                       │   essai Azure)       │
        │  + 1 DC Win 2025  │                       │  Conditional Access, │
        │  (non patché,     │                       │  App Registrations,  │
        │   pour dMSA)      │                       │  hybrid join         │
        └─────────┬─────────┘                       └──────────────────────┘
                  │ Sysmon + agent Wazuh
        ┌─────────▼─────────┐
        │  SIEM (Wazuh)     │
        │  règles + alertes │
        │  par technique    │
        └───────────────────┘
```

**Où ça tourne :** sur ta machine via des VM (VMware Workstation Pro). GOAD-Light se déploie avec Vagrant + Ansible.

**Prérequis logiciels (poste attaquant) :**
- Kali Linux (ou un Debian/Ubuntu avec les outils installés à la main)
- Docker (pour BloodHound CE / Neo4j)
- Python 3.11+
- Une clé API LLM (Claude, via `anthropic` en Python) pour l'agent

---

## 4. Cœur — Exploitation AD / Entra ID + détection

> Chaque module suit le même schéma : **objectif → étapes → outils → livrable**. Le livrable de chaque module est un writeup Markdown (`attack-writeups/xx-nom.md`) avec commandes, sorties, captures, **et une capture d'écran de l'alerte SIEM qui s'est réellement déclenchée** — pas seulement une phrase qui décrit la détection théorique.

### Module 1 — Construire le lab (semaine 1)

**Objectif :** un environnement hybride crédible et reproductible.

**Étapes :**
1. Cloner **GOAD** (Game of Active Directory) et déployer l'environnement **GOAD-Light** (2 domaines, 3 machines) via Vagrant + Ansible.
2. Ajouter **un DC Windows Server 2025** à la forêt, laissé **non patché** (pas de mises à jour post-août 2025) — pour reproduire BadSuccessor au module 6.
3. Créer un **tenant Entra ID gratuit** via un **essai Azure individuel**.
4. Installer **Microsoft Entra Connect** sur un serveur membre pour synchroniser l'AD on-prem vers le tenant. Configurer au moins un utilisateur en **hybrid join**.
5. Documenter la reconstruction complète dans `infra/` (les commandes Vagrant/Ansible, la config Entra Connect).

**Outils :** GOAD, Vagrant, VMware Workstation, Microsoft Entra Connect.

**Livrable :** `infra/README.md` — « comment reconstruire ce lab de zéro ».

---

### Module 2 — Instrumentation SIEM (semaine 1-2)

**Objectif :** transformer le lab d'un simple terrain d'attaque en un terrain d'attaque **observable**, comme le serait un vrai SI supervisé.

**Étapes :**
1. Déployer **Wazuh** (gratuit, open-source, installation "All-In-One") sur une VM Ubuntu Server dédiée, hors GOAD.
2. Installer l'agent Wazuh + **Sysmon** (config **Olaf Hartong — sysmon-modular**, structurée par technique MITRE ATT&CK) — pour l'instant sur **DC01 uniquement**, pour valider toute la chaîne de bout en bout avant d'étendre aux autres machines (DC02, DC03, SRV02) si le besoin s'en fait sentir pour un module donné.
3. Vérifier que les logs remontent bien : Event IDs de sécurité Windows (4624, 4625, 4768, 4769...), Sysmon (process creation, network connections).
4. Vérifier dans le dashboard Wazuh (Threat Hunting) que les groupes de règles Sysmon (`sysmon`, `sysmon_eid1_detections`) apparaissent bien pour l'agent concerné.

**Outils :** Wazuh, Sysmon (Sysinternals).

**Livrable :** `siem/setup.md` — comment le SIEM est déployé et connecté au lab. C'est la fondation qui rend tous les modules suivants "détection-crédibles".

---

### Module 3 — Reconnaissance moderne (semaine 2, début)

**Objectif :** cartographier le domaine avec les outils standards 2026.

**Étapes :**
1. Énumération réseau et services : **NetExec** (`nxc`, le successeur de CrackMapExec) pour le smb/ldap/winrm spraying et l'énumération.
2. Collecte du graphe : **SharpHound / BloodHound CE** (nouveau moteur graphe) → base **Neo4j**.
3. Écrire tes **propres requêtes Cypher** (pas seulement les pré-faites) pour identifier : comptes kerberoastables, chemins vers Tier0, ACL dangereuses, templates ADCS. Ces requêtes te resserviront directement comme « tools » de l'agent.
4. Optionnel : **AD Miner** pour un rapport d'audit automatisé de chemins.
5. Vérifie dans Wazuh à quoi ressemble (ou ne ressemble pas) cette phase de reconnaissance côté logs — beaucoup de reconnaissance passe inaperçue, c'est un constat à documenter aussi.

**Outils :** NetExec, BloodHound CE, Neo4j, AD Miner, Impacket, Wazuh.

**Livrable :** `attack-writeups/03-recon.md` + `agent/queries/*.cypher`.

---

### Module 4 — ADCS, de ESC1 à ESC16 (semaines 2-3)

**Objectif :** aller nettement au-delà des bases habituelles.

**Étapes :**
1. Sur ton lab, configure **plusieurs templates de certificats vulnérables** correspondant à différentes classes ESC.
2. Exploite au minimum **ESC1 → ESC11** avec **Certipy** : demande de certificat au nom d'un autre compte (ESC1), templates mal configurés, absence d'extension de sécurité (ESC9), relais NTLM vers les endpoints HTTP AD CS (ESC8/ESC11).
3. Si le temps le permet, pousse jusqu'à **ESC13, ESC14, ESC16**.
4. Pour chaque ESC : explique **pourquoi** la config est vulnérable, **comment** on la corrige côté défense, et **crée une règle de détection Wazuh** correspondante (au minimum sur les Event IDs de délivrance de certificat).

**Outils :** Certipy, NetExec, Impacket (`ntlmrelayx`), Wazuh.

**Livrable :** `attack-writeups/04-adcs-esc.md` — un tableau ESC par ESC (condition, exploitation, détection, remédiation, capture d'alerte).

---

### Module 5 — Kerberos & délégation, version complète (semaine 3-4)

**Objectif :** montrer la maîtrise des abus Kerberos avancés, pas juste le Kerberoasting.

**Étapes / techniques à chaîner :**
1. **Kerberoasting / AS-REP Roasting** ciblé et *justifié*.
2. **Délégation** : unconstrained, constrained (S4U2Self / S4U2Proxy), et **RBCD**.
3. **Shadow Credentials** : abus de `msDS-KeyCredentialLink` (Whisker / pyWhisker → PKINIT).
4. **Pass-the-Certificate** et **Overpass-the-Hash**.
5. **Abus de DACL/ACL** cartographiés via BloodHound puis exploités via **bloodyAD**.
6. Pour chaque technique : quelle alerte Wazuh se déclenche (ou pas) ? Documente les angles morts autant que les détections réussies — un vrai SOC junior doit savoir qu'une supervision n'est jamais parfaite.

**Outils :** Rubeus, Certipy, Whisker/pyWhisker, bloodyAD, Impacket, Wazuh.

**Livrable :** `attack-writeups/05-kerberos-delegation.md` avec une **chaîne complète** reliant un accès initial jusqu'à Domain Admin, et le statut de détection à chaque étape.

---

### Module 6 — BadSuccessor / dMSA (semaine 4)

**Objectif :** traiter un sujet 2025-2026 récent, en version « cycle complet ».

**Étapes :**
1. Sur le **DC Windows Server 2025 non patché**, obtiens le droit `CreateChild` sur une OU.
2. Crée un objet **dMSA** et manipule les attributs pour te faire passer pour un compte privilégié.
3. Obtiens un TGT au nom du compte cible → escalade jusqu'à Domain Admin.
4. **Démontre le correctif** : applique le patch (CVE-2025-53779), montre que l'attaque directe échoue.
5. **Construis la détection dans Wazuh** (Event ID 5137 sur création de dMSA) et capture l'alerte réelle qui se déclenche.

**Outils :** SharpSuccessor / scripts publics dMSA, Certipy, Rubeus/Impacket, Wazuh.

**Livrable :** `attack-writeups/06-badsuccessor-dmsa.md` — attaque + patch + détection instrumentée.

---

### Module 7 — Secrets, persistence, défense en profondeur (semaine 5)

**Objectif :** montrer que tu comprends le modèle de défense, pas juste l'attaque brute.

**Étapes / techniques :**
1. Extraction **NTDS.dit** (DCSync via secretsdump).
2. Abus **LAPS**.
3. **NTLM relay moderne** en tenant compte de l'**EPA**.
4. Discussion du **tiering model** (Tier0/Tier1/Tier2).
5. Alertes Wazuh sur DCSync (réplication anormale, Event ID 4662) et sur l'accès LAPS.

**Outils :** Impacket (secretsdump, ntlmrelayx), NetExec, BloodHound, Wazuh.

**Livrable :** `attack-writeups/07-secrets-persistence.md`.

---

### Module 8 — Identité hybride Entra ID (semaine 6)

**Objectif :** le pivot on-prem ↔ cloud — de l'exploitation d'identité pure.

**Étapes / techniques :**
1. **Device code phishing**.
2. **Vol de PRT** (Primary Refresh Token) sur un poste hybrid-joined.
3. **Abus du compte de synchronisation Entra Connect** (droits DCSync) → pivot **on-prem → cloud**.
4. **Cloud Kerberos Trust attack** (Dirk-jan Mollema, juillet 2026) : pivot dans le sens **inverse**, cloud → on-prem, via usurpation du compte MSOL_ depuis Entra ID.
5. **Contournement de Conditional Access**.
6. **Abus d'App Registrations / Service Principals** sur-privilégiés.
7. Documente ce qui est visible côté logs Entra ID (Sign-in logs, Audit logs) en plus de Wazuh côté on-prem — c'est un point de bascule intéressant pour un profil SOC : la supervision cloud et on-prem ne se font pas avec les mêmes outils.

**Outils :** **ROADtools** (ROADrecon, roadtx), **TokenTactics**, **AADInternals**, logs Entra ID natifs.

**Livrable :** `attack-writeups/08-entra-id-hybride.md` — une démo de pivot **dans les deux sens**, avec ce qui est détectable côté cloud et côté on-prem.

---

## 5. L'agent copilote (semaines 7-8)

> Message de posture : ce n'est pas « mon projet IA à côté », c'est **un outil d'aide à la priorisation** qui a deux usages complémentaires — un pentester s'en sert pour prioriser un chemin d'attaque, un analyste SOC pourrait s'en servir pour évaluer l'impact potentiel d'un compte compromis signalé par une alerte.

**Architecture (4 couches) :**

1. **Ingestion.** L'agent se connecte à la base **Neo4j** de BloodHound. Driver Neo4j Python.
2. **Tools structurés.** Fonctions précises que l'agent appelle via *tool use* (API Claude). Chaque tool correspond à une technique déjà exploitée à la main au cœur du projet, donc tu peux vérifier que l'agent ne raconte pas n'importe quoi :
   - `get_shortest_path(from, to)`
   - `list_kerberoastable_users()`
   - `check_esc_vulnerable_templates()`
   - `list_shadow_cred_candidates()`
   - `list_dmsa_abusable()`
   - `list_overprivileged_service_principals()`
   - `assess_compromised_account_impact(account)` — pense triage SOC ("ce compte a-t-il un accès direct ou indirect vers Domain Admin ? quelle sévérité recommandée ?").
3. **Raisonnement.** À partir d'une question en langage naturel, l'agent enchaîne plusieurs appels de tools et construit soit un **plan d'attaque priorisé** (usage pentest), soit une **évaluation de sévérité** (usage triage SOC), avec justification.
4. **Couche de validation.** Chaque réponse doit être **vérifiable** : confrontation aux données Neo4j réelles (anti-hallucination), toujours avec confirmation humaine avant toute action qui modifie quelque chose.

**Stack :** Python, API Claude (tool use), driver Neo4j.

**Étapes :**
1. Brancher l'agent sur Neo4j, coder 2-3 tools simples.
2. Faire fonctionner la boucle *question → appels de tools → réponse*.
3. Ajouter la couche de validation.
4. Étendre aux tools ADCS / dMSA / Entra ID + le tool de triage.
5. Enregistrer une **démo (GIF/vidéo)** montrant les deux usages (question pentest, question triage).

**Livrable :** `agent/` (code + tools + validation) + une démo dans le README principal.

---

## 6. Bonus optionnel — Pentest de ton propre agent (si le temps le permet, en dernier)

**Objectif :** retourner l'agent contre lui-même avec la méthodologie **OWASP Top 10 for LLM Applications**. À ne faire qu'après le cœur AD/détection et l'agent, jamais avant — ce n'est plus un pilier sur lequel compter pour l'employabilité, seulement un différenciateur bonus.

**Les 4 couches à tester :**
- **Planification / raisonnement du LLM** — prompt injection directe et indirecte.
- **Appels d'outils avec privilèges** — tool poisoning, escalade via chaînage d'outils.
- **Mémoire persistante** — memory poisoning.
- **Comportement non-déterministe** — reproductibilité des attaques.

**Livrable (si fait) :** `docs/pentest-agent-ia.md`.

---

## 7. Reporting (semaine 9)

**Objectif :** montrer que tu sais formaliser un constat, que ce soit pour un rapport de pentest ou une fiche d'incident SOC.

**Étapes :**
1. Rédiger un **rapport complet** sur les modules 1 à 8 : contexte, méthodologie, findings classés par criticité (CVSS), preuves, recommandations, **et statut de détection** (détecté / non détecté / partiellement détecté) pour chaque technique — cette dernière colonne est ce qui parle directement à un recruteur SOC.
2. **Bonus :** ton agent génère un **brouillon** de rapport à partir des findings structurés.

**Livrable :** `report/rapport-pentest-detection.pdf` (+ source Markdown).

---

## 8. Roadmap alignée sur ta deadline (PFE avril 2027)

Repère clé : **avoir un livrable présentable dès décembre 2026**.

| Période (indicative) | Focus | Statut « présentable » |
|---|---|---|
| Sem. 1 | Module 1 : lab GOAD-Light + DC Win2025 + sync Entra ID | — |
| Sem. 1-2 | Module 2 : instrumentation Wazuh/Sysmon | — |
| Sem. 2-3 | Modules 3-4 : recon moderne + ADCS ESC1→16 (+ détection) | — |
| Sem. 4 | Modules 5-6 : Kerberos/délégation + BadSuccessor (+ détection) | — |
| Sem. 5-6 | Modules 7-8 : secrets/persistence + Entra ID hybride (+ détection) | ✅ **Cœur AD/Entra ID + détection complet** |
| Sem. 7-8 | Agent copilote (double usage attaque/triage) + démo | ✅ **Projet présentable** |
| Sem. 9 | Reporting (avec statut de détection par technique) | ✅ **Version finale** |
| Sem. 10+ (si temps) | Bonus optionnel : pentest de l'agent (OWASP LLM) | ✅ Version différenciante |

**Si le temps manque, priorise dans cet ordre :** Module 1 → Module 2 (SIEM) → Modules 3-8 (cœur AD + détection) → Agent → Reporting → Bonus LLM. Le bonus LLM est la dernière chose à sacrifier si tu manques de temps, pas la première à faire.

> À raison de ~8-12 h/semaine, compte ~9-11 semaines calendaires pour le cœur + agent + reporting. En démarrant en septembre-octobre 2026, tu es largement dans les temps.

---

## 9. Structure du repo GitHub

```
ad-entraid-hybride-attaque-detection-2026/
├── README.md                    # Pitch, architecture, GIF de démo, "comment reproduire"
├── infra/                       # IaC du lab (Vagrant/Ansible), config Entra Connect — Module 1
│   └── README.md
├── siem/                        # Setup Wazuh, règles, captures d'alertes — Module 2
│   ├── setup.md
│   └── rules/
├── attack-writeups/             # Un writeup par module d'attaque (modules 3 à 8)
│   ├── 03-recon.md
│   ├── 04-adcs-esc.md
│   ├── 05-kerberos-delegation.md
│   ├── 06-badsuccessor-dmsa.md
│   ├── 07-secrets-persistence.md
│   └── 08-entra-id-hybride.md
├── agent/                       # L'agent copilote (attaque + triage)
│   ├── tools/
│   ├── queries/
│   ├── validation/
│   └── main.py
├── docs/
│   └── pentest-agent-ia.md      # Bonus optionnel, si fait
└── report/
    └── rapport-pentest-detection.pdf
```

**Soigne le README principal** : schéma d'architecture (avec le SIEM), GIF de démo de l'agent, et une phrase d'accroche claire.

---

## 10. Comment le présenter en entretien

**Pour un poste SOC / analyste sécurité (cible principale) :**
- Mets en avant l'instrumentation Wazuh et le tableau détection/non-détection par technique — c'est rare qu'un junior ait réellement observé une alerte se déclencher plutôt que de réciter un Event ID.
- Relie explicitement au **MITRE ATT&CK** : chaque technique de ton lab correspond à une technique référencée (T1558 Kerberoasting, T1003 credential dumping, etc.) — les recruteurs SOC parlent ce langage.
- Le tool `assess_compromised_account_impact` de ton agent est un excellent point d'entrée pour parler de triage et de priorisation d'incident.

**Pour un cabinet de conseil pentest (Wavestone…) ou un pure-player offensif (Synacktiv…), si tu gardes cette option ouverte :**
- Insiste sur la profondeur technique du cœur AD (ADCS ESC1-16, dMSA, délégation, Entra ID) et sur l'agent en usage "priorisation d'attaque".
- Leur process est très technique (challenge, CTF) : ton lab doit tenir la route sous questions pointues.

**Verdict marché (sans complaisance) :** ce projet te vend d'abord auprès de recruteurs SOC/analyste, grâce à l'instrumentation détection — c'est le marché le plus large et le plus accessible pour un junior. Il garde une porte ouverte, non garantie, vers les cabinets de pentest classiques. Il ne te qualifie **pas** pour des postes « AI safety researcher » de labo frontière — et ce n'est pas son but.

---

## 11. Ressources

### Références techniques offensives
- **GOAD (Game of Active Directory)** : https://github.com/Orange-Cyberdefense/GOAD
- **The Hacker Recipes — AD CS** : https://www.thehacker.recipes/ad/movement/adcs/
- **Certipy Wiki — Privilege Escalation (ESC)** : https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation
- **The Ultimate ADCS Attack Handbook (ESC1→ESC16)** : https://packetwanderer.com/posts/active-directory-certificate-services/
- **Akamai — BadSuccessor (attaque)** : https://www.akamai.com/blog/security-research/abusing-dmsa-for-privilege-escalation-in-active-directory
- **Akamai — BadSuccessor is Dead (le patch, CVE-2025-53779)** : https://www.akamai.com/blog/security-research/badsuccessor-is-dead-analyzing-badsuccessor-patch
- **NetExec (nxc)** : https://www.netexec.wiki/
- **ROADtools (Entra ID)** : https://github.com/dirkjanm/ROADtools

### Références détection / SOC
- **Wazuh — documentation officielle** : https://documentation.wazuh.com/
- **Sysmon config de référence (Olaf Hartong — sysmon-modular)** : https://github.com/olafhartong/sysmon-modular
- **MITRE ATT&CK — matrice Enterprise** : https://attack.mitre.org/matrices/enterprise/
- **MITRE ATT&CK — techniques AD spécifiques** (Kerberoasting T1558.003, DCSync T1003.006, etc.)
- Plateformes d'entraînement détection (pour compléter ton profil au-delà du lab) : **LetsDefend**, **Blue Team Labs Online**, **CyberDefenders**

### Offres et pages carrières à surveiller
Les offres individuelles expirent vite ; pour un PFE d'avril 2027, la plupart ouvriront entre septembre 2026 et janvier 2027.

- **Recherches SOC / analyste** : https://fr.indeed.com/q-stage-analyste-soc-emplois.html
- **Wavestone (toutes offres)** : https://careers.smartrecruiters.com/wavestone1
- **Synacktiv — offres & book de stages** : https://www.synacktiv.com/offres
- **Orange Cyberdefense — carrières** : https://www.orangecyberdefense.com/fr/carrieres
- Board spécialisé cyber : https://www.cyberjobs.fr/

---

*Document généré en septembre 2026. Vérifie toujours le statut des CVE, des patchs et des offres au moment où tu exécutes.*
