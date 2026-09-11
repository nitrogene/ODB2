# Capitalisation Technique & Découvertes API EasyEDA Pro

Ce document consigne de manière datée les comportements, astuces, contournements et spécificités de l'API EasyEDA Pro non documentés (ou insuffisamment détaillés) dans le skill officiel `easyeda-api`.

---

## [2026-09-03] Extraction de la nomenclature (BOM) et des empreintes depuis le PCB

* **Subtilité de lecture des valeurs :** `getState_Name()` ou `getState_Designator()` ne renvoient que le préfixe/désignateur du composant (`R1`, `C1`). La valeur réelle du composant (`10k`, `100nF`) ainsi que le part number LCSC se trouvent dans le dictionnaire `getState_OtherProperty()?.Value` ou `props.Device`.
* **Récupération de l'empreinte :** L'objet empreinte est accessible via `c.getState_Footprint()`. Pour résoudre le nom lisible de l'empreinte, il faut appeler `eda.lib_Footprint.get(fpInfo.uuid, fpInfo.libraryUuid)`.

```javascript
const ids = await eda.pcb_PrimitiveComponent.getAllPrimitiveId();
const components = await eda.pcb_PrimitiveComponent.get(ids);

const results = [];
for (const c of components) {
  const fpInfo = c.getState_Footprint?.() || c.footprint;
  let fpName = '';
  if (fpInfo?.uuid && fpInfo?.libraryUuid) {
    const fp = await eda.lib_Footprint.get(fpInfo.uuid, fpInfo.libraryUuid);
    fpName = fp?.name || fp || '';
  }

  const props = c.getState_OtherProperty?.() || {};
  results.push({
    designator: c.getState_Designator?.() || c.designator,
    value: props.Value || props.Device || '',
    footprint: fpName
  });
}
return results;
```

---

## [2026-09-03] Géométrie des pastilles (Pads) et unités de mesure

* **Unités runtime :** Les coordonnées de l'API PCB (`center.x`, `center.y`, `width`, `height`) sont exprimées en **mils** (1 mil = 0.0254 mm).
* **Lien composant - pastille :** L'identifiant `primitiveId` d'une pastille commence systématiquement par le `primitiveId` du composant parent (`pad.primitiveId.startsWith(comp.primitiveId)`).
* **Dimensions :** `pad.pad[1]` correspond à la largeur et `pad.pad[2]` à la hauteur en mils. `pad.pad[0]` indique la forme (`"RECT"`, `"OVAL"`, etc.).

```javascript
const targetNets = ['+12V', '+12V_PROT'];
const compIds = await eda.pcb_PrimitiveComponent.getAllPrimitiveId();
const comps = await eda.pcb_PrimitiveComponent.get(compIds);
const compMap = new Map();
for (const c of comps) {
  compMap.set(c.getState_PrimitiveId(), c.getState_Designator());
}

const padGeometry = [];
for (const net of targetNets) {
  const prims = await eda.pcb_Net.getAllPrimitivesByNet(net);
  for (const p of prims) {
    padGeometry.push({
      designator: compMap.get(p.parentId) || p.parentId,
      padNumber: p.num,
      net: p.net,
      x: Math.round(p.center.x * 10) / 10,
      y: Math.round(p.center.y * 10) / 10,
      width_mil: Math.round((p.topWidth || 0) * 10 * 10) / 10,
      height_mil: Math.round((p.topHeight || 0) * 10 * 10) / 10,
      shape: p.topType
    });
  }
}
return padGeometry;
```

---

## [2026-09-03] Identifiants numériques des couches (Layers) & Tracé de lignes

* **Identifiants runtime :** Dans le contexte navigateur EasyEDA Pro, les couches sont référencées par des entiers :
  * `1` = `Top Layer` (cuivre supérieur)
  * `2` = `Bottom Layer` (cuivre inférieur)
  * `11` = `Board Outline` (contour de carte)
* **API de tracé :** `eda.pcb_PrimitiveLine.create(net, layer, startX, startY, endX, endY, lineWidth, primitiveLock)`
* **API de via :** `eda.pcb_PrimitiveVia.create(net, x, y, holeDiameter, diameter)` (standard JLCPCB : perçage 12 mil ≈ 0.3 mm, diamètre 24 mil ≈ 0.6 mm).

---

## [2026-09-03] Capture haute fidélité du canvas PCB / Schéma (Blob vers Base64)

* `eda.pcb_Document.zoomToBoardOutline()` permet de cadrer parfaitement la vue sur le contour de carte avant export.
* `eda.dmt_EditorControl.getCurrentRenderedAreaImage(tabId)` renvoie un objet `Blob` inaccessible directement depuis l'extérieur du navigateur.
* **Astuce :** Utiliser un `FileReader` dans le code exécuté dans le navigateur pour convertir le `Blob` en chaîne `Base64`, puis l'écrire sur le disque côté système hôte (PowerShell / Node.js) :

```powershell
$code = @"
const doc = await eda.dmt_SelectControl.getCurrentDocumentInfo();
await eda.pcb_Document.zoomToBoardOutline();
await new Promise(r => setTimeout(r, 400));

const blob = await eda.dmt_EditorControl.getCurrentRenderedAreaImage(doc.tabId);
if (!blob) return { error: 'No blob' };

return new Promise((resolve) => {
  const reader = new FileReader();
  reader.onloadend = () => {
    const base64 = reader.result.split(',')[1];
    resolve({ success: true, base64: base64 });
  };
  reader.readAsDataURL(blob);
});
"@

$body = @{ code = $code } | ConvertTo-Json
$res = Invoke-RestMethod -Uri "http://localhost:49620/execute" -Method Post -Body $body -ContentType "application/json"

if ($res.result.success -and $res.result.base64) {
  $bytes = [Convert]::FromBase64String($res.result.base64)
  [IO.File]::WriteAllBytes("images/PCB.png", $bytes)
}
```

---

## [2026-09-05] Schématique : Raccordement impératif des drapeaux de réseau (`NetFlag`) par des fils (`Wire`)

* **Comportement découvert :** Si un drapeau de réseau (`netflag`, ex. `power-5v` ou `ground-gnd`) est positionné aux coordonnées exactes d'une broche de composant sans segment de fil physique (`eda.sch_PrimitiveWire`), le compilateur de schéma d'EasyEDA considère la broche comme **flottante** (`[Warn] : Found some components Pins floating, suggest placing No Connect Flag...`).
* **Impact critique :** Lors de l'import Schéma → PCB (`Design > Update PCB`), la pastille PCB associée ne reçoit aucun net ou reçoit un net temporaire découplé (ex. `$1N14`), risquant de laisser des diviseurs de tension de contre-réaction (comme `VSENSE` sur un Buck) complètement ouverts.
* **Règle à appliquer :** Toujours insérer au minimum un segment de fil (`eda.sch_PrimitiveWire.create([x1, y1, x2, y2], net)`) reliant explicitement la broche du composant au point de connexion du drapeau.

---

## [2026-09-05] Portée des namespaces API & Document actif

* `eda.sch_*` échoue (`获取所有器件的图元ID失败`) si le document actif affiché dans EasyEDA n'est pas une feuille de schéma (`documentType !== 1`).
* `eda.pcb_*` échoue si le document actif n'est pas un circuit imprimé (`documentType !== 3`).
* **Bonne pratique :** Avant d'exécuter des requêtes sur un domaine, basculer activement le document via :
  ```javascript
  await eda.dmt_EditorControl.openDocument(targetDocumentUuid);
  await new Promise(r => setTimeout(r, 600));
  ```

---

## [2026-09-05] Comportement de `eda.pcb_Document.importChanges()`

* L'appel programmatique `await eda.pcb_Document.importChanges(schUuid)` renvoie `false` lorsque des boîtes de dialogue interactives de confirmation de changements (liste des composants ajoutés/supprimés et nets modifiés) sont requises par la version desktop d'EasyEDA Pro.
* Pour une synchronisation fiable de nouveaux composants, privilégier l'action utilisateur via **Conception > Mettre à jour le PCB** (*Design > Update PCB*).

---

## [2026-09-05] Exportation automatisée du projet EasyEDA (`.epro2`)

* **API :** `eda.sys_FileManager.getProjectFile(fileName, password, fileType)`
* **Fonctionnement :** Renvoie un objet standard Web `File` (Blob). Pour sauvegarder le fichier sur le disque de la machine hôte via le pont WebSocket/HTTP :
  ```powershell
  $code = @"
  const projectFile = await eda.sys_FileManager.getProjectFile('ProPrj_ODB2-Scanner', undefined, 'epro2');
  if (!projectFile) return { error: 'Failed to get project file' };

  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onloadend = () => {
      const base64 = reader.result.split(',')[1];
      resolve({ success: true, name: projectFile.name, size: projectFile.size, base64: base64 });
    };
    reader.readAsDataURL(projectFile);
  });
  "@

  $body = @{ code = $code } | ConvertTo-Json
  $res = Invoke-RestMethod -Uri "http://localhost:49620/execute" -Method Post -Body $body -ContentType "application/json"

  if ($res.result.success -and $res.result.base64) {
    $bytes = [Convert]::FromBase64String($res.result.base64)
    [IO.File]::WriteAllBytes("easyeda/ProPrj_ODB2-Scanner.epro2", $bytes)
  }
  ```

---

## [2026-09-05] Exportation haute résolution du schéma (`PNG`)

* **API :** `eda.dmt_EditorControl.zoomToAllPrimitives()` puis `eda.dmt_EditorControl.getCurrentRenderedAreaImage(doc.tabId)`
* **Fonctionnement :**
  ```javascript
  await eda.dmt_EditorControl.openDocument(schPageUuid);
  await new Promise(r => setTimeout(r, 600));
  const doc = await eda.dmt_SelectControl.getCurrentDocumentInfo();
  await eda.dmt_EditorControl.zoomToAllPrimitives();
  await new Promise(r => setTimeout(r, 400));
  const pngBlob = await eda.dmt_EditorControl.getCurrentRenderedAreaImage(doc.tabId);
  // Lecture du blob via FileReader (base64) pour écriture sur le disque
  ```


---

## [2026-09-05] Manipulation 2D des composants PCB (Position & Rotation)

* **API directe :** `eda.pcb_PrimitiveComponent.modify(primitiveId, { x, y, rotation })`
  * *Note :* Privilégier cette méthode à l'approche `c.toAsync()`, car `get([id])` retourne un tableau, ce qui peut causer des erreurs de type si non déstructuré.
* **Repère d'orientation des pastilles (composants passifs à 2 broches ex. 0805, 0603, SMA) :**
  * `0°` : Horizontal standard — Pastille 1 à gauche, Pastille 2 à droite.
  * `90°` : Vertical — Pastille 1 en bas, Pastille 2 en haut.
  * `180°` : Horizontal inversé — Pastille 1 à droite, Pastille 2 à gauche.
  * `270°` : Vertical inversé — Pastille 1 en haut, Pastille 2 en bas.

---

## [2026-09-05] Contrôle DRC programmatique

* **API :** `await eda.pcb_Drc.check()`
* **Comportement :** Renvoie un booléen immédiat :
  * `true` : Aucun conflit DRC détecté (règles de dégagement, chevauchements et continuité respectées).
  * `false` : Présence de violations DRC (ex. discordance de nom de net entre pastille et piste, ou distance d'isolement insuffisante).

---

## [2026-09-06] Diagnostic approfondi des violations DRC (`includeVerboseError`)

* **API :** `await eda.pcb_Drc.check(true, false, true)`
* **Fonctionnement :** Le troisième paramètre `includeVerboseError = true` retourne la liste exhaustive des violations catégorisées (Clearance Error, Connection Error, Netlist Error) avec tous les détails géométriques :
  * Types et identifiants des objets incriminés (`obj1`, `obj2`, `objs`).
  * Coordonnées précises du conflit (`pos: { x, y }`).
  * Distances mesurées vs règles d'isolement requises (`minDistance`, `shouldBe`).
* **Utilité :** Permet à l'agent d'identifier et corriger précisément au mil près les violations d'isolement (ex. distance piste-pastille) sans intervention humaine.

---

## [2026-09-06] Réaffectation directe du réseau d'une pastille (`Pad Net`)

* **API directe :** `await eda.pcb_PrimitivePad.modify(primitiveId, { net: 'NOUVEAU_NET' })`
* **Contexte :** Permet de réconcilier ou forcer l'attribution de réseau sur une pastille PCB sans devoir refaire un import complet du schéma lorsque des incohérences mineures de synchronisation bloquent le DRC.

---

## [2026-09-06] Suppression ciblée de pistes (`PrimitiveLine`)

* **API :** `await eda.pcb_PrimitiveLine.delete([primitiveId1, primitiveId2, ...])`
* **Fonctionnement :** Supprime immédiatement les segments de ligne spécifiés par leur identifiant primitif (`primitiveId`), facilitant les ajustements de tracé et le ré-routage propre.

---

## [2026-09-06] Annotation textuelle du schéma (`sch_PrimitiveText`) & Système de coordonnées

* **API :** `eda.sch_PrimitiveText.create(x, y, text, angle, font, fontSize, color, bold, italic)`
  * Exemple : `await eda.sch_PrimitiveText.create(50, 480, "ALIMENTATION", 0, undefined, 9, "#003388", true, false)`
* **Repère et orientation du schéma (`documentType: 1`) :**
  * Format standard A4 : Largeur 1170, Hauteur 825.
  * **L'axe Y est orienté vers le HAUT** (`y = 0` en bas de page, `y = 825` en haut de page).
  * Attention : inversé par rapport au PCB où l'origine et l'orientation peuvent varier selon le cadrage.
* **Usage :** Permet d'insérer des titres et des annotations explicatives directement au-dessus des blocs de composants sans risque de court-circuit électrique (éléments purement graphiques).

---

## [2026-09-06] Robustesse des scripts d'exportation (Node.js vs PowerShell)

* Lors du transfert de gros volumes de données Base64 (ex. captures PNG haute résolution `getCurrentRenderedAreaImage`), les variables PowerShell `$base64` peuvent être interpolées et vidées si incluses par inadvertance dans des blocs `@"..."@`.
* L'exécution via un script Node.js exécuté via `node -e` ou un fichier scratch (`fetch('http://localhost:49620/execute')` et `Buffer.from(base64, 'base64')`) est plus robuste, gère directement les promesses JavaScript et évite tout conflit d'échappement shell.

---

## [2026-09-06] Rafraîchissement du rendu visuel des textes de pistes sur le canvas PCB (Cache WebGL)

* **Problème découvert :** Modifier la propriété `net` d'une piste via l'API (ou recréer la piste) met bien à jour la base de données interne du PCB (le panneau Propriétés / Détails de l'objet sélectionné affiche le nouveau nom), mais le texte imprimé le long de la piste dans le canvas WebGL continue d'afficher l'ancien nom de net (ex. `$1N15` au lieu de `+12V_PROT`).
* **Cause technique :** Le moteur graphique d'EasyEDA Pro met en cache les textures de rendu des textes vectoriels sur les pistes tant que l'onglet du document reste ouvert.
* **Solution programmatique éprouvée :**
  ```javascript
  // 1. Sauvegarder les modifications dans la base du document
  await eda.pcb_Document.save();
  await new Promise(r => setTimeout(r, 400));

  // 2. Fermer l'onglet actif
  const doc = await eda.dmt_SelectControl.getCurrentDocumentInfo();
  await eda.dmt_EditorControl.closeDocument(doc.tabId);
  await new Promise(r => setTimeout(r, 600));

  // 3. Réouvrir le document
  await eda.dmt_EditorControl.openDocument(pcbUuid);
  await new Promise(r => setTimeout(r, 1000));
  ```
* **Résultat :** La fermeture/réouverture force la destruction et la reconstruction complète du contexte WebGL et le rechargement propre des données depuis le stockage, répercutant immédiatement les nouveaux noms de nets sur tout le canvas.

---

## [2026-09-06] Schématique : Fusion automatique des fils (`Wire Merging`) et assignation de net

* **Fusion automatique de polylignes :** Dans le schéma EasyEDA Pro, dès que deux segments de fil (`sch_PrimitiveWire`) se touchent ou s'intersectent sur la grille, le compilateur les fusionne automatiquement en une seule entité polyline. Il faut veiller à ne pas faire transiter un fil d'alimentation brute (`+12V_FUSED`) à proximité immédiate d'un fil de sortie protégée (`+12V_PROT`) sous peine de court-circuiter le composant série (transistor de protection `Q1`).
* **API de modification de fil :** `await eda.sch_PrimitiveWire.modify(wireId, { net: 'NET_NAME' })` permet d'attribuer directement le nom de net électrique à une liaison filaire.
* **Fonctions stubs de `sch_PrimitiveAttribute` :** Dans la version actuelle de l'API embarquée, `eda.sch_PrimitiveAttribute.createNetLabel()` et `create()` sont des stubs vides (`async createNetLabel(t,i,n){}`). Pour modifier un label existant, utiliser `eda.sch_PrimitiveAttribute.modify(attrId, { value: 'NET_NAME' })`.

---

## [2026-09-06] Routage multicouche et topologie d'accès aux boîtiers SOIC-8

* **Franchissement de bus d'alimentation multicouche :** Lorsqu'un bus d'alimentation vertical est distribué sur la face inférieure (`Bottom Layer`, ex. `3.3V` sur l'axe `x = 3050`), toute liaison sécante (ex. distribution `+5V`) doit impérativement traverser ce corridor sur la face supérieure (`Top Layer`) avant d'effectuer une transition par via vers la couche inférieure, évitant ainsi tout conflit de dégagement ("Track to Track" / "Safe Spacing").
* **Topologie d'approche des boîtiers SOIC-8 (Transceivers CAN / K-Line) :**
  * Les broches SMD intermédiaires d'un boîtier SOIC-8 (pas standard 50 mil / 1.27 mm) ne peuvent pas être abordées horizontalement sur la même face sous peine d'intercepter les pastilles voisines.
  * **Règle géométrique d'accès :** L'accès doit s'effectuer soit verticalement (droit depuis le haut ou le bas sur la couche composant), soit en approchant par la couche opposée (`Bottom Layer`) jusqu'à un via situé dans l'alignement de la pastille cible à une distance de sécurité (> 25 mil du bord de pastille), avant de remonter directement en ligne droite.

---

## [2026-09-07] Schématique : Création programmatique de ports de réseau (`NetPort`)

* **Contournement des stubs `createNetLabel` :** L'API `eda.sch_PrimitiveAttribute.createNetLabel()` étant un stub vide dans le runtime EasyEDA Pro actuel, la création de ports de réseau via `eda.sch_PrimitiveComponent.createNetPort(direction, net, x, y, rotation, mirror)` est pleinement fonctionnelle et opérationnelle.
* **Connexion au schéma :** Le port de réseau dispose d'une broche électrique interne (au point d'insertion selon la rotation) qui connecte immédiatement tout fil (`Wire`) superposé ou adjacent et propage le net dans le compilateur de netlist EasyEDA sans nécessiter d'intervention manuelle.

---

## [2026-09-07] PCB : Invalidation de l'arbre de connectivité cuivre lors des modifications in-place (`modify` vs `create`)

* **Problème découvert :** Modifier directement l'attribut `net` de pistes ou vias existants via `eda.pcb_PrimitiveLine.modify(id, { net })` ou `eda.pcb_PrimitiveVia.modify(id, { net })` met à jour la propriété dans la base de données, mais le moteur de calcul DRC ne reconstruit pas toujours le graphe de continuité cuivre. Il en résulte de fausses alertes d'isolement DRC ("Track to Via distance is 0mm, should be >= 0.176mm") entre des éléments portant pourtant le même nom de net.
* **Solution robuste :** Pour réassigner un tracé existant vers un nouveau net, supprimer les primitives incriminées (`pcb_PrimitiveLine.delete()`, `pcb_PrimitiveVia.delete()`) et les recréer avec `create(net, ...)` garantit leur insertion immédiate dans l'arbre spatial du nouveau réseau, assurant un DRC à 0 erreur d'isolement.

---

## [2026-09-07] Exportation d'images haute résolution contrôlée via le contexte Canvas 2D

* **Spécificités runtime :** `eda.sch_ManufactureData.getPngFile()` n'est pas exposé dans tous les environnements desktop et `getExportDocumentFile()` requiert une validation modale bloquante.
* **Méthode d'export exacte :** Utiliser `eda.dmt_EditorControl.getCurrentRenderedAreaImage(doc.tabId)` pour obtenir le flux graphique brut rendu, puis projeter l'image dans un élément `<canvas>` dimensionné aux résolutions requises (ex. `2274 × 1236 px` pour le schéma, `1137 × 642 px` pour le PCB) avec `imageSmoothingQuality = 'high'` avant l'encodage PNG Base64. Cela garantit un respect strict et reproductible des dimensions sans dépendre de la taille de fenêtre du client.

---

## [2026-09-08] Contour de carte (`pcb_PrimitivePolyline`) et règles de dégagement

* **Lecture de la géométrie du contour :** Le contour de carte (Board Outline) est stocké sous la forme d'un objet `eda.pcb_PrimitivePolyline` sur la couche 11 (`EPCB_LayerId.BOARD_OUTLINE`).
* Sa géométrie est accessible via la méthode `poly.getState_Polygon()`, qui renvoie un descripteur géométrique standard, par exemple `["R", x, y, width, height, 0, 0]` pour un contour rectangulaire :
  * Dans ce projet : `["R", 1850, 50, 2600, 1250, 0, 0]`, ce qui correspond à une emprise physique de x dans [1850, 4450] (largeur 2600 mil = 66.04 mm) et y dans [-1200, 50] (hauteur 1250 mil = 31.75 mm).
* **Règle DRC de bordure :** La règle `Board Outline to Track` impose une distance d'isolement stricte de **0.300 mm (11.8 mil)**. Tout tracé ou via doit impérativement respecter cette marge par rapport aux 4 arêtes du contour.

---

## [2026-09-08] Connectivité pastille SMD vs Via traversant (Via-in-Pad et raccordement)

* **Comportement découvert :** Placer un via traversant (`pcb_PrimitiveVia.create()`) exactement aux coordonnées centrales d'une pastille CMS / SMD (`pcb_PrimitivePad`) de même nom de réseau peut ne pas être reconnu comme connecté par le vérificateur DRC si aucun segment de piste (`pcb_PrimitiveLine`) sur la couche de la pastille (couche 1 pour le Top) n'est rattaché physiquement au centre du via.
* **Solution robuste :** Décaler légèrement le via à l'extérieur de la pastille SMD (ex. à 25-30 mil de distance) et tracer explicitement un court segment de piste sur la couche du composant reliant le via à la pastille. Ce tracé assure une topologie claire dans le graphe de connectivité cuivre et élimine toute fausse détection de discontinuité.

---

## [2026-09-09] Routage USB-C : Paires différentielles, diodes ESD et ponts multicouches (Bridges)

* **Orientation des diodes TVS ESD (`U7`, `U8`) au plus près de `J2` :**
  * Positionner les diodes à 270° au plus près du connecteur USB-C (`y = -200`) place la pastille 1 (Signal) en haut (`y = -152.6 mil`) et la pastille 2 (`GND`) en bas (`y = -247.4 mil`).
  * Cela permet aux pistes `USB_D+` et `USB_D-` d'accéder directement aux broches de protection sans créer de boucle inductive ("stub"), puis de filer vers le corridor intérieur libre de l'ESP32 (`x = [3790, 3810]`) sur le Top Layer jusqu'aux broches IO20 et IO19 avec 0 erreur DRC.
* **Franchissement multicouche par pont (Bridge) pour les lignes de configuration CC :**
  * Lorsque deux signaux transversaux sur la couche inférieure (`USB_CC1` et `USB_CC2`) doivent croiser un bus d'alimentation transversal (`3.3V` à `y = -300`) ou une piste sécante (`USB_CC2` à `x = 3230`), l'insertion d'un pont de 30-40 mil sur la couche opposée (Top Layer) via 2 vias (perçage 12 mil, diamètre 20 mil) permet d'éliminer tout conflit d'isolement sans devoir re-router les bus d'alimentation.
* **Autoroute de contournement nord du connecteur USB-C :**
  * Entre les pastilles de blindage métallisées de `J2` (`y = 6.3`) et le bord supérieur de carte (`y = 50 mil`), la marge physique est étroite (dégagement au contour >= 11.8 mil, dégagement à la pastille >= 6.0 mil). Une piste de 5 mil de large centrée à `y = 34.5 mil` satisfait rigoureusement les deux contraintes avec une marge de sécurité (> 12.5 mil vers le bord, > 6.0 mil vers la pastille).

---

## [2026-09-09] Limite de résolution de la capture d'écran API (`getCurrentRenderedAreaImage`) vs Export UI natif

* **Problème identifié :** L'API `eda.dmt_EditorControl.getCurrentRenderedAreaImage()` capture uniquement le viewport du navigateur (environ 1631 × 618 px). Pour une feuille de schéma complète (format A4), les textes des composants ne mesurent que 3 à 4 pixels. Ré-étirer ce canvas en 2274 × 1236 px ne fait qu'agrandir les pixels sans ajouter de détail vectoriel, rendant le schéma illisible au zoom.
* **Solution retenue :** L'export d'images haute définition pour la documentation (`images/Schematic.png` et `images/PCB.png`) est confié à l'utilisateur via le menu natif de l'interface EasyEDA Pro (**Fichier > Exporter > Image / PDF** à 300 DPI ou largeur 4096 px), garantissant une netteté vectorielle irréprochable.

---

## [2026-09-11] Création d'une zone d'exclusion / Keepout multicouche (`pcb_PrimitiveRegion`)

* **API :** `eda.pcb_PrimitiveRegion.create(layer, complexPolygon, ruleType, regionName, lineWidth, primitiveLock)`
* **Couche multicouche (`EPCB_LayerId.MULTI`) :** Pour appliquer une zone d'exclusion sur toutes les couches (Top, Bottom, couches internes), utiliser l'identifiant de couche `12` (`EPCB_LayerId.MULTI`).
* **Géométrie rectangulaire :** Le polygone rectangulaire est instancié par `eda.pcb_MathPolygon.createPolygon(['R', x, y, width, height, rotation, cornerRadius])` où :
  * `x` est l'abscisse gauche (min X)
  * `y` est l'ordonnée supérieure (max Y)
  * `width` est la largeur (ΔX)
  * `height` est la hauteur descendante (ΔY)
* **Combinaison des règles d'exclusion (`ruleType`) :**
  * `EPCB_PrimitiveRegionRuleType.NO_WIRES` (`5`) : Interdiction formelle de passage de pistes de cuivre.
  * `EPCB_PrimitiveRegionRuleType.NO_FILLS` (`6`) : Interdiction des remplissages de cuivre (`Solid Fill`).
  * `EPCB_PrimitiveRegionRuleType.NO_POURS` (`7`) : Interdiction d'incursion des plans de masse ou de puissance (`Copper Pour`).
  * Spécifier `ruleType: [5, 6, 7]` garantit l'absence totale de tout conducteur sous l'antenne radio (2.4 GHz).

---

## [2026-09-11] Plans de masse (`pcb_PrimitivePour`), régénération et vias de couture (Stitching Vias)

* **Création des plans de masse :**
  * `eda.pcb_PrimitivePour.create(net, layer, complexPolygon, pourFillMethod, preserveSilos, pourName, pourPriority, lineWidth, primitiveLock)`
  * Pour couvrir l'ensemble du contour de carte, utiliser le polygone rectangulaire :
    `eda.pcb_MathPolygon.createPolygon(['R', 1850, 50, 2600, 1250, 0, 0])`
  * Couche Top (`1`), Couche Bottom (`2`), méthode `'solid'` (`EPCB_PrimitivePourFillMethod.SOLID`), `preserveSilos: false` pour supprimer les îlots de cuivre isolés.
* **Calcul et régénération du remplissage cuivre :**
  * Après création ou déplacement d'éléments, appeler impérativement `await pour.rebuildCopperRegion()` pour recalculer les zones de remplissage (`IPCB_PrimitivePoured`) et les freins thermiques (*thermal relief*).
* **Résorption des îlots isolés via pistes d'amorce et vias de couture :**
  * Lorsque `preserveSilos` est à `false`, les pastilles situées dans des zones étranglées par des pistes de signaux perdent leur frein thermique si le cuivre ne peut pas s'y infiltrer en respectant le dégagement (clearance).
  * L'insertion d'un via de couture (perçage 12 mil, diamètre 24 mil) raccordé par une courte piste de 10-12 mil sur la couche de la pastille (avec coordonnées exactes `pad.getState_X()`, `pad.getState_Y()`) permet de basculer immédiatement la continuité vers le plan de masse opposé sans aucun conflit DRC.
* **Matrice thermique pour boîtier QFN/LGA (ESP32 pad 41) :**
  * Pour les modules dotés d'un pad thermique composite divisé en sous-pastilles (ex. 9 pads 3×3 sous l'ESP32-S3), disposer une matrice de vias de masse 12/24 mil interconnectée par un quadrillage de pistes de cuivre sur les deux couches garantit une dissipation thermique optimale vers le plan inférieur et une continuité électrique parfaite (0 erreur DRC).
