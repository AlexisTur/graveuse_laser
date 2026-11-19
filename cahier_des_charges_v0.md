# Cahier des charges — Graveuse laser connectée (V0.1, brouillon de travail)

> Statut : **V0.1 à débattre**. **[À DÉCIDER]** = réponse de l'équipe attendue ; **[À VÉRIFIER]** = hypothèse technique à confirmer (datasheet, schéma, essai).
> Contexte : projet de synthèse « Objets connectés » ESEO (T. Bouvier / R. Perdriau).
>
> Historique :
> - V0 : première proposition.
> - V0.1 : caméra = ESP32-CAM existant (surveillance seule) ; laser 5 W optique pour découpe de contreplaqué ; stratégie « PCB final + firmware de démonstration » ; ajout des données du manuel MKS DLC32 V2.

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

La page web du téléphone, servie par l'ESP principal, affiche simplement l'image venant de l'adresse de l'ESP32-CAM (`<img src="http://<ip-cam>:81/stream">`).

Point de vigilance : l'ESP32-CAM a très peu de broches libres (la plupart sont prises par la caméra et la SD). Il faut identifier les 2 broches de l'UART **[À VÉRIFIER]**. La SD de l'ESP32-CAM ne sera pas utilisée : on garde la SD de la carte principale.

### 2.2 Compatibilité du brochage avec la DLC32 — **recommandé, [À DÉCIDER]**
Comme le PCB est le matériel final, je recommande de **reprendre le brochage ESP32 de la MKS DLC32 V2.1** pour toutes les fonctions qu'elle possède :
- moteurs via I2S + 74HC595 ;
- fins de course, laser, carte SD ;
- écran EXP1/EXP2 compatible **TS35** ;
- I2C, buzzer.

Nos fonctions en plus (LEDs, pompe, caméra, interlock, mesure d'énergie) vont sur les broches et sorties restantes.

Bénéfices :
- **Phase 2 quasi immédiate** : le firmware MKS DLC32 (dérivé open source de Grbl_Esp32, avec interface écran TS35 et compatible LaserGRBL) devrait tourner directement sur notre carte **[À VÉRIFIER : sources publiées sur github.com/makerbase-mks]**. Même chose pour Grbl_Esp32/FluidNC, qui ont des configurations DLC32.
- C'est exactement « l'open source facile à implémenter » qui vous a plu dans la vidéo.
- L'écran **TS35** se branche tel quel (2 nappes HE10), sans conception d'écran.

Action : récupérer le **schéma électrique DLC32 V2.1** (dépôt GitHub Makerbase) et le fichier de brochage des firmwares, puis faire le bilan des broches libres.

### 2.3 Stratégie firmware de cours (phase 1)
Le code est organisé en **deux couches** :
1. **Blocs pilotes (HAL)**, un par périphérique, chacun avec son programme de test :
   - moteurs, laser, pompe, LEDs ;
   - écran + tactile, SD ;
   - capteurs (flamme, interlock, fins de course, INA226) ;
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
| Supervision sécurité (flamme, interlock, watchdog) | Oui, en complément |
| Mesure d'énergie + gestion de veille | Oui (absent des firmwares open source) |
| Pilotage caméra, LEDs, pompe | Oui |
| Interface web téléphone (page + caméra) | Oui, en partie (adaptable à l'API du firmware open source) |
| Interface tactile | Probablement non si le firmware MKS est repris (il a déjà la sienne) |

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
| F12 | Détection flamme / ouverture capot → arrêt laser | M (sécurité) | Coupure < 100 ms |
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
                                                                                ├─► Écran TS35 (EXP1/EXP2)
                                                                                └─► MOSFET ─► ESP32-CAM (5 V)
ESP32-WROOM-32E
  ├─ I2S → 74HC595 ×2 → STEP/DIR/EN X,Y,Z + sorties lentes (pompe, buzzer, alim caméra…)   [comme DLC32]
  ├─ SPI → EXP1/EXP2 (écran TS35 + tactile)  |  SPI → micro-SD                               [comme DLC32]
  ├─ Entrées → fins de course X/Y/Z, capteur flamme (port « probe » DLC32), interlock capot
  ├─ PWM → laser TTL (buffer 5 V + pull-down), LEDs
  ├─ I2C → INA226 (+ extension)
  ├─ UART → ESP32-CAM (commandes)
  └─ USB (CH340 + reset auto) + HE10 8 pts (format ESEO) pour la programmation
Arrêt d'urgence : coupe matériellement l'alimentation du laser + entrée lue par l'ESP32
ESP32-CAM (module existant) ── Wi-Fi ──► téléphone (flux MJPEG)
```

### 4.2 Données reprises du manuel MKS DLC32 V2
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
| Drivers | Supports stepstick + TMC2209 (micro-pas par DIP) | Remplaçables en proto, silencieux, identiques à la machine de la vidéo |
| Extension de sorties | 74HC595 ×2 via I2S | Économise les GPIO, géré nativement par les firmwares phase 2 |
| Écran | MKS TS35 sur EXP1/EXP2 | Pas d'écran à concevoir ; compatible avec le firmware MKS |
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
| Mode réseau | **Point d'accès (AP) créé par la graveuse** : marche partout, y compris en salle de TP sans réseau. L'ESP32-CAM s'y connecte. Option : mode client (STA) sur la box. |
| Accès | `http://graveuse.local` (mDNS) ou IP fixe 192.168.4.1 |
| Interface | **Page web servie par l'ESP32** → rien à installer, Android + iOS |
| Commandes + état temps réel | **WebSocket** (explication ci-dessous) avec messages JSON |
| Vidéo | Flux MJPEG direct depuis l'ESP32-CAM |
| Sécurité | AP protégé WPA2 + mot de passe d'accès à l'interface |

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
- **Arrêt d'urgence matériel** coupant l'alimentation du laser (pas seulement un GPIO).
- **Interlock capot** en série dans la commande laser + lu par l'ESP32.
- Laser **éteint par défaut** : pull-down, broche hors strapping, PWM à 0 avant tout démarrage du Wi-Fi.
- **Watchdog** : plantage ou perte de connexion du téléphone pendant un tir → laser OFF.
- **Détection de flamme** obligatoire, puisque le pilotage à distance implique un fonctionnement sans surveillance directe. La caméra complète cette surveillance.
- Caisson avec fenêtre filtrante 445 nm et lunettes OD4+ adaptées à 445 nm **[À VÉRIFIER : références]**.
- Extraction des fumées pour la découpe de contreplaqué (usage réel).

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
8. Wi-Fi : connexion AP, latence WebSocket, flux caméra.
9. Intégration : tracé d'un carré, coupure sur flamme simulée (briquet devant le capteur, laser simulé).

---

## 8. Planning — **[À DÉCIDER : calendrier du cours]**
| Jalon | Contenu |
|---|---|
| J1 | Cahier des charges figé + synoptique validé par TB/RP |
| J2 | Proto sur cartes de dev (Wemos D1 R32 + TS35 + driver + ESP32-CAM) **pendant** la conception du schéma |
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
| Écran MKS TS35 | 15–25 € |
| ESP32-CAM | déjà en stock |
| PCB ×5 + composants | 30–60 € |
| TMC2209 ×3 | 10–15 € |
| Pompe d'aquarium | 10–20 € |
| Lunettes + fenêtre filtrante | 20–40 € |
| **Total** | **≈ 125–330 €** |

---

## 10. Questions ouvertes
1. Durée du projet, nombre de séances, date de rendu ?
2. Binôme accepté par les encadrants ?
3. Modèle exact de l'imprimante récupérée ?
4. Compatibilité du brochage avec la DLC32 (§2.2) : OK ?
5. Arduino/PlatformIO ou ESP-IDF : contrainte du cours ?
6. Assemblage PCB manuel ou externe ? 2 ou 4 couches ?
7. Connecteur laser 2 broches (12–24 V PWM) en plus du TTL : utile ?
8. Les encadrants attendent-ils un protocole IoT particulier (MQTT…) en plus du Wi-Fi/WebSocket ?
