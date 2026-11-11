# Four à refusion maison — Synthèse de projet (contexte prêt à donner à un LLM)

> **Rôle de ce document** : source unique de vérité sur l'état du projet « Four à refusion maison » (novembre 2026). Toutes les décisions de conception y sont figées. Le lecteur (LLM ou humain) doit pouvoir rédiger la documentation d'assemblage, d'utilisation ou de maintenance **sans relire l'historique de conception**. Les sections « écarté » ne sont pas à rouvrir sauf nouveau fait matériel.

## 1. Objectif

Reflow de pâte à souder **basse température Sn42Bi58** (fusion 138 °C) dans un **four à convection domestique 1500 W / 230 V** recalibré (carrosserie remontée, cavité mesurée jusqu'à 250 °C, courant résistif 6,5 A). Pente mesurée : 36–40 °C/min (≈ 0,6–0,67 °C/s) — suffisante ; l'enjeu réel est l'overshoot au pic, pas la montée.

**Profil cible SnBi :**


| Phase           | Consigne                                  |
| --------------- | ----------------------------------------- |
| Rampe libre     | \~0,6 °C/s jusqu'à \~145 °C               |
| Soak            | 145–150 °C, 60–90 s (activation du flux)  |
| Pic             | 170–180 °C                                |
| TAL             | 30–60 s au-dessus de 138 °C               |
| Refroidissement | porte ouverte + ventilateur de convection |


## 2. Architecture — 3 couches, 1 règle d'or

Pilote : **Raspberry Pi Pico**, C++ / pico-sdk.

**Règle d'or : la boucle de contrôle et les sécurités vivent sur le Pico — le PC ne supervise jamais la chaude chaîne.** Un cycle en cours survit à une coupure USB ; toute liaison PC/Bluetooth est un simple observateur (télémétrie, réglages, lancement). Chute de liaison pendant un run = le run continue.

1. **Couche 1 — Contrôle + sécurité autonomes** : machine à états (RAMP/SOAK/PEAK/COOL/ALARM), PID 1 Hz sur la sonde martyre, PWM lent (période 200 ms – 5 s), interverrouillages, cutoffs logiques. Fonctionne seule, sans PC ni IHM.
2. **Couche 2 — Protocole USB** (CDC, `/dev/ttyACM0`, texte ligne par ligne) : commandes `PROFILE`, `START`, `ABORT`, `GAINS`, `MODE MANUAL x%` ; télémétrie CSV 1 Hz : `t,sp,tc_board,tc_air,duty,STATE`. Le Pico ne reset pas à l'ouverture du port (CDC natif) — la capture peut s'ouvrir en cours de cycle.
3. **Couche 3 — IHM locale** : OLED SSD1306 + encodeur + 2 boutons (détail § 6).

Choix Pico vs Arduino Nano : Pico retenu (CDC sans reset DTR, 2 UART pour HC-05 simultané, 264 KB RAM pour l'IHM).

## 3. Section puissance

### 3.1 Carte fille HT (réutilisée du projet « NanoFridgeThermostat »)

Carte fille seule : **MOC3042M** (opto-triac à détection de passage à zéro intégrée), snubber **R4 360 Ω + C1 10 nF/500 V AC**, borniers TE 5,08 mm, connecteurs 2,54 mm **J5/J6** (J5 = `TriacDriverEnable`, J6 = retour LED), R8 = 330 Ω **inchangé** (point de design : limite 1 A crête du MOC).

### 3.2 Triac : BTA316Y-800CTQ (WeEn)

- 16 A / 800 V, 3Q « Hi-Commutation », **IGT 35 mA**, Tj 150 °C, package **IITO-220 onglet isolé d'usine** (2500 V RMS) — plaque et boîtier jamais sous tension via l'onglet. Brochage TO-220 : 1 = MT1, 2 = MT2, 3 = G.
- Remplace le FT0814MH de stock (8 A / **600 V**, onglet nu) utilisé en phase de validation seulement. Brochage identique, même 3Q, même IGT 35 mA → **drop-in prouvé**, aucune vérif d'orientation MOC (la carte frigo fonctionnelle prouve la config Q1/Q3).
- Vérif Q1/Q3 du MOC : **LEVÉE** — ne revient que si la carte fille est modifiée (MOC remplacé/déplacé).
- Dissipation à 6,5 A : \~7,5–8,5 W → dissipateur requis ; critère R ≤ 3 °C/W. Sur plaque/radiateur dans le flux : Tj ≈ 75 °C à pleine puissance.
- **Dissipateur** : trois candidats — plaque alu 100×100 mm (récup), Fischer SK610-84AL, Alutronic PR33/25.4/SE (les 2 derniers commandés) — le test empirique `MODE MANUAL 100` 20–30 min, base ≤ 70 °C, tranche. Montage vertical, air sur les deux faces, pâte thermique.
- Phase 1 (FT0814MH, onglet nu) uniquement : kit d'isolement obligatoire (mica Ninigi SMICA-TO220-2 + collets NIPPEL-TO220), **vis nylon ou collet isolant** (la vis traverse l'onglet ET la plaque = court-circuit classique), plaque traitée comme conducteur actif (MT2 = phase dès la minuterie ON), plaque jamais posée sur le PCB (soudures = points de perforation du kapton).

### 3.3 Commande opto — montage RETENU : NPN low-side

Pattern projet : « **le GPIO est un signal, pas une source** ».

**Chaîne LED : 5 V\_EXT → R 330 Ω → J5 → LED du MOC3042 → J6 → collecteur NPN → émetteur → GND.**

- Base pilotée par **GP15 via 2,2–4,7 kΩ**, **10 kΩ pull-down** (Hi-Z au boot = OFF explicite). Courant base \~1 mA → défaut GPIO 4 mA du pico-sdk suffit, **pas de drive strength**.
- LED \~11,4 mA sur le rail 5 V\_EXT → **zéro sur le budget I/O du Pico** (budget total ≈ 10 mA, brown-out impossible).
- Non inverseur : GP15 haut = chauffe.
- NPN quelconque de récup : BC547 / 2N3904 / 2N2222.
- **J6 n'est plus à la masse** : c'est le retour commuté de la LED → **au repos, J6 mesure ≈ 5 V — c'est normal**, pas une panne.
- Vérifier la polarité de J5 au multimètre (mode diode, \~1,1 V dans le sens passant).
- Même pattern NPN appliqué au buzzer (GP8, base via 1 kΩ) et à l'option ventilateur boîtier (GP6, base via 470 Ω + 10 k pull-down).
- Alternative écartée : GPIO direct avec drive strength 12 mA — valide mais consommait le seul gros poste du budget I/O.

### 3.4 Chaîne secteur

Boucle : prise murale → minuterie mécanique → thermo-rupteur d'origine (mode autonome) → presse-étoupe → J2 → triac → J3 → presse-étoupe → résistances. Souple 1,5 mm², embouts serties, pattes de maintien. **Ventilateur de convection alimenté en continu, hors boucle triac** (phase directe depuis la minuterie). Pistes IPC-2221 : phase 4 mm, neutre 5,8 mm (1 oz).

### 3.5 Écartés (ne pas rouvrir)

- **SSR-25DA** : contrefaçons massives, aucun revendeur officiel identifiable en France, dissipation identique (\~8 W à 6,5 A) → plaque obligatoire quand même ; SSR authentiques (Omron G3NA, Crydom) à 25–45 € = enterrés.
- **Fusible et fusible thermique** : voir § 5 — décision assumée, aucun fusible dans le projet.
- **Ventilateur dans l'interstice** cavité/carrosserie : le PTFE 260 °C supprime le besoin, et souffler dedans abaisserait plafond et pente (contraires à l'objectif bench). Recette en réserve si réintroduit : soufflante 230 V de récup micro-ondes sur le nœud du ventilateur de convection, jamais en GPIO.

## 4. Mesure &amp; alimentation

- **2 sondes K + 2× MAX6675** (SPI0 partagé : SCK GP18, MISO GP16 ; CS GP17 martyre, CS GP20 joue). Lecture ≤ 4 Hz (conversion 0,17–0,22 s), détection TC ouvert intégrée.
- **Sonde martyre** (PCB FR4 percé, même épaisseur que les cartes réelles, pointe plaquée + Kapton, position reproductible) = **contrôle PID**. **Sonde joue** (écrou d'origine) = supervision : coupure absolue \~230 °C (autonome) / \~300 °C (bench), santé chauffage.
- **Alarme directionnelle** : en chauffe l'air doit mener la carte (écart &gt; \~60 °C = emballement) ; en refroidissement l'inversion est normale.
- **Calibration uniformité** (une fois) : martyre en 3 positions ; gradient acceptable &lt; 5–8 °C au pic.
- **Alim** : chargeur USB certifié 1 A → Schottky 1N5819/SS34 → VSYS (pin 39) ; PC = données uniquement (alim de secours via D1 interne). Budget \~275 mA tournants / \~400 mA en pointe → marge ≥ 2,5×. Condo 470–1000 µF optionnel.
- **HC-05 (option, supervision Bluetooth)** : UART1 GP4 TX / GP5 RX, 115200 Bd, 3,3 V natif sans pont diviseur, VCC sur rail 5 V. Config une fois : `AT+UART=115200,0,0`, `AT+NAME=ReflowPico`. Jamais dans la chaîne de contrôle ; télémétrie dupliquée USB + UART (coût nul à 1 Hz).

## 5. Sécurité — architecture à deux modes (décision non négociable)

**Pas de fusible, pas de fusible thermique.** Justification : un triac mort en court-circuit appelle le même courant que la charge (6,5 A) — aucun fusible ne voit cette panne ; le four d'origine n'avait que le thermo-rupteur ; un fusible thermique honnête exigerait une chirurgie du faisceau refusée. Le bornage matériel est assuré ainsi :

1. **Mode autonome (reflow, par défaut)** : thermo-rupteur d'origine **remis en série** (réarmable \~210–220 °C, au-dessus du profil SnBi 180 °C, indépendant du Pico et du triac). Court-circuité, le four oscille borné sous la consigne du rupteur.
2. **Mode bench (essais &gt; 250 °C, rupteur débranché)** : four vide, sonde martyre retirée (Kapton \~200 °C), gaines des cosses de résistances inspectées (mica/verre = OK, plastique = stop), **opérateur présent en permanence**, minuterie armée, cutoffs logiques \~300 °C.
3. **Minuterie mécanique (auto-off) armée à chaque run** : switch maître ET borne temporelle — tout emballement meurt avec le cadran.
4. **Après chaque session bench : rebrancher le rupteur avant tout cycle autonome** (vérification physique) — le seul moyen d'oublier la sécurité allumée.
5. Règles générales : contrôles thermiques hors tension uniquement (couper au mur, toucher à la chaleur résiduelle) ; rien d'imprimé 3D dans la cavité (l'air atteint 170–180 °C au pic ; intérieur = FR4 + métal) ; **aucun PVC dans l'interstice** (&gt; 100 °C près de la paroi) ; séparation HT/BT ≥ 6 mm dans le boîtier, flux d'air traversant le dissipateur (entrée basse / sortie haute).

## 6. IHM locale &amp; brochage Pico

**Brochage figé (proposition gelée à la génération firmware) :**


| Fonction                  | GP            | Remarque                                                                                    |
| ------------------------- | ------------- | ------------------------------------------------------------------------------------------- |
| Opto MOC (via NPN)        | GP15          | base via 2,2–4,7 kΩ + 10 k pull-down                                                        |
| SPI0 SCK / MISO           | GP18 / GP16   | 2× MAX6675 partagés                                                                         |
| CS martyre / CS joue      | GP17 / GP20   | contrôle PID / supervision                                                                  |
| I2C0 SDA / SCL            | GP0 / GP1     | OLED SSD1306 0x3C                                                                           |
| UART1 TX / RX             | GP4 / GP5     | HC-05 115200 Bd                                                                             |
| Encodeur A / B            | GP10 / GP11   | 100 nF sur A et B, décodage quadrature (IRQ ou scrut. ≥ 1 kHz)                              |
| Bouton **Edit**           | GP12          | NO → GND + 100 nF ; entrer / valider / START·STOP                                           |
| Bouton **Back**           | GP13          | NO → GND + 100 nF ; sortir / annuler / arrêt                                                |
| Réserve poussoir encodeur | GP14          | libre dans le modèle à 2 boutons                                                            |
| LED statut                | GP9           | via 330 Ω ; fixe = chauffe, clignote = alarme, off = idle                                   |
| Buzzer                    | GP8           | via NPN, `#define` firmware : actif (DC) vs passif/PWM — composant non acheté, récup prévue |
| Ventilo boîtier (option)  | GP6           | NPN + flyback + 10 k pull-down, ou direct sur le rail                                       |
| Alim                      | pins 39/38/36 | VSYS / GND / 3V3 sondes                                                                     |


**Modèle d'interaction minimaliste** : encodeur = naviguer / incrémenter-décrémenter ; **Edit** = entrer dans un menu, éditer, valider — et lancer le cycle quand le curseur est sur le champ Start ; **Back** = sortir / annuler. Sélection d'une courbe pré-paramétrée (SnBi standard, SnBi lent, manuel…) → ajustements éventuels → Start → Edit = cycle lancé. **Dès le lancement, le champ devient STOP : un appui Edit OU Back = arrêt immédiat** (redondance volontaire — l'arrêt ne dépend jamais d'un seul bouton). Pas d'appui long, pas de confirmation : arrêter = le geste le plus simple de l'IHM.

**Anti-rebond matériel** : 100 nF entre GPIO et GND sur chaque bouton et sur A/B de l'encodeur, pull-ups internes (\~50 kΩ, τ ≈ 5 ms) — le firmware lit des niveaux stables, simple détection de front, pas de code d'anti-rebond.

**OLED** : 0,96″ SSD1306 128×64 (en stock), adresse 0x3C (confirmer par scan), pull-ups normalement sur le module, rafraîchi à 1 Hz (frame I2C \~25 ms à 400 kHz — aucun conflit avec la boucle de contrôle).

## 7. Câblage &amp; fils

- **Interstice cavité/carrosserie** : **LAPP ÖLFLEX HEAT 350 SC 1,5 mm²** (fibre de verre 350 °C, commandé 2 m) ou fil d'usine classé 250 °C+. **Sertir (âme nickel), ne pas souder** — l'étain fond à 230 °C. Silicone 180 °C **interdit** dans l'interstice (élément limitant en bench). Alt. : PTFE 260 °C (ÖLFLEX HEAT 260 SC).
- **Extérieur** : silicone 180 °C, 1,5 mm² (TKD, commandé — vérifier section ≥ 1 mm²).
- **Cosses de résistances** : cosse standard sans isolant PVC + kapton à adhésif silicone (\~280 °C) ou gaine PTFE rétractable 260 °C — l'adhésif acrylique courant décolle à \~155 °C.
- Jonction PVC ↔ fil d'usine **à l'extérieur**, cosse sertie en boîte fermée ; presse-étoupes côté extérieur (BM4007 PG7, commandés ×5) ; toute épissure sous gaine de verre.
- Percer la carrosserie au plus près des cosses de résistances ; connecteur 2 broches en ligne sur l'alim des résistances (retour config stock en 30 s).

**Les 9 réflexes d'assemblage** : ① presse-étoupe + souple 1,5 mm² extérieur uniquement, embouts serties ; ② aucun PVC dans l'interstice ; ③ jonctions phase hors carrosserie en boîte fermée ; ④ ventilateur de convection hors boucle triac ; ⑤ onglet triac : serrage modéré + pâte, (BTA isolé = plaque non sous tension) ; ⑥ séparation HT/BT ≥ 6 mm, flux d'air traversant ; ⑦ ouvrir la capture série **avant** de lancer le cycle ; ⑧ premier contrôle thermique hors tension à la chaleur résiduelle ; ⑨ minuterie armée à chaque run + rupteur rebranché après bench.

**Test à vide avant remontage** : minuterie OFF → ohmmètre sur la boucle (continuité minuterie → rupteur → résistances ≈ 35 Ω ; triac bloqué = circuit ouvert) → `MODE MANUAL 100` 20–30 min → couper au mur → contrôle chaleur résiduelle (triac/plaque, borniers, presse-étoupes, câble contre la tôle).

## 8. Stock, commande TME &amp; budget

**En stock** : Pico · carte fille NanoFridgeThermostat · FT0814MH (phase 1) · 2 sondes K + MAX6675 · encodeurs, boutons, LED · OLED SSD1306 · relais 10 A (secours) · HC-05 + HC-06 · plaque alu 100×100 · chargeur USB 1 A · NPN de récup · platines prototypage (stock cartons ou Amazon, hors TME).

**Commande TME passée (nov. 2026, ≈ 41 € TTC, port éco 5,90 € HT, 2–5 jours)** :


| Article                                                          | Qté        | Prix            | Rôle                                                    |
| ---------------------------------------------------------------- | ---------- | --------------- | ------------------------------------------------------- |
| Triac **BTA316Y-800CTQ** (⚠️ le Y = isolé, pas le BTA316-800CTQ) | ×5         | 0,51 €/pc       | commutation + spares à vie                              |
| LAPP ÖLFLEX HEAT 350 SC 1,5 mm²                                  | 2 m        | \~5 €/m         | interstice, à sertir                                    |
| Ruban polyimide **IPT POL-12MM** 12 mm × 33 m                    | 1          | 7,67 €          | Kapton — **à vérifier : adhésif silicone vs acrylique** |
| Ninigi SMICA-TO220-2 (mica) + NIPPEL-TO220                       | ×10 chacun | 2,81 € le lot   | kit isolement phase 1 (FT0814MH onglet nu)              |
| Radiateurs Fischer SK610-84AL + Alutronic PR33/25.4/SE           | 1 + 1      | 4,68 € + 2,62 € | candidats vs plaque alu récup, test empirique           |
| Presse-étoupes BM4007 (PG7, IP68)                                | ×5         | —               | sortie carrosserie                                      |
| 1N5819                                                           | ×5         | —               | Schottky alim                                           |
| Câble silicone TKD 2 couleurs                                    | 2×1 m      | \~1,07 €/m      | extérieur — **à vérifier : section ≥ 1 mm²**            |


## 9. Firmware — reste à générer

Le brochage et l'IHM sont gelés ; le firmware (couches 1 + 2 + 3) sera généré au feu vert de l'utilisateur, une fois la livraison TME reçue et l'assemblage engagé. Points de spec déjà figés : machine d'états (RAMP/SOAK/PEAK/COOL/ALARM), PID 1 Hz + PWM lent, cutoffs (martyre \~230 °C autonome / \~300 °C bench, coupure joue absolue), alarme directionnelle air/carte, protocole texte USB + télémétrie CSV 1 Hz (dupliquée sur UART1), buzzer abstrait par `#define` (ACTIF vs PWM — composant en récup), anti-rebond matériel donc simple détection de front, quadrature par table d'états.

Côté PC Linux : `/dev/ttyACM0` (cdc\_acm natif), groupe `dialout`, piège ModemManager (désactiver ou règle udev `ID_MM_DEVICE_IGNORE`), capture `cat /dev/ttyACM0 | tee run.csv` ou script pyserial + matplotlib pour le réglage PID. **Rien n'est loggé avant ouverture du port.**

## 10. Plan d'exécution

1. Câblage puissance + protections matérielles (seule étape non compressible) ;
2. Génération firmware couche 1 → couche 2 → couche 3 ;
3. Runs à vide : `MODE MANUAL 100` 20–30 min → contrôle thermique hors tension → test dissipateur (base ≤ 70 °C) ;
4. Calibration : martyre en 3 positions (uniformité), puis runs de profil avec tracé PC ;
5. Réglage PID (l'overshoot se joue surtout au passage soak → pic) ;
6. Premier reflow réel, log enregistré.

**Vérifications à l'ouverture du colis TME** : kapton IPT adhésif silicone (pas acrylique \~155 °C) ; section câble TKD ≥ 1 mm² ; triac bien **BTA316Y** (onglet isolé).

**Checklist ouverte (hors items résolus)** : polarité J5 (mode diode) · connecteur 2 broches sur l'alim résistances · ventilateur de convection hors boucle triac après remontage · capture série ouverte avant premier run · (option HC-05) config AT avant câblage définitif · rupteur rebranché après chaque bench · avant tout essai &gt; 250 °C : conditions mode bench respectées.