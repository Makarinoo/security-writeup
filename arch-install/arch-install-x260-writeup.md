# Installation d'Arch Linux en dur sur ThinkPad X260 (sans archinstall)

Installation manuelle d'Arch sur un ThinkPad X260, en UEFI, avec systemd-boot, puis Hyprland et les dotfiles HyDE.

J'ai noté la procédure telle que je l'ai faite, avec les blocages que j'ai rencontrés. C'est surtout ça qui manque dans les guides habituels.

## Sommaire

1. [Préparation de la clé USB](#1-préparation-de-la-clé-usb-depuis-windows)
2. [BIOS du X260](#2-bios-du-x260)
3. [Environnement live](#3-environnement-live)
4. [Partitionnement](#4-partitionnement)
5. [Formatage et montage](#5-formatage-et-montage)
6. [Système de base](#6-système-de-base)
7. [Configuration dans le chroot](#7-configuration-dans-le-chroot)
8. [Bootloader systemd-boot](#8-bootloader-systemd-boot)
9. [Post-installation](#9-post-installation)
10. [Environnement graphique avec Hyprland](#10-environnement-graphique-avec-hyprland)
11. [Dotfiles HyDE](#11-dotfiles-hyde)
12. [PATH et ~/.local/bin](#12-path-et-localbin)
13. [Batterie : seuils de charge TLP](#13-batterie--seuils-de-charge-tlp)
- [Récapitulatif des pièges](#récapitulatif-des-pièges)
- [Ce que je referais](#ce-que-je-referais)

## Contexte

| | |
|---|---|
| Machine | ThinkPad X260 (Intel Skylake, 2016) |
| RAM | 8 Go |
| Disque | SATA SSD 238,5 Go (`/dev/sda`) |
| GPU | Intel HD 520 |
| Wi-Fi | Intel Wireless 8260 (`wlp4s0`) |
| ISO | `archlinux-2026.10.01-x86_64.iso` |
| État initial | un Linux préexistant sur LVM (groupe `Cyber-vg`) |

Objectif : effacement complet, Arch seul, démarrage UEFI, environnement tiling.

## 1. Préparation de la clé USB (depuis Windows)

### Télécharger et vérifier l'ISO

Télécharger ne suffit pas. Il faut vérifier l'empreinte publiée par le projet, sinon rien ne garantit l'intégrité de ce qu'on va écrire.

```bash
curl -sL https://geo.mirror.pkgbuild.com/iso/latest/sha256sums.txt
curl -L -o archlinux-2026.10.01-x86_64.iso \
  https://geo.mirror.pkgbuild.com/iso/latest/archlinux-2026.10.01-x86_64.iso
sha256sum archlinux-2026.10.01-x86_64.iso
```

Sous Windows, l'équivalent de `sha256sum` est :

```powershell
Get-FileHash .\archlinux-2026.10.01-x86_64.iso -Algorithm SHA256
```

Attendu :

```
684ded26c63240ff4a41e8c25ee84ea6da233f557364821f13d12c2b0a9059a5
Taille : 1 640 497 152 octets
```

Limite : l'empreinte vient du même miroir que l'ISO. Elle prouve que le fichier n'a pas été abîmé en route, pas qu'il a bien été publié par Arch. Un miroir compromis servirait les deux. Voir [Ce que je referais](#ce-que-je-referais).

### Identifier la bonne cible

Avant d'écrire, confirmer quel disque est la clé. Sous Windows :

```powershell
Get-Disk | Select-Object Number, FriendlyName, BusType,
  @{n='SizeGB';e={[math]::Round($_.Size/1GB,1)}}
```

La colonne `BusType` tranche : `NVMe` ou `SATA` pour les disques internes, `USB` pour la clé. Se fier à la taille seule est une mauvaise habitude.

### Écrire en mode DD

Avec Rufus : sélectionner la clé, l'ISO, puis `DÉMARRER`. Rufus propose alors *mode Image ISO* ou *mode Image DD*. Prendre DD.

Attention, ce choix n'apparaît pas sur l'écran principal. Il surgit dans une boîte de dialogue après le clic sur `DÉMARRER`. Tant qu'on est sur l'écran principal, les champs `MBR` et `FAT32` affichés ne reflètent que le mode ISO par défaut.

### Vérifier que l'écriture est complète

Rufus affiche `PRÊT` aussi bien avant de démarrer qu'après avoir fini, ce qui est ambigu. Deux contrôles objectifs :

```powershell
# Octets réellement écrits par le processus
(Get-CimInstance Win32_Process -Filter "ProcessId=<PID>").WriteTransferCount

# Disposition résultante
Get-Partition -DiskNumber 1 | Select-Object PartitionNumber, Offset, Size, MbrType
```

Une écriture DD réussie d'une ISO Arch donne :

```
Octets écrits  ≈ taille exacte de l'ISO
Partition ESP  : offset 1300 Mio, taille 264 Mio, MbrType 239 (0xEF)
                 1300 + 264 = 1564 Mio = taille de l'ISO
```

Si on voit à la place une grosse partition FAT32 de la taille de la clé, c'est que le mode ISO a été utilisé.

## 2. BIOS du X260

- `F1` au démarrage pour entrer dans le setup
- `Security` > `Secure Boot` > Disabled (l'ISO Arch n'est pas signée)
- `Startup` > `UEFI/Legacy Boot` > UEFI Only
- `F10` pour sauvegarder
- `F12` pour le menu de démarrage, choisir l'entrée USB marquée UEFI

## 3. Environnement live

### Clavier

```bash
loadkeys fr
```

### Confirmer le mode UEFI

```bash
cat /sys/firmware/efi/fw_platform_size
```

Doit renvoyer `64`. Une erreur `No such file or directory` signifie un démarrage en BIOS legacy, et il est inutile de continuer : toute la suite (ESP, systemd-boot) suppose l'UEFI.

### Réseau Wi-Fi

L'ISO ne se connecte à aucun réseau automatiquement.

```bash
rfkill unblock all
iwctl device list
iwctl station wlan0 scan
iwctl station wlan0 get-networks
iwctl station wlan0 connect "SSID"
ping -c 3 archlinux.org
```

Dans l'ISO, iwd nomme l'interface `wlan0` (à confirmer avec `iwctl device list`). Une fois Arch installé, elle prend son nom stable, `wlp4s0` sur le X260 : c'est celui du tableau de contexte.

`Temporary failure in name resolution` sur un ping est ambigu : ça peut être le DNS, ou l'absence totale de connexion. Pour trancher, pinguer une IP directement avec `ping -c 2 1.1.1.1`. Si ça passe, seul le DNS est en cause. Sinon il n'y a pas de route du tout.

## 4. Partitionnement

### Identifier le disque interne

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN
```

La colonne `TRAN` est décisive : `sata` pour le disque interne, `usb` pour la clé d'installation. Dans l'environnement live, la clé est un disque comme un autre, et rien ne garantit que l'interne soit `sda`.

### Piège : LVM préexistant qui verrouille le disque

Le disque contenait une ancienne installation sur LVM. L'ISO Arch active automatiquement les groupes de volumes qu'elle détecte, ce qui verrouille la partition sous-jacente :

```
sda                  238,5G
├─sda1                   1G  EFI System
└─sda3               236,6G
  ├─Cyber--vg-root   228,7G  lvm
  └─Cyber--vg-swap_1   7,9G  lvm
```

`cfdisk` affiche alors `Device is currently in use, repartitioning is probably a bad idea`. Il faut désactiver LVM avant :

```bash
swapoff -a
vgchange -an
# si ça résiste :
dmsetup remove_all
```

Vérifier que les lignes `lvm` ont disparu de `lsblk` avant de continuer.

### Créer la table

```bash
cfdisk /dev/sda
```

- Table GPT
- Supprimer toutes les partitions existantes
- `New` > `1G` > `Type` > EFI System
- `New` > tout le reste, laisser `Linux filesystem`
- `Write`, taper `yes` en toutes lettres, puis `Quit`

Résultat :

```
/dev/sda1   2048        2099199     1G      EFI System
/dev/sda2   2099200     500117503   237,5G  Linux filesystem
```

cfdisk n'écrit rien tant que `Write` n'a pas été validé. Le message final `Syncing disks.` confirme que la table est passée sur le disque.

## 5. Formatage et montage

```bash
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2

mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

Point de contrôle à ne pas sauter, sans quoi tout s'installe au mauvais endroit :

```bash
lsblk /dev/sda
```

La colonne `MOUNTPOINTS` doit afficher `/mnt` sur `sda2` et `/mnt/boot` sur `sda1`.

## 6. Système de base

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode base-devel \
  nano networkmanager sof-firmware man-db man-pages
```

| Paquet | Rôle |
|---|---|
| `linux-firmware` | firmware de la carte Wi-Fi Intel 8260 |
| `intel-ucode` | microcodes Skylake, chargés au démarrage |
| `networkmanager` | indispensable, sans lui pas de réseau après redémarrage |
| `sof-firmware` | audio |

Puis le fstab, avec deux chevrons sinon le fichier est écrasé :

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

## 7. Configuration dans le chroot

```bash
arch-chroot /mnt
```

### Heure

```bash
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

### Langue et clavier

```bash
sed -i 's/^#fr_FR.UTF-8 UTF-8/fr_FR.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=fr_FR.UTF-8" > /etc/locale.conf
echo "KEYMAP=fr"        > /etc/vconsole.conf
```

`/etc/vconsole.conf` ne concerne que la console TTY. Sous Wayland, la disposition clavier est gérée par le compositeur, voir la section 10.

### Machine et utilisateur

```bash
echo "Cyber" > /etc/hostname

cat > /etc/hosts <<'EOF'
127.0.0.1   localhost
::1         localhost
127.0.1.1   Cyber.localdomain Cyber
EOF

passwd                                   # mot de passe root
useradd -m -G wheel -s /bin/bash user
passwd user
EDITOR=nano visudo                       # décommenter %wheel ALL=(ALL:ALL) ALL

systemctl enable NetworkManager
```

## 8. Bootloader systemd-boot

```bash
bootctl install

cat > /boot/loader/loader.conf <<'EOF'
default arch.conf
timeout 3
console-mode max
editor no
EOF

cat > /boot/loader/entries/arch.conf <<EOF
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=UUID=$(blkid -s UUID -o value /dev/sda2) rw
EOF
```

La substitution `$(blkid ...)` insère l'UUID toute seule. Recopier 36 caractères à la main est l'erreur la plus courante à cette étape. Noter que ce second heredoc n'est pas quoté, justement pour que la substitution s'exécute. Vérifier ensuite :

```bash
cat /boot/loader/entries/arch.conf
```

### Piège : variables EFI non écrites depuis le chroot

`bootctl install` affiche :

```
Not booted with EFI or running in a container, skipping EFI variable modifications.
```

Le chroot empêche l'écriture de l'entrée de démarrage en NVRAM. Ce n'est pas bloquant : bootctl a copié le binaire dans `/boot/EFI/BOOT/BOOTX64.EFI`, le chemin de secours que tous les firmwares savent lire. La machine démarre quand même.

Après le premier démarrage réel, relancer la commande pour inscrire l'entrée définitivement :

```bash
sudo bootctl install
```

Elle répond cette fois :

```
Created EFI boot entry "Linux Boot Manager".
Created EFI boot entry "Fallback Linux Boot Manager".
```

Les avertissements `Random seed file ... is world accessible, which is a security hole!` sont normaux. La partition EFI est en FAT32, qui ne gère pas les permissions Unix. Rien à corriger.

### Redémarrer

```bash
exit
umount -R /mnt
reboot
```

## 9. Post-installation

```bash
# Wi-Fi
nmcli device wifi list
sudo nmcli device wifi connect "SSID" password "..."

# Mise à jour
sudo pacman -Syu

# Swap (il n'y en a plus après la suppression du LVM)
sudo mkswap -U clear --size 4G --file /swapfile
sudo swapon /swapfile
echo '/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab
```

## 10. Environnement graphique avec Hyprland

```bash
sudo pacman -S hyprland waybar wofi kitty mako hyprlock hypridle \
  xdg-desktop-portal-hyprland xdg-desktop-portal-gtk hyprpolkitagent \
  hyprpaper thunar grim slurp wl-clipboard \
  pipewire pipewire-pulse pipewire-alsa wireplumber
```

Lancement depuis la console avec `Hyprland`, H majuscule.

### Piège : configuration en Lua depuis Hyprland 0.55

Hyprland a remplacé `hyprland.conf` (hyprlang) par `hyprland.lua` :

| Version | État du `.conf` |
|---|---|
| ≤ 0.54 | seul format |
| 0.55 | Lua introduit, `.conf` encore accepté |
| 0.56.1 | `.conf` déprécié |
| ≥ 0.57 | suppression du `.conf` annoncée (pas encore sortie : Arch en est à 0.56.2 au moment où j'écris) |

Trois conséquences pratiques.

`hyprctl keyword` ne fonctionne plus :

```
keyword can't work with non-legacy parsers. Use eval.
```

La nouvelle syntaxe passe par eval :

```bash
hyprctl eval 'hl.config({ input = { kb_layout = "fr" } })'
```

Les dispatchers ont changé de nom. `movefocus` devient `hl.dsp.focus` :

```lua
hl.bind(mainMod .. " + Left", hl.dsp.focus({ direction = "left" }))
```

Et les noms de touches sont sensibles à la casse. Les binds générés avec des flèches en minuscules (`" + left"`) ne s'enregistrent pas, il faut les keysyms XKB capitalisés :

```bash
sed -i 's/ + left"/ + Left"/g; s/ + right"/ + Right"/g; s/ + up"/ + Up"/g; s/ + down"/ + Down"/g' \
  ~/.config/hypr/hyprland.lua
hyprctl reload
```

Le symptôme est caractéristique : `Super`+`Q` fonctionne, parce que le Q est déjà en majuscule dans la config, mais `Super`+flèches reste sans effet.

### Clavier sous Wayland

`KEYMAP=fr` dans `/etc/vconsole.conf` ne s'applique qu'au TTY. Pour Hyprland, il faut le déclarer dans `hyprland.lua` :

```lua
hl.config({
    input = {
        kb_layout  = "fr",
        kb_variant = "",
        kb_options = "",
    },
})
```

Pour basculer entre fr et us avec `Alt`+`Shift` :

```lua
kb_layout  = "fr,us",
kb_options = "grp:alt_shift_toggle",
```

La première disposition listée est celle utilisée pour interpréter les raccourcis.

## 11. Dotfiles HyDE

```bash
cp -r ~/.config/hypr ~/.config/hypr.bak     # l'installateur écrase sans demander
sudo pacman -S --needed git base-devel
git clone --depth 1 https://github.com/HyDE-Project/HyDE ~/HyDE
cd ~/HyDE/Scripts && ./install.sh
```

À lancer en utilisateur normal, pas avec sudo. Le script appelle sudo lui-même quand il en a besoin.

Mes choix pendant l'installation :

| Question | Réponse | Raison |
|---|---|---|
| Assistant AUR | `yay` | déjà installé |
| Fournisseur `ttf-font` | `noto-fonts` | couverture Unicode la plus large |
| Backend `qt6-multimedia` | `qt6-multimedia-ffmpeg` | codecs inclus, pas de plugins à ajouter |
| Dots à déployer | `all` | ensemble cohérent, système neuf |

Le script ajoute le dépôt tiers Chaotic-AUR à `/etc/pacman.conf`. Ce sont des paquets AUR précompilés, donc beaucoup plus rapide sur un Skylake, mais ça revient à faire confiance à un dépôt non officiel pour des binaires installés en root. Réversible en commentant sa section.

### Faux blocage en fin d'installation

Le script rend la main sans bannière de fin, après plusieurs minutes silencieuses pendant lesquelles il génère les palettes via ImageMagick et reconstruit le cache de polices. Ne pas l'interrompre trop vite : un Ctrl+C pendant une transaction pacman laisse un verrou et des paquets à moitié installés. Vérifier d'abord l'activité réelle :

```bash
top -b -n 1 | head -15
```

Load average qui retombe et CPU majoritairement inactif veut dire terminé, pas figé.

### Waybar

HyDE fournit plusieurs dispositions de barre :

```bash
hyde-shell waybar -S        # sélecteur rofi
```

`Super`+`Alt`+`↑` et `↓` pour les faire défiler sans passer par le menu.

### Piège : icônes absentes dans la Waybar

Symptôme : la barre n'affiche que des chiffres bruts, sans les icônes de batterie ni de réseau.

Cause : `XDG_CONFIG_HOME` et `XDG_DATA_HOME` n'étaient définis nulle part. La config de Waybar inclut `$XDG_CONFIG_HOME/waybar/includes/includes.json`. Comme la variable est vide, le chemin reste littéral, le fichier n'est jamais trouvé et aucun module personnalisé ne se charge.

Correction : déclarer les variables pour toute la session dans `~/.config/environment.d/10-xdg.conf`, que `systemd --user` lit à la connexion :

```ini
XDG_CONFIG_HOME=/home/user/.config
XDG_DATA_HOME=/home/user/.local/share
XDG_STATE_HOME=/home/user/.local/state
XDG_CACHE_HOME=/home/user/.cache
XDG_BIN_HOME=/home/user/.local/bin
```

Puis se déconnecter et se reconnecter. Si une autre application de HyDE se comporte bizarrement, vérifier d'abord ces variables avec `printenv | grep XDG`.

## 12. PATH et ~/.local/bin

Plusieurs outils s'installent dans `~/.local/bin`, qui n'est pas dans le PATH par défaut. Le symptôme est un `commande introuvable` juste après une installation pourtant réussie, par exemple avec `hyde-shell` ou `claude`.

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

Ça vaut aussi pour les programmes lancés par les raccourcis de Hyprland, mais seulement parce que le `~/.bash_profile` par défaut d'Arch charge `~/.bashrc`, et que Hyprland est démarré depuis cette connexion en TTY. Avec un gestionnaire de connexion graphique (SDDM, GDM), `~/.bashrc` n'est pas lu : il faut alors déclarer le PATH dans `~/.config/environment.d/`, comme pour les variables XDG.

## 13. Batterie : seuils de charge TLP

Le X260 a deux batteries : `BAT0` (interne) et `BAT1` (amovible). Comme le portable reste souvent branché, j'ai limité la charge entre 40 et 80 % pour réduire l'usure.

```bash
sudo pacman -S tlp
sudo systemctl enable --now tlp
```

`/etc/tlp.d/01-thresholds.conf` :

```ini
# Seuils de charge ThinkPad X260 (double batterie)
START_CHARGE_THRESH_BAT0=40
STOP_CHARGE_THRESH_BAT0=80
START_CHARGE_THRESH_BAT1=40
STOP_CHARGE_THRESH_BAT1=80
```

Appliquer puis vérifier ce que le noyau a réellement reçu :

```bash
sudo tlp start
cat /sys/class/power_supply/BAT*/charge_control_{start,end}_threshold
```

### Piège : le mauvais dossier est ignoré sans erreur

TLP 1.10 lit `/etc/tlp.d/`. Un fichier placé dans `/etc/tlp.conf.d/` n'est jamais lu, et rien ne le signale : les seuils restent ceux du constructeur. Seule la lecture de `/sys` (ci-dessus) le révèle.

### Conséquence au quotidien

Branché, la charge s'arrête à 80 % (`Not charging` dans `/sys/class/power_supply/BAT0/status`) et ne reprend que sous 40 %. Pour une charge complète avant un déplacement :

```bash
sudo tlp fullcharge BAT0
sudo tlp fullcharge BAT1
```

Les seuils configurés reviennent tout seuls au redémarrage suivant.

## Récapitulatif des pièges

| # | Piège | Signe | Résolution |
|---|---|---|---|
| 1 | Mode ISO au lieu de DD | grosse partition FAT32 sur la clé | relancer Rufus, choisir DD dans la boîte de dialogue |
| 2 | LVM préexistant actif | `Device is currently in use` | `swapoff -a` puis `vgchange -an` |
| 3 | Mauvais disque ciblé | | se fier à la colonne `TRAN` de `lsblk` |
| 4 | Démarrage en BIOS legacy | `fw_platform_size` absent | UEFI Only dans le BIOS |
| 5 | Pas de Wi-Fi après reboot | aucune interface | `systemctl enable NetworkManager` dans le chroot |
| 6 | Entrée EFI non inscrite | `skipping EFI variable modifications` | relancer `bootctl install` après le premier démarrage |
| 7 | QWERTY sous Wayland | `vconsole.conf` ignoré | `kb_layout` dans `hyprland.lua` |
| 8 | `hyprctl keyword` rejeté | `non-legacy parsers` | utiliser `hyprctl eval` |
| 9 | Binds flèches inactifs | `Super`+`Q` marche, pas les flèches | capitaliser `Left`, `Right`, `Up`, `Down` |
| 10 | Commande introuvable | après une installation réussie | ajouter `~/.local/bin` au PATH |
| 11 | Waybar sans icônes | chiffres bruts seulement | variables XDG dans `~/.config/environment.d/` |
| 12 | Seuils TLP ignorés | `/sys` affiche encore les valeurs constructeur | fichier dans `/etc/tlp.d/`, pas `/etc/tlp.conf.d/` |

## Ce que je referais

Avec le recul, l'installation fonctionne mais laisse cinq points faibles. Je ne les ai pas encore corrigés sur cette machine.

### 1. Chiffrer le disque

Sans chiffrement, un portable volé se lit en branchant son SSD sur une autre machine. À la réinstallation, je mettrais la partition système en LUKS2 :

```bash
cryptsetup luksFormat /dev/sda2
cryptsetup open /dev/sda2 root
mkfs.ext4 /dev/mapper/root
mount /dev/mapper/root /mnt
```

Puis le hook `sd-encrypt` dans `/etc/mkinitcpio.conf`, après `block` :

```
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```

et l'entrée systemd-boot qui pointe vers le volume déchiffré :

```
options rd.luks.name=<UUID de sda2>=root root=/dev/mapper/root rw
```

### 2. Réactiver Secure Boot avec mes propres clés

Je l'ai désactivé pour démarrer l'ISO et je ne l'ai jamais remis. `sbctl` permet de créer ses propres clés et de signer systemd-boot et le noyau. Il faut d'abord passer Secure Boot en *Setup Mode* dans le BIOS, puis :

```bash
sudo sbctl create-keys
sudo sbctl enroll-keys -m        # -m garde les clés Microsoft (firmwares de certaines cartes)
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sudo sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sudo sbctl verify
```

Le hook pacman de sbctl re-signe ces fichiers à chaque mise à jour. Secure Boot et chiffrement vont ensemble : l'un empêche de démarrer un système modifié, l'autre empêche de lire le disque.

### 3. Vérifier la signature de l'ISO, pas seulement son empreinte

Télécharger aussi `archlinux-2026.10.01-x86_64.iso.sig`, puis :

```bash
gpg --keyserver-options auto-key-retrieve --verify archlinux-2026.10.01-x86_64.iso.sig
```

et comparer l'empreinte de la clé affichée avec celle publiée sur la page de téléchargement d'archlinux.org. Depuis une machine déjà sous Arch : `pacman-key -v archlinux-2026.10.01-x86_64.iso.sig`.

### 4. Ne pas laisser de secrets dans l'historique

`nmcli device wifi connect "SSID" password "..."` laisse le mot de passe du Wi-Fi dans `~/.bash_history`. Mieux :

```bash
nmcli --ask device wifi connect "SSID"
```

Et une fois sudo vérifié, verrouiller le compte root, qui n'a plus de raison d'avoir un mot de passe utilisable :

```bash
sudo passwd -l root
```

### 5. Activer l'entretien automatique

```bash
sudo systemctl enable systemd-boot-update.service   # met systemd-boot à jour quand systemd change
sudo systemctl enable --now fstrim.timer            # TRIM hebdomadaire du SSD
```

## Durée indicative

| Étape | Temps |
|---|---|
| Téléchargement et écriture de la clé | ~15 min |
| Partitionnement, pacstrap, bootloader | ~30 min |
| Hyprland et configuration | ~20 min |
| Dotfiles HyDE | 30 à 60 min |

## Références

- [Guide d'installation, Arch Wiki](https://wiki.archlinux.org/title/Installation_guide)
- [ThinkPad X260, Arch Wiki](https://wiki.archlinux.org/title/Lenovo_ThinkPad_X260)
- [TLP, Arch Wiki](https://wiki.archlinux.org/title/TLP)
- [XDG Base Directory, Arch Wiki](https://wiki.archlinux.org/title/XDG_Base_Directory)
- [Chiffrer tout le système, Arch Wiki](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system)
- [Secure Boot et sbctl, Arch Wiki](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [Documentation Hyprland](https://wiki.hypr.land/)
- [Migration Lua, discussion Hyprland](https://github.com/hyprwm/Hyprland/discussions/15410)
- [hyprlang2lua, convertisseur .conf vers Lua](https://github.com/EIonTusk/hyprlang2lua)
- [HyDE](https://github.com/HyDE-Project/HyDE)
