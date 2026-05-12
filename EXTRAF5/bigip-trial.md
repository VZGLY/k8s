# Marche à suivre — Trial BIG-IP VE pour le labo F5 CIS

Procédure pour obtenir, déployer et configurer un BIG-IP VE avec une licence d'évaluation, prêt à être piloté par F5 CIS.

---

## 1. Choix de la licence

Deux options gratuites :

| Option | Durée | Débit | Avantage | Où |
|---|---|---|---|---|
| **Eval / Trial** | 30 jours | Illimité | Toutes features | https://www.f5.com/trials/big-ip-virtual-edition |
| **F5 Developer / Lab License** | Sans expiration | **25 Mbps** | Idéal labo long-terme | https://www.f5.com/trials → "Get a Developer/Lab license" |

**Recommandation labo** : Developer/Lab. 25 Mbps suffit largement pour CIS + nginx.

---

## 2. Création du compte F5

1. Aller sur https://account.f5.com/ → **Create Account**
2. Email pro de préférence (un email perso passe aussi, mais validation manuelle parfois).
3. Valider l'email.
4. Une fois connecté : profil → vérifier que l'accès **downloads.f5.com** est actif (peut prendre 1-2 h après inscription).

---

## 3. Téléchargement de l'image BIG-IP VE

1. https://downloads.f5.com → **Find a Download** → **BIG-IP** → choisir la dernière LTS (ex. `17.1.x`).
2. Onglet **Virtual-Edition** → choisir le format :
   - **`.ova`** → VMware Workstation / ESXi
   - **`.qcow2`** → KVM / Proxmox
   - **`.vhd`** → **Hyper-V** ← choix recommandé sur Windows 11 Pro
3. Taille du fichier : ~3-4 Go. Garde aussi le **MD5** affiché à côté pour vérifier l'intégrité.

---

## 4. Spécs VM requises (minimales pour labo)

| Ressource | Minimum | Recommandé labo |
|---|---|---|
| vCPU | 2 | 4 |
| RAM | 4 Go | **8 Go** (sinon TMM crash) |
| Disque | 82 Go (image préformatée) | laisser tel quel |
| Réseaux | 2 NICs (mgmt + data) | 3 (mgmt, data interne, data externe) |

> **Note** : la VM démarre avec un disque préformaté à 82 Go. Ne pas le réduire, ça casse le boot.

---

## 5. Déploiement sur Hyper-V (cas le plus simple)

```powershell
# Depuis PowerShell admin sur Windows
# 1. Importer le VHD
$vmName = "bigip-ve"
$vhdPath = "C:\VMs\bigip-ve\disk0.vhd"   # chemin où tu as extrait l'image

New-VM -Name $vmName `
       -MemoryStartupBytes 8GB `
       -Generation 1 `
       -VHDPath $vhdPath `
       -SwitchName "Default Switch"

# 2. CPU
Set-VMProcessor -VMName $vmName -Count 4

# 3. Ajouter une 2e NIC pour le data plane (créer un vSwitch interne avant)
# New-VMSwitch -Name "f5-data" -SwitchType Internal
Add-VMNetworkAdapter -VMName $vmName -SwitchName "f5-data"

# 4. Démarrer
Start-VM -Name $vmName
vmconnect localhost $vmName
```

> **Si BIG-IP refuse de booter sur Gen2** : utiliser Gen1 (UEFI pas supporté sur certaines versions BIG-IP VE).

---

## 6. Premier login

1. Console série/vmconnect : login `root` / mot de passe `default`.
2. La CLI demande un nouveau mot de passe root → définis-le.
3. Configurer l'IP de management :
   ```
   config
   ```
   (utilitaire `config`) → entrer IP, masque, gateway de la NIC mgmt.
4. Tester ping depuis Windows vers cette IP.
5. Ouvrir https://`<mgmt-ip>` dans le navigateur (ignorer warning cert auto-signé).
6. Login GUI : `admin` / mot de passe `admin` → la GUI force le changement immédiat.

---

## 7. Activation de la licence

1. GUI → **System > License > Activate**.
2. Coller la **registration key** reçue par mail F5 (format `XXXX-XXXX-XXXX-...`).
3. Méthode : **Automatic** (la VM doit avoir un accès Internet) — sinon Manual (offline).
4. Accepter l'EULA → la VM redémarre les services (~2 min).
5. Vérifier : **System > License** → statut **Licensed**, fonctionnalités listées.

---

## 8. Configuration réseau minimale BIG-IP

| Étape | Chemin GUI | Valeur exemple |
|---|---|---|
| VLAN interne | Network > VLANs > Create | name=`internal`, interface=`1.1` |
| VLAN externe | Network > VLANs > Create | name=`external`, interface=`1.2` |
| Self-IP interne | Network > Self IPs > Create | `10.10.20.1/24` sur VLAN internal |
| Self-IP externe | Network > Self IPs > Create | `192.168.10.1/24` sur VLAN external |
| Route par défaut | Network > Routes > Create | gateway = gateway de external |

> **Le VIP** que tu mettras dans `VirtualServer.spec.virtualServerAddress` doit être **dans le subnet de la Self-IP externe** (ex. `192.168.10.100`).

---

## 9. Préparer BIG-IP pour CIS

### 9.1. Partition dédiée
```
System > Users > Partition List > Create
  Name: k8s
  Description: Partition gérée par F5 CIS
```

### 9.2. Compte de service pour CIS
```
System > Users > User List > Create
  Name: cis
  Password: <fort>
  Partition Access:
    - Partition: k8s,  Role: Administrator
    - Partition: Common, Role: Resource Administrator   (nécessaire pour AS3)
  Terminal Access: tmsh
```

### 9.3. Installer l'extension AS3
AS3 = Application Services 3 — c'est le format JSON que CIS pousse au BIG-IP.

1. Télécharger le RPM AS3 : https://github.com/F5Networks/f5-appsvcs-extension/releases
2. Upload via iControl REST (depuis WSL) :
   ```bash
   FN=f5-appsvcs-3.x.x-x.noarch.rpm
   CREDS="admin:<motdepasse>"
   BIGIP=10.10.10.10

   # Upload du RPM
   curl -sku $CREDS \
     -H "Content-Type: application/octet-stream" \
     -H "Content-Range: 0-$(($(stat -c%s $FN) - 1))/$(stat -c%s $FN)" \
     -T $FN \
     https://$BIGIP/mgmt/shared/file-transfer/uploads/$FN

   # Installation
   curl -sku $CREDS \
     -H "Content-Type: application/json" \
     -d "{\"operation\":\"INSTALL\",\"packageFilePath\":\"/var/config/rest/downloads/$FN\"}" \
     https://$BIGIP/mgmt/shared/iapp/package-management-tasks
   ```
3. Vérifier l'install :
   ```bash
   curl -sku $CREDS https://$BIGIP/mgmt/shared/appsvcs/info | jq
   ```
   Doit retourner la version d'AS3.

---

## 10. Test de bout en bout

1. **Côté K8s** (WSL) :
   ```bash
   # CRDs F5
   kubectl apply -f https://raw.githubusercontent.com/F5Networks/k8s-bigip-ctlr/master/docs/config_examples/customResourceDefinitions/customresourcedefinitions.yml

   # Namespace, RBAC, Secret CIS
   kubectl apply -f 10-cis/01-namespace.yaml
   kubectl apply -f 10-cis/02-rbac.yaml
   kubectl create secret generic bigip-login \
     --namespace f5-cis \
     --from-literal=username=cis \
     --from-literal=password='<motdepasse-cis>'

   # CIS via Helm
   helm repo add f5-stable https://f5networks.github.io/charts/stable
   helm install f5-bigip-ctlr f5-stable/f5-bigip-ctlr \
     --namespace f5-cis \
     --values 10-cis/04-cis-values.yaml \
     --set bigip.url=https://10.10.10.10

   # App de test
   kubectl apply -f 20-app/
   ```

2. **Côté BIG-IP** (GUI) :
   - **Local Traffic > Virtual Servers** → partition `k8s` → tu dois voir `nginx_demo_nginx-vs_80` apparaître en ~30s.
   - **Local Traffic > Pools** → un pool avec 2 ou 3 pool members (IP des workers Kind:NodePort).

3. **Test trafic** :
   ```bash
   curl -H "Host: nginx.demo.local" http://192.168.10.100/
   ```
   → page d'accueil nginx, et requêtes réparties round-robin entre les pods.

---

## 11. Logs utiles pour débugger

```bash
# Logs CIS — voir ce que le contrôleur pousse à BIG-IP
kubectl logs -n f5-cis -l app=f5-bigip-ctlr -f

# Côté BIG-IP, logs AS3 (SSH au BIG-IP)
ssh admin@10.10.10.10
tail -f /var/log/restnoded/restnoded.log
```

---

## 12. Pièges classiques

| Symptôme | Cause probable | Fix |
|---|---|---|
| CIS log : `401 Unauthorized` | mauvais user/pass dans le Secret | recréer le Secret, redémarrer pod CIS |
| CIS log : `partition does not exist` | partition `k8s` pas créée côté BIG-IP | créer la partition (étape 9.1) |
| CIS log : `AS3 extension not found` | RPM AS3 pas installé | étape 9.3 |
| VS apparaît mais pool members `down` | health monitor échoue (BIG-IP ne joint pas les nodes) | vérifier route BIG-IP → IPs nodes K8s, firewall, et que le Service est bien NodePort |
| VS pas créé du tout | CIS regarde le mauvais namespace | retirer `--namespace=xxx` ou ajouter le ns à `--namespace` |
