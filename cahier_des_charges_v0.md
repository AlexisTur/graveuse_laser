# Cahier des charges — Graveuse laser connectée (V0.1, brouillon de travail)

> Statut : **V0.1 à débattre**. **[À DÉCIDER]** = réponse de l'équipe attendue ; **[À VÉRIFIER]** = hypothèse technique à confirmer (datasheet, schéma, essai).
> Contexte : projet de synthèse « Objets connectés » ESEO (T. Bouvier / R. Perdriau).
>
> Historique :
> - V0 : première proposition.
> - V0.1 : caméra = ESP32-CAM existant (surveillance seule) ; laser 5 W optique pour découpe de contreplaqué ; stratégie « PCB final + firmware de démonstration » ; ajout des données du manuel MKS DLC32 V2.
> - V0.2 : électronique conçue librement (DLC32 = simple référence) ; c'est le firmware open source qu'on adapte (fichier machine + code personnalisé Grbl_Esp32) ; liste des 7 contraintes de compatibilité.

---

## 1. Contexte et objectifs

### 1.1 Objectif produit (long terme)
Graveuse/découpeuse laser à diode, construite sur la mécanique d'une imprimante 3D récupérée. Elle est pilotée par une **carte conçue par l'équipe**, fonctionnellement inspirée de la **MKS DLC32 V2** (machine de référence : vidéo YouTube = DLC32 + écran TS35 + TMC2209 + laser 5 W + pompe d'aquarium + éclairage). Elle est contrôlable :
- en local, par un écran tactile ;
- à distance, depuis un téléphone (Wi-Fi), avec une caméra de surveillance ;
- à terme, depuis **LaserGRBL**.

### 1.2 Découpage en deux phases — principe directeur

| | **Phase 1 — cours (évaluée)** | **Phase 2 — après le cours** |
|---|---|---|
| Matériel | **PCB final et complet** : tout le hard de la machine, tel qu'il servira en phase 2 | Inchangé (éventuelle révision B) |
| Firmware | **Firmware de démonstration maison** : simple, modulaire, pilote chaque périphérique et tout se commande à distance. **Pas besoin d'être compatible LaserGRBL.** | Firmware open source (Grbl_Esp32 / FluidNC / firmware MKS) + nos blocs logiciels réutilisables |
| Évaluation | PCB, code ESP32, communication sans fil, tests unitaires | — |

**Conséquence majeure** : le PCB doit être conçu dès maintenant pour la phase 2. Le brochage doit donc rester compatible avec les firmwares open source (voir §2.2), même si notre code de cours ne les utilise pas.

### 1.3 Livrables évalués (phase 1)
1. **Synoptique** validé par les encadrants (livrable intermédiaire obligatoire).
2. **Schéma électrique + PCB** (fabriqué en externe), monté et testé.
3. **Firmware ESP32 de démonstration** + **interface téléphone en Wi-Fi**.
4. **Rapport avec les résultats des tests unitaires** (obligatoire).

### 1.4 Contraintes du cours à lever tôt
| Contrainte | Situation | Action |
|---|---|---|
| Travail en trinôme « sauf exception » | Vous êtes 2 | Faire valider le binôme **[À DÉCIDER]** |
| Protocole sans fil validé par TB/RP | Wi-Fi + WebSocket | Faire valider avec l'argumentaire du §5 |
| « Solution économe en énergie » | Machine sur secteur 24 V | Traitée explicitement au §6.6 |
| Laser classe 4 en salle de TP | Probablement interdit ou encadré | Mode démo sans tir laser réel (§7) |

---

## 2. Choix structurants

### 2.1 Architecture microcontrôleurs — **DÉCIDÉ : option C (caméra = ESP32-CAM existant)**
- **ESP32-WROOM-32E** sur notre PCB = contrôleur machine (même rôle que la DLC32).
- **ESP32-CAM** (déjà en stock, avec sa carte de programmation MB) = caméra de **surveillance seule**. Il **diffuse lui-même son flux en Wi-Fi** vers le téléphone.

Pourquoi ne pas faire transiter la vidéo par l'ESP principal en série (idée « petit IC caméra + SD → UART → ESP ») :
- Débit : un flux VGA MJPEG ≈ 2–5 Mbit/s. Un UART fiable entre deux cartes plafonne autour de 1–2 Mbit/s. → ça ne passe pas.
- Charge : l'ESP principal devrait recevoir puis réémettre la vidéo. Cela consomme son CPU, sa RAM et le Wi-Fi au détriment de la génération des pas moteurs, ce qui entraîne des défauts de gravure.
- L'ESP32-CAM sait déjà streamer en Wi-Fi (exemple officiel `CameraWebServer`). C'est la solution la plus simple et la plus robuste.

Intégration sur notre PCB (l'ESP32-CAM n'est pas soudé : il est enfiché ou relié par un câble court) :
| Liaison | Rôle |
|---|---|
| 5 V commuté par MOSFET (commandé par l'ESP principal) | **Couper la caméra quand personne ne regarde** → argument « économe en énergie » |
| UART (2 fils) ESP principal ↔ ESP32-CAM | Commandes : marche/arrêt du flux, LED flash, état, adresse IP |
| GND commun | — |

**Une seule page sur le téléphone — DÉCIDÉ : servie par l'ESP principal.**
La page (widgets de commande + état des capteurs + vidéo) est servie par l'ESP principal. La vidéo y est intégrée par une simple balise `<img src="http://<ip-cam>:81/stream">` : c'est **le navigateur du téléphone** qui va chercher le flux directement chez l'ESP32-CAM. Les deux ESP n'échangent jamais la vidéo entre eux. Une balise `<img>` peut afficher une image venant d'une autre adresse sans configuration particulière (pas de blocage CORS pour une image).

Pourquoi pas l'inverse (page servie par l'ESP32-CAM, qui relaierait les ordres en UART) :
- la commande de la machine dépendrait de la caméra : caméra coupée ou plantée = plus d'interface. Or on veut justement pouvoir couper la caméra pour économiser l'énergie ;
- l'ESP32-CAM est déjà le plus chargé (compression JPEG + envoi du flux) ;
- les ordres feraient un détour Wi-Fi → ESP32-CAM → UART → ESP principal : plus de latence, un point de panne de plus, et le contrôleur machine ne serait plus maître de sa propre interface.

**Scénario d'allumage de la caméra :**
1. Le téléphone affiche la page de l'ESP principal (caméra éteinte, 0 mA).
2. L'utilisateur appuie sur « Caméra » → message WebSocket `{"cmd":"cam","on":true}`.
3. L'ESP principal ferme le MOSFET d'alimentation 5 V de l'ESP32-CAM.
4. L'ESP32-CAM démarre et se connecte au réseau Wi-Fi (voir §5), en **client**. Elle lance son serveur de flux.
5. L'ESP32-CAM envoie « prête + son adresse IP » à l'ESP principal par l'UART.
6. L'ESP principal transmet l'adresse à la page par WebSocket ; la page insère la balise `<img>`. Durée attendue : quelques secondes **[À MESURER]**.
7. Arrêt : bouton « Caméra off », fermeture de la page, ou délai d'inactivité → l'ESP principal coupe le MOSFET.

Rôle exact de l'UART (pas de vidéo dessus) : signalisation seulement. Adresse IP de la caméra, « je suis prête », état, LED flash, résolution. Elle permet à l'ESP principal de savoir que la caméra fonctionne **sans dépendre du Wi-Fi**. Avec le mode client retenu (§5), l'UART est **indispensable** : l'adresse IP de la caméra n'est pas connue d'avance, et c'est aussi par l'UART que l'ESP principal donne à la caméra le nom et le mot de passe du réseau.

Points de vigilance électriques :
- **Alimentation fantôme** : quand l'ESP32-CAM est coupée, la broche TX de l'ESP principal (au niveau haut au repos) l'alimente partiellement par ses diodes de protection internes. Il faut une résistance série (~1 kΩ) sur chaque fil UART **et** passer la broche TX en entrée quand la caméra est coupée.
- **Commutation côté +5 V** (interrupteur « high-side » : MOSFET P ou load switch), pas côté masse. Couper la masse d'une carte reliée par l'UART ferait circuler le courant de retour par les fils de données.
- Pics de courant de l'ESP32-CAM au démarrage du Wi-Fi : condensateur local de quelques centaines de µF côté caméra **[À VÉRIFIER sur datasheet]**.
- L'ESP32-CAM a très peu de broches libres (la plupart sont prises par la caméra et la SD). Il faut identifier les 2 broches de l'UART **[À VÉRIFIER]**. La SD de l'ESP32-CAM ne sera pas utilisée : on garde la SD de la carte principale.
- L'exemple `CameraWebServer` ne sert en pratique le flux qu'à **un seul spectateur à la fois** **[À VÉRIFIER]**. Ça suffit pour la surveillance depuis un téléphone.

### 2.2 Conception électronique libre + firmware open source adapté — **DÉCIDÉ**
La DLC32 n'est qu'une **référence fonctionnelle**. On conçoit notre propre électronique (brochage, choix des composants, architecture) et **c'est le firmware open source qu'on adapte à notre carte**, pas l'inverse.

C'est possible simplement avec Grbl_Esp32 (vérifié dans le code présent dans `Grbl_Esp32-main/`) :

| Ce qu'on adapte | Où / comment | Effort |
|---|---|---|
| **Brochage** (step/dir, fins de course, laser, SD, sondes…) | Un fichier de définition machine `src/Machines/notre_carte.h` : une ligne `#define X_STEP_PIN GPIO_NUM_xx` par signal (voir `mpcnc_laser_module_v1p2.h`, ~40 lignes) | Faible |
| **Pas/mm, vitesses, accélérations, courses, micro-pas** | Paramètres `$100`, `$110`, `$120`, `$130`…, modifiables **sans recompiler**, même depuis LaserGRBL. Valeurs par défaut dans le fichier machine (`DEFAULT_X_STEPS_PER_MM`…) | Nul |
| **Drivers TMC2209 en UART** (courant réglé par logiciel) | Pris en charge (`TrinamicUartDriver`, exemple `TMC2209_4x.h`) | Faible |
| **Notre écran, LEDs, caméra, mesure d'énergie, sécurité** | Points d'extension prévus par Grbl_Esp32 : `machine_init()`, `display_init()` (exemple `Custom/oled_basic.cpp` : une tâche FreeRTOS qui lit l'état de Grbl et l'affiche), `user_defined_macro()`, `user_m30()`… déclarés via `CUSTOM_CODE_FILENAME` / `DISPLAY_CODE_FILENAME` | Moyen — **c'est là que nos blocs de phase 1 viennent se brancher** |

Remarque : Grbl_Esp32 est archivé au profit de **FluidNC**, où le brochage se décrit dans un fichier YAML sans recompiler. FluidNC est moins pratique pour greffer du code personnel **[À VÉRIFIER]**. Le choix Grbl_Esp32 / FluidNC se fera en phase 2.

**Les vraies contraintes de compatibilité** (ce que le PCB doit respecter, et rien d'autre).
Niveau de preuve :
- **Grbl_Esp32** : vérifié dans le code local.
- **FluidNC** : seulement en partie (documentation communautaire et notes de version ; le wiki officiel n'était pas joignable).

On retient l'**intersection des deux**, c'est-à-dire le choix le plus prudent, qui laisse les deux firmwares possibles.
1. **µC = ESP32 classique** (WROOM-32E).
   - Grbl_Esp32 : ESP32 classique uniquement.
   - FluidNC : l'ESP32-S3 est pris en charge depuis la pré-version v4.0.0. Le fonctionnement des moteurs sur S3 n'est pas documenté.
   - → L'ESP32 classique est le seul choix compatible avec les deux.
2. **STEP/DIR des moteurs** sur des **GPIO directs** *ou* sur une chaîne de **74HC595 pilotée par l'I2S** (`I2S_OUT_BCK/WS/DATA`). Pas d'expandeur I2C/SPI pour les pas (trop lent, non pris en charge).
   - FluidNC : modes de pas RMT et TIMED (GPIO natifs), I2SO_STATIC et I2SO_STREAM (matériel I2S externe) → compatible.
   - FluidNC accepte en plus des **extensions d'E/S lentes** (ex. carte « Airedale » reliée en UART) **[À VÉRIFIER pour I2C/PCA95xx]**.
   - Contraintes 3 à 7 : non vérifiées pour FluidNC **[À VÉRIFIER]**.
3. **Drivers** : step/dir standard (A4988, TMC2209 autonome) ou **TMC2209 en UART** / TMC2130-5160 en SPI.
4. **Laser** sur une sortie GPIO capable de PWM (toutes les sorties de l'ESP32 le sont via LEDC), hors broches de strapping.
5. **Carte SD sur bus SPI** (broche CS configurable).
6. **Fins de course / flamme / arrêt d'urgence** sur des entrées GPIO (34–39 utilisables, avec pull-up externe).
7. Une **liaison USB-série sur l'UART0** (GPIO1/3) pour LaserGRBL en filaire.

Tout le reste est **libre et à concevoir par nous** : alimentation et protections, étages de puissance, choix et brochage de l'écran, gestion des sorties lentes (pompe, LEDs, buzzer, alim caméra), mesure d'énergie, sécurité matérielle, connectique, placement-routage.

Libertés de conception par rapport à la DLC32 (exemples à trancher en revue de schéma) :
| Fonction | DLC32 | Nous (proposition) |
|---|---|---|
| Drivers | TMC2209 autonomes, micro-pas par DIP | **TMC2209 en UART** : courant et micro-pas réglés par logiciel, diagnostic, homing sans capteur possible |
| Écran | TS35 imposé (EXP1/EXP2) | **Écran choisi par nous** (ex. 3,5" ILI9488 SPI + tactile capacitif FT6236 en I2C) et notre propre IHM |
| Sorties lentes | Sur 74HC595 | 74HC595 en I2S *ou* expandeur I2C (PCA9555) **[À DÉCIDER selon le bilan GPIO]** |
| Sécurité | Entrée « probe » = flamme | Chaîne clé + arrêt d'urgence + réarmement (relais) coupant le laser, flamme analogique |
| Mesure | — | INA226 |
| Caméra | — | Alimentation commutée + UART vers ESP32-CAM |

Action suivante : faire le **bilan GPIO** de notre propre carte (table de brochage) en respectant les 7 contraintes.

### 2.3 Stratégie firmware de cours (phase 1)
Le code est organisé en **deux couches** :
1. **Blocs pilotes (HAL)**, un par périphérique, chacun avec son programme de test :
   - moteurs, laser, pompe, LEDs ;
   - écran + tactile, SD ;
   - capteurs (flamme, chaîne de sécurité OK, fins de course, INA226) ;
   - liaison avec l'ESP32-CAM ;
   - Wi-Fi.

   Ces blocs **servent aux tests unitaires exigés par le cours** et prouvent que le PCB fonctionne.
2. **Application de démonstration** :
   - interface tactile ;
   - interface web téléphone ;
   - machine d'états sécurité ;
   - gestion de la veille ;
   - quelques mouvements simples (jog, prise d'origine, tracé d'un carré).

   Pas d'interpréteur G-code complet.

Ce qui sera **réutilisé en phase 2** (soyons honnêtes) :
| Bloc | Réutilisé ? |
|---|---|
| Moteurs, laser, SD, Wi-Fi de base | Non : le firmware open source les gère déjà. Ils auront servi à valider le PCB. |
| Supervision sécurité (flamme, chaîne de sécurité, watchdog) | Oui, en complément |
| Mesure d'énergie + gestion de veille | Oui (absent des firmwares open source) |
| Pilotage caméra, LEDs, pompe | Oui |
| Interface web téléphone (page + caméra) | Oui, en partie (adaptable à l'API du firmware open source) |
| Interface tactile (IHM écran) | **Oui** : branchée sur `display_init()` de Grbl_Esp32 ; elle lit l'état machine et envoie des commandes au lieu d'appeler nos pilotes |

Règle de conception pour que ce soit vrai : **l'IHM et l'interface web ne parlent jamais directement aux moteurs ou au laser**. Elles passent par une interface « contrôleur machine » (ex. `machine_jog()`, `machine_get_state()`). En phase 1, cette interface est notre code de démo ; en phase 2, elle appelle Grbl.

Ce que vous écrivez vous-mêmes reste largement suffisant pour être évalué sur le code. Utiliser des bibliothèques (LVGL ou TFT_eSPI, FATFS, serveur web) reste votre code applicatif. Le cours a d'ailleurs comme objectif explicite de « s'approprier et réutiliser du code existant ».

Framework : **Arduino-ESP32 sous PlatformIO** (même environnement que Grbl_Esp32/FluidNC/MKS, donc fusion plus facile en phase 2). Il donne aussi accès aux fonctions FreeRTOS et ESP-IDF. **[À DÉCIDER : faire valider par les encadrants, la présentation cite ESP-IDF]**.

### 2.4 Mécanique — **[À DÉCIDER : modèle exact de l'imprimante]**
- On garde : châssis, moteurs NEMA17, courroies, poulies, fins de course, alimentation 24 V.
- On retire : carte mère, hotend, extrudeur, plateau chauffant (remplacé par un nid d'abeille ou une tôle, indispensable pour la découpe).
- Bedslinger type Ender 3 : axe Y lourd → vitesse raster limitée. Sans gravité pour la **découpe**, qui se fait lentement.
- **Axe Z conservé** : réglage de la focale. En découpe multi-passes, on peut descendre le focus à chaque passe (gain réel sur 3 mm).

### 2.5 Laser — **DÉCIDÉ : module diode 5 W optique, entrée TTL PWM**
- Usage : gravure bois/cuir/carton + **découpe de contreplaqué ≈ 3 mm en plusieurs passes** (ordre de grandeur : 3 à 6 passes à 100–200 mm/min avec air assist) **[À VÉRIFIER par essais]**. Le 4–5 mm est possible mais lent et carbonise les bords.
- Consommation électrique ≈ 20–40 W (rendement de diode ~25 %) **[À VÉRIFIER selon le module acheté]**.
- Interface identique à la DLC32 (manuel §VII) :
  - connecteur 3 broches **V / G / TTL** ;
  - TTL = PWM 0–5 V, 1 kHz par défaut, réglable de 1 à 10 kHz.
- Option : connecteur 2 broches « 12–24 V PWM » de la DLC32 (pour lasers sans entrée TTL) **[À DÉCIDER]**.
- La découpe de contreplaqué dégage beaucoup de fumée (colles) : **extraction des fumées obligatoire** en usage réel (hors cours).

---

## 3. Exigences fonctionnelles

**M** = indispensable pour l'évaluation, **S** = devrait, **C** = bonus, **W** = phase 2.

| ID | Fonction | Prio | Critère de validation |
|---|---|---|---|
| F1 | Piloter 3 axes pas-à-pas (X, Y, Z) | M | Déplacement de 100 mm mesuré, erreur < 0,2 mm |
| F2 | Prise d'origine sur fins de course X/Y/Z | M | Répétable ±0,1 mm sur 10 essais |
| F3 | Commander le laser en PWM 0–100 % (1–10 kHz) | M | Rapport cyclique vérifié à l'oscilloscope ; laser éteint au boot/reset |
| F4 | Commander la pompe d'air assist (ON/OFF) | M | Commutation mesurée sous charge |
| F5 | Commander l'éclairage LED (PWM, gradation) | S | 0–100 % visible |
| F6 | Écran tactile : état machine, jog, laser, pompe, LEDs, réglages | M | Toutes les pages navigables, réponse au toucher < 200 ms |
| F7 | Lire/écrire des fichiers sur micro-SD (FAT32) | M | Écriture d'un journal + lecture d'un fichier |
| F8 | **Tout piloter à distance** depuis un téléphone en Wi-Fi | M | Android + iOS, sans installation, latence de commande < 200 ms |
| F9 | Afficher la caméra de surveillance sur le téléphone | M | ≥ 5 img/s en VGA, latence < 1 s |
| F10 | Allumer/éteindre la caméra à distance (économie d'énergie) | S | Conso caméra = 0 mA quand coupée |
| F11 | Mesurer la consommation (tension/courant 24 V) | S | ±5 % vs multimètre ; affichée à l'écran et sur le téléphone |
| F12 | Détection flamme → arrêt laser ; arrêt d'urgence → coupure matérielle du laser avec réarmement | M (sécurité) | Coupure < 100 ms ; pas de redémarrage au déverrouillage de l'AU |
| F13 | Mode veille économe | M (cours) | Conso en veille mesurée et documentée |
| F14 | Tracé démo (carré, cercle) depuis l'écran ou le téléphone | S | Tracé visible au crayon ou au laser basse puissance |
| F15 | Upload d'un fichier vers la SD depuis le téléphone | C | Fichier de 1 Mo transféré |
| F16 | Gravure de fichiers G-code / compatibilité LaserGRBL (USB + Wi-Fi) | W | Image gravée depuis LaserGRBL |

---

## 4. Architecture matérielle

### 4.1 Synoptique (texte, à transformer en schéma pour le livrable)
```
24 V (alim imprimante) ─► Fusible + anti-inversion + TVS ─┬─► Drivers moteurs ×3 (VM)
                                                           ├─► Connecteur laser V/G/TTL
                                                           ├─► MOSFET ─► pompe air assist
                                                           ├─► MOSFET (PWM) ─► LEDs éclairage
                                                           ├─► INA226 (mesure U/I, I2C)
                                                           └─► Buck 24→5 V 3 A ─┬─► LDO/Buck 3,3 V ─► ESP32, logique
                                                                                ├─► Écran tactile 3,5" (choix §4.3)
                                                                                └─► MOSFET ─► ESP32-CAM (5 V)
ESP32-WROOM-32E
  ├─ STEP/DIR X,Y,Z : GPIO directs ou 74HC595 en I2S (contrainte firmware n°2)
  ├─ UART → TMC2209 ×3 (adressés)
  ├─ Sorties lentes (pompe, buzzer, alim caméra) : 74HC595 ou PCA9555 [À DÉCIDER]
  ├─ SPI → écran + micro-SD (CS séparés)  |  tactile : SPI ou I2C [À DÉCIDER]
  ├─ Entrées → fins de course X/Y/Z, capteur flamme, « chaîne de sécurité OK »
  ├─ PWM → laser TTL (buffer 5 V + pull-down), LEDs
  ├─ I2C → INA226 (+ extension)
  ├─ UART → ESP32-CAM (commandes)
  └─ USB (CH340 + reset auto) + HE10 8 pts (format ESEO) pour la programmation
Chaîne de sécurité (clé + arrêt d'urgence + réarmement → relais K1) : coupe matériellement le laser et la pompe, force le TTL laser à 0 et les drivers en désactivé, informe l'ESP32 (§6.1)
ESP32-CAM (module existant) ── Wi-Fi ──► téléphone (flux MJPEG)
```

### 4.2 Données du manuel MKS DLC32 V2 (référence, pas un modèle à copier)
| Élément DLC32 | Valeur | Ce qu'on en fait |
|---|---|---|
| Alimentation | 12–24 V, 5 A max, fusible 10 A, jack DC 2,1 mm | **24 V** ; bornier à vis ou XT30 plutôt qu'un jack (un jack 5 A est juste) ; fusible dimensionné au §6.2 |
| Indicateurs | LEDs VIN, 5 V, 3,3 V | Repris (utile pour les tests unitaires) |
| USB | CH340, programmation + pilotage en ligne | Repris (LaserGRBL en filaire en phase 2) + HE10 ESEO |
| Carte TF | FAT32, .nc/.gc/.gcode | Repris |
| Reset externe | Sert d'arrêt d'urgence | Repris, **plus** coupure matérielle du laser |
| Drivers | Supports stepstick 8+8 broches, micro-pas par DIP M0/M1/M2, connecteur pour driver externe (TB6600/TMC2160) | Supports + **TMC2209 en mode autonome** (micro-pas par DIP, comme la DLC32). Driver externe : **W** |
| Écran | EXP1/EXP2, compatible TS24/TS35 uniquement | Repris à l'identique pour utiliser le **TS35** |
| Laser | 3 broches V/G/TTL (0–5 V, 1–10 kHz) ; 2 broches 12–24 V PWM | 3 broches obligatoire ; 2 broches en option |
| Broche « probe » | Utilisable en détection de flamme | Reprise pour le capteur de flamme |
| I2C | SCL réutilisable en sortie « cooler » ; SDA sert à forcer le mode programmation | Repris (INA226 dessus) — attention au strapping de SDA **[À VÉRIFIER]** |
| Buzzer | Partagé avec celui de l'écran | Repris |
| Spindle CNC | Sortie 0–24 V | **Non repris** (inutile pour un laser), ou recyclé en sortie pompe |

Le manuel ne contient **pas le schéma** : il faut récupérer celui publié par Makerbase sur GitHub.

### 4.3 Choix composants (proposition)
| Bloc | Choix | Justification |
|---|---|---|
| Drivers | Supports stepstick + **TMC2209 en UART** | Remplaçables en proto, silencieux, réglage logiciel ; pris en charge par Grbl_Esp32 |
| Extension de sorties | 74HC595 via I2S **ou** PCA9555 en I2C **[À DÉCIDER]** | Selon le bilan GPIO (§2.2) |
| Écran | Écran SPI 3,5" (ILI9488) + tactile **[À DÉCIDER : résistif XPT2046 sur SPI ou capacitif FT6236 sur I2C]** | Choisi et intégré par nous ; IHM maison réutilisable en phase 2 |
| Pompe / LEDs / alim caméra | MOSFET N logique (AO3400 ou équivalent) + diode de roue libre pour la pompe | Standard, robuste |
| Sortie laser | Buffer 5 V (74AHCT1G125 ou transistor) + pull-down | Le module attend 0–5 V ; état bas garanti au démarrage |
| Mesure d'énergie | INA226 + shunt sur le 24 V | Capteur numérique → « interfacer des capteurs » + preuve de sobriété |
| Flamme | Module capteur IR de flamme (sortie analogique + numérique) | Capteur analogique demandé par le cours + vraie sécurité |
| Alimentation | Buck 24→5 V 3 A + régulateur 3,3 V | ESP32 + ESP32-CAM : pics Wi-Fi ~500 mA chacun ; TS35 avec rétroéclairage |

### 4.4 Bilan électrique 24 V (estimation) **[À VÉRIFIER]**
| Charge | Puissance |
|---|---|
| Laser 5 W optique | 20–40 W |
| 3 moteurs NEMA17 | ~15–25 W |
| Pompe d'aquarium | 3–10 W |
| LEDs | 5–10 W |
| Logique + écran + 2 ESP | ~5 W |
| **Total** | **≈ 50–90 W → 2–4 A sous 24 V** (l'alimentation d'imprimante, ~350 W, est largement suffisante) |

---

## 5. Communication sans fil

**Choix : Wi-Fi 2,4 GHz.**
- Nécessaire pour la vidéo, qui est hors de portée du BLE.
- Le téléphone sait le faire sans installer d'application.
- Réutilisé en phase 2 (LaserGRBL en Wi-Fi).

| Aspect | Proposition |
|---|---|
| Mode réseau | **DÉCIDÉ : mode client (STA)** : les deux ESP rejoignent un réseau existant (box ou partage de connexion du téléphone). Point d'accès (AP) seulement pour la première configuration et en secours. Détails ci-dessous. |
| Accès | `http://graveuse.local` (mDNS) **[À VÉRIFIER sur Android]**, sinon adresse IP affichée sur l'écran tactile |
| Interface | **Page web servie par l'ESP32** → rien à installer, Android + iOS |
| Commandes + état temps réel | **WebSocket** (explication ci-dessous) avec messages JSON |
| Vidéo | Flux MJPEG direct depuis l'ESP32-CAM |
| Sécurité | Réseau WPA2 + mot de passe d'accès à l'interface ; AP de configuration protégé par mot de passe |

**Choix du mode réseau — DÉCIDÉ : B (mode client).**

| | A. L'ESP principal crée le réseau (AP) | B. Les deux ESP rejoignent un réseau existant (STA) : box ou partage de connexion du téléphone |
|---|---|---|
| Le téléphone garde Internet | **Non** (il est connecté à un réseau sans Internet ; Android/iOS peuvent même s'en déconnecter tout seuls) | **Oui** (par la box, ou par les données mobiles en partage de connexion) |
| Marche sans box (salle de TP, atelier) | Oui | Oui avec le partage de connexion du téléphone |
| Chemin de la vidéo | Caméra → **radio de l'ESP principal** → téléphone : en Wi-Fi, tout le trafic d'un point d'accès passe par lui. La vidéo occupe la radio et le cœur 0 de l'ESP principal | Caméra → box/téléphone → téléphone : **l'ESP principal n'est pas traversé** |
| Risque sur la gravure | Faible si la génération des pas est matérielle (RMT/I2S) sur le cœur 1, mais à **mesurer** (gigue des pas avec flux actif) | Aucun |
| Adresses IP | Connues d'avance (192.168.4.x) | Inconnues → la caméra les communique par l'UART (§2.1) |
| Configuration | Aucune | Saisir le nom et le mot de passe du réseau (page de configuration) |

Raisons du choix B : le téléphone **garde son accès à Internet** pendant l'utilisation, et la vidéo ne traverse plus l'ESP principal (aucun risque sur la génération des pas).

Conséquences à traiter :
1. **Première configuration (ou réseau introuvable)** : l'ESP principal ouvre un point d'accès de configuration (« Graveuse-Setup ») avec une page de saisie du nom du réseau et du mot de passe, enregistrés en mémoire flash. Principe classique (bibliothèque type WiFiManager). On peut aussi saisir le réseau sur l'écran tactile.
2. **L'ESP32-CAM n'est configurée nulle part** : l'ESP principal lui transmet le nom et le mot de passe du réseau **par l'UART** à chaque démarrage. Un seul point de configuration, et l'UART est pleinement justifiée.
3. **Trouver l'ESP principal depuis le téléphone** (adresse IP attribuée par la box, donc inconnue) : nom `graveuse.local` (mDNS : fonctionne sur iOS et PC, support Android variable **[À VÉRIFIER]**). En secours, l'adresse IP s'affiche sur l'écran tactile. Sur une box, on peut aussi réserver une adresse fixe.
4. **Trouver la caméra** : elle envoie son adresse IP à l'ESP principal par l'UART, qui la transmet à la page (§2.1).
5. **Réseau de l'école** : un réseau d'entreprise (type eduroam, identifiant + mot de passe personnels) est difficile à utiliser depuis un ESP32, et ces réseaux isolent souvent les appareils entre eux. Le téléphone ne pourrait alors pas joindre la graveuse. **En TP et en soutenance, on utilise le partage de connexion du téléphone.** **[À VÉRIFIER : le téléphone joint bien les appareils connectés à son propre partage, Android et iOS]**
6. **Secours** : si aucun réseau connu n'est trouvé après ~20 s, retour au point d'accès de configuration. La graveuse reste toujours pilotable.

**Le WebSocket en bref.**
- En HTTP classique, le téléphone pose une question, l'ESP répond, et la connexion se ferme. Pour afficher la position en direct, il faudrait reposer la question 10 fois par seconde.
- Un WebSocket est une **connexion qui reste ouverte**, comme un appel téléphonique. Les deux côtés peuvent envoyer un message à tout moment :
  - téléphone → ESP : `{"cmd":"jog","axe":"X","mm":10}` ;
  - ESP → téléphone, 10 fois par seconde : `{"x":42.0,"y":10.5,"laser":0,"flamme":false,"P":31.2}`.
- Alerte flamme : l'ESP **pousse** l'info immédiatement, sans attendre que le téléphone la demande.
- C'est standard dans tous les navigateurs, disponible en bibliothèque pour ESP32, et c'est le mécanisme utilisé par l'interface web de Grbl_Esp32/FluidNC.

Alternatives écartées :
- HTTP seul avec interrogation périodique : plus simple, mais latence et trafic inutiles.
- MQTT : nécessite un serveur intermédiaire (broker), inutile pour un pilotage local. **[À DÉCIDER]** si les encadrants attendent un protocole IoT « classique ».
- Application native : trop long, sans valeur ajoutée pour le cours.

---

## 6. Exigences non fonctionnelles

### 6.1 Sécurité laser — non négociable
- **Machine ouverte, sans capot — DÉCIDÉ.** Conséquences : le faisceau réfléchi ou diffusé est dangereux pour les yeux dans toute la pièce, donc **lunettes adaptées obligatoires pour toute personne présente**. Écran de protection orange autour de la tête laser recommandé. Pas de tir réel en salle de TP.
- **Chaîne de sécurité matérielle — [À DÉCIDER]** (discussion reportée le 2026-11-20, après lecture de documentation).
  - Acquis : bouton AU à 2 contacts NF ; aucune reprise au simple déverrouillage (EN ISO 13850) ; l'ESP32 informe mais n'est pas la barrière de sécurité.
  - Variantes candidates :
    - **V5** : relais K1 à auto-maintien + bouton de réarmement, deux voies (schéma ci-dessous) ;
    - **V6** : contact AU 1 en coupure directe de l'alimentation laser + bascule (74LVC1G74) sur le /OE du buffer TTL + porte ET « PWM = 0 », réarmement depuis l'écran ;
    - **V7** : comme V6 sans la porte ET. Le dialogue « Réarmer ? » de l'écran envoie l'impulsion qui libère la bascule ; un ESP32 figé ne peut pas réarmer.
  - Schéma d'exemple (V5) :
  ```
  24 V ──[Clé]──[AU contact NF 1]──[Verrou. distant (cavalier)]──┬──[Réarmement NO]──┬── bobine relais K1 ── GND
                                                                 └───[contact K1]────┘   (auto-maintien)
  Voie 1 (puissance) : contact K1 ──► alimentation laser + pompe d'air assist
                       sortie K1  ──► pont diviseur ──► entrée « puissance laser présente » de l'ESP32 ; désactivation des drivers (EN)

  Voie 2 (commande, indépendante du relais) :
  5 V ──[R pull-up]──┬──[AU contact NF 2]── GND        (AU enfoncé → contact ouvert → niveau haut)
                     ├──► /OE du buffer TTL laser (74AHCT1G125) → sortie en haute impédance → pull-down → TTL = 0
                     └──► entrée « AU enfoncé » de l'ESP32 (via adaptation 3,3 V)
  ```
  - **Arrêt d'urgence** : bouton coup de poing Ø 22 mm à accrochage, **2 contacts NF** (ex. S208-031, 10 A 400 V AC). Contact 1 → voie 1 (relais), contact 2 → voie 2 (blocage du signal TTL).
  - **Pourquoi deux voies** (redondance + diversité) : chaque voie couvre une panne que l'autre ne voit pas.
    - *Contacts du relais soudés* (appel de courant des condensateurs du driver laser) : la voie 1 ne coupe plus rien, et l'ESP32 lit toujours « puissance présente ». Seule la voie 2, qui ne passe pas par le relais, arrête le faisceau.
    - *Condensateurs internes du module laser* : après ouverture du relais, ils peuvent alimenter la diode quelques ms à quelques centaines de ms **[À MESURER]**. La voie 2 coupe le TTL instantanément.
    - *ESP32 planté, firmware buggé ou en cours de flashage* : le PWM est généré par un périphérique matériel (LEDC) qui **continue à tourner même si le processeur est bloqué**. La surveillance logicielle par le pont diviseur ne sert alors à rien. En phase 2, le firmware est en plus un code open source que nous ne maîtrisons pas entièrement.
    - *Broche TTL court-circuitée au niveau haut* : la voie 1 coupe l'alimentation du laser.
  - **Coût de la voie 2 : quasi nul.** Le buffer 5 V du TTL est déjà prévu (§4.3) et le 74AHCT1G125 possède une entrée de validation /OE. On l'utilise au lieu d'ajouter une porte ET.
  - La surveillance par l'ESP32 reste utile, mais pour **l'information** (arrêter le travail, PWM à 0, alerte au téléphone, prise d'origine exigée), pas comme barrière de sécurité.
  - **Pas de doublement du relais ni du bouton de réarmement.** Un second relais identique tomberait en panne de la même façon que le premier (même technologie, même appel de courant). La voie 2, d'une autre technologie, offre une meilleure redondance. Le réarmement est une fonction de **démarrage**, pas d'arrêt : sa panne ne peut pas empêcher un arrêt.
  - **Détection de soudure du relais (logiciel, gratuit)** : si la voie 2 indique « AU enfoncé » alors que le pont diviseur lit encore « puissance présente », l'ESP32 signale « relais K1 soudé », refuse toute reprise et alerte le téléphone.
  - **Bouton de réarmement bloqué fermé** : le relais se recollerait dès le déverrouillage de l'AU. Parade : l'ESP32 lit aussi le bouton de réarmement et se met en défaut s'il le voit appuyé en permanence. Le PWM reste à 0 tant que l'utilisateur n'a pas relancé un travail.
  - **Clé et verrouillage à distance sur les deux voies** **[À DÉCIDER]** : dans le schéma ci-dessus, seul l'AU agit sur les deux voies. Si le relais est soudé, ouvrir la porte ou tourner la clé ne coupe plus rien. Proposition : interrupteur à clé à **2 contacts NF** (même format Ø 22 mm que l'AU) et connecteur de verrouillage à distance à **4 points** (2 boucles, double cavalier par défaut, contact de porte à 2 contacts plus tard).
  - **Le bouton ne coupe que la bobine du relais** (quelques dizaines de mA). Son calibre continu, non indiqué par le fabricant, n'a donc pas d'importance.
  - **Réarmement obligatoire** : déverrouiller l'arrêt d'urgence ne relance rien. Il faut appuyer sur « Réarmement » (EN ISO 13850). L'ESP32 exige ensuite une prise d'origine.
  - **Sécurité positive** : relais collé au repos. Un fil coupé fait retomber le relais.
  - **Interrupteur à clé** et **connecteur de verrouillage à distance** (shunté par un cavalier en l'absence de capot), demandés par l'IEC 60825-1 pour un appareil de classe 4 **[À VÉRIFIER : texte exact]**. Le connecteur permet d'ajouter plus tard un capot ou un contact de porte sans refaire le PCB.
  - **Restent alimentés** : ESP32, écran, caméra, extraction (pour évacuer la fumée et voir ce qui se passe).
  - Relais : bobine 24 V, contact ≥ 5 A DC **[À VÉRIFIER : courant réel du module laser, ~1 A attendu sous 24 V]**.
- Laser **éteint par défaut** : pull-down, broche hors strapping, PWM à 0 avant tout démarrage du Wi-Fi.
- **Watchdog** : plantage ou perte de connexion du téléphone pendant un tir → laser OFF.
- **Détection de flamme** obligatoire, puisque le pilotage à distance implique un fonctionnement sans surveillance directe. La caméra complète cette surveillance.
- Lunettes certifiées **EN 207** couvrant 445 nm, OD ≥ 4 (ne pas se fier aux lunettes vertes livrées avec les modules bon marché, dont la protection réelle est souvent insuffisante) **[À VÉRIFIER : références]**. Zone de travail contrôlée : personne sans lunettes dans la pièce pendant un tir.
- **Pas de protection de la peau nécessaire** en usage normal. Justification : l'œil concentre la lumière sur la rétine (son cristallin multiplie l'éclairement reçu par environ 100 000), alors que la peau ne concentre rien. Les seuils d'exposition admissibles sont donc bien plus élevés pour la peau : de quelques centaines à plusieurs milliers de fois selon la durée et la longueur d'onde (ordre de grandeur ANSI Z136.1 : ~200 mW/cm² pour la peau en exposition longue) **[À VÉRIFIER : valeurs exactes de la norme]**. Ordre de grandeur pour 5 W optiques : un reflet diffus sur du bois donne ~3 mW/cm² à 10 cm, très en dessous du seuil peau. Restent dangereux pour la peau : le **faisceau direct** (jamais les mains dans la zone pendant un tir) et le **reflet spéculaire** sur une surface brillante (métal poli, miroir, verre : à proscrire sans précautions).
- **Extraction des fumées** (ventilateur 24 V sur une sortie MOSFET commandée, distincte de la pompe d'air assist) pour la découpe de contreplaqué en usage réel. Sans capot, elle aspire près de la zone de travail (hotte ou gaine), ou on travaille dans un local ventilé ou à l'extérieur.

### 6.2 Électrique
- Entrée 24 V, fusible (≈ 5 A d'après le §4.4), anti-inversion, TVS.
- Masses puissance / logique séparées, retours de courant moteurs hors de la zone ESP32.
- Antenne de l'ESP32 en bord de carte, sans cuivre dessous.

### 6.3 PCB
- 2 ou 4 couches **[À DÉCIDER]** (4 couches ≈ même prix chez JLCPCB, bien meilleur pour les masses).
- Assemblage manuel (CMS ≥ 0603) ou externe **[À DÉCIDER]**.
- Points de test sur chaque rail et bus ; connecteurs détrompés (JST-XH comme la DLC32).
- Format et trous de fixation compatibles avec le boîtier électronique de l'imprimante **[À VÉRIFIER]**.

### 6.4 Logiciel (phase 1)
- Tâches FreeRTOS : mouvement (priorité haute, cœur 1), réseau (cœur 0), interface, supervision sécurité.
- Blocs pilotes indépendants de l'application (§2.3), un programme de test par bloc.
- Git, un dépôt partagé.

### 6.5 Sobriété énergétique (exigence du cours)
- Drivers moteurs désactivés (EN) après inactivité : c'est le plus gros gain, car les moteurs consomment même à l'arrêt.
- Rétroéclairage TS35 atténué puis éteint.
- **ESP32-CAM coupée par MOSFET** tant que le flux n'est pas regardé.
- Wi-Fi en modem-sleep au repos ; veille profonde avec réveil par l'écran ou un bouton.
- **Tableau de mesures** (INA226) : gravure / repos / veille dans le rapport.

---

## 7. Plan de validation (tests unitaires → rapport)
1. Continuité de masse et courts-circuits à l'ohmmètre.
2. Alimentation seule : 24 → 5 → 3,3 V, ondulation à l'oscilloscope, LEDs témoins.
3. Programmation ESP32 (USB et HE10), clignotement d'une LED.
4. Bus : SPI écran, SPI SD, I2C INA226, I2S/74HC595, UART ESP32-CAM (analyseur logique).
5. Moteurs un par un, puis prise d'origine.
6. Sorties MOSFET pompe / LEDs / alimentation caméra, sous charge.
7. **Laser en mode démo** : PWM vérifié à l'oscilloscope ou sur une LED. Essai de tir réel uniquement hors salle de TP, avec protections.
8. Wi-Fi : connexion à la box et au partage de connexion du téléphone, bascule vers le point d'accès de configuration, latence WebSocket, flux caméra (délai d'allumage, images/s, courant caméra coupée = 0 mA).
9. Influence du flux vidéo sur la gravure : gigue des pas avec et sans flux (contrôle, attendu nul en mode client).
10. Chaîne de sécurité : AU enfoncé → relais retombé, laser et pompe coupés, TTL à 0, ESP32 informé ; chaque voie testée seule (relais shunté → la voie 2 bloque le TTL ; buffer forcé → la voie 1 coupe la puissance) ; temps de coupure du faisceau mesuré (photodiode ou oscilloscope) ; AU déverrouillé → rien ne redémarre avant « Réarmement » ; fil coupé → arrêt.
11. Intégration : tracé d'un carré, coupure sur flamme simulée (briquet devant le capteur, laser simulé).

---

## 8. Planning — **[À DÉCIDER : calendrier du cours]**
| Jalon | Contenu |
|---|---|
| J1 | Cahier des charges figé + synoptique validé par TB/RP |
| J2 | Proto sur DevKit ESP32-E **pendant** la conception du schéma. **Banc firmware (1 soirée)** : driver TMC2209 + 1 moteur + LED à la place du laser. FluidNC (YAML) puis Grbl_Esp32 (`notre_carte.h`), testés avec LaserGRBL en USB et Telnet (`$$`, déplacements, puissance) → choix du firmware de phase 2 et du mode de pilotage des pas **avant le routage** |
| J3 | Schéma électrique + revue |
| J4 | Routage + commande du PCB (fabrication ~1–2 semaines) |
| J5 | Montage + tests unitaires |
| J6 | Firmware démo + intégration + rapport |

---

## 9. Budget estimatif **[À VÉRIFIER]**
| Poste | Estimation |
|---|---|
| Imprimante 3D d'occasion | 0–80 € |
| Module laser 5 W optique | 40–90 € |
| Écran 3,5" SPI tactile | 10–20 € |
| ESP32-CAM | déjà en stock |
| PCB ×5 + composants | 30–60 € |
| TMC2209 ×3 | 10–15 € |
| Pompe d'aquarium | 10–20 € |
| Lunettes OD4+ 445 nm + écran orange de tête | 20–40 € |
| **Total** | **≈ 125–330 €** |

---

## 10. Questions ouvertes
1. Durée du projet, nombre de séances, date de rendu ?
2. Binôme accepté par les encadrants ?
3. Modèle exact de l'imprimante récupérée ?
4. Sorties lentes : 74HC595 en I2S ou expandeur I2C ? Tactile résistif ou capacitif ? (§2.2, après le bilan GPIO)
5. Arduino/PlatformIO ou ESP-IDF : contrainte du cours ?
6. Assemblage PCB manuel ou externe ? 2 ou 4 couches ?
7. Connecteur laser 2 broches (12–24 V PWM) en plus du TTL : utile ?
8. Les encadrants attendent-ils un protocole IoT particulier (MQTT…) en plus du Wi-Fi/WebSocket ?
