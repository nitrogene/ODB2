# Scanner OBD-II ESP32 (Daewoo Kalos)

Projet de conception matérielle (schématique et PCB) d'un scanner de diagnostic automobile OBD-II intelligent et communicant, optimisé pour une **Daewoo Kalos (2003, moteur 1.4L)**.

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
  * Double étage de conversion : abaisseur à découpage performant 12V $\rightarrow$ 5V (`U4` / `L1`) suivi d'un régulateur linéaire ultra-propre 3.3V (`U5` / `FB1`) pour l'ESP32 et la logique.

---

## 2. Architecture Matérielle & Composants Clés

| Fonction | Réf. Composant | Boîtier | Description |
| :--- | :--- | :--- | :--- |
| **Microcontrôleur & RF** | `U1` (ESP32-WROOM-32E) | SMD Module | MCU Wi-Fi/Bluetooth double cœur avec antenne PCB intégrée. |
| **Transceiver K-Line** | `U3` (L9637D013TR) | SOIC-8 | Émetteur-récepteur ISO 9141/KWP2000 avec protection thermique et court-circuit. |
| **Transceiver CAN** | `U2` (TJA1051T/3/1J) | SOIC-8 | Transceiver CAN haute vitesse avec broche d'adaptation de niveau `VIO` (3.3V). |
| **Régulateur Buck (12V $\rightarrow$ 5V)** | `U4` (TPS54331DR) | SOIC-8 | Convertisseur step-down synchrone 3.5V-28V, 3A. |
| **Régulateur LDO (5V $\rightarrow$ 3.3V)** | `U5` (LDL1117S33R) | SOT-223 | LDO à faible chute de tension et fort réjection de bruit (PSRR). |
| **Amortissement UART** | `R2`, `R3` ($10\,\Omega$) | 0603 | Résistances série d'amortissement sur les lignes `UART_RX` et `UART_TX` vers l'ESP32. |
| **Terminaison CAN** | `R11` ($120\,\Omega$) | 0603 | Résistance de terminaison de ligne du bus différentiel CAN. |
| **Protection ESD USB** | `U7`, `U8` (SD05C / BB05C) | SOD-323 | Diodes de protection ESD sur les lignes différentielles USB D+/D-. |
| **Connecteur USB-C** | `J2` (U263-161N-5BVZ15-2) | 16-pin SMD | Port USB-C pour alimentation de test, programmation et débogage. |

---

## 3. État Actuel de l'Implémentation

* **Schématique :** Schéma complet validé sous EasyEDA Pro (feuille `P1`).
* **Placement des composants (PCB) :** Placement 2D compact validé et sectorisé :
  * *Bloc Puissance (à gauche) :* Protections 12V, convertisseur buck `TPS54331` et régulateur `LDL1117`.
  * *Bloc Interfaces (au centre) :* Puces CAN `U2` et K-Line `U3`.
  * *Bloc Logique & Antenne (à droite) :* Module ESP32 avec son antenne orientée vers le bord extérieur libre.
* **Contour de carte (Board Outline) :** Défini et tracé sur la couche dédiée.

### Schéma

![Schéma ODB2 Scanner](./images/schematic.png)

### PCB

![PCB ODB2 Scanner](./images/pcb.png)
---

## 4. Feuille de Route & Checklist (TODO)

### 4.1 Routage des Pistes d'Alimentation
- [ ] **Rail +12V et Protection :**
  - [ ] Piste large ($0.8\text{ mm}$ à $1.0\text{ mm}$) reliant la broche 16 OBD $\rightarrow$ Fusible `F1` $\rightarrow$ Diode TVS `D1` $\rightarrow$ MOSFETs `Q1`/`Q2`.
  - [ ] Piste $12\text{V}$ vers la broche 7 (`VS`) de `U3` ($0.5\text{ mm}$).
- [ ] **Étage Buck 12V $\rightarrow$ 5V (`TPS54331DR` / `U4`) :**
  - [ ] Boucle de commutation courte et large : `U4` (broche 8 PH), inductance `L1` ($10\,\mu\text{H}$) et diode Schottky `D2` (`SS34`).
  - [ ] Condensateur d'entrée `C7` ($22\,\mu\text{F}$) au plus près de la broche 2 (`VIN`) de `U4`.
  - [ ] Condensateur de sortie `C8` ($22\,\mu\text{F}$) juste après `L1`.
  - [ ] Condensateur de bootstrap `C5` ($1\,\mu\text{F}$) entre broche 1 (`BOOT`) et broche 8 (`PH`).
- [ ] **Étage Régulation 3.3V (`LDL1117S33R` / `U5`) :**
  - [ ] Entrée `VIN` reliée au $5\text{V}$ (`L1` / `C8`).
  - [ ] Sortie `VOUT` vers perle de ferrite `FB1` et condensateur de filtrage `C6` ($1\,\mu\text{F}$).
  - [ ] Distribution du rail $3.3\text{V}$ vers le réseau de découplage `C1` à `C4` ($100\text{ nF}$) puis broche 2 de l'ESP32 `U1`.
  - [ ] Distribution $3.3\text{V}$ vers broche 5 (`VIO`) de `U2` et broche 3 (`VCC`) de `U3`.
- [ ] **Alimentation 5V :**
  - [ ] Rail $5\text{V}$ vers broche 3 (`VCC`) du transceiver CAN `U2`.

### 4.2 Routage des Signaux de Communication
- [ ] **Ligne K-Line (`U3` - `L9637D013TR`) :**
  - [ ] `UART_RX` : Broche 1 (`RX`) de `U3` $\rightarrow$ `R2` ($10\,\Omega$) $\rightarrow$ Broche 4 (`IO4`) de l'ESP32.
  - [ ] `UART_TX` : Broche 4 (`TX`) de `U3` $\rightarrow$ `R3` ($10\,\Omega$) $\rightarrow$ Broche 5 (`IO5`) de l'ESP32.
  - [ ] Ligne physique `K` : Broche 6 (`K`) de `U3` $\rightarrow$ Broche 7 de la prise OBD-II ($0.4\text{ mm}$ à $0.5\text{ mm}$).
- [ ] **Bus CAN (`U2` - `TJA1051T/3/1J`) :**
  - [ ] Lignes logiques : `TXD` (broche 1) $\rightarrow$ Broche 37 (`TXD0`) ESP32 ; `RXD` (broche 4) $\rightarrow$ Broche 36 (`RXD0`) ESP32.
  - [ ] Mode normal : Broche 8 (`S`) $\rightarrow$ Masse `GND`.
  - [ ] Paire différentielle CAN : `CANH` (broche 7) et `CANL` (broche 6) $\rightarrow$ Terminaison `R11` ($120\,\Omega$) $\rightarrow$ Broches 6 et 14 OBD-II *(pistes parallèles, symétriques et de même longueur)*.
- [ ] **Port USB-C & Programmation (`J2`) :**
  - [ ] Signaux `USB_D-` et `USB_D+` depuis `J2` via protections ESD `U8` / `U7` vers broches 13 (`IO19`) et 14 (`IO20`) de l'ESP32.
  - [ ] Résistances de configuration CC : Broches `CC1` et `CC2` de `J2` vers `R4` et `R5` ($5.1\,\text{k}\Omega$) $\rightarrow$ `GND`.
- [ ] **LED d'État (`LED1`) :**
  - [ ] Broche 38 (`IO2`) ESP32 $\rightarrow$ `R8` ($1.8\,\text{k}\Omega$) $\rightarrow$ Anode `LED1` $\rightarrow$ Cathode `GND`.

### 4.3 Plan de Masse & Gestion RF
- [ ] **Zone d'exclusion d'antenne (Keep-out Zone) :**
  - [ ] Définir une zone `Copper Keepout` sur **toutes les couches (All Layers)** sous et autour de l'antenne méandre de l'ESP32 (coin supérieur droit).
  - [ ] Garantir l'absence totale de cuivre (aucun plan de masse ni piste) pour préserver les performances radio.
- [ ] **Plans de masse (Copper Area) :**
  - [ ] Plan `GND` sur **Top Layer** (remplissage Solid, dégagement $0.254\text{ mm} - 0.3\text{ mm}$, Thermal Relief).
  - [ ] Plan `GND` sur **Bottom Layer** (remplissage Solid, dégagement $0.254\text{ mm} - 0.3\text{ mm}$, Thermal Relief).
- [ ] **Vias de couture (Stitching Vias) :**
  - [ ] Vias de masse sous le pad thermique central de l'ESP32 (broches 41).
  - [ ] Vias de masse au niveau des condensateurs de découplage et du bloc de découpage `U4`/`D2`.
  - [ ] Vias de masse réguliers le long du contour de carte.

### 4.4 Contrôles Finaux & Fabrication
- [ ] **Contrôle DRC (Design Rule Check) :** Exécuter la vérification des règles de conception sous EasyEDA Pro (`Shift + R`) et corriger les erreurs éventuelles.
- [ ] **Visualisation 3D :** Vérification visuelle globale de l'assemblage et du dégagement mécanique.
- [ ] **Export des fichiers de production :**
  - [ ] Fichiers Gerber & Perçage (Drill).
  - [ ] Fichier de nomenclature (BOM).
  - [ ] Fichier de placement des composants (CPL / Pick & Place).

---

## 5. Fichiers du Dépôt

* `ODB2-Scanner.eprj2` : Projet natif EasyEDA Pro v2 (contenant le schéma `P1` et le circuit imprimé `PCB1`).
* `README.md` : Documentation technique complète et suivi du projet.

---

## 6. Automatisation IA via Serveur MCP (EasyEDA Pro)

Pour permettre à un assistant IA (**Antigravity**, Claude Desktop, etc.) de manipuler directement le schéma et le PCB en temps réel (routage autonome des pistes, placement, création des zones de cuivre, etc.), le projet peut être piloté via le serveur MCP open-source [EasyEDA_MCP](https://github.com/Atmel2005/EasyEDA_MCP).

### 6.1 Principe de Fonctionnement
* **Extension EasyEDA Pro (`easyeda_plugin.eext`) :** Injecte un client WebSocket dans l'éditeur EasyEDA Pro qui se connecte sur `ws://127.0.0.1:8787`.
* **Serveur MCP Python (`easyeda_mcp.py`) :** Fournit les outils MCP standards (`stdio`) à l'IA et relaie les commandes vers EasyEDA Pro.
* **Outil Principal `execute_js` :** Permet à l'IA d'exécuter du JavaScript asynchrone interagissant avec plus de 670 commandes internes de l'API EasyEDA Pro (`eda.pcb_...`, `eda.sch_...`).

```
+---------------+              stdio             +--------------------+
|  Assistant IA | <============================> |  easyeda_mcp.py    |
| (Antigravity) |                                | (Serveur MCP stdio)|
+---------------+                                +--------------------+
                                                           ^
                                             WebSocket     | (Port 8787)
                                                           v
                                                 +--------------------+
                                                 | Extension EasyEDA  |
                                                 | (Éditeur Pro / CAD)|
                                                 +--------------------+
```

### 6.2 Prérequis & Installation

1. **Cloner ou télécharger le dépôt MCP :**
   ```bash
   git clone https://github.com/Atmel2005/EasyEDA_MCP.git
   ```

2. **Installer les dépendances Python (Python 3.10+) :**
   ```bash
   pip install mcp websockets
   ```

3. **Installer l'extension dans EasyEDA Pro :**
   * Ouvrir EasyEDA Pro (version Desktop ou Web : `https://pro.easyeda.com/editor`).
   * Aller dans le menu **Extensions** > **Gestionnaire d'extensions** (*Extension Manager*).
   * Cliquer sur **Charger une extension** (*Load Extension*) et sélectionner le fichier `easyeda_plugin.eext` (ou l'archive `easyeda_plugin.zip`).
   * Activer l'extension.

### 6.3 Configuration du Client IA (Antigravity / Claude)

Ajouter la configuration suivante dans le fichier de configuration MCP de votre client IA (par exemple `mcp_config.json` ou la configuration des serveurs MCP) :

```json
{
  "mcpServers": {
    "easyeda_pro": {
      "command": "python",
      "args": [
        "C:/chemin/vers/EasyEDA_MCP/easyeda_mcp.py"
      ]
    }
  }
}
```

### 6.4 Utilisation Interactive
1. Démarrer le client IA avec le serveur MCP configuré.
2. Ouvrir le projet `ODB2-Scanner.eprj2` dans EasyEDA Pro. Le plugin affiche un message de connexion confirmée (`[INFO] ATM_MCP WebSocket connected`).
3. Demander à l'assistant d'exécuter des actions précises :
   * *"Récupère la liste des composants du projet via `get_components`."*
   * *"Trace automatiquement les pistes UART entre U3, R2/R3 et l'ESP32."*
   * *"Déploie un plan de masse GND Top et Bottom en évitant l'antenne méandre de l'ESP32."*
