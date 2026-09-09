# Scanner OBD-II ESP32

Projet de conception matérielle (schématique et PCB) d'un scanner de diagnostic automobile OBD-II intelligent et communicant.

---

## 1. Objectifs du Projet

* **Diagnostic embarqué :** Lecture des données moteur en temps réel et des codes défauts (DTC) via la prise standard OBD-II (16 broches).
* **Connectivité sans fil :** Module **ESP32** assurant la liaison sans fil (Wi-Fi / Bluetooth) vers une application smartphone Android dédiée.
* **Support multi-protocoles :**
  * **Ligne K-Line (ISO 9141-2 / ISO 14230 KWP2000) :** Protocole principal du calculateur moteur (ECU) de la Daewoo Kalos.
  * **Bus CAN (ISO 15765-4) :** Diagnostic haute vitesse et compatibilité véhicules modernes.
* **Alimentation robuste & sécurisée :**
  * Alimentation directe depuis le 12V batterie de la prise diagnostic.
  * Protection contre les surtensions, inversions de polarité et surintensités (fusible réarmable PPTC `F1`, diode TVS `D1`, MOSFETs de protection `Q1`/`Q2`).
  * Double étage de conversion : abaisseur à découpage performant 12V → 5V (`U4` / `L1`) suivi d'un régulateur linéaire ultra-propre 3.3V (`U5` / `FB1`) pour l'ESP32 et la logique.

---

## 2. Architecture Matérielle & Anatomie Électronique

Pour comprendre le fonctionnement de ce scanner, il faut d'abord appréhender l'environnement très particulier d'un véhicule automobile : la batterie 12V d'une voiture n'est ni stable, ni propre (pics de surtension de l'alternateur, étincelles d'allumage, bruit des injecteurs), et les calculateurs (ECU) communiquent via des protocoles spécifiques (K-Line à 12V et bus différentiel CAN).

Le circuit imprimé est découpé en **8 blocs fonctionnels interconnectés**, organisés pour purifier l'énergie, protéger les composants sensibles et assurer une communication bidirectionnelle infaillible.

---

### 2.1 Schéma Fonctionnel Global & Arbre d'Énergie

```
               PRISE DIAGNOSTIC OBD-II (16 BROCHES)
                │                  │               │
  Broche 16 (+12V Batterie)   Broches 6 & 14    Broche 7 (K-Line)
                │              (Bus CAN Diff)          │
                ▼                  │                   ▼
    ┌─────────────────────────┐    │       ┌───────────────────────┐
    │  1. PROTECTION 12V      │    │       │  6. TRANSCEIVER       │
    │  • Fusible PPTC F1      │    │       │     K-LINE (U3)       │
    │  • Diode TVS D1         │    │       │  Traduction 12V ↔ 3.3V│
    │  • Anti-inversion Q1/Q2 │    │       └───────────┬───────────┘
    └───────────┬─────────────┘    │                   │ UART_RX / TX
                │ +12V_PROT        │                   │ (avec amortisseurs R2/R3)
                ▼                  ▼                   │
    ┌─────────────────────────┐  ┌────────────────┐    │
    │  2. CONVERTISSEUR BUCK  │  │ 5. TRANSCEIVER │    │
    │     12V → 5V (U4)       │  │    CAN (U2)    │    │
    │  • Inductance L1 (10µH) │  │ • Term. R11    │    │
    │  • Diode Schottky D2    │  │ • Adapt. VIO   │    │
    │  • Bootstrap C5         │  └────────┬───────┘    │
    │  • Feedback R12/R13     │           │ TWAI_RX/TX │
    └───────────┬─────────────┘           │ (CAN)      │
                │ +5V                     │            │
                ├─────────────────────────┼────────────┤
                │                         │            │
                ▼                         │            │
    ┌─────────────────────────┐           │            │
    │  3. RÉGULATEUR LDO 3.3V │           │            │
    │     (U5 - LDL1117S33R)  │           │            │
    │  • Filtrage HF (FB1)    │           │            │
    │  • Condensateur C6      │           │            │
    └───────────┬─────────────┘           │            │
                │ +3.3V Logique           │            │
                ▼                         ▼            ▼
    ┌──────────────────────────────────────────────────────────────┐
    │  4 & 8. CŒUR DE TRAITEMENT : ESP32-S3-WROOM-1 (U1)           │
    │  • Microcontrôleur 32-bit dual-core Xtensa LX7               │
    │  • Radio Wi-Fi 2.4 GHz & Bluetooth 5.0 (Antenne PCB intégrée)│
    │  • Réseau de condensateurs de découplage local (C1 à C4)     │
    │  • Témoin lumineux LED1 d'activité (piloté par IO2 via R8)   │
    └──────────────────────────────┬───────────────────────────────┘
                                   │ USB_D+ / USB_D-
                                   ▼
    ┌──────────────────────────────────────────────────────────────┐
    │  7. INTERFACE USB-C & DÉBOGAGE (J2)                          │
    │  • Résistances de configuration CC1/CC2 (R4, R5 : 5.1 kΩ)    │
    │  • Diodes de protection antistatique ESD (U7, U8 : SD05C)    │
    └──────────────────────────────────────────────────────────────┘
```

---

### 2.2 Guide Pédagogique : Le Rôle de Chaque Groupe de Composants

---

#### Bloc 1 : Protection d'Entrée 12V Automobile (`F1`, `D1`, `Q1`, `Q2`, `R6`, `R10`)

Le réseau électrique d'une voiture est l'un des environnements les plus agressifs pour l'électronique :
* Démarrage du moteur : chutes brutales de tension (cranking).
* Déconnexion accidentelle d'une batterie en charge (*Load Dump*) : pics d'énergie inductifs pouvant dépasser **40V à 60V**.
* Mauvaise manipulation : inversion des pinces de démarrage (+12V et masse inversés).

```
   +12V OBD (Pin 16) ────► [ Fusible F1 ] ──┬──► [ P-MOSFET Q1 (Drain) ] ──► +12V_PROT
                             (0.5A PPTC)    │        ▲ (Source)
                                            │        │ Grille pilotée
                                         [ D1 ]      │ par Q2 + R10
                                        (TVS 24V)    │
                                            │        ▼
                                           GND   [ N-MOSFET Q2 ] ◄── Polarisation R6
```

* **Fusible Réarmable PPTC `F1` (0.5A - `MF-MSMF050-2`) :**
  * *Principe :* Contrairement à un fusible traditionnel à fil fusible qui brûle définitivement, un PPTC (*Polymeric Positive Temperature Coefficient*) est constitué d'un polymère conducteur. En cas de surintensité (> 500 mA), l'échauffement interne par effet Joule fait brutalement exploser sa résistance électrique, bloquant le courant. Une fois le court-circuit éliminé et le composant refroidi, il redevient conducteur automatiquement.
* **Diode TVS de Protection contre les Surtensions `D1` (24V - `SMBJ24A`) :**
  * *Principe :* Une diode TVS (*Transient Voltage Suppressor*) reste totalement transparente en temps normal. Dès qu'une impulsion transitoire dépasse sa tension d'avalanche (24V), elle devient conductrice en quelques picosecondes et court-circuite l'excédent d'énergie directement vers la terre (GND), protégeant ainsi tous les composants situés en aval.
* **Protection Anti-Inversion par P-MOSFET `Q1` (`IRLML2244`) et N-MOSFET `Q2` (`2N7002`) :**
  * *Pourquoi pas une simple diode ?* Une diode de redressement classique provoquerait une chute de tension permanente de 0.7V à 1.0V et dissiperait inutilement de la chaleur (P = V × I).
  * *Fonctionnement ingénieux des MOSFETs :*
    * **En polarité normale (+12V branché correctement) :** La tension positive arrive sur la grille de `Q2` via la résistance `R6`. `Q2` devient passant et tire la grille du P-MOSFET `Q1` vers la masse (0V). La différence de potentiel Grille-Source Vgs de `Q1` devient fortement négative (~ -12V), ce qui sature complètement `Q1`. Sa résistance interne Rds(on) n'est que de **0.05 Ω**, entraînant une perte de tension quasi-nulle (< 0.02V) !
    * **En cas d'inversion accidentelle de polarité :** La grille de `Q2` n'est pas alimentée, `Q2` reste bloqué, la grille de `Q1` reste au même potentiel que sa source (Vgs = 0V) : `Q1` est hermétiquement ouvert. Aucun courant inverse destructeur ne pénètre dans la carte.

---

#### Bloc 2 : Convertisseur à Découpage Buck 12V → 5V (`U4`, `L1`, `D2`, `C5`, `C7`, `C8`, `R12`, `R13`, `R14`, `C9`)

L'ESP32 et les circuits logiques consomment jusqu'à 300 mA à 500 mA lors des transmissions radio Wi-Fi.
* Si on utilisait un simple régulateur linéaire pour abaisser 12V en 5V sous 500 mA, la puissance perdue en pure chaleur serait de :
  > **P_dissipée = (Vin - Vout) × I = (12V - 5V) × 0.5A = 3.5 Watts !**
  Le régulateur brûlerait en quelques secondes sans un radiateur métallique volumineux.
* Le **convertisseur Buck (`U4` - TPS54331DR)** découpe la tension à haute fréquence (**570 kHz**, soit 570 000 fois par seconde) avec un rendement exceptionnel supérieur à **85%**.

```
                        Inductance L1 (10µH)
                     ┌───── 3000000 ─────┐
                     │                   │
  VIN (12V) ──► [ Interrupteur ] ──┬─────┴───────────────► Sortie +5V
                Interne U4 (PH)    │                         │
                                   ▼                         ▼
                                [ D2 ] (Schottky)       [ C8 ] (22µF)
                                 Roue libre              Filtrage
                                   │                         │
                                  GND                       GND
```

* **Inductance de Puissance `L1` (10 µH blindée - `YNR6045-100M`) :**
  * *Rôle :* C'est le réservoir d'inertie magnétique. Quand le transistor interne de `U4` est fermé (ON), le courant traverse `L1` et charge son champ magnétique tout en alimentant la charge. Quand le transistor s'ouvre (OFF), l'inductance s'oppose à l'interruption du courant (V = L · di/dt) et restitue son énergie emmagasinée.
* **Diode Schottky de Roue Libre `D2` (3A / 40V - `SS34`) :**
  * *Rôle :* Quand le transistor interne de `U4` s'ouvre, l'inductance cherche à puiser du courant. La diode `D2` (cathode sur `PH`, anode sur `GND`) devient alors passante et permet au courant de circuler en boucle fermée depuis la masse vers l'inductance sans interruption.
  * *Pourquoi une diode Schottky ?* Elle offre un temps de commutation ultra-rapide (< 10 ns) et une chute de tension minime (~0.35V à 0.4V), minimisant drastiquement les pertes de puissance.
* **Condensateur de Bootstrap `C5` (1 µF - `C0603`) :**
  * *Rôle :* Le transistor de découpage interne de `U4` est un N-MOSFET placé côté "haut" (High-Side). Pour saturer un N-MOSFET dont la source est à 5V, il faut appliquer sur sa grille une tension supérieure à son drain (Vg ≈ 12V + 5V = 17V). Le condensateur `C5` connecté entre `BOOT` (broche 1) et `PH` (broche 8) forme une **pompe de charge** qui emmagasine de l'énergie et rehausse le potentiel de commande de grille.
* **Condensateurs Réservoirs d'Entrée et Sortie `C7` et `C8` (22 µF céramique 25V X5R - `C1206`) :**
  * `C7` fournit instantanément les fortes impulsions de courant demandées par le hachage à 570 kHz, évitant que les variations de courant ne polluent la ligne 12V amont.
  * `C8` accumule le courant triangulaire issu de l'inductance `L1` et lisse la tension de sortie pour ne laisser qu'une ondulation résiduelle (*ripple*) infime (< 20 mV).
* **Pont Diviseur de Contre-Réaction `R12` (10 kΩ) et `R13` (1.91 kΩ) :**
  * *Rôle :* Le régulateur `U4` possède un amplificateur interne comparant la tension sur sa broche `VSENSE` à une référence interne très stable de **0.800 V**.
  * La formule de calcul de la tension de sortie régulée est :
    > **Vout = 0.8V × (1 + R12 / R13) = 0.8V × (1 + 10 000 / 1 910) = 0.8 × 6.2356 ≈ 4.988 V ≈ 5.0 V**
* **Réseau de Compensation de Boucle `R14` (10 kΩ) et `C9` (3.3 nF) :**
  * *Rôle :* Dans tout système asservi en boucle fermée, un déphasage excessif entre la commande et la sortie peut transformer le régulateur en oscillateur instable. Le circuit RC série sur la broche `COMP` compense la réponse en fréquence (correcteur proportionnel-intégral) et garantit une marge de phase sécurisée quelles que soient les fluctuations de charge de l'ESP32.

---

#### Bloc 3 : Régulateur Linéaire LDO 5V → 3.3V & Filtrage RF (`U5`, `FB1`, `C6`)

Bien que le régulateur Buck soit très efficace, son découpage haute fréquence génère des bruits harmoniques. Le microcontrôleur ESP32-S3 et son transceiver radio 2.4 GHz exigent une alimentation d'une pureté absolue pour garantir une portée Wi-Fi/Bluetooth maximale et éviter les erreurs de conversion analogique.

```
  +5V (Buck) ──► [ Régulateur LDO U5 ] ──► [ Perle Ferrite FB1 ] ──┬──► Rail Logique +3.3V
                 (LDL1117S33R : 1.2A)       (Filtre bruit HF)     │
                                                               [ C6 ] (1µF)
                                                                  │
                                                                 GND
```

* **Régulateur Linéaire LDO `U5` (`LDL1117S33R` - SOT-223) :**
  * *Rôle :* Un régulateur LDO (*Low Drop-Out*) agit comme une résistance variable ultrarapide asservie. Il "rabote" le 5V pour fournir un **3.3V continu parfaitement plat**, avec une réjection de bruit (PSRR) de plus de 75 dB. La chute de tension n'étant que de 5V - 3.3V = 1.7V, l'échauffement reste très faible et facilement dissipé par le plan de masse du PCB.
* **Perle de Ferrite `FB1` (`BLM18PG121SN1D` - boîtier 0603) :**
  * *Principe :* Une perle de ferrite se comporte comme un fil ordinaire à résistance nulle pour le courant continu (DC), mais présente une impédance inductive élevée (**120 Ω à 100 MHz**) pour les parasites électromagnétiques et le bruit radiofréquence. Elle agit comme une barrière étanche empêchant le bruit numérique de l'ESP32 de refluer vers les capteurs et transceivers.
* **Condensateur Réservoir `C6` (1 µF céramique - `C0603`) :**
  * *Rôle :* Stabilise la boucle de régulation interne du LDL1117 et amortit les variations d'impédance de la perle de ferrite.

---

#### Bloc 4 : Condensateurs de Découplage Local Haute Fréquence (`C1` à `C4` - 100 nF)

* **Pourquoi a-t-on besoin de condensateurs de 100 nF au plus près de chaque puce ?**
  * Une piste de cuivre sur un circuit imprimé possède une inductance parasite naturelle d'environ 1 nanohenry par millimètre (L ≈ 1 nH/mm).
  * Quand l'ESP32 bascule l'état de ses transistors internes en moins d'une nanoseconde (Δt < 1 ns), l'appel de courant brusque di/dt provoque une chute de tension fugitive (V = L · di/dt) qui peut faire chuter le 3.3V local et provoquer un plantage ou un redémarrage intempestif du microcontrôleur (*brownout reset*).
  * **La solution :** Les condensateurs `C1` et `C2` (pour l'ESP32 `U1`), `C3` (pour la puce CAN `U2`) et `C4` (pour la puce K-Line `U3`) sont des condensateurs céramiques multi-couches (MLCC) placés à **moins de 2 mm des broches d'alimentation**. Ils agissent comme des micro-réservoirs d'énergie locale qui fournissent instantanément ces charges haute fréquence.

---

#### Bloc 5 : Interface de Bus Différentiel CAN (`U2` - TJA1051T, `R11`)

Le bus CAN (*Controller Area Network*) est la norme universelle de communication dans les véhicules récents (haute vitesse jusqu'à 1 Mbit/s).

```
   ESP32 (TWAI Controller)              Transceiver CAN U2                  Prise OBD-II
  ┌───────────────────────┐            ┌──────────────────┐               ┌──────────────┐
  │ IO37 (TXD) ───────────┼───────────►│ Pin 1 (TXD)      │               │              │
  │                       │            │       Pin 7 (CANH) ───┬─────────►│ Broche 6     │
  │ IO36 (RXD) ◄──────────┼────────────┤ Pin 4 (RXD)      │   [ R11 ]     │              │
  │                       │            │       Pin 6 (CANL) ───┴─────────►│ Broche 14    │
  └───────────────────────┘            │                  │  (120 Ω)      └──────────────┘
                                       │ Pin 5 (VIO=3.3V) │  Terminaison
                                       │ Pin 3 (VCC=5.0V) │
                                       └──────────────────┘
```

* **Principe du Signal Différentiel (`CANH` et `CANL`) :**
  * Au lieu de transmettre un signal par rapport à la masse (vulnérable aux parasites), le bus CAN utilise deux fils torsadés symétriques :
    * **Bit récessif (Niveau logique "1") :** CANH = 2.5V, CANL = 2.5V → Différence = 0V.
    * **Bit dominant (Niveau logique "0") :** CANH = 3.5V, CANL = 1.5V → Différence = **+2.0V**.
  * *Immunité totale au bruit :* Si un parasite électromagnétique (ex. étincelle d'allumage) frappe le faisceau automobile, il affecte simultanément et identiquement les deux fils (CANH = 8.5V, CANL = 6.5V). À l'arrivée, le récepteur soustrait les deux tensions : (8.5V - 6.5V) = 2.0V ! **Le parasite est mathématiquement éliminé.**
* **Résistance de Terminaison de Ligne `R11` (120 Ω - `R0805`) :**
  * *Rôle :* Un câble de transmission se comporte comme un guide d'ondes à haute fréquence. Si l'extrémité de la ligne est ouverte, les signaux électriques rebondissent à l'extrémité du câble (phénomène d'écho ou de réflexion d'onde) et viennent corrompre les trames suivantes. La résistance de 120 Ω équivaut à l'impédance caractéristique de la paire torsadée et absorbe l'onde sans aucune réflexion.
* **Broche d'Adaptation de Niveau Logique `VIO` (Broche 5 de `U2`) :**
  * Le circuit analogique de `U2` nécessite 5V sur sa broche `VCC` pour émettre les tensions requises sur le bus automobile.
  * Cependant, les broches de l'ESP32 ne tolèrent que **3.3V max**. En connectant `VIO` au rail 3.3V, `U2` adapte automatiquement ses signaux logiques `TXD` et `RXD` à 3.3V, garantissant une sécurité absolue pour le microcontrôleur.

---

#### Bloc 6 : Interface de Ligne K-Line ISO 9141-2 (`U3` - L9637D, `R2`, `R3`)

La Daewoo Kalos (2003) utilise principalement la ligne **K-Line** pour son calculateur moteur (ECU).

```
   ESP32 (UART)                        Transceiver K-Line U3               Prise OBD-II
  ┌─────────────────────┐            ┌──────────────────────┐             ┌──────────────┐
  │ IO5 (TX) ──► [ R3 ] ─┼───────────►│ Pin 4 (TX)           │             │              │
  │              (10 Ω) │            │                      │             │              │
  │ IO4 (RX) ◄── [ R2 ] ─┼────────────┤ Pin 1 (RX)           │             │              │
  │              (10 Ω) │            │                      │             │              │
  └─────────────────────┘            │ Pin 6 (K) ───────────┼────────────►│ Broche 7     │
                                     │                      │             │ (Ligne K 12V)│
                                     │ Pin 7 (VS = 12V)     │             └──────────────┘
                                     │ Pin 3 (VCC = 3.3V)   │
                                     └──────────────────────┘
```

* **Principe de la Ligne K-Line (ISO 9141 / ISO 14230 KWP2000) :**
  * C'est une ligne de communication **bidirectionnelle mono-fil** (*half-duplex*) fonctionnant aux niveaux de tension de la batterie automobile (**0V = état bas, 12V = état haut**).
* **Transceiver Spécialisé `U3` (`L9637D013TR`) :**
  * Il convertit les niveaux 12V de la voiture en signaux logiques 3.3V pour l'ESP32, et inversement. Il intègre des protections contre les courts-circuits permanents à la masse ou au 12V et une sécurité thermique.
* **Résistances d'Amortissement Série `R2` et `R3` (10 Ω - `R0805`) :**
  * *Rôle :* Placés en série sur les lignes numériques `UART_RX` et `UART_TX`, ces résistances étouffent les réflexions parasites (*damping*) et limitent le courant d'éventuelles décharges électrostatiques sur les broches GPIO de l'ESP32.

---

#### Bloc 7 : Port USB-C, Programmation & Protections ESD (`J2`, `U7`, `U8`, `R4`, `R5`, `VBUS_5V`)

Le connecteur USB-C permet de flasher le firmware dans l'ESP32, d'afficher les logs série de débogage et de tester la carte sur un banc de test sans être branché sur la voiture.

```
       Prise USB-C (J2)                     Protections ESD             ESP32-S3 (U1)
  ┌─────────────────────────┐             ┌─────────────────┐         ┌───────────────┐
  │ Broche A6/B6 (D+) ──────┼──────────┬──┤ U7 (SD05C TVS)  │────────►│ IO20 (USB D+) │
  │                         │          │  └────────┬────────┘         │               │
  │ Broche A7/B7 (D-) ──────┼────┬─────┼──┤ U8 (SD05C TVS)  │────────►│ IO19 (USB D-) │
  │                         │    │     │  └────────┬────────┘         └───────────────┘
  │ Broches A5 (CC1) ──[R4]─┼─┐  │     │           │
  │ Broches B5 (CC2) ──[R5]─┼─┤  │     │          GND
  │              (5.1 kΩ)   │ │  │     │
  │                         │ ▼  ▼     ▼
  └─────────────────────────┴─┴──┴─────┴───────────────────────────────────────────────
```

* **Résistances de Configuration `R4` et `R5` (5.1 kΩ pull-down - `R0805`) :**
  * *Pourquoi sont-elles obligatoires en USB-C ?* Dans la norme USB-C, les broches `CC1` et `CC2` déterminent qui alimente qui. Les alimentations et chargeurs modernes USB-C (Power Delivery / chargeurs intelligents) ne délivrent **aucun courant** tant qu'ils ne détectent pas une résistance de 5.1 kΩ reliée à la masse sur la broche CC. Sans `R4` et `R5`, la carte ne recevrait jamais de courant 5V sur un chargeur USB-C !
* **Diodes de Protection Antistatique ESD `U7` et `U8` (`SD05C` - boîtier SOD-323) :**
  * *Danger de l'électricité statique :* En touchant les contacts métalliques d'un câble USB, le corps humain peut décharger des milliers de volts (décharge électrostatique ESD).
  * Les diodes `U7` et `U8` sont des diodes TVS bidirectionnelles ultra-rapides capables de canaliser une décharge de **±30 000 Volts** à la masse en moins d'une nanoseconde, tout en présentant une capacité parasite quasi-nulle (< 3 pF) pour ne pas déformer les signaux USB haute vitesse (12 Mbit/s).
* **Point de Test Cuivre `VBUS_5V` (`Test-Point-0.5mm`) :**
  * Pad cuivre rond permettant de vérifier facilement au multimètre ou à l'oscilloscope la présence de la tension d'alimentation 5V issue du câble USB lors de la mise au point sur table.

---

#### Bloc 8 : Cœur de Traitement et Télécommunications (`U1` - ESP32-S3, `LED1`, `R8`)

* **SoC `U1` (`ESP32-S3-WROOM-1-N16R8`) :**
  * Processeur 32-bit double cœur cadencé à 240 MHz avec **16 Mo de mémoire Flash** et **8 Mo de PSRAM**.
  * Intègre nativement le contrôleur USB OTG (pas besoin de puce convertisseur série externe type CH340/CP2102).
  * Intègre le contrôleur matériel **TWAI** (*Two-Wire Automotive Interface*), 100% compatible avec la norme CAN 2.0B.
  * Antenne 2.4 GHz gravée sur le PCB assurant la liaison sans fil Wi-Fi et Bluetooth Low Energy (BLE) avec l'application mobile.
* **LED d'État `LED1` (Verte - `0603`) & Résistance de Limitation `R8` (1.8 kΩ) :**
  * Pilotée par la broche `IO2` de l'ESP32.
  * *Calcul de la résistance de limitation :* Avec une tension de sortie de 3.3V et une tension de seuil de LED verte de Vf ≈ 2.1V :
    > **I_LED = (Vio - Vf) / R8 = (3.3V - 2.1V) / 1 800 Ω = 1.2V / 1 800 Ω ≈ 0.67 mA**
    Cette valeur garantit un voyant parfaitement visible tout en consommant un courant dérisoire sans échauffement ni surcharge de la broche du microcontrôleur.

---

### 2.3 Tableau de Synthèse : "Contraintes du Véhicule vs Solutions Électroniques"

| Contrainte du Véhicule | Risque pour l'Électronique | Solution Technique Implémentée | Composants Dédiés |
| :--- | :--- | :--- | :--- |
| **Pics de surtension alternateur (*Load Dump*)** | Destruction instantanée par claquage diélectrique (> 40V) | Écrêtage transitoire à 24V vers la masse | Diode TVS 24V [`D1`](file:///D:/Dev/ODB/README.md#L58) |
| **Inversion accidentelle de polarité batterie** | Court-circuit destructeur de tous les circuits intégrés | Commutation automatique sans perte par MOSFET | P-MOSFET [`Q1`](file:///D:/Dev/ODB/README.md#L65) + N-MOSFET [`Q2`](file:///D:/Dev/ODB/README.md#L66) |
| **Court-circuit accidentel sur le faisceau** | Échauffement critique, fonte des pistes, risque d'incendie | Coupure thermique réarmable sans intervention | Fusible PPTC réarmable [`F1`](file:///D:/Dev/ODB/README.md#L60) |
| **Chute de tension 12V → 5V à fort courant** | Surchauffe extrême si régulateur linéaire classique (3.5W dissipés) | Conversion à découpage haute fréquence (570 kHz, rdt > 85%) | Étage Buck [`U4`](file:///D:/Dev/ODB/README.md#L81) + Inductance [`L1`](file:///D:/Dev/ODB/README.md#L63) + Diode [`D2`](file:///D:/Dev/ODB/README.md#L59) |
| **Bruit de hachage électromagnétique sur la radio** | Portée Wi-Fi/Bluetooth dégradée, instabilité analogique | Double filtrage : Régulateur linéaire LDO + Perle de ferrite | LDO 3.3V [`U5`](file:///D:/Dev/ODB/README.md#L82) + Ferrite [`FB1`](file:///D:/Dev/ODB/README.md#L61) + [`C6`](file:///D:/Dev/ODB/README.md#L54) |
| **Micro-coupures lors des commutations de l'ESP32** | Chute fugitive de tension locale, redémarrage (*brownout*) | Réservoirs d'énergie locale placés à < 2 mm des broches | Condensateurs céramiques [`C1`, `C2`, `C3`, `C4`](file:///D:/Dev/ODB/README.md#L49-L52) |
| **Parasites d'allumage moteur sur le bus CAN** | Trames de diagnostic corrompues ou illisibles | Transmission différentielle symétrique + adaptation d'impédance | Transceiver CAN [`U2`](file:///D:/Dev/ODB/README.md#L79) + Terminaison 120 Ω [`R11`](file:///D:/Dev/ODB/README.md#L74) |
| **Signaux 12V de la ligne K-Line Daewoo Kalos** | Destruction des broches du microcontrôleur limitées à 3.3V | Translation de niveau bidirectionnelle 12V ↔ 3.3V | Transceiver K-Line [`U3`](file:///D:/Dev/ODB/README.md#L80) + Résistances série [`R2`, `R3`](file:///D:/Dev/ODB/README.md#L67-L68) |
| **Décharges électrostatiques (ESD) lors du branchement USB** | Claquage des broches USB internes du silicium ESP32 | Dérivation des étincelles (jusqu'à 30 kV) en < 1 ns | Diodes ESD bidirectionnelles [`U7`, `U8`](file:///D:/Dev/ODB/README.md#L83-L84) |
| **Négociation de charge USB Type-C** | Pas de tension 5V délivrée par les chargeurs récents | Détection automatique d'appareil consommateur (Sink) | Résistances pull-down 5.1 kΩ [`R4`, `R5`](file:///D:/Dev/ODB/README.md#L69-L70) |

---

### 2.4 Nomenclature Complète du Schéma & PCB (37 composants)

Inventaire extrait directement du projet actif via l'API EasyEDA Pro :

| Désignateur | Valeur (`Value`) | Référence Fabricant (`Device`) | Empreinte (`Footprint`) | Description / Fonction |
| :--- | :--- | :--- | :--- | :--- |
| **C1** | 100nF | CL10B104KB8NNNC | `C0603` | Découplage alimentation ESP32 (rail 3.3V) |
| **C2** | 100nF | CL10B104KB8NNNC | `C0603` | Découplage alimentation ESP32 (rail 3.3V) |
| **C3** | 100nF | CL10B104KB8NNNC | `C0603` | Découplage alimentation transceiver CAN `U2` |
| **C4** | 100nF | CL10B104KB8NNNC | `C0603` | Découplage alimentation transceiver K-Line `U3` |
| **C5** | 1uF | CL10A105KA8NNNC | `C0603` | Bootstrap convertisseur Buck `U4` (broches BOOT → PH) |
| **C6** | 1uF | CL10A105KA8NNNC | `C0603` | Filtrage sortie régulateur LDO `U5` (rail 3.3V) |
| **C7** | 22uF | TCC1206X5R226K250HT | `C1206` | Condensateur réservoir entrée Buck `U4` (rail 12V VIN) |
| **C8** | 22uF | TCC1206X5R226K250HT | `C1206` | Condensateur filtrage sortie Buck `U4` (dérivation rail 5V vers GND) |
| **C9** | 3.3nF | CL10B332KB8NNNC | `C0603` | Condensateur de compensation de boucle Buck `U4` (broche COMP vers GND) |
| **D1** | *—* | SMBJ24A_C19077578 | `SMB_L4.3-W3.6-LS5.3-RD` | Diode TVS 24V de protection contre les surtensions OBD-II |
| **D2** | *—* | SS34_C52023881 | `SMA_L4.3-W2.6-LS5.1-RD` | Diode Schottky 40V 3A de roue libre (Cathode sur PH, Anode sur GND) pour convertisseur Buck `U4` |
| **F1** | *—* | MF-MSMF050-2 | `F1812` | Fusible réarmable PPTC 0.5A protection ligne 12V |
| **FB1** | *—* | BLM18PG121SN1D_C14709 | `L0603` | Perle de ferrite pour filtrage HF du rail 3.3V LDO |
| **J2** | *—* | U263-161N-5BVZ15-2 | `USB-TH-TYPE-C_U263-161N-5BVZ14-2` | Connecteur USB Type-C 16 broches (flash, debug et test 5V) |
| **L1** | 10uH | YNR6045-100M | `IND-SMD_L6.0-W6.0` | Inductance blindée 10µH étage Buck `U4` |
| **LED1** | *—* | PSC-1608U52GC-G4 | `LED0603-RD_GREEN` | LED d'état verte pilotée par la broche IO2 de l'ESP32 |
| **Q1** | *—* | IRLML2244TRPBF | `SOT-23-3_L2.9-W1.6-P1.90-LS2.8-BR` | P-MOSFET protection contre l'inversion de polarité 12V |
| **Q2** | *—* | 2N7002_C50176485 | `SOT-23-3_L2.9-W1.3-P1.90-LS2.4-BR` | N-MOSFET commande et commutation alimentation |
| **R2** | 10Ω | FRC0805F10R0TS | `R0805` | Résistance série amortissement ligne K-Line RX |
| **R3** | 10Ω | FRC0805F10R0TS | `R0805` | Résistance série amortissement ligne K-Line TX |
| **R4** | 5.1kΩ | 0805W8F5101T5E | `R0805` | Résistance pull-down USB-C configuration CC1 |
| **R5** | 5.1kΩ | 0805W8F5101T5E | `R0805` | Résistance pull-down USB-C configuration CC2 |
| **R6** | 10kΩ | 0805W8F1002T5E | `R0805` | Résistance de polarisation / pull-up |
| **R8** | 1.8kΩ | FRC0805J182 TS | `R0805` | Résistance de limitation de courant LED1 |
| **R10** | 10kΩ | 0805W8F1002T5E | `R0805` | Résistance de polarisation / pull-up |
| **R11** | 120Ω | 0805W8F1200T5E | `R0805` | Résistance de terminaison de ligne différentielle CAN |
| **R12** | 10kΩ | 0805W8F1002T5E | `R0805` | Résistance haute pont diviseur feedback Buck `U4` (rail 5V vers VSENSE) |
| **R13** | 1.91kΩ | 0805W8F1911T5E | `R0805` | Résistance basse pont diviseur feedback Buck `U4` (VSENSE vers GND) |
| **R14** | 10kΩ | 0805W8F1002T5E | `R0805` | Résistance série compensation de boucle Buck `U4` (broche COMP) |
| **U1** | 2.4GHz | ESP32-S3-WROOM-1-N16R8 | `WIRELM-SMD_ESP32-S3-WROOM-1` | SoC ESP32-S3 Wi-Fi 2.4 GHz + BLE 5.0 (16MB Flash / 8MB PSRAM) |
| **U2** | *—* | TJA1051T/3/1J | `SOIC-8_L4.9-W3.9-P1.27-LS6.0-BL` | Transceiver CAN haute vitesse avec broche VIO (3.3V) |
| **U3** | *—* | E-L9637D013TR | `SOIC-8_L4.9-W3.9-P1.27-LS6.0-BL` | Transceiver K-Line ISO 9141 / KWP2000 |
| **U4** | *—* | TPS54331DR | `SOIC-8_L5.0-W4.0-P1.27-LS6.0-BL` | Régulateur abaisseur Step-Down Buck 12V → 5V, 3A |
| **U5** | *—* | LDL1117S33R | `SOT-223-4_L6.5-W3.5-P2.30-LS7.0-BR` | Régulateur linéaire LDO 5V → 3.3V faible bruit, 1.2A |
| **U7** | *—* | SD05C_C53238084 | `SOD-323_L1.7-W1.3-LS2.5-BI` | Diode ESD bidirectionnelle protection ligne USB D+ |
| **U8** | *—* | SD05C_C53238084 | `SOD-323_L1.7-W1.3-LS2.5-BI` | Diode ESD bidirectionnelle protection ligne USB D- |
| **VBUS_5V**| *—* | Test-Point | `Test-Point-0.5mm` | Point de test pad cuivre pour le rail 5V USB |

---

### 2.5 Répertoire des Équipotentielles & Signaux du Scanner (Nets)

Pour garantir une lisibilité absolue lors de la conception, du débogage et du routage, chaque liaison électrique (Net) du projet est rigoureusement identifiée par un nom fonctionnel explicite, éliminant tout identifiant anonyme générique auto-généré :

| Nom du Net | Composants Reliés (Broches) | Rôle & Fonction Électrique | Domaine / Bloc |
| :--- | :--- | :--- | :--- |
| **`+12V`** | OBD-II (Pin 16), `D1(1)`, `F1(1)` | Alimentation batterie brute issue de la prise OBD-II (écrêtée à 24V par la diode TVS `D1`). | Alimentation / Entrée |
| **`+12V_FUSED`** | `F1(2)`, `Q1(2)` (Source) | Alimentation 12V protégée en surintensité en sortie du fusible réarmable PPTC 0.5A. | Alimentation / Sécurité |
| **`+12V_PROT`** | `Q1(3)` (Drain), `C7(1)`, `U4(2)` (`VIN`) | Rail 12V sécurisé contre l'inversion de polarité, alimentant le convertisseur Buck et son condensateur réservoir. | Alimentation / Sécurité |
| **`PH_BUCK`** | `U4(8)` (`PH`), `L1(1)`, `D2(1)` (Cathode), `C5(2)` | Nœud de découpage haute fréquence (570 kHz) reliant le transistor interne, l'inductance et la diode Schottky de roue libre. | Alimentation / Buck |
| **`BOOT_BUCK`** | `U4(1)` (`BOOT`), `C5(1)` | Ligne de pompe de charge bootstrap rehaussant la tension de grille pour piloter le MOSFET High-Side interne. | Alimentation / Buck |
| **`VSENSE_BUCK`** | `U4(5)` (`VSENSE`), `R12(1)`, `R13(1)` | Point milieu du diviseur de tension de contre-réaction asservissant la sortie 5.0V sur la référence interne 0.800V. | Alimentation / Buck |
| **`COMP_BUCK`** | `U4(6)` (`COMP`), `R14(1)` | Sortie de l'amplificateur d'erreur transconductance reliée au filtre de compensation de boucle de régulation. | Alimentation / Buck |
| **`RC_COMP`** | `R14(2)`, `C9(1)` | Nœud intermédiaire série du réseau RC de compensation de phase (stabilité dynamique). | Alimentation / Buck |
| **`+5V`** | `L1(2)`, `C8(1)`, `U5(3)` (`VIN`), `R12(2)`, `U2(3)` (`VCC`) | Rail d'alimentation 5.0V régulé issu de l'étage Buck, distribuant la puissance au régulateur LDO et au transceiver CAN. | Alimentation / Rail 5V |
| **`3.3V_PRE`** | `U5(4)` (`VOUT` / Tab), `FB1(1)` | Sortie 3.3V brute du régulateur linéaire LDO avant élimination des harmoniques radiofréquences. | Alimentation / LDO |
| **`3.3V`** | `FB1(2)`, `C6(1)`, `C1(1)`, `C2(1)`, `C3(1)`, `C4(1)`, `U1(2)`, `U2(5)` (`VIO`), `U3(3)` (`VCC`) | Rail d'alimentation logique 3.3V purifié et filtré, alimentant le microcontrôleur ESP32 et les étages logiques. | Alimentation / Rail 3.3V |
| **`GND`** | Plan de masse, pads thermiques, blindages, condensateurs, transceivers | Potentiel de référence zéro volt (0V) commun assurant le retour des courants et le blindage électromagnétique. | Référence / Masse |
| **`GATE_PMOS`** | `Q1(1)` (Grille), `Q2(3)` (Drain), `R10(2)` | Commande de grille du P-MOSFET tirée à la masse par le N-MOSFET `Q2` pour autoriser le passage du 12V. | Commutation / Contrôle |
| **`GATE_NMOS`** | `Q2(1)` (Grille), `R6(2)` | Polarisation de grille du N-MOSFET de commande depuis le 12V à travers la résistance `R6`. | Commutation / Contrôle |
| **`LED_STATUS`** | `U1(38)` (`IO2`), `R8(1)` | Sortie numérique du microcontrôleur pilotant l'allumage du témoin visuel de fonctionnement. | Interface / Statut |
| **`LED_ANODE`** | `R8(2)`, `LED1(1)` (Anode) | Liaison à courant limité (0.67 mA) entre la résistance de limitation `R8` et la LED d'état verte. | Interface / Statut |
| **`VBUS_USB`** | `J2` (`A4, B9, A9, B4`), `VBUS_5V` (Point de test) | Tension d'alimentation 5V issue du câble USB-C hôte, accessible sur pad de test pour les mesures sur banc. | Interface / USB-C |
| **`USB_CC1`** | `J2(A5)` (`CC1`), `R4(1)` (5.1 kΩ) | Ligne de configuration USB Type-C canal 1 permettant la détection d'un appareil récepteur (*Sink*). | Interface / USB-C |
| **`USB_CC2`** | `J2(B5)` (`CC2`), `R5(1)` (5.1 kΩ) | Ligne de configuration USB Type-C canal 2 permettant la détection d'un appareil récepteur (*Sink*). | Interface / USB-C |
| **`USB_D+`** | `J2` (`A6, B6`), `U7(1)` (TVS ESD), `U1(14)` (`IO20`) | Ligne de données différentielle USB positive haute vitesse avec protection antistatique 30 kV. | Interface / USB-C |
| **`USB_D-`** | `J2` (`A7, B7`), `U8(1)` (TVS ESD), `U1(13)` (`IO19`) | Ligne de données différentielle USB négative haute vitesse avec protection antistatique 30 kV. | Interface / USB-C |
| **`CANH`** | `U2(7)` (`CANH`), `R11(1)` (120 Ω), OBD-II (Pin 6) | Ligne de bus CAN différentielle niveau haut (2.5V récessif / 3.5V dominant). | Communication / CAN |
| **`CANL`** | `U2(6)` (`CANL`), `R11(2)` (120 Ω), OBD-II (Pin 14) | Ligne de bus CAN différentielle niveau bas (2.5V récessif / 1.5V dominant). | Communication / CAN |
| **`TXD`** | `U1(37)` (`TXD0`), `U2(1)` (`TXD`) | Signal d'émission logique 3.3V du contrôleur TWAI de l'ESP32 vers le transceiver CAN. | Communication / CAN |
| **`RXD`** | `U1(36)` (`RXD0`), `U2(4)` (`RXD`) | Signal de réception logique 3.3V du transceiver CAN vers le contrôleur TWAI de l'ESP32. | Communication / CAN |
| **`K_LINE`** | `U3(6)` (`K`), OBD-II (Pin 7) | Ligne de diagnostic bidirectionnelle automobile 12V (protocole ISO 9141-2 / Daewoo Kalos). | Communication / K-Line |
| **`K_RX_IC`** | `U3(1)` (`RX`), `R2(1)` (10 Ω) | Sortie numérique 3.3V du récepteur K-Line avant amortissement de ligne. | Communication / K-Line |
| **`UART_RX_MCU`** | `R2(2)` (10 Ω), `U1(4)` (`IO4`) | Signal de réception UART amorti arrivant sur la broche du microcontrôleur ESP32. | Communication / K-Line |
| **`K_TX_IC`** | `U3(4)` (`TX`), `R3(1)` (10 Ω) | Entrée d'émission du transceiver K-Line après amortissement de ligne. | Communication / K-Line |
| **`UART_TX_MCU`** | `R3(2)` (10 Ω), `U1(5)` (`IO5`) | Signal d'émission UART issu du microcontrôleur ESP32 vers la résistance d'amortissement. | Communication / K-Line |

---

## 3. État Actuel de l'Implémentation

* **Schématique :** Schéma complet validé sous EasyEDA Pro (feuille `P1`), mis à jour et corrigé le 03/09/2026 :
  * *Étage Buck TPS54331 (`U4`) :* Recâblage conforme de la diode Schottky de roue libre `D2` (`SS34` : cathode sur `PH`, anode sur `GND`), condensateur de sortie `C8` (22 µF) en dérivation vers la masse, ajout du pont diviseur de feedback `R12` (10 kΩ) / `R13` (1.91 kΩ) fixant la régulation à 5.0V sur `VSENSE`, et du réseau série de compensation `R14` (10 kΩ) / `C9` (3.3 nF) sur `COMP`.
  * *Rail +5V :* Alimentation de l'étage LDO `U5` et de la broche 3 (`VCC`) du transceiver CAN `U2`.
  * *Visibilité & Raccordement :* Toutes les étiquettes (`R12`, `R13`, `R14`, `C9` et leurs valeurs) et les continuités physiques vers les broches et drapeaux `GND` sont vérifiées.
* **Placement des composants (PCB) :** Placement 2D compact validé pour l'ensemble des 37 composants (retouches et disposition finale validées le 05/09/2026) :
  * *Bloc Puissance (à gauche) :* Protections 12V, convertisseur buck `TPS54331` (avec `D2`, `C5`, `C7`, `C8`, `R12`, `R13`, `R14`, `C9`) et régulateur `LDL1117`.
  * *Bloc Interfaces (au centre) :* Puces CAN `U2` et K-Line `U3`.
  * *Bloc Logique & Antenne (à droite) :* Module ESP32 avec son antenne orientée vers le bord extérieur libre.
* **Contour de carte (Board Outline) :** Défini et tracé sur la couche dédiée.

### Schéma

![Schéma ODB2 Scanner](./images/Schematic.png)

### PCB

![PCB ODB2 Scanner](./images/PCB.png)

---

### 3.1 Prochaine Étape Immédiate

> [!IMPORTANT]
> **Reprise immédiate : Synchronisation Netlist & Plan de Masse / Gestion RF (Section 4.3)**
>
> 1. **Vérification préliminaire de la synchronisation Netlist :**
>    * La Section 4.2 est entièrement terminée, validée et commitée (0 erreur d'isolement / clearance).
>    * Suite à l'ouverture de l'onglet Schéma pour les exports HD, un drapeau *Netlist Error* ("PCB and schematic netlist does not match") est réapparu dans le panneau DRC d'EasyEDA Pro.
>    * **Action à la reprise :** Cliquer sur *Import Changes* dans le panneau DRC ou faire *Conception > Mettre à jour le PCB...* (`Alt + D` puis `U`) pour réconcilier le jeton de synchronisation Schéma/PCB.
>
> 2. **Démarrage de la Section 4.3 (Plan de Masse & Gestion RF) :**
>    * **Zone d'exclusion RF (Keepout d'antenne) :** Définir la zone `Copper Keepout` (`NO_POURS` / `NO_FILLS`) sur toutes les couches sous l'antenne méandre 2.4 GHz de l'ESP32 (`U1`) dans le coin supérieur droit (`x ∈ [3850, 4450], y ∈ [-220, 50]`).
>    * **Plans de masse (`GND`) :** Couler le plan de cuivre `GND` sur **Top Layer** (Layer 1) et sur **Bottom Layer** (Layer 2) avec remplissage Solid et dégagement thermique (Thermal Relief).
>    * **Vias de couture (Stitching Vias) :** Disposer la matrice de vias sous le pad thermique central de l'ESP32 (`U1_41`), autour de la boucle de découpage buck (`U4`/`D2`/`L1`) et le long du périmètre de la carte pour interconnecter les plans haut et bas.
>    * **Contrôle DRC final :** Valider 0 erreur d'isolement (Clearance), 0 broche non connectée (les 40 broches GND seront résolues par les plans de masse) et 0 erreur de netlist.

---

## 4. Feuille de Route & Checklist (TODO)

### 4.1 Étage Buck & Routage des Pistes d'Alimentation
- [x] **Rail +12V et Protection :** *(Routage réalisé via l'API EasyEDA Pro)*
  - [x] Piste large (0.8 mm à 1.0 mm) reliant la broche 16 OBD → Fusible `F1` → Diode TVS `D1` → MOSFETs `Q1`/`Q2`. *(Pistes de puissance de 35 mil / ~0.89 mm tracées sur Top Layer, reliant l'entrée F1(1) à D1(1), F1(2) vers Q1(2) et polarisation R10/R6)*
  - [x] Piste 12V vers la broche 7 (`VS`) de `U3` (0.5 mm). *(Piste 20 mil / ~0.50 mm routée via Bottom Layer et 2 vias de 24/12 mil pour franchir l'étage de découpage central)*
  - [x] Polarisation et commande de grille des MOSFETs : `GATE_NMOS` (Q2 broche 1 vers R6) et `GATE_PMOS` (Q1 broche 1 vers Q2 broche 3 et R10). *(Routage Top/Bottom Layer dédié, 0 erreur DRC)*
- [x] **Synchronisation & Placement Buck :**
  - [x] Exécuter « Update PCB from Schematic » pour importer `R12`, `R13`, `R14`, `C9` et les nouveaux chevelus nets sur le PCB.
  - [x] Placer `D2`, `C8`, `R12`, `R13`, `R14`, `C9` sur le PCB selon les règles de minimisation des boucles d'induction et de bruit.
- [x] **Routage Étage Buck 12V → 5V (`TPS54331DR` / `U4`) :** *(Routage réalisé via l'API EasyEDA Pro, 0 erreur DRC)*
  - [x] Boucle de commutation courte et large : `U4` (broche 8 PH), inductance `L1` (10 µH) et diode Schottky `D2` (`SS34` : cathode sur PH, anode sur GND) (pistes de 32-35 mil sur Top Layer, longueur totale 843 mil).
  - [x] Condensateur d'entrée `C7` (22 µF) au plus près de la broche 2 (`VIN`) de `U4` et liaison 12V protégée depuis `Q1(3)` (piste large 32 mil, 818 mil).
  - [x] Condensateur de sortie `C8` (22 µF) et distribution du rail 5V vers `U5(3)` et `R12(2)` (piste 32 mil / 18 mil, 1309 mil).
  - [x] Condensateur de bootstrap `C5` (1 µF) entre broche 1 (`BOOT`) et broche 8 (`PH`) (piste 16 mil, 202 mil).
  - [x] Pont diviseur de feedback : `R12` (10 kΩ) et `R13` (1.91 kΩ) au plus près de la broche 5 (`VSENSE`) (piste 16 mil, 253 mil).
  - [x] Réseau de compensation : `R14` (10 kΩ) et `C9` (3.3 nF) au plus près de la broche 6 (`COMP`) (pistes 16 mil ultra-courtes de 130 mil et 63 mil).
- [x] **Étage Régulation 3.3V (`LDL1117S33R` / `U5`) :** *(Routage réalisé via l'API EasyEDA Pro, 0 erreur DRC)*
  - [x] Entrée `VIN` reliée au 5V (`L1` / `C8`).
  - [x] Sortie `VOUT` vers perle de ferrite `FB1` et condensateur de filtrage `C6` (1 µF). *(Piste 30/24 mil via Bottom Layer et 2 vias 24/12 mil pour 3.3V_PRE, puis bus 3.3V descendant à C6(1))*
  - [x] Distribution du rail 3.3V vers le réseau de découplage `C1` à `C4` (100 nF) puis broche 2 de l'ESP32 `U1`. *(Liaison Bottom Layer vers colonne C1-C4 et dérivation directe Top Layer 24 mil vers ESP32 U1_2)*
  - [x] Distribution 3.3V vers broche 5 (`VIO`) de `U2` et broche 3 (`VCC`) de `U3`. *(Dérivation Top Layer directe vers U2_5 et dérivation Bottom Layer avec 2 vias 24/12 mil abordant U3_3 verticalement par le sud avec dégagement > 28 mil)*
- [x] **Alimentation 5V :** *(Routage réalisé via l'API EasyEDA Pro, 0 erreur DRC)*
  - [x] Distribution du rail 5V vers broche 3 (`VIN`) de `U5` et résistance de contre-réaction `R12`.
  - [x] Distribution du rail 5V vers broche 3 (`VCC`) du transceiver CAN `U2`. *(Traversée Top Layer au-dessus du bus 3.3V, puis passage sous broches U2(1)-U2(2) via Bottom Layer et remontée verticale Top Layer dans U2_3)*
  - [x] Raccordement du point de test `VBUS_5V` depuis le connecteur USB-C `J2` (`VBUS_USB`). *(Piste 10 mil sur Top Layer, 0 erreur DRC)*

### 4.2 Routage des Signaux de Communication
- [x] **Ligne K-Line (`U3` - `L9637D013TR`) :** *(Routage réalisé via l'API EasyEDA Pro, 0 erreur DRC)*
  - [x] `UART_RX` : Broche 1 (`RX`) de `U3` → `R2` (10 Ω) → Broche 4 (`IO4`) de l'ESP32. *(Liaison K_RX_IC 10 mil sur Top Layer jusqu'à R2, puis UART_RX_MCU 8 mil sur Top Layer le long du corridor x = 3635 vers U1_4)*
  - [x] `UART_TX` : Broche 4 (`TX`) de `U3` → `R3` (10 Ω) → Broche 5 (`IO5`) de l'ESP32. *(Liaison K_TX_IC via Bottom Layer et 2 vias 24/12 mil vers R3_2, puis UART_TX_MCU 8 mil sur Top Layer le long du corridor x = 3655 vers U1_5)*
  - [x] Ligne physique `K` : Broche 6 (`K`) de `U3` vers broche 7 de la prise OBD-II. *(Broche assignée au net K_LINE)*
- [x] **Bus CAN (`U2` - `TJA1051T/3/1J`) :** *(Routage réalisé via l'API EasyEDA Pro, 0 erreur DRC)*
  - [x] Lignes logiques : `TXD` (broche 1) → Broche 37 (`TXD0`) ESP32 ; `RXD` (broche 4) → Broche 36 (`RXD0`) ESP32. *(Routage 10 mil via Bottom Layer sous l'ESP32 et vias 24/12 mil à x = 4320 et x = 4260)*
  - [x] Mode normal : Broche 8 (`S`) → Masse `GND`. *(Broche assignée au net GND)*
  - [x] Paire différentielle CAN : `CANH` (broche 7) et `CANL` (broche 6) → Terminaison `R11` (120 Ω) → Broches 6 et 14 OBD-II *(routage 10 mil 100% sur Top Layer sans aucun via ni croisement, symétrie préservée)*.
- [x] **LED d'État (`LED1`) :** *(Routage réalisé via l'API EasyEDA Pro, 0 erreur DRC)*
  - [x] Broche 38 (`IO2`) ESP32 → `R8` (1.8 kΩ) → Anode `LED1` → Cathode `GND`. *(Liaison LED_STATUS depuis U1_38 via Bottom Layer à x = 4350 vers R8_1, puis liaison directe LED_ANODE sur Top Layer vers l'anode de LED1)*
- [x] **Port USB-C & Programmation (`J2`) :** *(Routage réalisé via l'API EasyEDA Pro, 0 erreur DRC)*
  - [x] Signaux `USB_D-` et `USB_D+` depuis `J2` via protections ESD `U8` / `U7` vers broches 13 (`IO19`) et 14 (`IO20`) de l'ESP32. *(Paire différentielle 8 mil / espacement 12 mil sur Top Layer passant par les diodes TVS U7/U8 pivotées à 270° au plus près de J2, puis corridor x = 3790/3810 jusqu'à U1_13 et U1_14)*
  - [x] Résistances de configuration CC : Broches `CC1` et `CC2` de `J2` vers `R4` et `R5` (5.1 kΩ) → `GND`. *(Ligne USB_CC1 via autoroute nord puis pont multicouche Layer 1/2 vers R4_1 ; ligne USB_CC2 sur Bottom Layer avec pont Layer 1 au-dessus du rail 3.3V vers R5_1)*
  - [x] Raccordement `VBUS_USB` : Pastilles d'alimentation A4B9 et A9B4 reliées à la diode D5. *(Liaison Top Layer 10 mil sur autoroute nord reliant A4B9, A9B4 et D5_1)*


### 4.3 Plan de Masse & Gestion RF
- [ ] **Zone d'exclusion d'antenne (Keep-out Zone) :**
  - [ ] **Définir le keepout AVANT de couler les plans GND** (sinon reprise manuelle du remplissage après coup).
  - [ ] Définir une zone `Copper Keepout` sur **toutes les couches (All Layers)** sous et autour de l'antenne méandre de l'ESP32 (coin supérieur droit).
  - [ ] Garantir l'absence totale de cuivre (aucun plan de masse ni piste) pour préserver les performances radio.
- [ ] **Plans de masse (Copper Area) :**
  - [ ] Plan `GND` sur **Top Layer** (remplissage Solid, dégagement 0.254 mm - 0.3 mm, Thermal Relief).
  - [ ] Plan `GND` sur **Bottom Layer** (remplissage Solid, dégagement 0.254 mm - 0.3 mm, Thermal Relief).
- [ ] **Vias de couture (Stitching Vias) :**
  - [ ] Vias de masse sous le pad thermique central de l'ESP32 (broches 41).
  - [ ] Vérifier la densité de vias pour la **dissipation thermique**, pas seulement pour passer le DRC.
  - [ ] Vias de masse au niveau des condensateurs de découplage et du bloc de découpage `U4`/`D2`.
  - [ ] Vias de masse réguliers le long du contour de carte.

### 4.4 Contrôle Intermédiaire (post plan de masse)
- [ ] **Contrôle DRC (Design Rule Check) :** Exécuter la vérification des règles de conception sous EasyEDA Pro (`Shift + R`) et corriger les erreurs éventuelles.
- [ ] **Visualisation 3D :** Vérification visuelle globale de l'assemblage et du dégagement mécanique.

### 4.5 Recherche d'optimisations du PCB
- [ ] **Optimisations du PCB :** Analyser le PCB à la recherche d'optimisations: déplacer des composants pour améliorer les performances / la stabilité, pour raccourcir des pistes ou supprimer des vias. Découper en sous tâches de 4.6

### 4.6 Implémentations des optimisations du PCB 

### 4.7 Contrôle Final
- [ ] **Silkscreen**: rajouter des informations sur le PCB pour délimiter des zones logiques (ie Alimentation). Decouper en sous tache de ce point
- [ ] **Contrôle DRC final** après optimisations (vérifier qu'aucune reprise n'a cassé un clearance ou un thermal relief).
- [ ] **Visualisation 3D finale**.

### 4.8 Fabrication
- [ ] **Export des fichiers de production :**
  - [ ] Fichiers Gerber & Perçage (Drill).
  - [ ] Fichier de nomenclature (BOM).
  - [ ] Fichier de placement des composants (CPL / Pick & Place).

---

## 5. Fichiers du Dépôt

* `ODB2-Scanner.eprj2` : Projet natif EasyEDA Pro v2 (contenant le schéma `P1` et le circuit imprimé `PCB1`).
* `README.md` : Documentation technique complète et suivi du projet.
* `LEARNINGS.md` : Journal de capitalisation technique et découvertes sur l'API EasyEDA Pro.
* `AGENTS.md` : Règles de gouvernance et consignes strictes pour les agents IA.

---

## 6. Automatisation IA via EasyEDA Pro

Pour permettre à un assistant IA (Claude Code, Codex, Antigravity, OpenCode...) de manipuler directement le schéma et le PCB en temps réel (routage autonome des pistes, placement, création des zones de cuivre, etc.), ce projet s'appuie sur le **skill officiel EasyEDA** : [easyeda/easyeda-api-skill](https://github.com/easyeda/easyeda-api-skill), maintenu par l'éditeur lui-même.

### 6.1 Architecture

Le principe repose sur un **pont local** qui fait le lien entre l'IA et l'API interne d'EasyEDA, laquelle n'existe que dans le contexte JavaScript de l'onglet navigateur :

```
IA (Claude Code / Copilot CLI / Antigravity / Codex)
        │  Agent Skill (SKILL.md) + API HTTP/WebSocket
        ▼
Serveur Node.js (pont local)  ─────  tourne sur le PC, port auto 49620-49629
        │  WebSocket (localhost)
        ▼
Extension .eext (run-api-gateway) ──  JavaScript, injectée dans l'onglet
        │  appel direct                     navigateur EasyEDA Pro
        ▼
API interne EasyEDA (eda.pcb_..., eda.sch_..., eda.dmt_...)
```

Deux briques distinctes, deux cycles de vie :
- Le **serveur Node.js** est relancé à chaque session (par le client IA ou manuellement).
- L'**extension `.eext`** est importée **une seule fois** dans EasyEDA Pro (Extensions → Extension Manager → Import Extension) et reste active tant qu'elle n'est pas désinstallée.

### 6.2 Installation

```bash
git clone https://github.com/easyeda/easyeda-api-skill
cd easyeda-api-skill
npm install
npm run build:docs   # génère la documentation API structurée dans docs/
npm run server       # démarre le pont WebSocket/HTTP (port auto 49620-49629)
```

### 6.3 Installation de l'extension EasyEDA

1. Télécharger `run-api-gateway.eext` depuis <https://jlc-ext.com/item/oshwhub/run-api-gateway>.
2. Dans EasyEDA Pro : **Settings → Extensions → Extension Manager → Import Extension**.
3. Sélectionner le fichier téléchargé et vérifier que **"Allow External Interaction"** reste activé.
4. Ouvrir `ODB2-Scanner.eprj2` : l'extension se connecte automatiquement au serveur en scannant la plage de ports et en validant le handshake (`service: "easyeda-bridge"`).

### 6.4 Connexion depuis le client IA & Démarrage automatique

Aucune configuration `mcp_config.json` n'est nécessaire. Les outils compatibles **Agent Skills** (Claude Code, OpenCode, QwenCode, Antigravity...) lisent automatiquement `SKILL.md` à la racine du dépôt cloné et disposent alors des instructions et de la documentation API.

#### Automatisation sous Antigravity (Lifecycle Hook)
Pour éviter de devoir lancer manuellement `npm run server` ou de valider des invites de permissions de commande shell à chaque session :
* **Hook de cycle de vie** : Le projet inclut un hook configuré dans [`.agents/hooks.json`](.agents/hooks.json) appelant le script [`.agents/ensure-bridge.mjs`](.agents/ensure-bridge.mjs).
* **Déclenchement automatique** : Dès qu'une interaction commence dans `agy` (`PreInvocation`), le script teste si le port `49620` (ou plage `49620-49629`) répond. Si le pont est inactif, il est démarré automatiquement en arrière-plan détaché (logs consignés dans `.agents/easyeda-bridge.log`).
* **Comportement lors d'un arrêt forcé (`kill`)** : Si les processus `node` sont arrêtés manuellement, le serveur reste coupé pendant l'inactivité. Dès que vous envoyez une nouvelle commande ou invite à l'IA, le hook détecte l'absence du serveur et le relance automatiquement avant de traiter la requête.
* **Désactivation du démarrage automatique** : Pour désactiver ce comportement et empêcher le démarrage en arrière-plan, il suffit de passer `"enabled": false` dans [`.agents/hooks.json`](.agents/hooks.json).

Pour un appel manuel ou un test (le port exact est affiché au démarrage du serveur, ex. `49620`) :

```bash
# Vérifier la connexion à EasyEDA
curl http://localhost:49620/health

# Exécuter du code EasyEDA à distance
curl -X POST http://localhost:49620/execute \
  -H "Content-Type: application/json" \
  -d '{"code": "return await eda.dmt_Project.getCurrentProjectInfo();"}'
```

### 6.5 Modules API pertinents pour ce projet

| Préfixe | Domaine | Classes clés utiles au routage de l'`ODB2-Scanner` |
|---|---|---|
| `PCB_` | PCB & Footprint | `PrimitiveLine` (pistes), `PrimitiveVia` (vias), `PrimitivePour` (plans de masse), `PrimitivePad`, `Drc` (vérification des règles), `Net`, `Layer` |
| `DMT_` | Gestion de document | `Project`, `Pcb`, `Board`, `EditorControl` |
| `SCH_` | Schématique | `PrimitiveComponent`, `PrimitiveWire` |
| `EPCB_` / `ESCH_` | Énumérations | `LayerId`, `PrimitiveType`, `PadType` |

Exemple de tracé de piste, tiré de la documentation du skill (unités en mil) :

```javascript
// Créer une piste cuivre sur la couche Top pour le net GND
await eda.pcb_PrimitiveLine.create(
  "GND",              // nom du net
  EPCB_LayerId.TOP,   // couche (énum, pas un nombre brut)
  0, 0,               // startX, startY
  100, 0              // endX, endY
);
```

Pour déplacer un élément existant (via, composant), le pattern asynchrone recommandé par le skill est :

```javascript
const prim = await eda.pcb_PrimitiveVia.get([viaId]);
const asyncPrim = prim.toAsync();
asyncPrim.setState_X(newX);
asyncPrim.setState_Y(newY);
asyncPrim.done();
```

### 6.6 Capitalisation & Découvertes Techniques

Pour retrouver l'ensemble des subtilités d'implémentation, astuces et découvertes sur l'API EasyEDA Pro (manipulation des pastilles, unités en mil, typage des couches, scripts de capture du canvas en Base64, et pièges de raccordement de schématique), se référer au document dédié :
👉 **[`./LEARNINGS.md`](./LEARNINGS.md)**.

### 6.7 Bonnes pratiques pour un routage PCB piloté par IA

En cohérence avec la checklist de routage de la Section 4 de ce document :

- **Ne pas s'appuyer sur l'auto-routeur intégré d'EasyEDA pour un résultat final** : la documentation officielle d'EasyEDA le déconseille elle-même *("Auto router is not good enough! Suggest routing manually!")*. Un agent IA doit raisonner piste par piste, pas déclencher l'auto-routeur en aveugle.
- **Toujours relire les positions réelles des pads avant de router** (`pcb_PrimitivePad.get(...)`) plutôt que de faire router l'IA sur des coordonnées supposées — c'est la méthode qui a fait ses preuves dans les retours d'expérience communautaires sur ce skill.
- **Vérifier le DRC après chaque lot de pistes tracées** (`pcb_Drc`), pas seulement à la fin du projet, pour détecter les courts-circuits ou chevauchements au plus tôt.
- **Sauvegarder ou versionner le fichier `.eprj2`** avant toute session de routage automatisé en masse — un script IA mal formulé peut modifier plusieurs pistes en une seule commande `execute`.
- **Router en dernier les rails de puissance** (12V, 5V, 3.3V — voir §4.1) avec des largeurs de piste explicitement spécifiées à l'agent, les erreurs de largeur de piste sur ces rails étant plus difficiles à repérer visuellement qu'un DRC de court-circuit.

## 7. Architecture logicielle

(TODO)