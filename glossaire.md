# Glossaire technique — Active Directory, Kerberos, Attaques offensives, Détection & SIEM

> Notes personnelles compilées au fil du projet `ad-entraid-hybride-attaque-detection-2026`. Organisées en 4 parties : fondamentaux AD, attaques Kerberos, outils/notions spécifiques au projet, détection & SIEM.

---

# Partie 1 — Fondamentaux Active Directory

## 1.1 Le problème de départ : la gestion à grande échelle

Une entreprise avec 5 ordinateurs peut se gérer à la main : un compte par employé sur chaque PC, un mot de passe, terminé. Avec **2000 ordinateurs et 3000 employés**, gérer chaque PC séparément devient ingérable :
- un changement de mot de passe devrait se répercuter sur 2000 machines,
- un nouvel employé nécessiterait 2000 créations de compte,
- un départ nécessiterait 2000 désactivations.

Il faut **centraliser** — c'est le problème qu'Active Directory résout.

## 1.2 Le Forest — le niveau le plus large

Le conteneur logique le plus haut dans AD. Une forêt regroupe un ou plusieurs **domaines** qui partagent :
- un **schéma** commun (structure des objets),
- une **configuration** commune,
- un **catalogue global** (Global Catalog) indexant tous les objets de tous les domaines,
- des **trusts** automatiques bidirectionnels et transitifs entre tous les domaines de la forêt.

```
Forest: orange-group.com
 ├── Tree: orange.fr
 │    ├── Domain: orange.fr              (racine du tree, France, siège)
 │    ├── Domain: rennes.orange.fr       (enfant, France)
 │    └── Domain: paris.orange.fr        (enfant, France)
 ├── Tree: orange.es
 │    └── Domain: orange.es              (Espagne)
 └── Tree: orange.pl
      └── Domain: orange.pl              (Pologne)
```

**En pentest** : compromettre le domaine le moins bien sécurisé (souvent une petite filiale) permet parfois de pivoter vers les autres domaines de la forêt en abusant les trusts (SID history, trust key compromise).

## 1.3 Le Tree — le niveau intermédiaire

Un regroupement de domaines qui partagent schéma/configuration (comme dans une forêt), ont des trusts automatiques, et **partagent un namespace DNS contigu** — c'est la caractéristique qui définit un tree : chaque domaine enfant est littéralement un sous-domaine DNS du parent.

```
orange.fr                    ← domaine racine du tree
 ├── rennes.orange.fr        ← domaine enfant (contigu)
 └── paris.orange.fr         ← domaine enfant (contigu)
```

**Une Forest peut contenir plusieurs trees sans namespace DNS commun entre eux** — c'est ce qui distingue une forêt multi-tree d'un simple tree multi-domaine. Dans l'exemple ci-dessus, `orange.fr`, `orange.es`, `orange.pl` sont dans la même forêt mais forment trois trees distincts (noms DNS non contigus), alors que `rennes.orange.fr`/`paris.orange.fr` restent dans le même tree que `orange.fr`.

| Niveau | Ce qui est partagé | Contrainte de nommage DNS |
|---|---|---|
| **Domain** | Base AD, DCs, `krbtgt`, password policy | Un seul namespace DNS |
| **Tree** | + schéma, configuration | Namespace DNS **contigu** obligatoire |
| **Forest** | Idem tree + Global Catalog + trusts globaux | Aucune contrainte de contiguïté entre trees |

Dans la majorité des environnements d'entreprise, il n'y a qu'un seul domaine dans un seul tree dans une seule forêt — le multi-tree/multi-domaine sert surtout aux très grandes organisations (fusions, filiales à marques distinctes, forte séparation géographique).

**En pentest** : deux domaines du même tree ont des trusts parent-enfant automatiques bidirectionnels transitifs, mais c'est en réalité vrai pour tous les domaines d'une même forêt, tree ou pas. La vraie limite de confiance automatique se situe au niveau de la **forêt**, pas du tree — pour du pivoting, ce qui compte c'est "est-on dans la même forêt", le découpage en trees étant plus une question d'organisation DNS que de sécurité.

## 1.4 Le Domain — la vraie frontière de sécurité

L'unité de base d'administration : un ensemble d'objets (users, groups, computers) partageant la même base AD, les mêmes DCs, une politique de sécurité de base commune, et un même Kerberos realm.

```
Domaine: orange.fr
 ├── DCs: DC01-Rennes, DC02-Paris
 ├── Users: Saad, Layda, ~5000 employés
 ├── Groups: Domain Admins, DTSI-Team...
 └── Computers: tous les PCs joints au domaine
```

Deux domaines de la même forêt ont chacun leurs propres admins, leurs propres DCs, leur propre compte `krbtgt`. Compromettre le domaine A ne compromet pas automatiquement le domaine B — c'est **la** vraie frontière de sécurité.

## 1.5 Le Domain Controller (DC)

Le serveur hébergeant le rôle **AD DS**, qui exécute les services d'authentification et valide chaque tentative de connexion.

**Pourquoi plusieurs DCs par domaine :**
1. Redondance/haute disponibilité,
2. répartition de charge,
3. distribution géographique (latence, résilience réseau),
4. réplication multi-master continue entre tous les DCs.

**En pentest** : tous les DCs d'un même domaine partagent le **même hash krbtgt** (répliqué). Compromettre un seul DC — souvent un site distant, moins surveillé — suffit à forger un Golden Ticket valable sur tout le domaine.

## 1.6 Active Directory (AD) — le service d'annuaire

AD n'est pas juste "le software" : c'est un **service d'annuaire** (base de données + services tournant sur le(s) DC(s)). Analogie : AD est le registre central de l'entreprise ; le DC est le serveur qui l'héberge et répond aux demandes.

**Contenu concret** : comptes utilisateurs (mots de passe chiffrés), comptes machine, groupes, OUs, GPOs, ACLs.

**Directory Services** — le mécanisme sous-jacent, le rôle Windows Server AD DS qui fournit : la base NTDS.dit, le protocole LDAP (Get-ADUser, Get-DomainUser...), la réplication entre DCs, l'intégration DNS/Kerberos.

## 1.7 Les OUs (Organizational Units)

Un conteneur **à l'intérieur d'un domaine**, purement organisationnel.

```
Domain: orange.fr
 ├── OU: Rennes
 │    ├── OU: DTSI
 │    │    ├── Users: Saad, Layda...
 │    │    └── Computers: PC-DTSI-01...
 │    └── OU: RH
 ├── OU: Paris
 └── OU: Comptes_Admins
```

Sert à ranger les objets logiquement, appliquer des GPOs ciblées, déléguer des droits d'administration.

**Différence clé avec un groupe** : un objet appartient à **une seule** OU (comme une adresse), mais peut être membre de **plusieurs** groupes (comme des abonnements).

### OU vs conteneurs par défaut

Un nouveau domaine crée automatiquement des conteneurs (`CN=Users`, `CN=Computers`) où atterrissent les nouveaux comptes. **Ces conteneurs ne sont pas des OU**, et les GPO ne peuvent être liées qu'à un site, un domaine, ou une **OU** — jamais à un conteneur `CN=...`. Pour appliquer des GPO : créer une OU, y déplacer les objets, puis lier la GPO.

### Délégation d'administration par OU

Via le **Delegation of Control Wizard** (clic droit sur l'OU → Delegate Control), on peut assigner des tâches précises (réinitialiser des mots de passe, créer/supprimer des comptes, modifier des attributs, gérer des groupes, lier des GPOs) à une OU spécifique — ex. un responsable IT junior peut réinitialiser des mots de passe dans `OU=Marketing` mais pas dans `OU=Finance`.

**Utilité** : moindre privilège, décentralisation dans une grande organisation, réduction de la surface d'attaque en cas de compromission d'un compte à délégation limitée.

**En pentest** : une délégation trop permissive (ex. `GenericAll` sur une OU contenant des comptes admin) est une voie d'escalade classique, que BloodHound cartographie bien.

## 1.8 Authentication vs Authorization

**Authentication = prouver qui on est.** Login/password → vérification via Kerberos (AS-REQ/AS-REP) → obtention d'un TGT. Répond à : *es-tu bien Saad ?*

**Authorization = déterminer ce qu'on a le droit de faire.** Une fois authentifié, le système compare le token (SIDs de groupes) à l'**ACL** de la ressource demandée. Répond à : *Saad a-t-il le droit de lire ce dossier ?*

```
1. Saad se logue → authentification → TGT obtenu
2. Saad ouvre \\SERVEUR\Partage_DTSI
   → autorisation → ACL "DTSI-Team: Read/Write" → Saad est dans DTSI-Team → accès accordé
3. Saad ouvre \\SERVEUR\Partage_RH
   → autorisation → Saad absent des groupes autorisés → refusé
```

**En pentest** : ce sont deux surfaces d'attaque distinctes — pass-the-hash/Kerberoasting attaquent l'authentification, exploiter des ACLs mal configurées (BloodHound) attaque l'autorisation, sans casser de mot de passe.

### SSO (Single Sign-On)

S'authentifier une seule fois pour accéder à plusieurs services sans retaper son mot de passe — exactement ce que permet Kerberos :

```
1. Login le matin → AS-REQ/AS-REP → TGT obtenu
2. Ouverture d'un dossier partagé → TGT présenté au TGS → Service Ticket → accès
3. Ouverture d'une appli interne → même TGT → nouveau Service Ticket → accès
```

Le mot de passe n'est tapé qu'une fois ; le TGT sert ensuite de preuve d'identité réutilisable. (Hors monde Windows : SAML/OAuth2/OIDC pour le SSO web.)

### RBAC (Role-Based Access Control)

Modèle où les permissions sont attachées à des **rôles** ; les users héritent des permissions en étant assignés à un rôle plutôt que d'avoir des droits individuels.

```
Rôle "DTSI-Developer" :
 - Read/Write sur \\SERVEUR\Projets
 - Accès SSH aux serveurs de dev
Saad → assigné au rôle → hérite automatiquement de ces droits
```

Dans AD, **le groupe est l'implémentation concrète du RBAC** : groupe = rôle, ACLs = permissions du rôle, ajout au groupe = assignation du rôle.

### Comment SSO et RBAC s'articulent

Le SSO gère l'authentication, le RBAC gère l'authorization — les deux mêmes étapes, incarnées dans Kerberos :

```
1. SSO : authentification une fois → TGT (AS)
2. Accès à une ressource → TGT sert à obtenir un Service Ticket, sans re-taper le mdp
3. Le Service Ticket contient le PAC, qui liste les groupes
4. RBAC : le serveur lit le PAC → compare aux ACL de la ressource → accès accordé ou refusé
```

Le SSO **transporte** l'identité, le RBAC **décide** de l'accès. Il faut être authentifié avant que le système évalue le rôle.

**En pentest** : un vol de ticket (pass-the-ticket) est si puissant parce qu'en volant le ticket SSO d'un user, on hérite instantanément de son évaluation RBAC (groupes déjà encodés dans le PAC), sans deviner ses permissions.

## 1.9 Kerberos — le protocole d'authentification

Permet de prouver son identité **sans envoyer le mot de passe en clair**, par un système de tickets.

**Composants** : KDC (Key Distribution Center, rôle du DC), divisé en AS (Authentication Service) et TGS (Ticket Granting Service).

```
1. User → AS : authentification → reçoit un TGT (Ticket Granting Ticket)
2. User → TGS : présente le TGT → reçoit un Service Ticket
3. User → Service voulu : présente le Service Ticket → accès
```

**Le PAC (Privilege Attribute Certificate)** : à l'intérieur du ticket, contient le token d'accès de l'user (tous ses SIDs de groupes). Le service compare ces SIDs à l'ACL de la ressource pour l'autorisation.

**En pentest**, ce mécanisme est exploité par : Kerberoasting (extraire/casser des service tickets), Golden Ticket (forger un PAC avec le hash `krbtgt`), pass-the-hash (réutiliser un hash NTLM).

## 1.10 Resource Management

AD centralise la gestion des ressources partagées via les groupes, pas ressource par ressource :

```
\\SERVEUR-FICHIERS\Projets       → accès géré par groupe "Projets-RW"
\\SERVEUR-FICHIERS\Confidentiel  → accès géré par groupe "Direction-Only"
Imprimante-Rennes-3e-etage        → accès géré par groupe "Rennes-Employees"
```

L'admin gère les droits une seule fois au niveau du groupe ; un nouvel arrivant ajouté au groupe hérite automatiquement des accès.

## 1.11 Group Policy Management (GPO)

Des règles appliquées automatiquement aux users/machines selon leur OU.

```
GPO "Sécurité-Rennes" → OU:Rennes
 - Verrouillage d'écran après 5 min
 - Mot de passe minimum 14 caractères
 - Interdiction d'installer des logiciels non signés

GPO "Restriction-Stagiaires" → OU:Stagiaires
 - Pas d'accès USB
 - Pas d'accès PowerShell
```

**En pentest** : le droit de modifier une GPO (souvent mal délégué) permet de pousser un script malveillant exécuté sur tous les PCs de l'OU visée au prochain refresh — mouvement latéral/persistance classique.

## 1.12 Global Catalog (GC)

Service tournant sur certains DCs (pas forcément tous), contenant un **index partiel de tous les objets de toute la forêt**, pas juste du domaine local.

**Problème résolu** : un DC ne connaît en détail que son propre domaine. Chercher un user d'un autre domaine de la forêt nécessite d'interroger le GC.

**Contenu du GC** : tous les attributs des objets du domaine local (complet), et seulement quelques attributs clés des objets des autres domaines de la forêt (nom, email, UPN — pas tout).

```
Recherche de "jkowalski@orange.pl" depuis orange.fr dans Outlook
→ le DC local n'a pas ce user en détail → interroge le GC
→ le GC répond "oui il existe, voici son email/nom" (attributs limités)
```

Utilisé aussi pour résoudre les appartenances aux groupes universels (membres potentiels de plusieurs domaines).

**En pentest** : bonne source de recon — interroger le GC permet de cartographier rapidement tous les domaines/users de la forêt sans se connecter à chaque domaine séparément.

**Précision complémentaire (LDAP/partition Configuration)** : la partition **Configuration** (structure globale de la forêt : domaines existants, schéma, trusts) est répliquée intégralement et identiquement sur **chaque DC**, GC ou non, quel que soit son domaine. Interroger n'importe quel DC, même celui d'un domaine enfant périphérique, suffit donc pour obtenir la liste complète des domaines de la forêt — sans besoin d'un GC pour cette information précise (contrairement aux attributs détaillés d'objets d'un autre domaine, qui eux nécessitent un vrai GC).

| Ce que tu cherches | Disponible sur quel DC |
|---|---|
| Quels domaines existent dans la forêt (structure, noms) | N'importe quel DC, GC ou pas — via Configuration |
| Détails (attributs limités) sur les objets d'un autre domaine | Seulement un DC avec le rôle GC |

## 1.13 kpasswd

Un service Kerberos séparé, dédié uniquement au changement de mot de passe, sur son propre port. Cohérent avec la distinction authentication/authorization : Kerberos gère la preuve d'identité (AS/TGS), mais changer un mot de passe est une opération à part avec ses propres implications de sécurité (s'assurer que c'est bien l'utilisateur qui change SON mot de passe, pas quelqu'un qui rejoue une preuve d'identité volée).

## 1.14 Trust Relationships

Un lien permettant aux users d'un domaine de s'authentifier sur des ressources d'un autre domaine. Sans trust, deux domaines sont étanches, même dans la même forêt.

**Direction** : one-way (A fait confiance à B, pas l'inverse) ou two-way (mutuelle).
**Transitivité** : transitif (A confiance B, B confiance C ⇒ A confiance C automatiquement) ou non-transitif (la confiance s'arrête strictement entre les deux domaines concernés).

**Dans une forêt (par défaut)** : tous les domaines ont des trusts bidirectionnels et transitifs automatiques (trust parent-enfant / tree-root).

**Entre deux forêts différentes** : trust manuel, souvent one-way et pas forcément transitif, pour limiter l'exposition.

```
Orange rachète NetPlus, forêt netplus.com séparée
Trust configuré : netplus.com → fait confiance → orange-group.com
(one-way : les users Orange NE peuvent PAS s'authentifier sur netplus.com,
 mais les users netplus.com PEUVENT accéder à certaines ressources Orange)
```

**En pentest**, les trusts sont une cible de choix pour le pivoting inter-domaines/inter-forêts :
- Compromettre un domaine ayant un trust entrant permet parfois d'abuser ce trust pour accéder à l'autre côté.
- **SID History injection** : abuser un attribut normalement utilisé pour la migration d'users entre domaines, pour usurper l'appartenance à un groupe privilégié de l'autre domaine.
- Un trust mal configuré (trop permissif, transitif là où il ne devrait pas l'être) peut permettre de sauter d'un domaine compromis à un domaine propre sans autre effort.

## 1.15 Security Groups vs Distribution Groups

**Security Groups** : servent à donner des accès/permissions. Ont un **SID**, qui apparaît dans le token d'accès (et donc dans le PAC du ticket Kerberos) — c'est ce qui permet la vérification des permissions à l'autorisation.

```
Groupe "DTSI-Team" utilisé dans une ACL : "DTSI-Team: Read/Write sur \\SERVEUR\Projets"
```

**Distribution Groups** : servent uniquement à envoyer des emails à plusieurs personnes (liste de diffusion). Aucun rôle de sécurité, **pas de SID de sécurité utilisable dans une ACL**.

```
Security Group     → a un SID → utilisable dans les ACLs → contrôle les ACCÈS
Distribution Group  → pas de SID de sécu → utilisable seulement pour l'EMAIL
```

**Piège** : les deux types partagent la même structure de scope (Domain Local, Global, Universal) et se ressemblent visuellement — la seule vraie différence est le type coché à la création.

**En pentest** : aucun intérêt à attaquer un groupe de distribution (pas de droits dessus) — en énumération, on se concentre uniquement sur les security groups.

## 1.16 Groupes de sécurité privilégiés (built-in)

| Groupe | Rôle |
|---|---|
| **Domain Admins** | Contrôle total sur tout le domaine |
| **Enterprise Admins** | Contrôle total sur toute la forêt (existe seulement dans le domaine racine) |
| **Server Operators** | Administre les serveurs membres (services, partages, backup) sans être Domain Admin |
| **Backup Operators** | Sauvegarde/restaure n'importe quel fichier sur DCs/serveurs — accès complet de fait aux données |
| **Account Operators** | Crée/modifie/supprime la plupart des comptes et groupes (sauf ceux à hauts privilèges) |
| **Domain Users** | Contient automatiquement tous les comptes utilisateurs du domaine |
| **Domain Computers** | Contient automatiquement toutes les machines jointes au domaine |
| **Domain Controllers** | Contient les comptes machine des DCs eux-mêmes |

## 1.17 Workgroup vs Domaine

Un ordinateur Windows appartient soit à un **workgroup**, soit à un **domaine (AD)** — jamais les deux à la fois.
- **Workgroup** : mode par défaut, comptes locaux par machine, pas d'authentification centralisée. Petits réseaux.
- **Domaine (AD)** : authentification centralisée (Kerberos/NTLM), gestion centralisée des comptes, GPO, etc.

## 1.18 Étude de cas — Server Manager

**Server Manager** : outil d'administration centrale sur Windows Server, permettant d'installer/configurer/surveiller les rôles/fonctionnalités d'un serveur (local ou distant) depuis une seule interface. Notamment : promouvoir un serveur en DC (installation du rôle AD DS puis assistant de promotion).

**Cas observé** : un serveur avec 3 rôles installés — **AD DS**, **DNS**, **File and Storage Services** — est configuré comme contrôleur de domaine actif :
- **AD DS** → gère l'annuaire (users, groupes, ordinateurs, GPO, authentification Kerberos),
- **DNS** → un DC héberge quasi-systématiquement le DNS, car AD en dépend entièrement pour localiser ses services (enregistrements SRV pour DC/LDAP/Kerberos),
- **File and Storage Services** → héberge probablement le **SYSVOL** (partage réseau des GPO et scripts de connexion, répliqué entre DCs).

## 1.19 Récapitulatif — hiérarchie complète

```
FOREST (orange-group.com)
 └── TREE (orange.fr)                       ← domaines au namespace DNS contigu
      └── DOMAIN (orange.fr)                ← frontière de sécurité
           ├── DOMAIN CONTROLLER (DC01, DC02...) ← le(s) serveur(s)
           │    └── héberge → Active Directory (AD DS)
           │         ├── Directory Service (NTDS.dit, LDAP, réplication, DNS)
           │         ├── KDC (Kerberos) → AS + TGS
           │         └── ACLs (autorisation)
           ├── OU (Rennes → DTSI)                ← organisation + délégation + GPO
           │    └── USER (Saad), COMPUTER (PC-DTSI-01)
           ├── GROUPS (DTSI-Team, Domain Admins) ← gestion des ressources
           └── GPOs (Sécurité-Rennes...)          ← politiques appliquées par OU
```

- **Forest** = plusieurs trees/domaines qui se font confiance
- **Tree** = domaines partageant un namespace DNS contigu
- **Domain** = la vraie frontière de sécurité/administration
- **DC** = le serveur qui héberge et fait tourner AD
- **AD** = l'annuaire (base + services) qui stocke tout
- **OU** = organisation interne + point d'application des GPOs/délégation
- **Kerberos** = le protocole d'authentification par tickets
- **Authentication** = prouver son identité / **Authorization** = vérifier ses droits
- **Resource Management** = gestion centralisée des accès via groupes
- **GPO** = règles appliquées automatiquement par OU

---

# Partie 2 — Attaques Kerberos et techniques offensives avancées

## 2.1 Password spraying

```powershell
. .\DomainPasswordSpray.ps1
notepad DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -UserList .\users.txt -Password 123456 -Verbose
```

## 2.2 Golden Ticket — forger un TGT (niveau domaine entier)

**Ce qu'il faut voler** : le hash du compte `krbtgt` — le compte qui signe tous les TGT du domaine (répliqué sur tous les DCs).

**Principe** :
1. Compromission d'un DC (ou DCSync) → récupération du hash krbtgt.
2. Forge d'un TGT complet, sans jamais passer par l'AS — username, groupes (ex. "Domain Admins"), durée de validité choisis librement.
3. Ce faux TGT est signé avec le vrai hash krbtgt → le DC lui fait confiance sans vérification supplémentaire.
4. Présentation au TGS → Service Tickets pour n'importe quel service du domaine.

**Pourquoi c'est catastrophique** : fonctionne pour n'importe quel service, usurpation de n'importe quel utilisateur (même inexistant), reste valable même après changement du mot de passe usurpé (le hash krbtgt change rarement). Outil : Mimikatz → `kerberos::golden`.

## 2.3 Silver Ticket — forger un Service Ticket (niveau un seul service)

**Ce qu'il faut voler** : le hash du compte de service (compte machine/service associé à un serveur précis, ex. `SERVEUR-FICHIERS$`).

**Principe** :
1. Compromission d'un serveur → récupération du hash du compte de service local.
2. Forge directe d'un Service Ticket pour ce service précis — jamais besoin de passer par l'AS ni le TGS (pas de TGT du tout).
3. Le ticket, signé avec le hash du compte de service, est accepté par ce service.
4. Accès au service ciblé en se faisant passer pour n'importe quel user.

**Note** : le KDC connaît le hash de TOUS les comptes du domaine, pas seulement KRBTGT.

## 2.4 Le protocole des Service Tickets et le Kerberoasting

Un utilisateur déjà authentifié (TGT en poche) qui veut accéder à une ressource (base SQL, partage exposé via un compte de service) envoie une **TGS-REQ** au TGS, avec son TGT + un authenticator chiffré avec sa clé de session + le SPN du service visé. Le TGS déchiffre le TGT (via le hash krbtgt), en extrait la clé de session, valide l'authenticator, puis génère un **Service Ticket (ST)** contenant l'identité de l'utilisateur, son PAC, et une nouvelle clé de session client-serveur.

**Le point essentiel** : ce ST est chiffré non pas avec le hash krbtgt, mais avec le **hash du mot de passe du compte associé au SPN demandé** (souvent un compte de service, parfois un compte utilisateur standard configuré avec un SPN). Le TGS renvoie le ST au client, qui le transmet directement au serveur cible (sans repasser par le DC) — c'est ce serveur, connaissant son propre hash, qui déchiffre le ST et décide de l'accès.

**Le problème** : le KDC délivre ce ST à **n'importe quel utilisateur authentifié** qui le demande pour un SPN donné, sans droits particuliers sur ce service — il suffit d'avoir un TGT valide. Un attaquant peut donc énumérer les comptes à SPN, demander un TGS pour chacun, récupérer le ST correspondant, puis **extraire hors ligne** la partie chiffrée du ticket et tenter de la casser (bruteforce/dictionnaire), sans générer de trafic suspect supplémentaire vers le DC après la récupération initiale. Les comptes de service ayant historiquement des mots de passe rarement changés, souvent faibles, le cassage peut réussir — et si ce compte a des privilèges élevés (fréquent), la compromission s'étend largement.

**Différence avec l'AS-REP Roasting** : le Kerberoasting n'exploite pas une mauvaise configuration comme l'absence de préauthentification — il exploite un comportement **normal et attendu** du protocole (délivrance de ST chiffrés avec le hash du compte de service à toute personne authentifiée), combiné à la faiblesse fréquente des mots de passe de service. **Mitigation principale** : mots de passe longs/aléatoires pour les comptes de service, ou **gMSA** (Group Managed Service Accounts, mot de passe géré et changé automatiquement par AD) rendant le cassage offline quasi infaisable.

**Prérequis pratiques** : un compte valide (peu privilégié) dans le domaine — légitime ou compromis (phishing, credentials leakés). Une fois connecté, TGT obtenu automatiquement, authenticator généré via la clé de session. Il ne manque que le SPN de la cible, récupérable par énumération (l'attribut `servicePrincipalName` est lisible par tout utilisateur authentifié) via `GetUserSPNs.py` (Impacket) ou `Get-DomainUser -SPN` (PowerView). Cibles typiques : `MSSQLSvc/sqlserver01.corp.local:1433`, `HTTP/webapp01.corp.local`.

## 2.5 Authentification Kerberos et AS-REP Roasting

**Historique** : dans les premières versions d'AD, il n'existait pas de préauthentification — un utilisateur envoyait juste son nom au KDC, sans preuve préalable. Le KDC générait alors un TGT (identité + PAC + clé de session, chiffré avec le hash krbtgt) et renvoyait aussi la clé de session, chiffrée avec le hash du mot de passe de l'utilisateur lui-même.

**Le problème d'origine** : un attaquant pouvait se faire passer pour n'importe quel utilisateur, sans rien prouver, recevoir cette clé de session chiffrée avec le hash de la cible, et tenter de la casser hors ligne — succès = mot de passe découvert.

**La correction** : introduction de la **préauthentification**. Le client doit prouver qu'il connaît le hash de son mot de passe *avant* toute réponse — il chiffre un horodatage courant avec ce hash et l'envoie dans sa requête initiale. Le KDC (qui connaît ce hash) tente de déchiffrer ; si ça marche et que l'horodatage est récent, il en déduit un client légitime, et alors seulement génère et renvoie le TGT + clé de session.

Cette clé de session sert ensuite pour chaque demande de TGS : le client envoie son TGT + un message chiffré avec cette clé (un **authenticator**) ; le KDC déchiffre le TGT via le hash krbtgt, en extrait la clé de session, puis l'utilise pour vérifier l'authenticator.

**L'attaque AS-REP Roasting** exploite précisément l'**absence de cette préauthentification** : si elle est désactivée sur un compte, l'attaquant peut envoyer une requête en se faisant passer pour la cible, récupérer la clé de session chiffrée avec son hash, et la casser hors ligne. Si la préauthentification est activée, l'attaque devient impossible.

**Différence fondamentale avec le Kerberoasting** : ici, l'attaquant n'a même pas besoin d'un compte domaine valide — connaître (ou deviner) un nom d'utilisateur vulnérable suffit. Souvent la toute première attaque tentée en pentest AD, avant même d'avoir des creds.

## 2.6 Pass-the-Ticket (PtT)

Contrairement au cassage de hash, le PtT consiste à **voler un ticket Kerberos déjà valide** sur une machine compromise et à le réutiliser tel quel, sans jamais connaître mot de passe ni hash.

**Principe** :
1. Compromission d'une machine où un user (ex. un admin) est/était connecté.
2. Windows garde les tickets Kerberos en mémoire (LSASS) tant que la session existe.
3. Extraction de ces tickets (Mimikatz, Rubeus).
4. Injection du ticket volé dans sa propre session → l'attaquant devient littéralement cet utilisateur.

```bash
# Extraction
mimikatz # sekurlsa::tickets /export
# Injection (impersonation)
mimikatz # kerberos::ptt ticket.kirbi
```

C'est exactement le principe SSO détourné : un ticket volé sert de "preuve d'identité réutilisable" — mais réutilisée par l'attaquant, pas le vrai propriétaire.

## 2.7 DCSync — extraire des hashs via la réplication AD

Permet d'extraire les hashs de mots de passe de n'importe quel compte AD (y compris krbtgt) sans accès direct au DC lui-même, en abusant d'une fonctionnalité légitime de réplication.

**Principe** : les DCs se répliquent entre eux les données AD via le protocole **MS-DRSR**. DCSync consiste à se faire passer pour un DC et demander cette réplication à un vrai DC, qui répond normalement — la requête paraît légitime de son point de vue.

```
lsadump::dcsync /domain:security.local /user:krbtgt
```
Le DC renvoie le hash NTLM du compte demandé.

**Privilèges nécessaires** : pas un accès admin classique, mais deux droits étendus (extended rights) sur l'objet domaine — `DS-Replication-Get-Changes` et `DS-Replication-Get-Changes-All`. Normalement réservés aux Domain Admins/Enterprise Admins/comptes DC, mais souvent trouvés par erreur sur d'autres comptes (mauvaise délégation, groupe imbriqué mal audité) — exactement ce que BloodHound repère via les ACLs.

**Pourquoi c'est critique** : extraire le hash krbtgt permet de forger des Golden Tickets (accès domaine illimité, persistant, quasi indétectable) ; aucun besoin de RCE ni de dump LSASS sur le DC ; beaucoup plus discret qu'un accès interactif au DC.

**Dans BloodHound** : chercher les nœuds avec les edges `GetChanges` et `GetChangesAll` vers le domaine — la combinaison des deux (pas un seul) est nécessaire pour un DCSync complet.

**Chemin d'attaque typique** :
```
Compte compromis → membre d'un groupe → groupe a droits GetChanges + GetChangesAll → DCSync → krbtgt hash → Golden Ticket
```
Souvent la dernière étape d'un chemin d'attaque AD, le "game over" du domaine — quasi invisible en audit manuel, mais que BloodHound rend visible en quelques clics.

## 2.8 High value groups dans BloodHound

Un objet marqué "high value" est critique parce qu'il donne (directement ou indirectement) un chemin vers le contrôle du domaine — BloodHound les tague d'une étoile et les colore différemment.

**Groupes marqués high value par défaut** : Domain Admins, Enterprise Admins, Administrators, Schema Admins, Account Operators, Backup Operators, Print Operators, Server Operators, DnsAdmins, etc.

**Pourquoi Account Operators est high value** : ce groupe peut créer/modifier/supprimer la plupart des comptes et groupes (sauf ceux des OU protégées comme Domain Admins) — un membre peut donc ajouter un compte qu'il contrôle dans un groupe privilégié, réinitialiser des mots de passe, modifier des appartenances de groupes. Résultat : escalade indirecte quasi équivalente à être admin du domaine.

## 2.9 Rubeus — le couteau suisse des attaques Kerberos

Créé par l'équipe SpecterOps (les mêmes que BloodHound/SharpHound), regroupe dans un seul binaire tout ce qui touche à l'abus du protocole Kerberos.

| Fonction | Commande |
|---|---|
| Kerberoasting | `Rubeus.exe kerberoast` |
| AS-REP Roasting | `Rubeus.exe asreproast` |
| Pass-the-Ticket | `Rubeus.exe ptt` |
| Overpass-the-Hash | `Rubeus.exe asktgt` |
| Golden/Silver Ticket | `Rubeus.exe golden` / `Rubeus.exe silver` |
| Extraction de tickets en mémoire | `Rubeus.exe dump` |
| Renouvellement de TGT | `Rubeus.exe renew` |
| S4U abuse (delegation) | `Rubeus.exe s4u` |

---

# Partie 3 — Outils et notions spécifiques au projet

## 3.1 Vagrant — décrire des VMs en code

Analogue à un `docker-compose.yml` pour des VMs. **Problème résolu** : monter un lab AD à la main (cliquer dans VMware, insérer chaque ISO, suivre chaque assistant) prend une journée pour 4-5 VM, et il faut tout refaire en cas de casse. Vagrant lit un fichier texte (`Vagrantfile`) décrivant les VM voulues, et les crée automatiquement via l'hyperviseur (VMware/VirtualBox).

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "kingslanding" do |dc|
    dc.vm.box = "windows_server_2019"
    dc.vm.hostname = "kingslanding"
    dc.vm.network "private_network", ip: "192.168.56.10"
  end
end
```
`vagrant up` → la VM existe, démarrée, avec le bon hostname et la bonne IP, sans un clic.

**Le vrai gain** : la reproductibilité — `vagrant destroy && vagrant up` recrée un lab identique et propre en quelques minutes après avoir cassé un DC en testant une attaque.

## 3.2 Ansible — configurer les machines automatiquement

Analogue à un `Dockerfile` pour ce qui est *dans* les containers. **Problème résolu** : Vagrant donne des VM vides ; il faut ensuite promouvoir en DC, créer des OUs, créer 50 comptes avec mots de passe différents, configurer des ACLs volontairement mal foutues, installer ADCS avec un template vulnérable — fait à la main, ça prend des jours et n'est jamais reproductible.

Ansible exécute une liste d'instructions (un **playbook**, en YAML) sur une ou plusieurs machines à distance (WinRM pour Windows, SSH pour Linux).

```yaml
- name: Créer l'utilisateur Arya Stark
  win_domain_user:
    name: arya.stark
    password: "P@ssw0rd123"
    state: present
    groups: ["Domain Users"]

- name: Rendre le template de certificat vulnérable (ESC1)
  win_certificate_template:
    name: "VulnTemplate"
    enrollee_supplies_subject: yes
```

| | Vagrant | Ansible |
|---|---|---|
| Rôle | Crée la VM (OS installé, réseau) | Configure ce qu'il y a **dans** la VM (comptes, rôles, permissions) |
| Analogie | Construire la maison | Meubler et câbler l'intérieur |

Les deux travaillent souvent ensemble : Vagrant crée les VM, puis appelle Ansible comme "provisioner".

## 3.3 GOAD — le lab AD prêt à l'emploi

Projet open-source ayant déjà écrit le Vagrantfile et les playbooks Ansible pour construire un environnement AD réaliste et volontairement vulnérable.

**Topologie réelle (thème Game of Thrones)** :
```
Forest : sevenkingdoms.local
 ├── Domain: sevenkingdoms.local (racine)
 │    └── DC: kingslanding
 ├── Domain: north.sevenkingdoms.local (enfant)
 │    ├── DC: winterfell
 │    └── Serveur membre: castelblack
 └── Domain: essos.local (forêt séparée, avec un trust)
      └── DC: meereen
```

Fournit prêt à l'emploi : plusieurs domaines dans une même forêt, un trust inter-forêts (pivoting), des ACLs mal configurées à trouver via BloodHound, des templates ADCS vulnérables.

**GOAD-Light** : version allégée, seulement `sevenkingdoms.local` + `north.sevenkingdoms.local` (2 domaines, 3 machines), pour une machine avec moins de RAM (~20 Go suffisent).

## 3.4 BloodHound et Neo4j

BloodHound ne stocke pas ses données dans une base classique (SQL) mais dans **Neo4j**, une base de données **en graphe** : chaque utilisateur/groupe/ordinateur est un **nœud**, chaque relation (« est membre de », « a GenericAll sur », « a une session ouverte sur ») est une **arête (edge)**.

```
(Arya Stark) --[MemberOf]--> (IT-Team)
(IT-Team)    --[GenericAll]--> (Domain Admins)
```

Ce format en graphe permet de répondre à des questions comme *"quel est le chemin le plus court entre Arya Stark et Domain Admins ?"* — lourd en SQL, naturel en graphe.

## 3.5 Les requêtes Cypher

Cypher est le **langage de requête** de Neo4j — l'équivalent de SQL, pensé pour naviguer dans un graphe. L'interface BloodHound ("shortest path") lance en réalité une requête Cypher en coulisses ; savoir l'écrire soi-même permet d'aller chercher ce que l'interface ne propose pas par défaut.

**Exemple — ce que fait l'interface pour "chemin le plus court vers Domain Admins" :**
```cypher
MATCH p = shortestPath(
  (u:User {name:"ARYA.STARK@SEVENKINGDOMS.LOCAL"})-[*1..]->(g:Group {name:"DOMAIN ADMINS@SEVENKINGDOMS.LOCAL"})
)
RETURN p
```
*"Trouve le chemin le plus court entre Arya Stark et Domain Admins, en suivant n'importe quelle relation."*

**Exemple — requête personnalisée, non proposée par l'interface :**
```cypher
// Comptes kerberoastables (SPN défini) qui sont aussi Domain Admin
MATCH (u:User {hasspn:true})-[:MemberOf]->(g:Group {name:"DOMAIN ADMINS@SEVENKINGDOMS.LOCAL"})
RETURN u.name
```
Ce type de requête sert directement de "tool" pour l'agent IA du projet (`list_kerberoastable_users()` = littéralement cette requête, exécutée automatiquement en Python).

## 3.6 AD Miner — l'audit automatique par-dessus BloodHound

**Problème résolu** : écrire soi-même une requête Cypher pour chaque question type ("comptes avec mot de passe qui n'expire jamais ?", "délégations non contraintes ?", "templates ADCS vulnérables ?") prend du temps.

AD Miner a déjà écrit des dizaines de requêtes Cypher standards (les questions qu'un auditeur AD pose systématiquement), les lance automatiquement sur la base Neo4j remplie par BloodHound, et sort un rapport HTML classé par risque.

**Analogie** : BloodHound + Cypher = poser des questions une par une, à la main. AD Miner = un questionnaire d'audit complet passé automatiquement, résultats en rapport.

**Usage type** : lancé en tout début de mission (module Recon), donne une vue d'ensemble rapide des faiblesses avant de creuser à la main les chemins les plus intéressants.

## 3.7 Le modèle Tier0 / Tier1 / Tier2

**Problème résolu** : tous les DCs d'un même domaine partagent le hash krbtgt (vu en 1.5) ; le modèle de Tiering formalise une règle pour éviter qu'un compte très privilégié se fasse voler en se connectant sur une machine moins protégée.

**Principe** : classer chaque compte/machine par niveau de criticité, interdire qu'un compte d'un niveau élevé se connecte sur une machine d'un niveau inférieur.

```
Tier 0  → contrôle TOUT le domaine
          Ex: Domain Admins, les DCs eux-mêmes, la CA (ADCS), le serveur Azure AD Connect
Tier 1  → contrôle des serveurs/applications
          Ex: administrateurs de serveurs SQL, serveurs de fichiers
Tier 2  → postes de travail utilisateurs
          Ex: comptes employés standards, leurs PC
```

**Exemple du problème que Tier0 empêche** :
```
Un admin Domain Admin (Tier0) se connecte en RDP sur PC-Comptabilité (Tier2)
pour dépanner un problème d'imprimante.
→ Son ticket Kerberos / ses identifiants restent en mémoire sur ce PC (LSASS).
→ Un attaquant ayant déjà compromis PC-Comptabilité peut dumper ces identifiants (Mimikatz).
→ L'attaquant a maintenant des creds Domain Admin, juste parce que
  l'admin a "descendu" son compte Tier0 sur une machine Tier2.
```

**La règle Tier0** : un compte Tier0 ne doit jamais se logger sur une machine Tier1/Tier2 — il utilise un compte séparé pour ces tâches, et une station d'administration dédiée pour ses tâches Tier0.

**Point central pour ce projet** : le serveur hébergeant Azure AD/Microsoft Entra Connect est un asset **Tier0**, même s'il "n'a l'air que d'un serveur de synchro" — son compte de synchro a souvent des droits **DCSync**, le compromettre équivaut à compromettre un DC. C'est le pivot on-prem ↔ cloud du module Entra ID.

## 3.8 ADCS — quand un certificat remplace un mot de passe

Kerberos classique authentifie via mot de passe (hash NTLM dérivé). **PKINIT** est une méthode alternative utilisant un **certificat numérique** — utile pour carte à puce, comptes de service, authentification machine.

**ADCS (Active Directory Certificate Services)** : le rôle Windows Server jouant le rôle de **CA (Certificate Authority)** interne, qui émet ces certificats.

**Précision importante** : le certificat ne contient PAS les permissions — c'est juste une carte d'identité ("je certifie que cette clé publique appartient à Saad Agoumi / UPN saad@sevenkingdoms.local"), signée par la CA.

**Pourquoi c'est une surface d'attaque énorme** : un certificat, une fois émis, prouve l'identité tout seul — pas besoin de mot de passe ni hash. Se faire délivrer un certificat au nom de quelqu'un d'autre (template mal configuré) permet de s'authentifier comme cette personne, même si son mot de passe change ensuite (le certificat reste valide jusqu'à expiration/révocation).

```
Authentification classique : User → AS-REQ (hash du mdp) → AS-REP → TGT
Authentification PKINIT :    User → AS-REQ (certificat) → AS-REP → TGT
```
Le TGT final est **identique** dans les deux cas — seule la méthode de preuve change. D'où un certificat volé/abusé donne accès à exactement les mêmes privilèges qu'un mot de passe volé.

Le serveur où ADCS est installé/configuré **devient** la CA.

### PKINIT en détail — token PKI et déroulement

**Ce que contient le token** : un certificat (identité + clé publique, signé par la CA), et une clé privée générée dans la puce, qui ne sort jamais du matériel.

**Déroulement** :
```
1. Token branché → Windows génère un horodatage frais à signer (anti-rejeu).
2. La puce signe cet horodatage avec la clé privée (opération interne, clé privée jamais exposée) → une SIGNATURE.
3. Le PC envoie au KDC : le certificat + la signature.
```

**Propriété mathématique** : avec signature + clé publique, on peut *vérifier* la signature. Sans la clé privée, impossible de la produire soi-même. (Précision technique : dans les schémas modernes ECDSA/RSA-PSS, le vérificateur recalcule le hash du message et vérifie la correspondance — même principe de sécurité, mécanique légèrement différente de l'intuition RSA "textbook".)

**Ce que fait le KDC** :
```
1. Vérifie que le certificat est signé par une CA du NTAuth Store (magasin de confiance du domaine).
2. Extrait identité (UPN/SAN) et clé publique du certificat.
3. Vérifie la signature contre l'horodatage attendu (frais, récent).
4. Conclut à l'identité, construit le PAC (en interrogeant AD pour les groupes/droits réels —
   le certificat ne contient PAS les permissions), émet le TGT.
```

**Ce qu'un attaquant qui intercepte le trafic peut extraire** :

| Élément intercepté | Sensible ? | Pourquoi |
|---|---|---|
| Le certificat | Non | Identité + clé publique, public par nature |
| La clé publique | Non | Faite pour être partagée |
| La signature seule | Non | Inerte sans la clé publique, valable pour cet horodatage précis uniquement |

**Conclusion** : intercepter tout le trafic PKINIT ne donne aucun élément réutilisable pour usurper l'identité — contrairement à Kerberos classique où le secret (hash) circule et est stocké des deux côtés.

**Le certificat peut-il être falsifié ?** Non, pas en forgeant la cryptographie — le KDC fait confiance à une liste fermée de CA (le NTAuth Store), pas à "un certificat qui a l'air valide". Un certificat auto-signé ou signé par une CA externe non reconnue est rejeté immédiatement (même logique qu'un navigateur qui rejette un certificat HTTPS auto-signé). **Conséquence** : la seule façon de faire accepter un faux certificat est de le faire signer par la vraie CA de confiance — exactement ce que ciblent ESC1/ESC16 : tromper la CA légitime pour qu'elle signe un certificat qu'elle n'aurait pas dû signer, sans jamais casser la cryptographie elle-même.

### Golden Certificate

Le pire scénario possible, parallèle direct avec le Golden Ticket :

| | Golden Ticket | Golden Certificate |
|---|---|---|
| Ce qui est volé | Le hash du compte krbtgt | La clé privée de la CA elle-même |
| Ce que ça permet | Forger un TGT valide pour n'importe qui | Signer un certificat valide pour n'importe quelle identité |
| Pourquoi le domaine fait confiance | Le KDC fait confiance à tout ticket signé krbtgt | Le KDC fait confiance à tout certificat signé par la CA |
| Durée de compromission | Jusqu'au changement du mot de passe krbtgt (souvent négligé) | Jusqu'à révocation/remplacement de la CA (opération lourde) |

**Mécanisme** : compromission du serveur ADCS + extraction de la clé privée de la CA (Mimikatz, vol du .pfx) → l'attaquant signe lui-même n'importe quel certificat, pour n'importe qui, parfaitement valide aux yeux du KDC.

**Pourquoi c'est pire qu'ESC1-16** : ESC1-16 exploitent une mauvaise configuration corrigible (template, extension de sécurité) ; le Golden Certificate exploite le vol direct de la racine de confiance — remédiation = révoquer/régénérer toute la CA, cassant potentiellement tous les certificats déjà émis. Le serveur ADCS est donc un asset **Tier0**, au même titre qu'un DC.

**Autres risques (hors cryptographie)** :

| Risque | Nature | Description |
|---|---|---|
| Golden Certificate | Vol de la racine de confiance | Le plus grave |
| Révocation mal vérifiée | Défaut opérationnel | Un certificat révoqué (CRL) continue de fonctionner si le KDC ne vérifie pas correctement |
| Vol physique + PIN faible | Facteur humain | Token volé + PIN deviné/sans limite de tentatives |
| ESC1 → ESC16 | Abus de la CA légitime | Config permissive du template ou de la CA |

## 3.9 ESC1

**ESC** = Escalation Sub-CA/Certificate — nomenclature des différentes façons d'abuser ADCS.

**ESC1** : un template mal configuré permet à n'importe quel utilisateur à faible privilège de (1) demander un certificat via ce template, (2) **choisir lui-même** l'identité inscrite dans le certificat (champ SAN), et (3) ce certificat est valide pour l'authentification client (PKINIT).

```
Arya Stark (aucun privilège) demande un certificat via "VulnTemplate", en précisant :
"ce certificat, c'est pour administrator@sevenkingdoms.local"
→ Le template accepte (enrollee_supplies_subject = yes, mal configuré)
→ La CA émet le certificat, valide, au nom d'Administrator
→ Arya s'authentifie via PKINIT → TGT au nom d'Administrator, Domain Admin
```
Jamais besoin du mot de passe d'Administrator — le certificat seul suffit.

## 3.10 ESC16

**Différence avec ESC1** : ESC1 est une erreur sur un template précis ; ESC16 est une erreur globale sur la CA elle-même — plus discrète et large, même sur des templates bien configurés.

**Mécanisme** : chaque certificat émis contient normalement une "extension de sécurité" liant fermement le certificat au **SID** du compte pour qui il a été émis. ESC16 = cette extension **désactivée globalement** sur la CA (mauvais réglage registre côté serveur CA).

```
1. L'attaquant a un droit d'écriture sur l'attribut UPN d'un compte qu'il contrôle déjà.
2. Il modifie temporairement son UPN pour pointer vers "administrator@sevenkingdoms.local".
3. Il demande un certificat (template légitime) — l'extension de sécurité étant désactivée
   sur la CA, aucun garde-fou pour vérifier la cohérence UPN ↔ SID réel.
4. Il remet son UPN d'origine (discrétion).
5. Il obtient un certificat parfaitement valide, l'authentifiant comme Administrator,
   sans jamais avoir touché au compte Administrator lui-même.
```

**Point commun avec BadSuccessor** : le même pattern — au lieu de compromettre directement le compte cible, l'attaquant manipule un **attribut de liaison/référence** pour se faire passer pour lui aux yeux du système d'authentification.

## 3.11 Certipy

L'outil de référence (Python, utilisable depuis Kali/Linux) pour trouver et exploiter les templates/CA vulnérables.

```bash
# Reconnaissance — templates vulnérables
certipy find -u 'arya.stark@sevenkingdoms.local' -p 'P@ssw0rd123' -dc-ip 192.168.56.10 -vulnerable

# Exploitation ESC1 — certificat au nom d'Administrator
certipy req -u 'arya.stark@sevenkingdoms.local' -p 'P@ssw0rd123' \
  -ca 'sevenkingdoms-CA' -template 'VulnTemplate' \
  -upn 'administrator@sevenkingdoms.local'

# Authentification avec le certificat obtenu → TGT + hash NTLM
certipy auth -pfx administrator.pfx -dc-ip 192.168.56.10
```
Fait le pont complet, de "je soupçonne un problème ADCS" jusqu'à "j'ai un TGT valide de Domain Admin".

## 3.12 Managed Service Accounts et dMSA

**Problème historique** : un compte de service a besoin d'un mot de passe, comme un compte utilisateur — mais ces mots de passe sont souvent fixés une fois pour toutes (personne ne veut casser un service en production en le changeant), ce qui en fait une cible de choix pour du Kerberoasting à long terme.

**gMSA (2012)** : Windows gère et change automatiquement le mot de passe tous les 30 jours, sans intervention humaine.

**dMSA (delegated Managed Service Account, nouveau Windows Server 2025)** : pensé pour migrer en douceur un vieux compte de service (mot de passe statique) vers ce système géré, sans casser le service pendant la transition.

**Mécanisme de migration légitime** :
```
Ancien compte : svc-sql (mot de passe statique depuis 5 ans)
1. L'admin crée un nouveau dMSA : svc-sql-dmsa
2. Il "lie" ce dMSA à l'ancien compte via l'attribut msDS-ManagedAccountPrecededByLink → pointe vers svc-sql
3. Les services basculent progressivement vers svc-sql-dmsa, qui HÉRITE des mêmes droits que svc-sql
4. Migration terminée, svc-sql-dmsa a tout hérité, l'ancien compte peut être désactivé
```
Un dMSA peut, **par design**, hériter des droits d'un autre compte via cet attribut de liaison — fonctionnalité légitime, exactement ce que BadSuccessor détourne.

## 3.13 BadSuccessor

**L'idée** : au lieu de suivre le processus légitime de migration, un attaquant crée lui-même un dMSA et ment sur les attributs de liaison, pour se faire passer pour n'importe quel compte du domaine — y compris Domain Admin — sans jamais toucher au compte cible.

**Prérequis (plus faible qu'on ne croit)** : juste le droit `CreateChild` (ou `msDS-DelegatedManagedServiceAccount`) sur **une seule OU**. Beaucoup de comptes non-admin ont ce genre de droit délégué sans s'en rendre compte.

**Étapes de l'attaque** :
```
1. Arya Stark a le droit CreateChild sur OU=ServiceAccounts.
2. Elle crée un nouvel objet dMSA : "fake-dmsa" dans cette OU.
3. Elle modifie deux attributs sur son propre objet fake-dmsa (elle en est propriétaire) :
   - msDS-ManagedAccountPrecededByLink = DN de "Administrator" ("je prétends succéder à Administrator")
   - msDS-DelegatedMSAState = 2 ("la migration est terminée" — mensonge)
4. Elle demande un TGT pour fake-dmsa (AS-REQ).
5. Le KDC lit msDS-ManagedAccountPrecededByLink, voit qu'il pointe vers Administrator, et
   — avant le patch de 2025 — fait confiance sans vérifier une vraie migration.
   Il émet un TGT avec le PAC d'Administrator.
6. Arya a un TGT valide de Domain Admin, sans mot de passe, sans hash, sans jamais
   avoir interagi avec le vrai compte Administrator.
```

**Pourquoi c'est si fort** : contrairement à Kerberoasting/Golden Ticket qui attaquent quelque chose lié directement au compte ciblé (son hash, ou le hash krbtgt), ici le compte Administrator n'est **jamais touché** — l'attaquant crée un objet entièrement nouveau, sous son propre contrôle, et c'est la confiance mal placée du KDC dans un attribut de liaison qui fait tout le travail.

**Le patch (CVE-2025-53779)** : Microsoft a imposé une **validation par lien mutuel** — le KDC exige que le compte cible référence *lui aussi* le dMSA en retour, preuve d'une vraie migration bidirectionnelle. Sur un DC patché, l'étape 5 échoue. D'où le lab monte volontairement un DC Windows Server 2025 non patché — pour rejouer l'attaque originale, puis démontrer que le patch la bloque (triptyque attaque → patch → détection).

## 3.14 DEF CON 2025 — origine de la découverte

DEF CON : la plus grande conférence de hacking au monde, chaque année en août à Las Vegas — où les chercheurs présentent leurs découvertes les plus marquantes de l'année. Équivalent, en beaucoup plus gros, des BSides visées plus tard pour présenter son propre projet.

**Lien avec BadSuccessor** : vulnérabilité publiée d'abord par un chercheur Akamai (Yuval Gordon) en mai 2025, puis présentée à DEF CON 2025 (août 2025) — l'exposition massive a mis la pression sur Microsoft pour sortir un correctif en moins d'une semaine, délai très rapide pour ce type de faille structurelle.

**Intérêt pour le projet** : citer "présenté à DEF CON 2025" dans le writeup n'est pas de la décoration — ça prouve en une ligne un suivi de l'actualité de recherche offensive en temps réel, pas juste des cours de certification datés.

## 3.15 Entra ID — AD dans le cloud

**Problème résolu** : l'AD classique authentifie pour des ressources internes au réseau (partages, PC du domaine). Une entreprise utilise aussi des services cloud (Office 365, Teams, SaaS) — Kerberos ne fonctionne pas nativement sur Internet pour ce genre de service.

**Entra ID** (anciennement Azure AD, même service renommé) : équivalent d'AD hébergé par Microsoft dans le cloud, avec son propre annuaire, parlant des protocoles web modernes plutôt que Kerberos.

| Monde AD classique | Monde Entra ID (équivalent cloud) |
|---|---|
| Forest / Domain | **Tenant** |
| Domain Controller (DC) | Pas de serveur physique — service 100% géré par Microsoft |
| Kerberos (AS-REQ/AS-REP, TGT) | **OAuth2 / OIDC** (tokens) |
| GPO | **Conditional Access** (règles évaluées à chaque connexion) |
| Groupes de sécurité + ACLs | **App Registrations** + permissions Graph API |
| Compte machine joint au domaine | **Device (hybrid/Entra joined)** |

```
AD classique : Arya se connecte le matin → Kerberos AS-REQ → TGT → accès \\SERVEUR\Projets
Entra ID     : Arya ouvre Teams → authentification OAuth2 → access token → accès accordé
```

**En pentest**, les classes d'attaque changent de nature : voler un token plutôt qu'un hash NTLM (device code phishing, vol de PRT) ; contourner du Conditional Access plutôt qu'une GPO.

## 3.16 Azure AD Connect / Microsoft Entra Connect

**Problème résolu** : la plupart des entreprises ne veulent pas gérer deux annuaires séparés (AD local + Entra ID cloud avec des comptes différents) — un employé devrait changer son mot de passe deux fois sinon.

**Fonctionnement** : logiciel installé sur un serveur du réseau local (souvent un serveur membre, pas forcément un DC) qui synchronise en continu les comptes de l'AD local vers le tenant Entra ID (utilisateurs, groupes, mots de passe re-hashés, pas en clair) — un même utilisateur a UN SEUL compte, utilisable des deux côtés.

```
1. Lit l'AD local via LDAP
2. Pousse les objets vers Entra ID via l'API Microsoft Graph (HTTPS)
3. Tourne toutes les 30 min par défaut (synchro delta)
4. Résultat : un seul mot de passe, valable AD local (Kerberos) ET Entra ID (OAuth2) → "hybrid join"
```

**Point le plus sensible du projet (rappel Tier0)** : Azure AD Connect utilise un compte de service créé automatiquement (souvent nommé `MSOL_xxxxx`), qui a très souvent des droits **DCSync** (répliquer toutes les données AD, y compris hashes de mots de passe — la même primitive que la réplication native entre DCs).

**Pivot concret** :
```
Attaquant compromet le serveur Azure AD Connect (souvent moins surveillé, "juste un serveur de synchro")
  ↓
Récupère les identifiants du compte de synchro (souvent stockés en clair/déchiffrables
localement sur ce serveur, par conception)
  ↓
Avec ces identifiants + droits DCSync → extrait TOUS les hashes du domaine on-prem
  ↓
Compromission complète du domaine on-prem, en partant d'un serveur "cloud"
```
C'est la démo "pivot on-prem ↔ cloud" du module Entra ID — le serveur Azure AD Connect doit être traité comme un asset Tier0, au même niveau qu'un DC.

## 3.17 Cloud Kerberos Trust Attack — pivot Entra ID → AD on-prem

> Recherche de Dirk-jan Mollema (auteur de ROADtools), publiée juillet 2026 — plus récente que BadSuccessor.

**Contexte légitime** : **Windows Hello for Business**, mode "Cloud Kerberos Trust", permet à un appareil hybrid-joined d'obtenir un ticket Kerberos on-prem sans ligne de vue réseau directe vers un DC — pratique en télétravail.

**Ce que l'activation installe** (`Set-AzureADKerberosServer`) :
```
AzureADKerberos$   → compte machine se comportant comme un RODC (Read-Only DC)
krbtgt_AzureAD     → compte détenant une clé de signature PARTAGÉE entre Entra ID et l'AD on-prem
```

**Le point clé — inversion du sens de confiance** :
```
AAD Connect (connu) :          AD on-prem --[source de vérité]--> Entra ID (copie synchronisée)
Cloud Kerberos Trust (nouveau) : Entra ID --["je certifie cette identité"]--> AD on-prem (fait confiance)
```
La plupart des défenseurs pensent la confiance hybride à sens unique (cloud = copie, jamais source) ; cette fonctionnalité crée un second pont, dans l'autre sens.

**Mécanisme de l'attaque** — prérequis : être (ou compromettre) un **Global Admin** côté Entra ID.
```
1. Le Global Admin réécrit, via l'API de sync, le SID on-prem d'un utilisateur hybride qu'il
   contrôle, pour qu'il corresponde au SID du compte de synchro MSOL_ (droits DCSync).
   → Même schéma que BadSuccessor : jamais de contact direct avec le compte cible,
     manipulation d'un ATTRIBUT DE LIAISON (le SID) pour usurper son identité.
2. Il demande à Entra ID un "Partial TGT" en se présentant comme ce SID usurpé.
   → Entra ID accepte (autorité cloud du Global Admin).
3. Ce Partial TGT est envoyé à un DC on-prem, qui le complète en TGT PLEINEMENT VALIDE
   — le DC fait confiance à ce qu'Entra ID certifie (pont Cloud Kerberos Trust).
4. L'attaquant a un vrai TGT pour MSOL_ (droits DCSync) → extraction de tous les hashes
   du domaine on-prem (secretsdump).
```

**Pourquoi le compte MSOL_ en particulier** : les comptes vraiment privilégiés (Domain Admins) sont généralement dans une liste de refus RODC (`AzureADKerberos$` ne peut pas émettre de tickets pour eux). Le compte de synchro MSOL_, très puissant (droits DCSync) mais rarement ajouté à cette liste, est l'angle mort exploité.

**Pourquoi ce n'est PAS patché (et ne le sera jamais)** : contrairement à BadSuccessor (corrigé en une semaine), Microsoft ne traite pas ceci comme un bug — leur position : "Global Admin" côté cloud est déjà considéré, par design, comme l'équivalent d'un Domain Admin sur l'AD on-prem lié. Ce n'est pas un abus d'un défaut technique, c'est le modèle de confiance fonctionnant comme prévu. **Conséquence pour le lab** : contrairement à BadSuccessor (qui exige un DC volontairement non patché), cette attaque fonctionne aujourd'hui, sur un environnement à jour — il n'y a rien à patcher.

| | AAD Connect | Cloud Kerberos Trust |
|---|---|---|
| Sens du pivot | on-prem → cloud | **cloud → on-prem** |
| Compromis en premier | Serveur AAD Connect (Tier0 caché) | Un compte **Global Admin** (rôle purement cloud) |
| Mécanisme | Vol des creds du compte de sync, déjà en clair localement | Réécriture d'un attribut SID pour usurper le compte de sync, via l'API cloud |
| Statut du correctif | N/A (bonne hygiène = séparer les rôles Tier0) | **Non patché, ne le sera jamais** (comportement voulu) |
| Révélation sur Tier0 | Le serveur AAD Connect est un asset Tier0 caché | **Le rôle Global Admin cloud est, lui aussi, un asset Tier0** |

**Point à retenir pour le modèle Tier0** : la liste d'assets Tier0 doit inclure, en plus du DC et du serveur ADCS/AAD Connect, **le rôle Global Admin lui-même côté Entra ID** — une compromission 100% cloud peut aboutir à une compromission complète on-prem, sans qu'aucune machine locale n'ait jamais été touchée directement.

**Outils** : `roadtx` (module de la suite **ROADtools**, même auteur que la recherche) intègre le tooling pour manipuler ce flow Partial TGT / Cloud Kerberos Trust — même outil que celui prévu pour ROADrecon, pas de nouvel outil à apprendre.

---

# Partie 4 — Détection, SIEM et supervision

## 4.1 SIEM et agents — vue générale

**SIEM** : un logiciel installé sur un serveur, jouant le rôle de serveur central qui récupère les logs de toutes les machines surveillées et, en fonction de règles, génère des alertes. Wazuh en est un exemple.

**Agent (agent SIEM)** : un petit logiciel installé sur chaque machine à surveiller. C'est cet agent qui envoie les logs vers le manager (le serveur central).

**Limite des agents "de base"** : par défaut, un agent SIEM ne récupère que les journaux système Windows natifs (Application, Security, System) — des logs relativement **pauvres** pour de la vraie détection d'attaque. Beaucoup d'événements utiles (ligne de commande complète d'un processus, accès mémoire à un autre processus, connexions réseau détaillées) ne sont tout simplement pas journalisés par ces trois journaux natifs.

**D'où le rôle de Sysmon** : un agent complémentaire, plus riche, installé sur la machine, qui génère des événements bien plus détaillés, en se basant sur un fichier de règles de configuration (comme celui d'**Olaf Hartong**) qui définit précisément *quoi* logger.

**Comment Wazuh récupère les logs Sysmon** : l'agent Wazuh va lire ("intercepter le canal de") l'agent Sysmon — concrètement, il lit le journal d'événements que Sysmon écrit localement (`Microsoft-Windows-Sysmon/Operational`) — puis transmet ces événements au manager. C'est ensuite dans le dashboard du manager que ces logs deviennent consultables et exploitables.

```
Agent Wazuh natif (seul)   → lit Application/Security/System → logs pauvres, peu détaillés
Sysmon (config Olaf Hartong) → génère des événements riches, orientés détection
Agent Wazuh                 → lit AUSSI le canal Sysmon en plus des 3 journaux natifs
                             → transmet le tout, chiffré, au manager
Manager Wazuh                → applique ses propres règles de détection sur tout ce qui arrive
                             → génère les alertes visibles dans le dashboard
```

**Point de clarification important** : Olaf Hartong n'a rien à voir avec le mécanisme "si tel événement, alors telle alerte". Olaf Hartong est seulement l'auteur du fichier de configuration Sysmon (`sysmonconfig.xml`), qui définit quoi Sysmon doit surveiller/générer comme événements. Le mécanisme "si X alors alerte" est un étage complètement différent : c'est le **moteur de règles du manager Wazuh**, côté serveur, qui décide si un événement reçu mérite une alerte. Sysmon génère les événements bruts ; Wazuh décide, via ses propres règles, si un événement mérite une alerte. Ce sont deux étages séparés, pas un seul mécanisme.

## 4.2 Les journaux d'événements Windows natifs

Windows, même sans rien installer de tiers, garde en permanence une trace de ce qui se passe, dans des journaux d'événements (Event Logs), consultables via l'Observateur d'événements (`eventvwr.msc`). Trois journaux intéressent particulièrement la sécurité :

- **Application** : événements liés aux logiciels installés (plantage, mise à jour...).
- **System** : événements liés à l'OS lui-même (service qui démarre/s'arrête, pilote qui charge...).
- **Security** : événements liés à la sécurité — le plus important en pentest/SOC (connexions, déconnexions, changements de mot de passe, création de comptes...).

Un événement est une entrée avec un numéro (**Event ID**) et des détails. Ex. : l'**Event ID 4624** dans Security = "connexion réussie", avec nom d'utilisateur, heure, machine source.

**Le problème** : ces journaux natifs sont assez pauvres en détail pour une vraie détection d'attaque. Ex. l'Event ID 4688 (création de processus) peut exister mais souvent sans la ligne de commande complète — on sait que `powershell.exe` a démarré, pas avec quels arguments, exactement ce qui différencie un usage normal d'une attaque.

## 4.3 Le fichier `ossec.conf`

Le fichier de configuration de l'agent Wazuh installé sur une machine (ex. `C:\Program Files (x86)\ossec-agent\ossec.conf` sur DC01). Il indique à l'agent où trouver le manager (adresse IP), et quels journaux lire et transmettre.

La partie concernée : les blocs `<localfile>`. Chacun dit "va lire ce journal précis". Par défaut, l'installeur en met déjà plusieurs :

```xml
<localfile>
  <location>Application</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>

<localfile>
  <location>System</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Dès l'installation de l'agent, sans rien configurer de plus, ces trois journaux sont déjà lus et transmis au manager — d'où des logons/logoffs déjà visibles dans le dashboard avant même d'ajouter Sysmon. Le problème : ces trois journaux natifs ne suffisent pas pour une vraie détection fine d'attaque AD, d'où l'ajout de Sysmon.

## 4.4 Sysmon — l'œil qui voit ce que les logs Windows normaux ne voient pas

**Problème résolu** : l'Observateur d'événements natif est trop pauvre pour de la détection sérieuse — il ne logue quasiment rien sur ce qui compte vraiment (processus lançant quel autre processus, connexions réseau sortantes, modifications de registre). Si `mimikatz.exe` tourne sur un DC, le journal Windows classique ne donne quasiment rien d'exploitable.

**Ce que fait Sysmon (Sysinternals, Microsoft, gratuit)** : s'installe comme un service Windows sur une machine, et logue en détail des événements que Windows n'enregistre pas nativement — création de processus (ligne de commande complète), connexions réseau, création/modification de fichiers, accès à LSASS (le processus contenant les hashs en mémoire — exactement ce que dump Mimikatz), modifications du registre, etc.

**"Agent Sysmon"** : dans le vocabulaire SOC, un agent est simplement le logiciel installé sur la machine surveillée elle-même, par opposition au serveur central qui collecte tout. "Déployer un agent Sysmon sur DC01" = installer le service Sysmon sur DC01, qui commence à générer ces logs détaillés localement, dans le journal `Microsoft-Windows-Sysmon/Operational`.

**Exemple concret** :
```
Sans Sysmon : Arya Stark lance Mimikatz sur DC01
  → Le journal Windows classique note vaguement "un processus a démarré"
     (Event ID 4688, si même activé — souvent pas assez détaillé)

Avec Sysmon installé sur DC01 :
  → Event ID 1 (Process Create) : mimikatz.exe lancé, ligne de commande complète,
     hash du binaire, parent process
  → Event ID 10 (ProcessAccess) : mimikatz.exe a ouvert un handle vers lsass.exe
     avec des droits d'accès mémoire — signature quasi certaine d'un dump de credentials
```
Ce deuxième événement (accès à LSASS) devient la preuve concrète d'attaque détectée — impossible sans Sysmon.

**Richesse concrète de ce que Sysmon capture, que les journaux natifs ne donnent pas ou mal :**

| Ce que Sysmon capture | Exemple concret |
|---|---|
| Création de processus, ligne de commande complète | Pas juste "powershell.exe a démarré" mais `powershell.exe -enc SGVsbG8...` (base64 caché — signature classique d'attaque) |
| Connexions réseau sortantes | Quel processus s'est connecté à quelle IP/port — exfiltration, C2 |
| Accès à un autre processus en mémoire | `mimikatz.exe` ouvrant un handle vers `lsass.exe` — vol de credentials |
| Création/modification de fichiers | Un `.ps1` créé dans un dossier temporaire suspect |
| Modifications du registre | Une clé de démarrage automatique modifiée (persistance) |
| Chargement de DLL | Une DLL chargée depuis un chemin inhabituel |

Sysmon écrit ces logs localement, mais ne les envoie nulle part tout seul — il faut un "transporteur" pour les faire remonter vers un serveur central : c'est exactement le rôle de l'agent Wazuh, qui lit le canal Sysmon et l'expédie.

## 4.5 Wazuh — le SIEM qui centralise, analyse et alerte

**Problème résolu** : même avec Sysmon loguant en détail sur plusieurs machines, ça ne sert à rien si ces logs restent éparpillés machine par machine. Un analyste a besoin d'une vue centralisée : tous les événements du parc, dans un seul endroit, avec des règles qui déclenchent automatiquement une alerte quand un pattern suspect apparaît.

**Ce qu'est Wazuh** : un SIEM (Security Information and Event Management) open-source, qui fait trois choses :
1. **Collecte** les logs de toutes les machines surveillées (agent sur chacune, qui lit — entre autres — le canal Sysmon local et l'envoie).
2. **Analyse** ces logs en les comparant à des règles de détection (ex. "un processus a ouvert LSASS avec tel niveau d'accès" → alerte "possible credential dumping").
3. **Affiche** tout dans un dashboard web centralisé, alertes classées par sévérité.

**Architecture manager/agent** :
```
DC01, DC02, SRV02, DC03 (chacun a un AGENT Wazuh installé)
        │ (chaque agent lit les logs Sysmon locaux + logs Windows, les envoie chiffrés)
        ▼
VM Wazuh dédiée — le MANAGER
   ├── Indexer  : stocke tous les événements reçus (base de données)
   ├── Manager  : applique les règles de détection sur les événements
   └── Dashboard: interface web où consulter les alertes
```

**Exemple complet, Mimikatz sur DC01** :
```
1. Sysmon sur DC01 génère l'Event ID 10 (accès à LSASS).
2. L'agent Wazuh sur DC01 lit cet événement dans le canal Sysmon local, l'envoie chiffré au manager.
3. Le manager compare l'événement à ses règles de détection (souvent basées sur Sigma,
   un format standard) — une règle "processus non-système accédant à LSASS avec droits
   mémoire élevés" matche.
4. Une alerte apparaît dans le dashboard : "Possible LSASS Memory Dump detected on DC01",
   avec la ligne de commande exacte, l'heure, l'utilisateur.
```
Cette alerte, capturée en screenshot dans le dashboard, devient la preuve concrète de détection pour chaque writeup de module.

## 4.6 SwiftOnSecurity vs Olaf Hartong

**Problème résolu** : Sysmon, tel qu'installé par défaut, ne logue quasiment rien — il faut un fichier de configuration XML précisant quoi logger (quels Event IDs activer, quels filtres pour éviter d'être noyé sous des millions d'événements inutiles). Écrire ce fichier de zéro prendrait des semaines.

**SwiftOnSecurity et Olaf Hartong** : deux configurations Sysmon prêtes à l'emploi, publiées gratuitement sur GitHub, téléchargées et appliquées directement plutôt que d'écrire sa propre config.

| | SwiftOnSecurity | Olaf Hartong (sysmon-modular) |
|---|---|---|
| Philosophie | Config généraliste "bon sens", poste de travail classique | Config **structurée par technique MITRE ATT&CK**, orientée détection d'attaques (dont AD) |
| Organisation | Un seul gros fichier XML | Découpée en modules (fichier par catégorie), activables/désactivables séparément |
| Exemple de règle | Filtre le bruit des mises à jour Windows/Office automatiques | Règles pensées pour repérer Kerberoasting, DCSync, création de comptes de service suspects |

**Pourquoi Olaf Hartong pour ce projet** : SwiftOnSecurity part du principe "je surveille un PC d'employé normal contre du malware classique" — pas pensé pour les attaques AD à générer dans les modules suivants (BadSuccessor, Kerberoasting, DCSync). Olaf Hartong a des modules explicitement alignés sur ces techniques, donc de meilleures chances qu'une règle capte l'événement pertinent lors d'un `certipy req` ou de la création d'un dMSA malveillant, plutôt que d'écrire soi-même une règle de zéro pour chaque attaque.

**Chaînage complet** :
```
Config Olaf Hartong (dit à Sysmon QUOI logger, orienté attaques AD)
        ↓ appliquée à
Sysmon installé sur DC01 (génère les événements détaillés localement)
        ↓ lus et transmis par
Agent Wazuh sur DC01 (envoie les événements au manager)
        ↓ reçus et analysés par
Manager Wazuh (192.168.56.40)
        ↓ déclenche
Une alerte visible dans le Dashboard Wazuh
```

## 4.7 La Cyber Kill Chain

**Problème résolu** : avant ce modèle (Lockheed Martin, 2011), on décrivait les attaques de façon floue. La Kill Chain découpe toute attaque en 7 étapes obligatoires — un défenseur n'a pas besoin de bloquer partout : bloquer **une seule** étape casse toute l'attaque.

```
1. Reconnaissance    → collecte d'infos sur la cible
2. Weaponization      → préparation de l'"arme" (payload, exploit)
3. Delivery            → livraison de l'arme à la cible (email, USB, accès réseau...)
4. Exploitation        → l'arme s'exécute, exploite une faille
5. Installation         → installation d'une persistance (backdoor, compte)
6. Command & Control    → canal de contrôle à distance
7. Actions on Objectives → objectif réel atteint (vol de données, sabotage...)
```

**Exemple mappé sur BadSuccessor** :
```
1. Reconnaissance    → BloodHound repère le droit CreateChild sur OU=ServiceAccounts
2. Weaponization      → préparation des attributs à écrire (msDS-ManagedAccountPrecededByLink...)
3. Delivery           → déjà franchi (accès via un compte déjà compromis)
4. Exploitation        → création du dMSA malveillant et écriture des attributs mensongers
5. Installation         → le dMSA reste en place comme objet AD légitime en apparence
6. Command & Control    → pas toujours nécessaire (attaque "locale" à l'AD)
7. Actions on Objectives → demande d'un TGT pour le dMSA → PAC d'Administrator → domaine compromis
```

**Pourquoi c'est central pour le module SIEM** : chaque règle de détection construite correspond à une tentative de repérer une étape précise de cette chaîne. Détecter l'accès à LSASS (Mimikatz) = intercepter l'étape 7. Détecter la création d'un compte de service suspect = intercepter l'étape 5. Le triptyque "attaque → patch → détection" de chaque module revient concrètement à identifier à quelle étape de la Kill Chain l'alerte Wazuh intervient.

**Limite du modèle** : pensé pour du malware classique avec livraison réseau (email, exploit) — colle moins bien aux attaques internes à un AD déjà compromis (comme BadSuccessor, sans "delivery" réseau) — une des raisons pour lesquelles MITRE ATT&CK est devenu plus populaire en pentest AD moderne.

## 4.8 MITRE ATT&CK

**Problème que la Kill Chain ne résout pas** : dire "on est à l'étape Exploitation" est vrai mais trop vague — il existe des centaines de façons de faire de l'exploitation (injection SQL, ESC1, Kerberoasting...) : des techniques radicalement différentes à détecter.

**Ce qu'est MITRE ATT&CK** : une base de connaissances immense, maintenue par MITRE, recensant toutes les techniques d'attaque connues, organisées en **tactiques** (le "pourquoi") et **techniques** (le "comment"), chacune avec un identifiant unique.

```
Tactique : Credential Access (TA0006) → "l'attaquant veut voler des identifiants"
  Technique : T1558.003 — Kerberoasting
  Technique : T1003.006 — DCSync
  Technique : T1649       — Steal or Forge Authentication Certificates (ESC1/ESC16)

Tactique : Privilege Escalation (TA0004) → "l'attaquant veut obtenir plus de droits"
  Technique : T1558.001 — Golden Ticket
```

| | Cyber Kill Chain | MITRE ATT&CK |
|---|---|---|
| Niveau de détail | 7 grandes étapes chronologiques | Des centaines de techniques précises, avec sous-techniques |
| Usage typique | Vue d'ensemble ("où en est l'attaque ?") | Référence technique pour écrire des règles de détection précises |
| Analogie | Les chapitres d'un livre | Chaque phrase précise à l'intérieur de ces chapitres |

**Pourquoi c'est le référentiel central du projet** : la config Sysmon Olaf Hartong est structurée par technique MITRE ATT&CK — chaque module de règles activé correspond directement à un identifiant technique (ex. le module surveillant les accès LSASS correspond à T1003.001). Citer l'ID MITRE ATT&CK de chaque attaque réalisée dans les writeups (ex. "T1558.003 — Kerberoasting") est un standard professionnel, exactement ce que fait un vrai rapport de pentest ou une vraie alerte SOC.

## 4.9 SIEM — rappel synthétique

Un SIEM collecte les logs de tout un parc de machines, les corrèle avec des règles de détection, et les affiche dans un dashboard centralisé avec des alertes. Wazuh **est** le SIEM du projet.

**Point important pour la suite (SOAR)** : un SIEM **détecte et alerte**, mais **n'agit pas tout seul**. Quand Wazuh affiche "Possible LSASS Memory Dump detected on DC01", c'est l'analyste humain qui doit décider quoi faire (isoler la machine ? désactiver le compte ? investiguer ?). Le SIEM s'arrête à l'alerte.

## 4.10 SOAR — l'automatisation de la réponse

**Problème résolu** : dans un vrai SOC, un analyste peut recevoir des centaines voire des milliers d'alertes par jour. Traiter chacune manuellement est impossible à cette échelle, et le temps de réaction humain (minutes à heures) laisse à l'attaquant largement le temps d'aller plus loin.

**Ce qu'est un SOAR (Security Orchestration, Automation and Response)** : un outil branché **en aval** du SIEM, qui automatise la réponse à certaines alertes selon des règles prédéfinies ("playbooks"), sans attendre qu'un humain clique.

**Exemple, Mimikatz sur DC01** :
```
SIEM seul :
  → Alerte "LSASS Memory Dump on DC01" apparaît
  → Un analyste doit la voir, investiguer, puis agir manuellement
  → Délai : minutes à heures

Avec un SOAR :
  → L'alerte déclenche AUTOMATIQUEMENT un playbook :
      1. Isoler DC01 du réseau
      2. Désactiver le compte utilisateur qui a lancé le process suspect
      3. Créer un ticket d'incident automatiquement
      4. Notifier l'équipe SOC (Slack/email)
  → Délai : secondes, sans intervention humaine initiale
```

| | SIEM (Wazuh) | SOAR |
|---|---|---|
| Rôle | Détecte et alerte | Automatise la réponse à l'alerte |
| Action sur le système | Aucune — observation | Agit réellement (isoler, bloquer, désactiver...) |
| Analogie | Le détecteur de fumée qui sonne | Le système qui coupe l'électricité et appelle les pompiers tout seul |

**Pour ce projet** : un SOAR est hors scope du module SIEM tel que défini (construction de la détection, pas la réponse automatisée) — mais le mentionner en conclusion/perspectives montre une compréhension de la chaîne complète d'un SOC réel.

## 4.11 IDS et IPS

**Problème résolu** : Sysmon et Wazuh surveillent surtout ce qui se passe à l'intérieur d'une machine (processus, mémoire, registre). Une partie des attaques se voit dans le trafic réseau lui-même (scan de ports, exfiltration massive, signature réseau connue).

**IDS (Intrusion Detection System)** : surveille le trafic réseau (souvent en copie, port miroir/SPAN) et compare à des signatures/comportements connus. **Il ne fait qu'alerter** — comme un SIEM, mais pour le réseau.

**IPS (Intrusion Prevention System)** : même principe, mais placé directement **en ligne** sur le chemin du trafic (pas en copie) — permet de **bloquer activement** en temps réel, avant que le paquet n'atteigne sa cible.

**Exemple** :
```
Scénario : un attaquant externe scanne les ports d'un DC avec nmap.

Avec un IDS :
  → Le trafic de scan est repéré et loggé → alerte "Port scan detected"
  → Le scan continue normalement — l'IDS n'a fait qu'observer et alerter

Avec un IPS :
  → Le trafic de scan est repéré en temps réel, sur le chemin du paquet
  → L'IPS bloque activement les paquets suivants de cette IP
  → Le scan est interrompu avant même de se terminer
```

| Outil | Que surveille-t-il ? | Que fait-il en cas de détection ? |
|---|---|---|
| **IDS** | Le trafic réseau (en copie) | Alerte seulement |
| **IPS** | Le trafic réseau (en ligne, direct) | Bloque activement |
| **SIEM** (Wazuh) | Les logs de toutes les machines/sources (dont Sysmon) | Alerte, centralise, corrèle |
| **SOAR** | Les alertes produites par le SIEM/IDS/IPS | Automatise une réponse (isoler, bloquer un compte...) |

**Où ça se situe dans le projet** : le module SIEM (Wazuh + Sysmon) couvre la brique SIEM, orientée machine/hôte (aussi appelée **HIDS**, Host-based IDS — un Sysmon+Wazuh joue en partie ce rôle, par opposition à un **NIDS**, Network IDS, qui surveille le trafic entre machines). Wazuh a un module de détection réseau basique intégré, mais un vrai NIDS dédié (Suricata, Snort) serait un ajout pertinent en perspective d'amélioration du lab — par exemple pour repérer un trafic LDAP anormal généré par `certipy find` ou `bloodhound-python` depuis une machine compromise.

## 4.12 Résumé — comment tout s'articule

```
MITRE ATT&CK  → le dictionnaire des techniques précises (T1558.003, T1003.006...)
      │           utilisé pour NOMMER et CLASSER chaque attaque réalisée
      ▼
Cyber Kill Chain → la vue chronologique globale (à quelle étape en est l'attaque ?)
      │              utile pour la narration des writeups
      ▼
Sysmon (config Olaf Hartong, alignée MITRE ATT&CK)
      │  génère les événements détaillés sur chaque machine
      ▼
Agent Wazuh → transporte ces événements vers le manager
      ▼
SIEM (Wazuh manager + dashboard) → corrèle, alerte, affiche
      │
      ├── (hors scope module SIEM, à mentionner en perspective)
      │    SOAR → automatiserait la réponse à ces alertes
      │
      └── (hors scope module SIEM, à mentionner en perspective)
           IDS/IPS réseau (Suricata/Snort) → compléterait la vue
           "trafic réseau" en plus de la vue "hôte" que donne Sysmon
```
