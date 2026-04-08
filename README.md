
---

# LSC / Anyka / Tuya — RTSP 100% Local (No Cloud)

**By HeartyGFX (Watson) & Holmes — April 2026**

---



<img src="https://github.com/user-attachments/assets/0103eac8-a083-4a93-9af8-da55f1f6acb8" width="600">




## **EN — Summary**

Some LSC Smart Connect cameras (Anyka-based, Tuya firmware) are heavily tied to cloud services. This repository documents a method to recover a local RTSP stream (LAN).

**Status:**

- **Main RTSP stream:** validated (2 cameras out of 2)
    
- **Sub stream:** not validated yet (contributions welcome)
    
- **Repeatability:** confirmed on 2 units, 3rd unit test in progress
    

**Important prerequisites:**

- The camera must first be configured via LSC Connect / Smart Connect to join the local WiFi.
    
- SD card compatibility is very sensitive (**MBR required**).
    

---

### ⚠️ Disclaimer

- For educational and personal use only.
    
- Use only on devices you own.
    
- No warranty.
    
- You are responsible for what you do with this information.
    

---

### 🙏 Credits & Prior Work

This work builds on prior research and tooling:

**[Guino / LSCOutdoor1080P](https://github.com/guino/LSCOutdoor1080P) : SD execution method and baseline hacking approach**

**[MuhammedKalkan / Anyka-Camera-Firmware](https://github.com/MuhammedKalkan/Anyka-Camera-Firmware) : Anyka firmware documentation and alternative RTSP stacks**
    
    

    

**Original contribution of this repo :**

- Identification of a factory-mode trigger path enabling local RTSP streaming
    
- Repeatable PoC on multiple devices
    
- Documentation of SD partition-table constraints (GPT vs MBR)
    

---

### 📋 Tested Hardware

|Field|Value|
|---|---|
|**Brand**|LSC Smart Connect Outdoor Camera (Action stores)|
|**Reference**|Article No. 3202092.2|
|**Chip**|Anyka AK3918EV330|
|**Firmware**|Tuya IPC SDK 4.9.18|

---

### 🔑 The Key Discovery

The Anyka/Tuya firmware contains a dormant RTSP server.

It is activated by creating **/tmp/_ak39_factory.ini** at runtime.

This triggers factory test mode and RTSP server starts on port **554**.

---

### 📦 Requirements

- LSC Smart Connect camera (configured on app first)
    
- SD Card 4GB to 32GB
    
- Windows PC with Telnet client enabled
    
- VLC Media Player
    

#### ⚠️ SD Card MUST be MBR formatted

Modern SD cards are GPT formatted by default and are rejected by the camera.

**Windows commando method — open CMD as Administrator :**

DOS

```
diskpart
list disk
select disk X
clean
convert mbr
create partition primary
format fs=fat32 quick
active
exit
```

|Command|Why|
|---|---|
|**clean**|Wipes everything including partition table|
|**convert mbr**|**Anyka cameras reject GPT**|
|**active**|Marks partition as bootable|

---

### 📁 SD Card File Structure

Plaintext

```
SD:/
├── hack.sh    <-- original from Guino repo (DO NOT MODIFY)
└── custom.sh  <-- our script
```

_Both files must be UTF-8 encoded with LF line endings (not CRLF)_

**📄 hack.sh**

Use the original unmodified **hack.sh** from Guino's repo.

**Do not modify it.**

**📄 custom.sh**

Bash

```
#!/bin/sh

sleep 8

# Permanent telnet via rc.local
if [ ! -f /etc/config/rc.local ]; then
    mount -o remount,rw /etc/config
    printf '#!/bin/sh\ntelnetd -p 24 -l /bin/sh &\n' > /etc/config/rc.local
    chmod +x /etc/config/rc.local
fi

telnetd -p 24 -l /bin/sh &
```

---

### 🚀 Step by Step Procedure

**STEP 1 — Prepare SD Card**

- Format SD : FAT32 + MBR (voir les commandes Diskpart plus haut)
    
- Copy **hack.sh** and **custom.sh** to SD root
    
- Check line endings : **LF only**
    

**STEP 2 — First Boot**

- Insert SD into powered-off camera
    
- Power on
    
- **Wait 60 seconds**
    

**STEP 3 — Enable Windows Telnet**

- Control Panel
    
- Programs
    
- Turn Windows features on or off
    
- Check **Telnet Client**
    

**STEP 4 — Connect via Telnet**

DOS

```
telnet 192.168.1.100 24
```

You get a direct root shell :

Plaintext

```
~ #
```

**STEP 5 — Activate RTSP**

Bash

```
cat > /tmp/_ak39_factory.ini << 'EOF'
[config]
rtsp_enable = 1
rtsp_port = 554
factory_mode = 1
video_enable = 1
audio_enable = 1

[rtsp]
port = 554
enable = 1
main_stream = 1
sub_stream = 1
EOF

killall anyka_ipc
```

**⚠️ Camera will speak and reboot. Normal.**

**STEP 6 — Reconnect After Reboot**

- Wait 90 seconds then reconnect :
    

DOS

```
telnet 192.168.1.100 24
```

- Verify :
    

Bash

```
netstat -tuln | grep 554
```

- Expected :
    

Plaintext

```
tcp 0.0.0.0:554 LISTEN
```

**STEP 7 — View in VLC**

- Media
    
- Open Network Stream
    
- **rtsp://192.168.1.100:554/main_ch**
    

---

### 📡 RTSP URLs

|Stream|URL|Status|
|---|---|---|
|**Main (HD)**|`rtsp://CAM_IP:554/main_ch`|✅ Validated|
|**Sub (SD)**|`rtsp://CAM_IP:554/sub_ch`|❓ Not validated yet|



---


**IF it doesn't work,** try running the following lines manually via **telnet connection**:

**Step 1:** Create the configuration file

Bash

```
cat > /tmp/_ak39_factory.ini << 'EOF'
[config]
rtsp_enable = 1
rtsp_port = 554
factory_mode = 1
EOF
```

**Step 2:** Verify the file content

Bash

```
cat /tmp/_ak39_factory.ini
```

**Step 3:** Restart the IPC process

Bash

```
killall anyka_ipc
```

**Step 4:** Check the network ports The camera will reboot (you will hear a voice prompt). Wait a few seconds, go back to your telnet session, and type:

Bash

```
sleep 15 && netstat -tuln
```

If you see the following output:

Plaintext

```
tcp        0      0 0.0.0.0:23              0.0.0.0:* LISTEN
tcp        0      0 0.0.0.0:24              0.0.0.0:* LISTEN
tcp        0      0 0.0.0.0:8090            0.0.0.0:* LISTEN
tcp        0      0 0.0.0.0:554             0.0.0.0:* LISTEN
```

**You won!** Now, check the main HD stream with VLC... 👌



---

### 🔄 Automatic RTSP on Every Boot

Bash

```
#!/bin/sh

sleep 8

# Permanent telnet
if [ ! -f /etc/config/rc.local ]; then
    mount -o remount,rw /etc/config
    printf '#!/bin/sh\ntelnetd -p 24 -l /bin/sh &\n' > /etc/config/rc.local
    chmod +x /etc/config/rc.local
fi

telnetd -p 24 -l /bin/sh &

# RTSP factory trigger once per boot
if [ ! -f /tmp/_ak39_factory_done ]; then
    cat > /tmp/_ak39_factory.ini << 'EOF'
[config]
rtsp_enable = 1
rtsp_port = 554
factory_mode = 1
video_enable = 1
audio_enable = 1

[rtsp]
port = 554
enable = 1
main_stream = 1
sub_stream = 1
EOF
    touch /tmp/_ak39_factory_done
    sleep 3
    killall anyka_ipc
fi
```

**⚠️ /tmp/ is cleared on reboot.** **Camera reboots once per power cycle to activate RTSP.**

---

### 🔍 Technical Details

**Firmware internals**

|Element|Value|
|---|---|
|**Binary**|`/usr/bin/anyka_ipc`|
|**RTSP trigger**|`/tmp/_ak39_factory.ini`|
|**RTSP port**|554|
|**Key symbols**|`ht_rtsp_start`, `ht_rtsp_stop`|

**Storage layout**

|Mount|Type|Access|
|---|---|---|
|**/usr**|squashfs|Read-only|
|**/etc/config**|jffs2|Persistent read-write|
|**/tmp**|tmpfs|Lost on reboot|

**Why Guino's patch doesn't work here**

- **Expected MD5 :** `5ac1f462bf039ec3c6c0a31d27ae652a` (v2.10.36)
    
- **Actual MD5 :** `36849ada9f7fc1e6ea27a986cfbee8d0` (older)
    
- Our method bypasses this completely.
    

---

### ✅ Compatible Software

- VLC Media Player
    
- Frigate (NVR)
    
- Home Assistant (Generic Camera)
    
- Blue Iris
    
- Shinobi
    
- FFmpeg
    

---

### ⚠️ Known Limitations

- SD card required for persistence
    
- One reboot per power cycle to activate RTSP
    
- Factory mode may disable some Tuya cloud features
    
- Tested on LSC Article No. 3202092.2 only
    
- Other LSC/Anyka variants : untested (feedback welcome)
    

---

### 🤝 Contributing

- Tested on a different variant ?
    
- Found the **sub_ch** URL ?
    
- Found permanent RTSP without SD ?
    

**Open an issue or submit a PR!**

---

### 📜 License

MIT — Free to use, share, modify.

If this helped you → ⭐ **Star the repo**

---

## **FR — Version Française**

**LSC / Anyka / Tuya — RTSP 100% Local (Sans Cloud)**

**Par HeartyGFX (Watson) et Holmes — Avril 2026**

---

### Résumé

Certaines caméras LSC Smart Connect basées sur une puce Anyka et un firmware Tuya sont entièrement verrouillées sur le cloud.

Ce dépôt documente une méthode permettant d'obtenir un flux vidéo RTSP 100% local, sans cloud, sans application officielle, sans clé API.

**Statut actuel :**

- **Flux principal RTSP :** validé sur 2 caméras sur 2
    
- **Flux secondaire :** non validé à ce stade, contributions bienvenues
    
- **Répétabilité :** 2 sur 2 confirmées, 3e test en cours
    

**Prérequis importants :**

- La caméra doit obligatoirement être configurée une première fois via l'application LSC Connect ou Smart Connect afin de rejoindre le réseau WiFi local.
    
- Ce hack ne remplace pas cette étape initiale.
    

---

### ⚠️ Avertissement

- Ce hack est fourni à des fins éducatives et pour usage personnel uniquement.
    
- Utilisez-le uniquement sur des appareils qui vous appartiennent.
    
- Aucune garantie.
    
- Vous êtes responsable de l'usage que vous en faites.
    

---

### 🙏 Crédits et travaux antérieurs

Ce travail s'appuie sur les recherches et outils suivants :

**[Guino / LSCOutdoor1080P](https://github.com/guino/LSCOutdoor1080P) : Méthode d'exécution via carte SD et approche de base pour le hack.**

**[MuhammedKalkan / Anyka-Camera-Firmware](https://github.com/MuhammedKalkan/Anyka-Camera-Firmware) : Documentation du firmware Anyka et piles RTSP alternatives.**
    

**Contribution originale de ce dépôt :**

- Découverte du fichier déclencheur **/tmp/_ak39_factory.ini** activant le mode usine et le serveur RTSP dormant
    
- Méthode sans patch binaire, compatible avec les versions de firmware non patchables
    
- Documentation des contraintes de compatibilité carte SD (MBR vs GPT)
    

---

### 📋 Matériel testé

|Champ|Valeur|
|---|---|
|**Marque**|LSC Smart Connect Outdoor Camera (Action)|
|**Référence**|Article No. 3202092.2|
|**Puce**|Anyka AK3918EV330|
|**Firmware**|Tuya IPC SDK 4.9.18|

---

### 🔑 La découverte clé

Le firmware Anyka/Tuya contient un serveur RTSP dormant compilé dans le binaire principal.

Il est activé par la création du fichier **/tmp/_ak39_factory.ini** au moment de l'exécution.

Ce fichier déclenche le mode test usine, ce qui démarre le serveur RTSP sur le port **554**.

---

### 📦 Prérequis

- Caméra LSC Smart Connect configurée via l'application officielle (WiFi rejoint)
    
- Carte SD de 4 Go à 32 Go
    
- PC Windows avec client Telnet activé
    
- VLC Media Player
    

#### ⚠️ Compatibilité carte SD — Point critique

Les cartes SD modernes sont formatées en GPT par défaut et sont rejetées par la caméra.

**Il faut impérativement une table de partition MBR.**

**Méthode Windows via diskpart — ouvrir CMD en Administrateur :**

DOS

```
diskpart
list disk
select disk X  
clean
convert mbr
create partition primary
format fs=fat32 quick
active
exit
```

_(Remplacez X par le numéro de votre carte SD)_

**Explication des commandes :**

- **clean :** efface tout, y compris la table de partition
    
- **convert mbr :** est l'étape cruciale, Anyka rejette le GPT
    
- **active :** marque la partition comme démarrable
    

---

### 📁 Structure de la carte SD

Copier à la racine de la SD :

- **hack.sh** — original du dépôt Guino, ne pas modifier
    
- **custom.sh** — notre script
    

_Les deux fichiers doivent être encodés en UTF-8 avec des fins de ligne LF et non CRLF._

**📄 hack.sh**

Utiliser le fichier **hack.sh** original et non modifié issu du dépôt de Guino.

**Ne pas le modifier.**

---

### 🚀 Procédure étape par étape

**Étape 1 — Préparer la carte SD**

- Formater en FAT32 avec table MBR
    
- Copier **hack.sh** et **custom.sh** à la racine
    
- Vérifier les fins de ligne : **LF uniquement**
    

**Étape 2 — Premier démarrage**

- Insérer la SD dans la caméra éteinte
    
- Allumer la caméra
    
- **Attendre 60 secondes**
    

**Étape 3 — Activer le client Telnet Windows**

- Panneau de configuration
    
- Programmes
    
- Activer ou désactiver des fonctionnalités Windows
    
- Cocher **Client Telnet**
    

**Étape 4 — Connexion Telnet**

Depuis CMD Windows :

DOS

```
telnet 192.168.1.100 24
```

_(Remplacer 192.168.1.100 par l'IP de votre caméra). Vous obtenez un shell root direct._

**Étape 5 — Activer le RTSP**

Depuis le shell Telnet, créer le fichier déclencheur et redémarrer le processus principal (utiliser les commandes fournies dans la section EN).

**⚠️ La caméra va parler et redémarrer. C'est normal.**

**Étape 6 — Reconnexion après reboot**

- Attendre 90 secondes puis se reconnecter :
    

DOS

```
telnet 192.168.1.100 24
```

- Vérifier que le port 554 est actif :
    

Bash

```
netstat -tuln
```

**Étape 7 — Flux vidéo dans VLC**

- Media
    
- Ouvrir un flux réseau
    
- Entrer : **rtsp://192.168.1.100:554/main_ch**
    

🎉 **Flux vidéo local en direct, sans cloud.**

---

### 📡 URLs RTSP

|Flux|URL|Statut|
|---|---|---|
|**Principal HD**|`rtsp://IP_CAM:554/main_ch`|Validé|
|**Secondaire SD**|`rtsp://IP_CAM:554/sub_ch`|Non validé|

---

### 🔍 Détails techniques

|Élément|Valeur|
|---|---|
|**Binaire principal**|`/usr/bin/anyka_ipc`|
|**Fichier déclencheur**|`/tmp/_ak39_factory.ini`|
|**Port RTSP**|554|
|**Symboles clés**|`ht_rtsp_start`, `ht_rtsp_stop`|

**Point de montage :**

- **/usr (squashfs) :** Lecture seule
    
- **/etc/config (jffs2) :** Lecture-écriture persistant
    
- **/tmp (tmpfs) :** Perdu au reboot
    

**Pourquoi le patch de Guino ne fonctionne pas ici :**

- **MD5 attendu :** `5ac1f462bf039ec3c6c0a31d27ae652a` (version 2.10.36)
    
- **MD5 réel :** `36849ada9f7fc1e6ea27a986cfbee8d0` (version antérieure)
    
- Notre méthode contourne complètement ce problème.
    

---

### ✅ Logiciels compatibles

- VLC Media Player
    
- Frigate (NVR)
    
- Home Assistant via intégration Generic Camera
    
- Blue Iris
    
- Shinobi
    
- FFmpeg
    

---

### ⚠️ Limitations connues

- Carte SD requise pour la persistance
    
- Un reboot par cycle d'alimentation pour activer le RTSP
    
- Le mode usine peut désactiver certaines fonctions Tuya cloud
    
- Testé uniquement sur LSC Article No. 3202092.2
    
- Autres variantes LSC/Anyka : non testées (feedback welcome)
    

---

### 🤝 Contribuer

- Testé sur une autre variante ?
    
- Trouvé l'URL **sub_ch** ?
    
- Trouvé comment rendre le RTSP permanent sans SD ?
    

**Ouvrir une Issue ou soumettre une PR.**

---

### 📜 Licence

MIT — Free to use, share, modify.

If this helped you → ⭐ **Star the repo**

---

> "Vise la lune, même si tu ne l'atteins pas, tu finiras parmi les étoiles." 🐦 _La Philosophie de l'Oiseau_

**HeartyGFX (Watson) & Holmes | 2026 🔍**
