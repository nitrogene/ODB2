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


