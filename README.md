# Write-ups — sécurité & administration système

Notes techniques et rapports rédigés au fil de ma formation en cybersécurité
(Bachelor Cybersécurité, EPITA) et de mes projets personnels.

Chaque document suit la même logique : le symptôme observé, l'analyse qui mène
à la cause racine, la procédure appliquée, et ce que j'en retire. L'objectif est
qu'un lecteur puisse reproduire le raisonnement, pas seulement les commandes.

---

## Incidents

| Document | Sujet |
|---|---|
| [Récupération d'un système Linux non bootable](incidents/kali-boot-recovery.md) | Kernel panic dû à un initramfs absent sur une partition `/boot` saturée. Diagnostic, réparation par chroot depuis un environnement live (LVM, bind mounts, `efivars`), réinstallation de GRUB en UEFI. |

## Labs

Lab Active Directory / SOC en cours de montage : infrastructure virtualisée sous
Proxmox VE, domaine Active Directory, scénarios offensifs et détection via Wazuh.
Le write-up sera publié ici une fois les premiers scénarios de détection en place.

## Autres dépôts

- [Chiffrement de Vigenère](https://github.com/Makarinoo/lsb-steganography) — implémentation en Python du
  chiffrement polyalphabétique : gestion de la clé, du décalage et des caractères
  hors alphabet.

---

## À propos

Étudiant en Bachelor Cybersécurité à l'EPITA (2025-2028), orienté sécurité des
systèmes d'information. Je recherche une alternance à partir de septembre 2027.

[LinkedIn](https://www.linkedin.com/in/ayman-elhasnaoui) · elhasnaoui.ayman20@gmail.com

Les documents de ce dépôt portent sur mes propres machines et sur des
environnements de lab montés à cette fin. Aucun test réalisé sur un système tiers
n'y figure.
