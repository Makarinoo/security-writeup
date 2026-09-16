# Rapport d'incident — Récupération d'un système Kali Linux non bootable

**Date :** septembre 2026
**Durée d'intervention :** ~2 h
**Résultat :** système restauré, aucune perte de données

---

## 1. Environnement

| Élément | Détail |
|---|---|
| Machine | Lenovo ThinkPad X260 (20F5), BIOS R02ET56W |
| Stockage | SSD Samsung MZ7TY256HDHP, 256 Go |
| OS | Kali Linux rolling, noyau 7.1.5+kali-amd64 |
| Partitionnement | `sda1` EFI (FAT32) · `sda2` `/boot` ext4 944 Mo · `sda3` LVM |
| LVM | VG `Cyber-vg` → LV `root` (ext4), LV `swap_1` |
| Chiffrement | aucun (pas de LUKS) |

---

## 2. Symptôme initial

Kernel panic systématique au démarrage :

```
VFS: Cannot open root device "/dev/mapper/Cyber--vg-root"
     or unknown-block(0,0): error -6
List of all bdev filesystems:
 fuseblk
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

---

## 3. Analyse

Le point déterminant est la ligne `List of all bdev filesystems: fuseblk`. Le noyau
ne dispose ni du pilote ext4 ni de device-mapper au moment du montage de la racine.
Ces modules ne sont pas compilés en dur : ils sont fournis par l'**initramfs**.

Conclusion : l'initramfs du noyau courant est absent ou n'a pas été chargé. Sans lui,
le volume logique `Cyber-vg/root` n'existe tout simplement pas du point de vue du noyau.

---

## 4. Cause racine

Inspection de `/boot` depuis un environnement live :

```
/dev/sda2   944M  885M  0  100%  /boot
```

La partition contenait quatre initramfs d'environ 200 Mo chacun (noyaux 6.16.8, 6.18.5,
6.18.9, 6.18.12), hérités de mises à jour successives sans purge.

`dpkg --list | grep linux-image` révélait l'état `iF` sur `linux-image-7.1.5+kali-amd64` :
paquet dépaqueté mais configuration échouée. Lors de la mise à jour du noyau,
`update-initramfs` n'avait pas pu écrire `initrd.img-7.1.5` faute d'espace. L'échec est
passé inaperçu dans le flot de sortie d'`apt`, et GRUB pointait vers un initrd inexistant.

**Cause racine : saturation de `/boot` provoquant l'échec silencieux de la génération
de l'initramfs lors d'une mise à jour de noyau.**

---

## 5. Complication rencontrée

Une tentative de réparation menée sans monter la partition EFI a réécrit l'amorçage
en mode BIOS/Legacy sur un système installé en UEFI. Conséquences successives :

1. Le menu de boot ne présentait plus que `ATA HDD0` et `PCI LAN`, sans entrée UEFI.
2. Puis, à l'amorçage : `error: symbol 'grub_memcpy' not found` → `grub rescue>`,
   signature d'une désynchronisation entre le code d'amorçage et les modules de `/boot/grub`.

Point retenu : `grub-install` doit toujours être exécuté avec `/boot/efi` monté et
`--target` explicite, sous peine d'écrire dans le mauvais mode d'amorçage.

---

## 6. Procédure de résolution

### 6.1 Accès au système

Démarrage sur clé Kali Live **en mode UEFI** (vérifié via l'en-tête du menu GRUB),
Secure Boot désactivé.

```bash
sudo -i
lsblk -f                          # cartographie des partitions avant toute action
```

### 6.2 Montage et chroot

```bash
vgchange -ay                      # activation des volumes logiques
mount /dev/Cyber-vg/root /mnt
mount /dev/sda2 /mnt/boot
mount /dev/sda1 /mnt/boot/efi
for d in dev dev/pts proc sys run; do mount --bind /$d /mnt/$d; done
mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars
chroot /mnt
export LC_ALL=C
```

Le montage de `efivars` est indispensable : sans lui, `grub-install` ne peut pas
enregistrer l'entrée de démarrage dans la NVRAM du firmware.

### 6.3 Libération d'espace

```bash
rm /boot/initrd.img-6.16.8+kali-amd64
rm /boot/initrd.img-6.18.5+kali-amd64
rm /boot/initrd.img-6.18.9+kali-amd64
df -h /boot                       # 100 % → 33 %
```

### 6.4 Régénération de l'initramfs

```bash
apt --fix-broken install
```

Reprise de la configuration de `linux-image-7.1.5+kali-amd64` et génération de
`/boot/initrd.img-7.1.5+kali-amd64` (206 Mo).

### 6.5 Nettoyage et réinstallation du bootloader

```bash
apt purge -y 'linux-image-6.16.8+kali-amd64' \
             'linux-image-6.18.5+kali-amd64' \
             'linux-image-6.18.9+kali-amd64'
apt autoremove --purge -y

grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=kali
update-grub
```

### 6.6 Sortie propre

```bash
exit
umount -R /mnt
reboot
```

---

## 7. Vérification

- `ls -lh /boot/initrd.img-*` → initramfs présent pour le noyau courant
- `grub-install` → *Installation finished. No error reported.*
- Démarrage nominal, système et données intacts

---

## 8. Mesures préventives

- Contrôler `df -h /boot` après chaque mise à jour de noyau
- Lire la fin de sortie d'`apt upgrade` : une erreur `update-initramfs` ne doit
  jamais être suivie d'un redémarrage
- Exécuter `apt autoremove --purge` régulièrement
- Conserver en permanence un support live bootable
- À moyen terme : agrandir `/boot` (944 Mo ne tiennent que 4 initramfs Kali)

---

## 9. Compétences mobilisées

- Lecture et interprétation d'une trace de kernel panic
- Chaîne d'amorçage Linux : firmware UEFI → GRUB → noyau → initramfs → montage racine
- Administration LVM (`vgchange`, volumes logiques)
- Récupération par chroot avec bind mounts (`/dev`, `/proc`, `/sys`, `/run`, `efivars`)
- Résolution d'états `dpkg` dégradés (`iF`, `--fix-broken`)
- Distinction et réparation des modes d'amorçage UEFI / Legacy
- Méthode : diagnostic avant action, sauvegarde de l'état, vérification à chaque étape
