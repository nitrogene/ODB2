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

![Schéma ODB2 Scanner](./images/Schematic.png)

### PCB

![PCB ODB2 Scanner](./images/PCB.png)
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

### 6.4 Connexion depuis le client IA

Aucune configuration `mcp_config.json` n'est nécessaire. Les outils compatibles **Agent Skills** (Claude Code, OpenCode, QwenCode...) lisent automatiquement `SKILL.md` à la racine du dépôt cloné et disposent alors des instructions et de la documentation API.

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

### 6.6 Exemples de requêtes pour ce projet

* *"Liste les pads de U3 (L9637D013TR) et de l'ESP32 (U1), puis trace la piste UART_RX entre la broche 1 de U3 et la broche 4 de U1 en passant par R2."*
* *"Trace la paire différentielle CAN (CANH/CANL) entre U2 et le connecteur OBD-II en pistes parallèles de même longueur."*
* *"Crée le plan de masse GND sur Top et Bottom Layer avec un dégagement de 0.254 mm, en respectant la zone d'exclusion sous l'antenne de l'ESP32."*
* *"Lance une vérification DRC et liste les erreurs restantes."*

### 6.7 Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| `Port 49620-49629 already in use` | Une autre instance du serveur tourne déjà | Fermer l'instance existante avant de relancer |
| `EasyEDA is not connected` / outil indisponible | L'extension `.eext` n'est pas active dans l'onglet EasyEDA Pro | Vérifier que le projet est ouvert avec l'extension chargée et activée |
| Timeout sur une requête `/execute` | Onglet EasyEDA Pro inactif, ou opération trop longue | Garder l'onglet actif ; relancer la requête |

> Si tu retombes sur une erreur du type `AttributeError: 'Server' object has no attribute 'list_tools'` ou `JSON-RPC error -32022: the initialize handshake is not accepted`, c'est un résidu de l'ancienne approche par serveur MCP custom (§ historique 6) — ces erreurs viennent d'incompatibilités de version du SDK Python `mcp` et ne concernent pas le skill officiel décrit ici, qui ne dépend pas de `mcp` du tout.

### 6.8 Bonnes pratiques pour un routage PCB piloté par IA

En cohérence avec la checklist de routage de la Section 4 de ce document :

- **Ne pas s'appuyer sur l'auto-routeur intégré d'EasyEDA pour un résultat final** : la documentation officielle d'EasyEDA le déconseille elle-même *("Auto router is not good enough! Suggest routing manually!")*. Un agent IA doit raisonner piste par piste, pas déclencher l'auto-routeur en aveugle.
- **Toujours relire les positions réelles des pads avant de router** (`pcb_PrimitivePad.get(...)`) plutôt que de faire router l'IA sur des coordonnées supposées — c'est la méthode qui a fait ses preuves dans les retours d'expérience communautaires sur ce skill.
- **Vérifier le DRC après chaque lot de pistes tracées** (`pcb_Drc`), pas seulement à la fin du projet, pour détecter les courts-circuits ou chevauchements au plus tôt.
- **Sauvegarder ou versionner le fichier `.eprj2`** avant toute session de routage automatisé en masse — un script IA mal formulé peut modifier plusieurs pistes en une seule commande `execute`.
- **Router en dernier les rails de puissance** (12V, 5V, 3.3V — voir §4.1) avec des largeurs de piste explicitement spécifiées à l'agent, les erreurs de largeur de piste sur ces rails étant plus difficiles à repérer visuellement qu'un DRC de court-circuit.
