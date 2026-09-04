# Pentest AD / Entra ID Hybride — Techniques Offensives 2026

> **Version 2026, vérifiée marché.** Document de cadrage et guide d'exécution étape par étape.
> Auteur : Saad Agoumi — IMT Atlantique — cible : stage de fin d'études (avril 2027) puis CDI en pentest / sécurité offensive.
> Repo GitHub : `pentest-ad-entraid-hybride-2026`

---

## 0. Règles du jeu (à lire une fois, à respecter toujours)

- **Tout se fait sur TON lab, dans TON tenant, avec TES machines.** Aucune technique n'est jamais lancée contre un système que tu ne possèdes pas ou pour lequel tu n'as pas une autorisation écrite. C'est la ligne rouge du métier, et c'est aussi ce qu'un recruteur veut voir : de la maturité, pas juste de la technique.
- Le tenant Entra ID et les ressources Azure utilisent des offres **gratuites** (Microsoft 365 Developer Program, free trial Azure). Zéro besoin de payer, zéro besoin d'accéder à quoi que ce soit de propriétaire.
- Chaque technique offensive que tu documentes, tu documentes **aussi la détection et la remédiation**. C'est ce qui te distingue d'un script-kiddie et te positionne comme quelqu'un qui comprend la défense.

---

## 1. Ce que j'ai vérifié et ce qui change par rapport à la première version

J'ai repris ta proposition à trois piliers et je l'ai confrontée au marché réel de septembre 2026 (recherche technique + offres d'emploi + contenus publiés par les cabinets que tu vises). **Le cœur du projet tient parfaitement la route.** Trois ajustements importants :

### 1.1 BadSuccessor / dMSA est désormais PATCHÉ — et c'est une bonne nouvelle pour toi

La première version présentait BadSuccessor comme « pas encore patché » et « le sujet le plus chaud ». Ce n'est plus exact au moment où tu lis ça :

- Microsoft a assigné le **CVE-2025-53779** et publié un correctif **moins d'une semaine après la présentation DEF CON 2025** (août 2025).
- Le patch impose une **validation KDC par lien mutuel** entre le dMSA et le compte cible : l'attribut `msDS-ManagedAccountPrecededByLink` peut toujours être écrit, mais le KDC n'émet plus de ticket sauf si le couplage ressemble à une vraie migration.
- **La voie d'escalade directe est donc fermée sur un DC à jour.**

**Ce que ça change concrètement pour ton projet :**
- Tu montes **volontairement un DC Windows Server 2025 non patché** dans ton lab pour reproduire l'attaque d'origine (c'est légitime et courant en lab).
- Mais tu ne la présentes **pas** comme une 0-day. Tu la présentes comme un **cycle complet** : (a) exploitation de la technique, (b) explication du patch (validation du lien mutuel côté KDC), (c) détection (Event ID 5137 sur création de dMSA, requêtes sur les attributs sensibles). Ce triptyque « attaque → correctif → détection » est **beaucoup plus impressionnant** en entretien qu'un simple « j'ai lancé l'exploit » : il prouve que tu es à jour en 2026, pas en mai 2025, et que tu penses comme un défenseur autant que comme un attaquant.

### 1.2 Profondeur > largeur : confirmé, on garde le cap

Le marché récompense la capacité **démontrable** à exploiter un AD moderne, pas une liste de 10 sujets effleurés. La structure « pilier 1 lourd, piliers 2 et 3 en soutien » est la bonne. On résiste à la tentation d'ajouter cloud pur, CI/CD, etc. — ces sujets diluent le signal principal.

### 1.3 Le timing dicte l'ordre de construction

Ton PFE démarre **le 1er avril 2027**. Les candidatures PFE des grands cabinets ouvrent en général **entre septembre 2026 et janvier 2027** (au fil de l'eau). Tu dois donc pouvoir montrer un livrable **présentable dès décembre 2026**, même partiel. D'où la règle : **on front-load le cœur AD** (piliers 1 + 3) pour qu'un recruteur croisé en novembre-décembre ait déjà quelque chose de solide à regarder. Le pilier 2 et les bonus viennent ensuite.

### 1.4 Ce que le marché confirme (et qui valide tes 3 piliers)

- **AD / Entra ID hybride = surface n°1 en pentest interne** : confirmé par le panorama outils AD 2026 de Wavestone, la formation PENTESTAD de HS2, et le stage « Deep Purple » de Synacktiv (AD + postures Windows/Linux/macOS + cloud Azure/AWS/GCP).
- **NetExec est bien le standard 2025-2026** qui a remplacé CrackMapExec (abandonné).
- **ADCS va bien de ESC1 à ESC16** aujourd'hui (la recherche a étendu la liste en 2025).
- **Le pentest augmenté par l'IA ET le pentest des applications IA/LLM sont deux demandes marché réelles** : Orange Cyberdefense l'écrit noir sur blanc dans son propre blog (les pentesters doivent désormais évaluer la robustesse des LLM de leurs clients, avec le référentiel OWASP LLM en tête), Wavestone publie un stage sur le « développement de capacités de tests de pénétration spécifiques aux systèmes d'IA agentiques », et des cabinets comme Vaadata vendent déjà un service « Pentest LLM » dédié (apps IA, agents, RAG, MCP).

---

## 2. Le projet en trois piliers

| Pilier | Nom | Poids | Ce que ça prouve à un recruteur |
|---|---|---|---|
| **1** | Exploitation AD / Entra ID moderne | **Lourd (≈60 %)** | « Je sais vraiment hacker un SI d'entreprise moderne, pas juste réciter un lab INE. » |
| **2** | Pentest de mon propre agent IA (OWASP LLM Top 10) | Moyen (≈20 %) | « Je sais tester un système IA avec une méthodologie reconnue » — le service que vendent déjà OCD, Wavestone, Vaadata. |
| **3** | L'agent copilote lui-même (BloodHound → chemins d'attaque) | Léger (≈20 %) | « J'ai automatisé la partie répétitive de mon propre travail de pentester » — tu penses productivité de mission, comme un consultant senior. |

**L'astuce narrative qui rend ce projet fort :** le **même artefact** (ton agent) sert deux compétences distinctes et très recherchées — tu l'as *construit* (pilier 3) **et** tu l'as *pentesté* (pilier 2). Un recruteur voit d'un coup les deux profils du moment sur un seul projet cohérent, sans que tu aies eu à faire deux projets séparés.

**Ordre de construction ≠ ordre des piliers :** tu construis le pilier 3 (l'agent) avant de pouvoir faire le pilier 2 (l'attaquer). Voir la roadmap en §7.

---

## 3. Architecture du lab

```
                        ┌─────────────────────────────┐
                        │   Poste attaquant (Kali)     │
                        │   + Agent IA copilote (P3)   │
                        └──────────────┬───────────────┘
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 │                                            │
        ┌────────▼─────────┐   Azure AD Connect    ┌──────────▼──────────┐
        │  AD on-prem       │◄─────── sync ────────►│   Entra ID           │
        │  (GOAD)           │                       │  (tenant gratuit     │
        │  DC01, DC02...    │                       │   M365 Dev Program)  │
        │  + 1 DC Win 2025  │                       │  Conditional Access, │
        │  (non patché,     │                       │  App Registrations,  │
        │   pour dMSA)      │                       │  hybrid join         │
        └───────────────────┘                       └──────────────────────┘
```

**Où ça tourne :** sur ta machine (16 Go de RAM confortables, 32 idéal) via des VM, ou sur un petit serveur/VPS. GOAD se déploie avec Vagrant + Ansible.

**Prérequis logiciels (poste attaquant) :**
- Kali Linux (ou un Debian/Ubuntu avec les outils installés à la main)
- Docker (pour BloodHound CE / Neo4j)
- Python 3.11+
- Une clé API LLM (Claude, via `anthropic` en Python) pour le pilier 3

---

## 4. Pilier 1 — Exploitation AD / Entra ID (le cœur)

> Chaque module suit le même schéma : **objectif → étapes → outils → livrable**. Le livrable de chaque module est un writeup Markdown (`attack-writeups/xx-nom.md`) avec commandes, sorties, captures, et une section « Détection & remédiation ».

### Module 0 — Construire le lab (semaine 1)

**Objectif :** un environnement hybride crédible et reproductible.

**Étapes :**
1. Cloner **GOAD** (Game of Active Directory) et déployer l'environnement `GOAD` complet (multi-forêt) via Vagrant + Ansible. Prends le temps de comprendre chaque VM (DC, serveurs membres, comptes).
2. Ajouter **un DC Windows Server 2025** à la forêt, laissé **non patché** (pas de mises à jour post-août 2025) — c'est lui qui te permettra de reproduire BadSuccessor au module 4.
3. Créer un **tenant Entra ID gratuit** via le **Microsoft 365 Developer Program** (licences E5 de test).
4. Installer **Azure AD Connect** sur un serveur membre pour synchroniser l'AD on-prem vers le tenant. Configure au moins un utilisateur en **hybrid join**.
5. Documenter la reconstruction complète dans `infra/` (les commandes Vagrant/Ansible, la config AAD Connect).

**Outils :** GOAD, Vagrant, VirtualBox/VMware, Azure AD Connect.

**Livrable :** `infra/README.md` — « comment reconstruire ce lab de zéro ». C'est déjà un point fort : ça montre que tu sais *construire*, pas seulement *casser*.

---

### Module 1 — Reconnaissance moderne (semaine 2, début)

**Objectif :** cartographier le domaine avec les outils standards 2026.

**Étapes :**
1. Énumération réseau et services : **NetExec** (`nxc`, le successeur de CrackMapExec) pour le smb/ldap/winrm spraying et l'énumération.
2. Collecte du graphe : **SharpHound / BloodHound CE** (nouveau moteur graphe) → base **Neo4j**.
3. Écrire tes **propres requêtes Cypher** (pas seulement les pré-faites) pour identifier : comptes kerberoastables, chemins vers Tier0, ACL dangereuses, templates ADCS. Ces requêtes te resserviront directement comme « tools » de l'agent au pilier 3.
4. Optionnel : **AD Miner** pour un rapport d'audit automatisé de chemins, en complément.

**Outils :** NetExec, BloodHound CE, Neo4j, AD Miner, Impacket.

**Livrable :** `attack-writeups/01-recon.md` + un fichier `agent/queries/*.cypher` avec tes requêtes commentées.

---

### Module 2 — ADCS, de ESC1 à ESC16 (semaines 2-3)

**Objectif :** aller nettement au-delà du tronc commun eCPPT (qui s'arrête souvent à ESC1-ESC8).

**Étapes :**
1. Sur ton lab, configure **plusieurs templates de certificats vulnérables** correspondant à différentes classes ESC (tu peux t'appuyer sur les configs de GOAD ou les créer à la main pour bien comprendre chaque faille).
2. Exploite au minimum **ESC1 → ESC11** avec **Certipy** : demande de certificat au nom d'un autre compte (ESC1), templates mal configurés, absence d'extension de sécurité (ESC9), relais NTLM vers les endpoints HTTP AD CS (ESC8/ESC11).
3. Si le temps le permet, pousse jusqu'à **ESC13, ESC14, ESC16** (mapping de certificat faible, extension de sécurité désactivée globalement sur la CA).
4. Pour chaque ESC : explique **pourquoi** la config est vulnérable et **comment** on la corrige côté défense.

**Outils :** Certipy, NetExec, Impacket (`ntlmrelayx`).

**Livrable :** `attack-writeups/02-adcs-esc.md` — un tableau ESC par ESC (condition, exploitation, détection, remédiation). C'est une pièce de référence qui impressionne parce que peu de juniors vont aussi loin.

---

### Module 3 — Kerberos & délégation, version complète (semaine 3-4)

**Objectif :** montrer la maîtrise des abus Kerberos avancés, pas juste le Kerberoasting.

**Étapes / techniques à chaîner :**
1. **Kerberoasting / AS-REP Roasting** ciblé et *justifié* (quels comptes, pourquoi).
2. **Délégation** : unconstrained, constrained (S4U2Self / S4U2Proxy), et **RBCD** (resource-based constrained delegation).
3. **Shadow Credentials** : abus de `msDS-KeyCredentialLink` (Whisker / pyWhisker → PKINIT).
4. **Pass-the-Certificate** (authentification PKINIT avec un certificat volé) et **Overpass-the-Hash**.
5. **Abus de DACL/ACL** (GenericAll, WriteDACL, ForceChangePassword…) cartographiés via BloodHound puis exploités via **bloodyAD**.

**Outils :** Rubeus (ou équivalents Linux), Certipy, Whisker/pyWhisker, bloodyAD, Impacket.

**Livrable :** `attack-writeups/03-kerberos-delegation.md` avec une **chaîne complète** reliant un accès initial jusqu'à Domain Admin.

---

### Module 4 — BadSuccessor / dMSA (semaine 4)

**Objectif :** traiter un sujet 2025-2026 que peu de pentesters confirmés ont manipulé en pratique — en version « cycle complet » (cf. §1.1).

**Étapes :**
1. Sur le **DC Windows Server 2025 non patché**, obtiens le droit `CreateChild` sur une OU (via un chemin d'ACL que tu as trouvé au module 3).
2. Crée un objet **dMSA** et manipule les attributs (`msDS-ManagedAccountPrecededByLink`, `msDS-DelegatedMSAState`) pour te faire passer pour un compte privilégié.
3. Obtiens un TGT au nom du compte cible → escalade jusqu'à Domain Admin.
4. **Puis démontre le correctif** : applique le patch (ou explique la validation du lien mutuel côté KDC via CVE-2025-53779), montre que l'attaque directe échoue, et documente la **détection** (création de dMSA, Event ID pertinents).

**Outils :** SharpSuccessor / scripts publics dMSA, Certipy, Rubeus/Impacket.

**Livrable :** `attack-writeups/04-badsuccessor-dmsa.md` — attaque + patch + détection. Ta pièce « je suis à jour 2026 ».

---

### Module 5 — Secrets, persistence, défense en profondeur (semaine 5)

**Objectif :** montrer que tu comprends le modèle de défense, pas juste l'attaque brute.

**Étapes / techniques :**
1. Extraction **NTDS.dit** (DCSync via secretsdump, à partir d'un compte à droits DCSync).
2. Abus **LAPS** (lecture des mots de passe admin locaux mal protégés).
3. **NTLM relay moderne** : relais vers **LDAP(S) / ADCS** en tenant compte de l'**EPA (Extended Protection for Authentication)** — plus représentatif d'un environnement durci 2026 qu'un relais NTLM basique.
4. Discussion du **tiering model** (Tier0/Tier1/Tier2) : comment tu franchis les tiers, et comment un tiering correct t'aurait bloqué.

**Outils :** Impacket (secretsdump, ntlmrelayx), NetExec, BloodHound.

**Livrable :** `attack-writeups/05-secrets-persistence.md`.

---

### Module 6 — Identité hybride Entra ID (semaine 6)

**Objectif :** LE différenciateur 2026 — le pivot on-prem ↔ cloud. C'est de l'exploitation d'identité pure, pas de la config cloud.

**Étapes / techniques :**
1. **Device code phishing** (abus du flow OAuth device code pour voler des tokens).
2. **Vol de PRT** (Primary Refresh Token) sur un poste hybrid-joined.
3. **Abus du compte de synchronisation AAD Connect** (le compte MSOL/sync, qui a souvent des droits **DCSync**) → pivot **on-prem → cloud**.
4. **Cloud Kerberos Trust attack (bonus 2026, la plus récente — pivot dans le sens INVERSE, cloud → on-prem).** Recherche publiée par Dirk-jan Mollema (auteur de ROADtools) en juillet 2026 : un Global Admin Entra ID réécrit le SID on-prem d'un utilisateur hybride pour usurper le compte de sync MSOL_ (celui à droits DCSync), demande un "Partial TGT" depuis Entra ID, qu'un DC on-prem complète en TGT valide → DCSync → tous les hashes du domaine. **Non patché** — Microsoft considère que c'est le modèle de confiance fonctionnant "comme prévu" (Global Admin cloud = équivalent Domain Admin on-prem, par design). Démontrable dès aujourd'hui, sans DC vieilli exprès (contrairement à BadSuccessor).
5. **Contournement de Conditional Access** (named locations, device compliance).
6. **Abus d'App Registrations / Service Principals** sur-privilégiés (permissions Graph API excessives).

**Outils :** **ROADtools** (ROADrecon), **TokenTactics**, **AADInternals**.

**Livrable :** `attack-writeups/06-entra-id-hybride.md` — une démo de pivot **dans les deux sens** (on-prem → cloud via AAD Connect, ET cloud → on-prem via Cloud Kerberos Trust). C'est ce double pivot qui colle exactement à ce que testent Wavestone et Orange Cyberdefense en mission, et qui montre que tu suis la recherche jusqu'à juillet 2026 — plus récent que BadSuccessor (DEF CON 2025).

---

## 5. Pilier 3 — L'agent copilote (semaines 7-8)

> On construit l'agent **avant** de l'attaquer (pilier 2). Message de posture : ce n'est pas « mon projet IA à côté », c'est **l'outil que j'ai construit pour accélérer mon propre travail de pentester** sur le pilier 1.

**Architecture (4 couches) :**

1. **Ingestion.** L'agent se connecte à la base **Neo4j** de BloodHound (pas de copier-coller de graphe brut dans un prompt). Driver Neo4j Python.
2. **Tools structurés.** Tu codes des fonctions précises que l'agent appelle via *tool use* (API Claude). Chaque tool = une technique que tu as **déjà exploitée à la main** au pilier 1, donc tu peux vérifier que l'agent ne raconte pas n'importe quoi :
   - `get_shortest_path(from, to)`
   - `list_kerberoastable_users()`
   - `check_esc_vulnerable_templates()`
   - `list_shadow_cred_candidates()`
   - `list_dmsa_abusable()`
   - `list_overprivileged_service_principals()`
3. **Raisonnement.** À partir d'une question en langage naturel (« j'ai un accès sur PC-COMPTA-04 avec j.dupont, comment j'atteins Domain Admin ? »), l'agent enchaîne plusieurs appels de tools et construit un **plan d'attaque priorisé**, avec les commandes concrètes à chaque étape.
4. **Couche de validation (le vrai travail d'ingénierie).** Chaque chemin proposé doit être **vérifiable** : soit en confrontant la sortie de l'agent aux données Neo4j réelles (anti-hallucination), soit via une exécution en **lecture seule** que l'agent déclenche lui-même, **toujours avec confirmation humaine** avant toute action qui modifie quelque chose.

**Stack :** Python, API Claude (tool use) ou boucle d'appels que tu codes toi-même pour bien comprendre le mécanisme (préférable à un gros framework au début), driver Neo4j.

**Étapes :**
1. Brancher l'agent sur Neo4j, coder 2-3 tools simples (`get_shortest_path`, `list_kerberoastable_users`).
2. Faire fonctionner la boucle *question → appels de tools → réponse*.
3. Ajouter la couche de validation (comparaison systématique aux données réelles).
4. Étendre aux tools ADCS / dMSA / Entra ID.
5. Enregistrer une **démo (GIF/vidéo)** : une question posée → un chemin exploitable et *vérifié* sur ton lab GOAD.

**Livrable :** `agent/` (code + tools + validation) + une démo dans le README principal.

---

## 6. Pilier 2 — Pentest de ton propre agent (semaine 9)

**Objectif :** retourner l'agent contre lui-même avec la méthodologie 2026, basée sur l'**OWASP Top 10 for LLM Applications**. Un vrai rapport de pentest applicatif IA.

**Les 4 couches à tester (méthodologie établie en 2026) :**
- **Planification / raisonnement du LLM** — prompt injection directe et **indirecte** (via des données AD malveillantes que l'agent ingère : nom d'objet, description d'un Service Principal piégée).
- **Appels d'outils avec privilèges** — **tool poisoning**, escalade de privilèges via **chaînage d'outils**.
- **Mémoire persistante** — memory poisoning (si ton agent garde un état entre requêtes).
- **Comportement non-déterministe** — reproductibilité des attaques.

**Étapes :**
1. Mapper chaque classe d'attaque OWASP LLM à ton agent.
2. Construire des payloads (ex : injecter une instruction dans la description d'un objet AD que l'agent va lire).
3. Tester si l'agent peut être détourné de sa tâche, appeler un tool non prévu, ou révéler des données sensibles.
4. Documenter chaque test comme un finding (criticité, preuve, remédiation : validation d'entrée, permissions lecture seule par défaut, confirmation humaine).

**Livrable :** `docs/pentest-agent-ia.md` — « j'ai aussi pentesté mon propre outil IA avec l'OWASP LLM Top 10 ». Phrase qui marque en entretien.

---

## 7. Reporting (semaine 10)

**Objectif :** montrer que tu comprends le vrai métier — le rapport, c'est la moitié du travail d'un pentester pro.

**Étapes :**
1. Rédiger un **rapport de pentest complet** sur les modules 1 à 6 : contexte, méthodologie, findings classés par criticité (CVSS), preuves, recommandations. C'est ta preuve « je sais hacker », indépendante de la couche IA.
2. **Bonus :** ton agent génère un **brouillon** de rapport à partir des findings structurés — toi tu relis et corriges. Tu présentes ça comme un gain de temps mesuré (« draft en X minutes vs Y heures manuellement »).

**Livrable :** `report/rapport-pentest.pdf` (+ source Markdown), au style d'un vrai rapport de mission.

---

## 8. Roadmap alignée sur ta deadline (PFE avril 2027)

Repère clé : **avoir un livrable présentable dès décembre 2026**, car les candidatures PFE ouvrent à ce moment-là.

| Période (indicative) | Focus | Statut « présentable » |
|---|---|---|
| Sem. 1 | Module 0 : lab GOAD + DC Win2025 + sync Entra ID | — |
| Sem. 2-3 | Modules 1-2 : recon moderne + ADCS ESC1→16 | — |
| Sem. 4 | Modules 3-4 : Kerberos/délégation + BadSuccessor | — |
| Sem. 5-6 | Modules 5-6 : secrets/persistence + Entra ID hybride | ✅ **Cœur AD/Entra ID complet** |
| Sem. 7-8 | Pilier 3 : agent copilote + démo | ✅ **Projet présentable (P1 + P3)** |
| Sem. 9 | Pilier 2 : pentest de l'agent (OWASP LLM) | ✅ Version différenciante |
| Sem. 10 | Reporting + polish du repo + README + vidéo | ✅ **Version finale** |

**Si le temps manque, priorise dans cet ordre :** Module 0 → Modules 1-6 (cœur AD) → Pilier 3 (agent) → Reporting → Pilier 2. Le pilier 2 est le bonus qui fait la différence, mais un projet « cœur AD profond + agent + rapport » est **déjà un livrable solide** pour candidater.

> À raison de ~8-12 h/semaine en parallèle de l'eCPPT, compte ~10-14 semaines calendaires. En démarrant en septembre-octobre 2026, tu es largement dans les temps pour les campagnes PFE et tu peux même itérer/enrichir jusqu'au printemps.

---

## 9. Structure du repo GitHub

```
pentest-ai-copilot/
├── README.md                    # Pitch, architecture, GIF de démo, "comment reproduire"
├── infra/                       # IaC du lab (Vagrant/Ansible), config AAD Connect
│   └── README.md
├── attack-writeups/             # Un writeup par module (1 à 6)
│   ├── 01-recon.md
│   ├── 02-adcs-esc.md
│   ├── 03-kerberos-delegation.md
│   ├── 04-badsuccessor-dmsa.md
│   ├── 05-secrets-persistence.md
│   └── 06-entra-id-hybride.md
├── agent/                       # Pilier 3 : l'agent copilote
│   ├── tools/                   # get_shortest_path, list_kerberoastable_users, ...
│   ├── queries/                 # tes requêtes Cypher
│   ├── validation/              # couche anti-hallucination
│   └── main.py
├── docs/
│   └── pentest-agent-ia.md      # Pilier 2 : OWASP LLM Top 10 contre ton agent
└── report/
    └── rapport-pentest.pdf      # Rapport de mission final (+ source .md)
```

**Soigne le README principal** : schéma d'architecture, GIF de démo de l'agent, et une phrase d'accroche claire. C'est la première chose qu'un recruteur regarde.

---

## 10. Comment le présenter en entretien (mappé sur de vraies offres)

- **Pour un cabinet de conseil (Wavestone…) :** insiste sur le pilier 3 (tu penses productivité de mission) et le reporting. Fais le lien avec leurs propres sujets : ils recrutent des stages sur la « sécurisation des accès à privilège » (ton pilier 1, Tier0/PAM) et sur le « développement de capacités de pentest spécifiques aux systèmes d'IA agentiques » (tes piliers 2 et 3).
- **Pour un pure-player offensif (Synacktiv…) :** insiste sur la profondeur technique du pilier 1 (ADCS ESC1-16, dMSA, délégation, Entra ID). Leur stage « Deep Purple » couvre exactement AD + postures + cloud. Leur process est très technique (challenge, CTF) : ton lab doit tenir la route sous questions pointues.
- **Pour Orange Cyberdefense (où tu as déjà un pied) :** ils écrivent eux-mêmes que leurs pentesters doivent évaluer la robustesse des LLM clients (OWASP LLM) *et* utiliser l'IA pour accélérer le pentest. Tes trois piliers répondent point par point.

**Verdict marché (sans complaisance) :** ce projet te vend surtout auprès des **cabinets de pentest classiques et pure-players** (ta cible réelle et la plus large), et t'ouvre une **porte crédible mais non garantie** vers les rôles d'outillage/pentest-IA en interne. Il ne te qualifie **pas** pour des postes « AI safety researcher » de labo frontière (autre métier, senior, exige recherche publiée/CTF spécialisés) — et ce n'est pas son but.

---

## 11. Ressources

### Références techniques (pour aller au fond de chaque module)
- **GOAD (Game of Active Directory)** — le lab de référence : https://github.com/Orange-Cyberdefense/GOAD
- **The Hacker Recipes — AD CS** : https://www.thehacker.recipes/ad/movement/adcs/
- **Certipy Wiki — Privilege Escalation (ESC)** : https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation
- **The Ultimate ADCS Attack Handbook (ESC1→ESC16)** : https://packetwanderer.com/posts/active-directory-certificate-services/
- **Akamai — BadSuccessor (attaque)** : https://www.akamai.com/blog/security-research/abusing-dmsa-for-privilege-escalation-in-active-directory
- **Akamai — BadSuccessor is Dead (le patch, CVE-2025-53779)** : https://www.akamai.com/blog/security-research/badsuccessor-is-dead-analyzing-badsuccessor-patch
- **NetExec (nxc)** : https://www.netexec.wiki/ — présentation red team : https://www.redfoxsec.com/blog/netexec-for-red-teamers-the-modern-toolkit-for-active-directory-exploitation
- **ROADtools (Entra ID)** : https://github.com/dirkjanm/ROADtools
- **Panorama outils AD 2026 (Wavestone RiskInsight)** : https://www.riskinsight-wavestone.com/en/2026/03/overview-of-active-directory-security-tools-version-2026/
- **OWASP Top 10 for LLM Applications** : https://genai.owasp.org/
- **Orange Cyberdefense — Pentesting & IA** : https://www.orangecyberdefense.com/fr/insights/blog/pentesting-et-intelligence-artificielle-ia-simuler-une-attaque-est-la-meilleure-defense

### Offres réelles (voir la note importante ci-dessous)

Les offres individuelles **expirent vite** (le stage Wavestone « nouvelles techniques d'attaque » qu'on avait repéré est déjà clos). Pour un PFE d'avril 2027, la plupart des offres ouvriront **entre septembre 2026 et janvier 2027**. Utilise donc surtout les **pages carrières** (toujours à jour) et les **recherches filtrées**, et surveille-les régulièrement.

**Pages carrières à surveiller (cibles prioritaires) :**
- **Wavestone (toutes offres)** : https://careers.smartrecruiters.com/wavestone1
  - Ex. actuellement ouvertes et pertinentes : « Stage - Intelligence artificielle & Cybersécurité » (techniques d'attaque sur les algorithmes de ML) et « Stage - Sécurisation des accès à privilège : protéger les clés du royaume » (PAM/Tier0).
- **Synacktiv — offres & book de stages** : https://www.synacktiv.com/offres — book stages (PDF) : https://www.synacktiv.com/book_stage_synacktiv.pdf (stage « Deep Purple » = AD + cloud red team)
- **Orange Cyberdefense — carrières** : https://www.orangecyberdefense.com/fr/carrieres — et sur Welcome to the Jungle : https://www.welcometothejungle.com/fr/companies-v1/orange-cyberdefense/jobs

**Recherches filtrées (toujours fraîches) :**
- Indeed — Stage pentest : https://fr.indeed.com/q-stage-pentest-emplois.html
- Indeed — Pentest junior : https://fr.indeed.com/q-securite-pentest-junior-emplois.html
- Indeed — CDI Pentest (Île-de-France) : https://fr.indeed.com/emplois?q=CDI+Pentest&l=%C3%8Ele-de-France
- Welcome to the Jungle — Cybersécurité : https://www.welcometothejungle.com/fr/pages/emploi-cybersecurite
- APEC — stages cyber : https://www.apec.fr/candidat/recherche-stage.html/stage?motsCles=Stage+Cybersécurité
- Cyberjobs.fr (board spécialisé cyber) : https://www.cyberjobs.fr/

---

*Document généré en septembre 2026. Vérifie toujours le statut des CVE, des patchs et des offres au moment où tu exécutes — c'est un réflexe de pentester, et c'est exactement l'état d'esprit que ce projet doit démontrer.*
