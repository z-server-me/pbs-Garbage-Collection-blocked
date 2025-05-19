# 🧹 Proxmox Backup Server – Résolution d'un blocage de Garbage Collection

## 🔍 Contexte

Le **Garbage Collector (GC)** de Proxmox Backup Server (PBS) était bloqué avec le message :

```
Error: marking used chunks failed: unexpected error on datastore traversal: Bad message (os error 74)
```

L'interface indiquait :

* `Pending Data`: **\~50 GiB**
* Impossible d’exécuter le GC
* Échec silencieux de la planification, même en forçant le job
* Erreur `Bad Request (400)` lors de la modification via l’interface PBS à cause d’un `datastore.cfg` mal interprété (ligne commentée non prise en compte).

---

## ⚒️ Cause identifiée

Une **corruption partielle du système de fichiers** sur le volume monté (`/mnt/ssd4to`) a empêché le GC d’accéder correctement aux chunks et index. Cela a causé :

* Une lecture invalide des fichiers `.fidx` ou `.didx`
* Des erreurs système type "Bad message"
* Un GC bloqué en phase 1 (mark used chunks)

---

## ✅ Solution appliquée

### 1. Mise en maintenance du datastore

```bash
proxmox-backup-manager datastore update marechal-pve --maintenance-mode offline
```

Ce mode empêche toute opération de lecture/écriture pendant les réparations.

---

### 2. Arrêt des services PBS

```bash
systemctl stop proxmox-backup
systemctl stop nfs-server  # si utilisé
```

---

### 3. Vérification de l’utilisation du montage

```bash
lsof +f -- /mnt/ssd4to
fuser -vm /mnt/ssd4to
```

---

### 4. Démontage du volume

```bash
umount /mnt/ssd4to
```

---

### 5. Réparation du système de fichiers

```bash
fsck -f -v /dev/sdb1
```

✅ Plusieurs inodes optimisés, système de fichiers marqué comme **modifié avec succès**.

---

### 6. Remontage du volume

```bash
mount /mnt/ssd4to
```

Puis redémarrage de PBS :

```bash
systemctl start proxmox-backup
```

---

## 🔄 Nouvelle exécution du Garbage Collection

Le GC a été relancé avec succès :

```bash
proxmox-backup-client garbage-collect --repository marechal-pve
```

Résultat :

```
Removed garbage: 135.322 GiB
Removed chunks: 52204
Pending removals: 39.853 MiB
Leftover bad chunks: 2
```

🎉 **Problème résolu** — le `Pending Data` est retombé à \~40 MiB.

---

## ⚙️ Options de tuning recommandées

Depuis l’interface PBS > Datastore > Options :

```
Chunk Order: inode
Sync Level: filesystem
GC Access-Time Cutoff: 1445 (24h 5min)
GC Cache Capacity: 1048576
```

---

## 💡 Recommandations

* Toujours exécuter un `fsck` si le GC échoue de manière incompréhensible.
* Surveiller la taille du `Pending Data`.
* S’assurer que `maintenance-mode` est activé **avant toute réparation disque**.
* Ne pas commenter de lignes invalides dans `/etc/proxmox-backup/datastore.cfg`, elles causent un `Bad Request`.
