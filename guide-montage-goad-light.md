# Guide de montage — Labo GOAD-Light sur VMware Workstation Pro

Procédure complète, du clonage du dépôt jusqu'au snapshot final.

## Étape 1 — Préparer l'hyperviseur

Désinstaller VirtualBox s'il est présent (évite les conflits réseau avec VMware).

Installer VMware Workstation Pro version **17.6.3** précisément (les versions plus récentes ne sont pas compatibles avec l'outillage Vagrant).

Installer Vagrant sur Windows directement (pas dans WSL2 — Vagrant pilote VMware depuis Windows, même si le reste du provisioning tourne dans WSL2).

Installer le plugin `vagrant-vmware-desktop` :

```
vagrant plugin install vagrant-vmware-desktop
```

Installer le **Vagrant VMware Utility** (paquet MSI séparé, à télécharger sur le site HashiCorp), et vérifier que le service `VagrantVMware` est bien démarré.

## Étape 2 — Configurer le réseau VMware

Ouvrir le "Virtual Network Editor" de VMware, configurer le réseau `VMnet2` en mode Host-only, sous-réseau `192.168.56.0/24`, DHCP désactivé.

Assigner manuellement une adresse IP statique côté hôte à l'adaptateur `VMnet2` (VMware ne le fait pas automatiquement, contrairement à VirtualBox) :

```powershell
New-NetIPAddress -InterfaceIndex <index_VMnet2> -IPAddress 192.168.56.1 -PrefixLength 24
```

## Étape 3 — Cloner GOAD et configurer le lab

Cloner le dépôt GOAD dans WSL2.

Lancer la console GOAD :

```
./goad.sh
```

Sélectionner le lab et le provider :

```
set_lab GOAD-Light
set_provider vmware
```

Vérifier que tous les prérequis sont satisfaits :

```
check
```

## Étape 4 — Créer et démarrer les machines virtuelles

Lancer la création complète du lab :

```
install
```

Si une VM échoue au tout premier démarrage avec une erreur du type "not ready for guest communication", relancer son provisioning manuellement une fois (bug connu de Vagrant/VMware sur Windows) :

```
vagrant.exe provision <nom-de-la-vm>
```

## Étape 5 — Vérifier la connectivité réseau

Tester l'accessibilité de chaque VM sur le port WinRM (5986), depuis Windows et depuis WSL2 :

```powershell
Test-NetConnection -ComputerName 192.168.56.X -Port 5986
```

```bash
nc -zv 192.168.56.X 5986
```

## Étape 6 — Lancer le provisioning Ansible complet

Exécuter l'ensemble des playbooks :

```
provision_lab
```

(ou playbook par playbook si besoin, ex. `provision servers.yml`).

Si le playbook `servers.yml` reste bloqué sans erreur sur une installation (SQL Server Express ou SSMS), c'est qu'une fenêtre d'installation invisible attend une validation. Se connecter en direct à la console VMware de la VM concernée, tuer le processus bloqué, puis relancer l'installeur manuellement en mode interactif (double-clic, pas de ligne de commande) pour valider les écrans à la main.

Si l'installation de SSMS boucle indéfiniment malgré une installation manuelle réussie, créer manuellement le dossier attendu par l'ancien check du playbook pour le satisfaire :

```powershell
New-Item -Path "C:\Program Files (x86)\Microsoft SQL Server Management Studio 18" -ItemType Directory -Force
```

Relancer ensuite `provision servers.yml`, puis enchaîner avec les deux derniers playbooks :

```
provision security.yml
provision vulnerabilities.yml
```

## Étape 7 — Vérifier le résultat

Confirmer que chaque playbook s'est terminé avec `failed=0` sur les trois machines (DC01, DC02, SRV02).

## Étape 8 — Faire le snapshot

Depuis la console GOAD, sur l'instance active :

```
snapshot
```

Un snapshot capture l'état complet des trois VMs (disque, mémoire, configuration) à cet instant précis. Il sert de point de restauration : si une manipulation ultérieure (exploitation, erreur, plantage) casse le lab, on restaure le snapshot au lieu de tout reconstruire depuis le début.
