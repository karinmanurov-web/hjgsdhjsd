# Stage 5 patch — composition centroid for balance

Status: **applied to the main HTML; verified by reading the committed file back from GitHub**.
Target branch: `67pocoyo`
Target file: `deepseek_html_20261008_fbe883.html`
Initial base blob SHA: `ed83d53303eafe06d5542781edf41d00d5a1a682`
Latest implementation commit: `8eb99820ef6d7d202b154835fb306b562fdda2f8`
Latest HTML blob SHA: `a37c4f6a0af9419a4e4d13969ced1cf9ff464089`

## Problem

In `Evaluator.evaluate()`, both rule-of-thirds and balance use only the main object's bounding-box center:

```js
const mainBBox = computeMainObjBBox(frame, cam);
const centroid = mainBBox ? { u: mainBBox.u, v: mainBBox.v } : { u:0.5, v:0.5 };
```

That makes the balance score ignore other visible objects. This patch introduces a screen-space composition centroid weighted by the visible projected area of each object, with a modest boost for the designated main object. The existing main-object bbox remains the input for focus scoring.

## Patch A — add these helpers inside the Evaluator IIFE, after computeMainObjBBox()

```js
function computeObjectScreenBBox(obj, frame, cam, root){
  const ov = frame.overrides?.[obj.id] || {};
  if((ov.visible !== undefined ? ov.visible : obj.visible) === false) return null;
  if(obj.role === 'guide') return null;

  const tr = { ...(obj.transform || {}), ...(ov.transform || {}) };
  const pos = tr.pos || [0,0,0];
  const size = AnalyzerLite.guessObjectSize(obj);
  if(!size || ![size.x,size.y,size.z].every(Number.isFinite)) return null;

  const center = new THREE.Vector3(pos[0],pos[1],pos[2]);
  const points = [];
  for(const [sx,sy,sz] of [
    [-1,-1,-1],[1,-1,-1],[-1,1,-1],[1,1,-1],
    [-1,-1,1],[1,-1,1],[-1,1,1],[1,1,1]
  ]){
    const p = new THREE.Vector3(
      center.x + sx*size.x/2,
      center.y + sy*size.y/2,
      center.z + sz*size.z/2
    ).applyMatrix4(root.matrixWorld).project(cam);
    // Ignore corners behind the camera or beyond the far clipping plane.
    if(!Number.isFinite(p.x) || !Number.isFinite(p.y) || !Number.isFinite(p.z) || p.z < -1 || p.z > 1) continue;
    points.push({u:(p.x+1)/2, v:(1-p.y)/2});
  }
  if(!points.length) return null;

  const u0 = Math.max(0, Math.min(...points.map(p=>p.u)));
  const u1 = Math.min(1, Math.max(...points.map(p=>p.u)));
  const v0 = Math.max(0, Math.min(...points.map(p=>p.v)));
  const v1 = Math.min(1, Math.max(...points.map(p=>p.v)));
  if(u1 <= u0 || v1 <= v0) return null;
  return {u:(u0+u1)/2, v:(v0+v1)/2, area:(u1-u0)*(v1-v0)};
}

function computeCompositionCentroid(frame, cam, mainBBox){
  const root = SceneManager.getRenderInfo().rootGroup;
  root.updateMatrixWorld(true);
  cam.updateMatrixWorld(true);

  const objects = State.get().objects || [];
  let sumW = 0, sumU = 0, sumV = 0;
  for(const obj of objects){
    const box = computeObjectScreenBBox(obj, frame, cam, root);
    if(!box) continue;
    // Area weighting captures the visual footprint without letting one huge
    // object completely dominate; the main subject receives a restrained boost.
    let weight = Math.sqrt(Math.max(0, box.area));
    if(mainBBox && obj.id === mainBBox.objId) weight *= 1.35;
    if(!Number.isFinite(weight) || weight <= 0) continue;
    sumW += weight;
    sumU += box.u * weight;
    sumV += box.v * weight;
  }
  return sumW > 0
    ? {u:sumU/sumW, v:sumV/sumW}
    : (mainBBox ? {u:mainBBox.u, v:mainBBox.v} : {u:0.5, v:0.5});
}
```

## Patch B — replace the centroid assignment in evaluate()

Replace:

```js
const centroid = mainBBox ? { u: mainBBox.u, v: mainBBox.v } : { u:0.5, v:0.5 };
```

with:

```js
const centroid = computeCompositionCentroid(frame, cam, mainBBox);
```

## Important validation before applying

- Check scene-object transform/size conventions against a few real objects.
- Verify that hidden objects and `role === 'guide'` do not affect the result.
- Test one centered object, a second object added on one side, and a hidden second object.
- This is a projected-bounding-box approximation, not a pixel-accurate ID/depth map. Full ID/depth maps remain a separate Stage 5 task.
- Do not change `EvaluatorV2` or `ImproveEngine` as part of this patch.


## Follow-up correction

The shared projected bounding-box helper now applies each object's Euler rotation and scale before projection. `computeMainObjBBox()` uses the same helper as the composition centroid, so the focus criterion and composition weighting share consistent screen-space bounds. Hidden objects and guides remain excluded. The file was fetched again after commit to verify the new blob SHA and helper definitions; live browser behavior has not yet been tested.


## Regression coverage

Added pure-function tests to the in-app test runner for: equal-weight objects centering at the midpoint, the main object receiving extra weight, and invalid/zero-area items being ignored. These tests are present in the committed file; they have not yet been executed in a live browser session.
