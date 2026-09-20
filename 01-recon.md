# Module 1 — Reconnaissance (GOAD-Light)

## Cible
Scope réseau `192.168.56.0/24`.

## 1. Découverte réseau

```bash
nmap -sn 192.168.56.0/24
```
→ 3 hôtes vivants : `.10`, `.11`, `.22`

```bash
nmap -p- --min-rate 5000 192.168.56.10 192.168.56.11 192.168.56.22
```

| IP | Hostname | Rôle | Ports clés |
|---|---|---|---|
| .10 | KINGSLANDING | DC racine `sevenkingdoms.local` | 88, 389, 464, 3268/69, 9389 |
| .11 | WINTERFELL | DC enfant `north.sevenkingdoms.local` | idem |
| .22 | CASTELBLACK | Serveur membre (north) | 445 (signing:False), 1433 (MSSQL) |

Absence de ports AD (88/389/464/9389) sur `.22` → confirme rôle non-DC.

## 2. Énumération anonyme (NetExec)

```bash
nxc smb 192.168.56.10 192.168.56.11 192.168.56.22
nxc smb 192.168.56.10 192.168.56.11 192.168.56.22 -u '' -p '' --users
```

- `Null Auth:True` sur les 2 DC (KINGSLANDING, WINTERFELL)
- Énumération SAMR anonyme réussie sur **WINTERFELL seulement** → 10 comptes users du domaine NORTH listés
- KINGSLANDING refuse l'énumération malgré Null Auth:True (asymétrie non expliquée, à creuser)
- CASTELBLACK : `STATUS_ACCESS_DENIED`

**Finding critique** : mot de passe en clair dans la description LDAP d'un compte :
```
samwell.tarly    Samwell Tarly (Password : Heartsbane)
```

## 3. Validation du credential

```bash
nxc smb 192.168.56.11 -u 'samwell.tarly' -p 'Heartsbane'
# → north.sevenkingdoms.local\samwell.tarly:Heartsbane  [+]

nxc smb 192.168.56.10 -u 'samwell.tarly' -p 'Heartsbane' -d north.sevenkingdoms.local
# → authentifie aussi sur le DC racine (trust forêt bidirectionnel/transitif confirmé)
```

## 4. Cartographie de la forêt (LDAP authentifié)

```bash
ldapsearch -x -H ldap://192.168.56.10 \
  -D 'samwell.tarly@north.sevenkingdoms.local' -w 'Heartsbane' \
  -b "CN=Partitions,CN=Configuration,DC=sevenkingdoms,DC=local" \
  "(&(objectClass=crossRef)(systemFlags=3))" dnsRoot
```
→ confirme les 2 domaines de la forêt :
- `sevenkingdoms.local`
- `north.sevenkingdoms.local`

Requête faite contre KINGSLANDING (domaine racine), avec un compte du domaine enfant → confirme que la partition Configuration est répliquée intégralement sur tous les DC, quel que soit leur domaine.

## 5. Collecte BloodHound (un run par domaine)

```bash
bloodhound-ce-python -u 'samwell.tarly@north.sevenkingdoms.local' -p 'Heartsbane' \
  -d sevenkingdoms.local -ns 192.168.56.10 -c All

bloodhound-ce-python -u 'samwell.tarly@north.sevenkingdoms.local' -p 'Heartsbane' \
  -d north.sevenkingdoms.local -ns 192.168.56.11 -c All
```

| Domaine | Users | Groups | OUs | GPOs | Computers | Trusts |
|---|---|---|---|---|---|---|
| sevenkingdoms.local | 16 | 59 | 9 | 2 | 1 | 1 |
| north.sevenkingdoms.local | 17 | 51 | 1 | 3 | 2 | 1 |

Import des 14 JSON dans BloodHound CE → graphe fusionné, trust inter-domaines visible.
