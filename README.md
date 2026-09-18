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
| [Récupération d'un système Linux non bootable](incidents/kali-recuperation-boot.md) | Kernel panic dû à un initramfs absent sur une partition `/boot` saturée. Diagnostic, réparation par chroot depuis un environnement live (LVM, bind mounts, `efivars`), réinstallation de GRUB en UEFI. |

## Labs

| Document | Sujet |
|---|---|
| [Lab SOC / Active Directory](https://github.com/Makarinoo/soc-ad-lab) | Infrastructure virtualisée sous Proxmox VE : réseau de lab isolé derrière OPNsense, domaine `ad.lab.internal` peuplé avec quatre faiblesses volontaires, SIEM Wazuh et Sysmon sur le poste client. Kerberoasting, AS-REP Roasting et DCSync menés depuis Kali, puis détectés par quatre règles personnalisées mappées MITRE ATT&CK. Durcissement des quatre faiblesses et rejeu des attaques pour en vérifier l'effet. |

## Autres dépôts

- [Chiffrement de Vigenère](https://github.com/Makarinoo/Chiffrement-Vigenere) — implémentation en Python du
  chiffrement polyalphabétique : gestion de la clé, du décalage et des caractères
  hors alphabet.

- [Stéganographie LSB](https://github.com/Makarinoo/lsb-steganography) — dissimulation de données dans les bits de
  poids faible d'une image : encodage, extraction et limites de la méthode.



  

---

## À propos

Étudiant en Bachelor Cybersécurité à l'EPITA (2025-2028), orienté sécurité des
systèmes d'information. Je recherche une alternance à partir de septembre 2027.

[LinkedIn](https://www.linkedin.com/in/ayman-elhasnaoui) · elhasnaoui.ayman20@gmail.com

Les documents de ce dépôt portent sur mes propres machines et sur des
environnements de lab montés à cette fin. Aucun test réalisé sur un système tiers
n'y figure.
