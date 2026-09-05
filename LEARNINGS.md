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
