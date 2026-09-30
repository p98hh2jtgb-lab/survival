# HUD PRO v5.9.4 — KODI I "AUTO FOTO" (i plote)

Burimi: `CreatorPack_v5.9.4.zip` — branch `arena/01a0f37d-survival`.
Numrat e rreshtave i pergjigjen versionit 5.9.4.

## 0) Si funksionon (rrjedha)

```
  Paneli "📸 Auto foto (Ctrl+1)"      Lua binding "HUD PRO: Auto photo"
  butoni "📸 Auto" per rrumullak             |
  Ctrl+1 (keydown ne UI)                     v
            |                       extensions.creatorpack.pfhudAutoPhoto()
            v                                    |
   bngApi.engineLua(AUTO_PHOTO_LUA, cb)          |
   (Lua direkt, kthen JSON)                      |
            |                                    v
            |                     sendAutoPhoto(auto)
            |                                    |
            |                     findVehiclePhoto()  -> lista e kandidateve
            |                                    |
            +----->  guihooks.trigger('creatorpack.autophoto',
                                      { paths = ..., name = ..., model = ..., auto = ... })
                                                 |
                            $rootScope.$on('creatorpack.autophoto')  (app.js)
                                                 v
                                  handleAutoPhoto(data)
                                                 v
                    Image() -> removeBackground() (flood-fill) -> crop
                                                 v
                        placeAutoPhoto(dataUrl, name) -> item.image -> CSS
                        background-image i rrumullakut (imageBgStyle)
```

---

## 1) LUA — `lua/ge/extensions/creatorpack.lua`

### 1.1 Ndihmesit qe perdoren nga auto foto (rreshtat 88-99)

```lua
local function normKey(v)
  if v == nil then return '' end
  v = tostring(v):lower():gsub('\\','/')
  v = v:gsub('%.pc$','')
  v = v:gsub('^vehicles/','')
  return v
end

local function basenameKey(v)
  v = normKey(v)
  return v:match('([^/]+)$') or v
end
```

### 1.2 `fileOk` — a ekziston fajlli (rreshtat 394-404)

```lua
-- v5.9.0: AUTO FOTO — merr foton zyrtare te makines aktuale
-- =====================================================
local function fileOk(path)
  if not path or path == '' then return false end
  local p = tostring(path):gsub('^/', '')
  if FS and FS.fileExists then
    local ok, r = pcall(function() return FS:fileExists(p) or FS:fileExists('/' .. p) end)
    return ok and r
  end
  return true
end
```

### 1.3 KETU ISHTE BUG-I — rregullimi v5.9.4 (rreshtat 406-437)

```lua
-- v5.9.4 RREGULLIM KRYESOR (auto foto):
-- be:getObject() merr INDEKS (0 .. be:getObjectCount()-1), JO ID!
-- Kodi i v5.9.0-v5.9.2 i kalonte ID-ne e makines aty -> kthente gjithmone nil
-- -> lista e fotove mbetej e zbrazet -> auto foto nuk punonte kurre.
-- Per ID perdoret be:getObjectByID() (dhe be:getPlayerVehicle() si rruge e dyte).
local function currentVehicle()
  local vehId = be:getPlayerVehicleID(0)
  if not vehId then return nil end
  local veh = nil
  if be.getObjectByID then
    local ok, v = pcall(be.getObjectByID, be, vehId)
    if ok then veh = v end
  end
  if not veh and be.getPlayerVehicle then
    local ok, v = pcall(be.getPlayerVehicle, be, 0)
    if ok then veh = v end
  end
  return veh
end

-- Lexon nje fushe te makines: si veti e drejtperdrejte, ose me getField()
-- (disa versione te lojes i ekspozojne vetem me getField('JBeam','')).
local function vehField(veh, field)
  if not veh then return nil end
  local ok, v = pcall(function() return veh[field] end)
  if ok and type(v) == 'string' and v ~= '' then return v end
  if veh.getField then
    ok, v = pcall(function() return veh:getField(field, '') end)
    if ok and type(v) == 'string' and v ~= '' then return v end
  end
  return nil
end
```

### 1.4 `findVehiclePhoto` — mbledh kandidatet (rreshtat 439-495)

```lua
local function findVehiclePhoto()
  local veh = currentVehicle()
  local model = vehField(veh, 'JBeam')
  if not model then return {}, nil, nil end
  local configKey = basenameKey(vehField(veh, 'partConfig') or '')
  local cands, name = {}, model

  local okCur, cur = pcall(core_vehicles.getCurrentVehicleDetails)
  if okCur and cur then
    local c = cur.configs or cur.config or (cur.current and cur.current.config)
    if type(c) == 'table' then table.insert(cands, c.preview); name = c.Name or c.Configuration or name end
    if cur.current and type(cur.current) == 'table' then table.insert(cands, cur.current.preview) end
    if cur.model and type(cur.model) == 'table' then
      if cur.model.Name then name = (cur.model.Brand and (cur.model.Brand .. ' ') or '') .. cur.model.Name end
    end
  end
  local okList, list = pcall(core_vehicles.getConfigList, true)
  if okList and list and list.configs then
    for _, cfg in pairs(list.configs) do
      if normKey(cfg.model_key) == normKey(model) and configKey and basenameKey(cfg.key or '') == configKey then
        table.insert(cands, 1, cfg.preview)
      end
    end
  end
  local okModel, m = pcall(core_vehicles.getModel, model)
  if okModel and m and m.model then table.insert(cands, m.model.preview) end
  for _, ext in ipairs({ 'jpg', 'png', 'jpeg' }) do
    if configKey then table.insert(cands, '/vehicles/' .. model .. '/' .. configKey .. '.' .. ext) end
  end
  for _, ext in ipairs({ 'jpg', 'png', 'jpeg' }) do table.insert(cands, '/vehicles/' .. model .. '/default.' .. ext) end

  -- v5.9.1: te gjitha fotot ne folderin e makines (edhe mod-et ne zip)
  if FS and FS.findFiles then
    for _, pat in ipairs({ '*.jpg', '*.png', '*.jpeg' }) do
      local okF, files = pcall(function() return FS:findFiles('/vehicles/' .. model .. '/', pat, 0, false, false) end)
      if okF and files then
        local def, rest = {}, {}
        for _, f in ipairs(files) do
          local low = string.lower(f)
          if configKey and low:find(string.lower(configKey), 1, true) then table.insert(def, 1, f)
          elseif low:find('default', 1, true) then table.insert(def, f)
          elseif not low:find('skin', 1, true) and not low:find('_d.', 1, true) and not low:find('_n.', 1, true) then table.insert(rest, f) end
        end
        for _, f in ipairs(def) do table.insert(cands, f) end
        for _, f in ipairs(rest) do table.insert(cands, f) end
      end
    end
  end
  local out, seen = {}, {}
  for _, c in ipairs(cands) do
    if type(c) == 'string' and c ~= '' then
      if c:sub(1, 1) ~= '/' then c = '/' .. c end
      if not seen[c] then seen[c] = true; table.insert(out, c) end
    end
  end
  return out, name, model
end
```

### 1.5 Dergimi te UI + hook-u i ndrrimit te makines (rreshtat 497-522)

```lua
local function sendAutoPhoto(auto)
  if guihooks == nil or guihooks.trigger == nil then return end
  local ok, paths, name, model = pcall(findVehiclePhoto)
  if not ok or type(paths) ~= 'table' then paths = {} end
  local lim = {}
  for i = 1, math.min(#paths, 40) do lim[i] = paths[i] end
  guihooks.trigger('creatorpack.autophoto', { paths = lim, name = name, model = model, auto = auto and true or false })
end
local function pfhudAutoPhoto() sendAutoPhoto(false) end

local function onVehicleSwitched(oldId, newId)
  pcall(sendAutoPhoto, true)
  resolveVehicleValue()
  -- v3.2: makina e re mund te kete tashme dem te grumbulluar (p.sh. e ke
  -- rrahur me pare). Nese e leme damageBase=0, ratio kercen menjehere lart
  -- dhe kostoja e makines se re duket e gabuar. Marrim demin AKTUAL si baze,
  -- keshtu qe cdo makine e re fillon nga $0.
  local vehId = be:getPlayerVehicleID(0)
  local obj = vehId and map.objects[vehId]
  damageBase = (obj and obj.damage) or 0
end

local function onVehicleResetted(vehId)
  -- pas nje reset-i fizik makina eshte e paprekur -> baza kthehet ne 0
  if vehId == be:getPlayerVehicleID(0) then damageBase = 0 end
end
```

### 1.6 Eksportet (rreshtat 547-548, 562-563)

```lua
M.pfhudAutoPhoto   = pfhudAutoPhoto
M.sendAutoPhoto    = sendAutoPhoto
M.onVehicleSwitched   = onVehicleSwitched
M.onVehicleResetted   = onVehicleResetted
```

### 1.7 Binding-u (tasti) — `lua/ge/extensions/core/input/actions/pfhud_actions.json`

```json
{
  "pfhud_auto_photo": {
    "cat": "gameplay",
    "order": 9112,
    "ctx": "tlua",
    "actionMap": "Global",
    "onDown": "extensions.load('creatorpack'); if extensions and extensions.creatorpack then extensions.creatorpack.pfhudAutoPhoto() end",
    "title": "HUD PRO: Auto photo of current car",
    "desc": "Put the current car photo (no background) into the selected circle. Suggested key: Ctrl+1.",
    "isBasic": true
  }
}
```

---

## 2) JAVASCRIPT — `ui/modules/apps/CreatorPack/app.js`

### 2.1 Cilesimet (rreshtat 655-657)

```javascript
        if (cfg.autoPhotoOnSwitch == null) cfg.autoPhotoOnSwitch = false;
        if (cfg.autoPhotoNext == null) cfg.autoPhotoNext = false;
        if (cfg.autoPhotoTolerance == null) cfg.autoPhotoTolerance = 28;
```

### 2.2 Leximi direkt ne Lua (v5.9.3+) + butoni (rreshtat 4379-4409)

Ne `app.js` kjo ruhet si **nje rresht i vetem** me `\n` brenda stringut JS.
Kodi i lexuar (i zbukuruar) eshte ky:

```lua
(function()
  local v = be:getPlayerVehicle(0)
  if not v then return {err='no vehicle'} end
  local function field(name)
    local ok, r = pcall(function() return v[name] end)
    if ok and type(r) == 'string' and r ~= '' then return r end
    if v.getField then
      ok, r = pcall(function() return v:getField(name, '') end)
      if ok and type(r) == 'string' and r ~= '' then return r end
    end
    return nil
  end
  local m = field('JBeam') or 'vehicle'
  local ck = string.match(field('partConfig') or '', '([^/\\]+)%.pc$')
  local out, seen = {}, {}
  local function add(p)
    if type(p) == 'string' and p ~= '' then
      if string.sub(p, 1, 1) ~= '/' then p = '/' .. p end
      if not seen[p] and #out < 40 then seen[p] = true; table.insert(out, p) end
    end
  end
  local name = m
  pcall(function()
    local d = core_vehicles.getCurrentVehicleDetails()
    if d then
      if type(d.configs) == 'table' then add(d.configs.preview); name = d.configs.Name or name end
      if type(d.current) == 'table' then add(d.current.preview) end
      if type(d.model) == 'table' then
        add(d.model.preview)
        if d.model.Name then name = (d.model.Brand and (d.model.Brand .. ' ') or '') .. d.model.Name end
      end
    end
  end)
  pcall(function() local mi = core_vehicles.getModel(m); if mi and mi.model then add(mi.model.preview) end end)
  if ck then add('/vehicles/' .. m .. '/' .. ck .. '.jpg'); add('/vehicles/' .. m .. '/' .. ck .. '.png') end
  add('/vehicles/' .. m .. '/default.jpg'); add('/vehicles/' .. m .. '/default.png'); add('/vehicles/' .. m .. '/default.jpeg')
  pcall(function()
    for _, pat in ipairs({'*.jpg', '*.png', '*.jpeg'}) do
      for _, f in ipairs(FS:findFiles('/vehicles/' .. m .. '/', pat, 0, false, false) or {}) do add(f) end
    end
  end)
  return {paths = out, model = m, name = name}
end)()
```

Funksioni qe e thire:

```javascript
      // =====================================================
      // v5.9.0: AUTO FOTO — foto e makines + fshirje sfondi + rrumullak
      // =====================================================
      var autoPhotoTarget = null;
      scope.autoPhotoMsg = '';
      function autoMsg(t) { scope.autoPhotoMsg = t; $timeout(function(){ if (scope.autoPhotoMsg === t) scope.autoPhotoMsg = ''; }, 3500); }
      var AUTO_PHOTO_LUA = "(function()\n  local v = be:getPlayerVehicle(0)\n  if not v then return {err='no vehicle'} end\n  local function field(name)\n    local ok, r = pcall(function() return v[name] end)\n    if ok and type(r) == 'string' and r ~= '' then return r end\n    if v.getField then\n      ok, r = pcall(function() return v:getField(name, '') end)\n      if ok and type(r) == 'string' and r ~= '' then return r end\n    end\n    return nil\n  end\n  local m = field('JBeam') or 'vehicle'\n  local ck = string.match(field('partConfig') or '', '([^/\\\\]+)%.pc$')\n  local out, seen = {}, {}\n  local function add(p)\n    if type(p) == 'string' and p ~= '' then\n      if string.sub(p, 1, 1) ~= '/' then p = '/' .. p end\n      if not seen[p] and #out < 40 then seen[p] = true; table.insert(out, p) end\n    end\n  end\n  local name = m\n  pcall(function()\n    local d = core_vehicles.getCurrentVehicleDetails()\n    if d then\n      if type(d.configs) == 'table' then add(d.configs.preview); name = d.configs.Name or name end\n      if type(d.current) == 'table' then add(d.current.preview) end\n      if type(d.model) == 'table' then\n        add(d.model.preview)\n        if d.model.Name then name = (d.model.Brand and (d.model.Brand .. ' ') or '') .. d.model.Name end\n      end\n    end\n  end)\n  pcall(function() local mi = core_vehicles.getModel(m); if mi and mi.model then add(mi.model.preview) end end)\n  if ck then add('/vehicles/' .. m .. '/' .. ck .. '.jpg'); add('/vehicles/' .. m .. '/' .. ck .. '.png') end\n  add('/vehicles/' .. m .. '/default.jpg'); add('/vehicles/' .. m .. '/default.png'); add('/vehicles/' .. m .. '/default.jpeg')\n  pcall(function()\n    for _, pat in ipairs({'*.jpg', '*.png', '*.jpeg'}) do\n      for _, f in ipairs(FS:findFiles('/vehicles/' .. m .. '/', pat, 0, false, false) or {}) do add(f) end\n    end\n  end)\n  return {paths = out, model = m, name = name}\nend)()";
      var autoPhotoWait = null;
      scope.autoPhoto = function(index) {
        autoPhotoTarget = (index == null) ? null : Number(index);
        autoMsg('📸 duke marrë foton…');
        if (autoPhotoWait) $timeout.cancel(autoPhotoWait);
        autoPhotoWait = $timeout(function(){ autoMsg('✗ Loja s\'u përgjigj. Fshij versionet e vjetra të CreatorPack te mods dhe rindize lojën.'); }, 6000);
        var sent = false;
        try {
          if (window.bngApi && typeof window.bngApi.engineLua === 'function') {
            window.bngApi.engineLua(AUTO_PHOTO_LUA, function(res) {
              scope.$applyAsync(function() {
                if (autoPhotoWait) { $timeout.cancel(autoPhotoWait); autoPhotoWait = null; }
                if (typeof res === 'string') { try { res = JSON.parse(res); } catch (e) {} }
                if (!res || res.err) { autoMsg('✗ ' + ((res && res.err) || 'pa përgjigje') ); return; }
                var paths = res.paths; if (paths && !Array.isArray(paths)) paths = Object.keys(paths).map(function(k){ return paths[k]; });
                if (window.console && console.log) console.log('[HUD PRO] autofoto (Lua): model=' + (res.model || '?') + ', foto=' + ((paths && paths.length) || 0) + ', emri=' + (res.name || '?'));
                handleAutoPhoto({ paths: paths || [], name: res.name, model: res.model, auto: false });
              });
            });
            sent = true;
          }
        } catch (e) {}
        if (!sent) engineLua("extensions.load('creatorpack'); if extensions.creatorpack and extensions.creatorpack.pfhudAutoPhoto then extensions.creatorpack.pfhudAutoPhoto() end");
      };
```

### 2.3 Fshirja e sfondit (rreshtat 4410-4451)

```javascript
      function colDist(d, i, r, g, b) { var dr = d[i]-r, dg = d[i+1]-g, db = d[i+2]-b; return Math.sqrt(dr*dr + dg*dg + db*db); }
      function removeBackground(img, tol) {
        var MAX = 512, sc = Math.min(1, MAX / Math.max(img.naturalWidth || img.width, img.naturalHeight || img.height));
        var w = Math.max(1, Math.round((img.naturalWidth || img.width) * sc)), h = Math.max(1, Math.round((img.naturalHeight || img.height) * sc));
        var c = document.createElement('canvas'); c.width = w; c.height = h;
        var x = c.getContext('2d'); x.drawImage(img, 0, 0, w, h);
        var id = x.getImageData(0, 0, w, h), d = id.data, n = w * h;
        var bg = new Uint8Array(n), stack = [];
        var step = tol * 0.45;           // ndryshim lokal (gradient i sfondit)
        function seed(p) { if (!bg[p]) { bg[p] = 1; stack.push(p); } }
        for (var i = 0; i < w; i++) { seed(i); seed((h - 1) * w + i); }
        for (var k = 0; k < h; k++) { seed(k * w); seed(k * w + w - 1); }
        // ngjyra mesatare e skajeve
        var sr = 0, sg = 0, sb = 0, sn = 0;
        stack.forEach(function(p){ sr += d[p*4]; sg += d[p*4+1]; sb += d[p*4+2]; sn++; });
        sr /= sn; sg /= sn; sb /= sn;
        while (stack.length) {
          var p = stack.pop(), px = p % w, py = (p / w) | 0, pi = p * 4;
          var nb = [px > 0 ? p - 1 : -1, px < w - 1 ? p + 1 : -1, py > 0 ? p - w : -1, py < h - 1 ? p + w : -1];
          for (var q = 0; q < 4; q++) {
            var o = nb[q]; if (o < 0 || bg[o]) continue;
            var oi = o * 4;
            if (d[oi+3] < 20 || (colDist(d, oi, d[pi], d[pi+1], d[pi+2]) < step && colDist(d, oi, sr, sg, sb) < tol * 2.2)) { bg[o] = 1; stack.push(o); }
          }
        }
        // alpha + skaj i bute
        var minX = w, minY = h, maxX = -1, maxY = -1;
        for (var y = 0; y < h; y++) for (var xx = 0; xx < w; xx++) {
          var p2 = y * w + xx;
          if (bg[p2]) { d[p2*4+3] = 0; continue; }
          var edge = (xx > 0 && bg[p2-1]) || (xx < w-1 && bg[p2+1]) || (y > 0 && bg[p2-w]) || (y < h-1 && bg[p2+w]);
          if (edge) d[p2*4+3] = Math.min(d[p2*4+3], 150);
          if (xx < minX) minX = xx; if (xx > maxX) maxX = xx; if (y < minY) minY = y; if (y > maxY) maxY = y;
        }
        x.putImageData(id, 0, 0);
        var fg = 0; for (var z = 0; z < n; z++) if (!bg[z]) fg++;
        if (maxX < 0 || fg < n * 0.08) return null;   // v5.9.2: e hengri makinen -> provo me pak
        var bw = maxX - minX + 1, bh = maxY - minY + 1, pad = Math.round(Math.max(bw, bh) * 0.06);
        var out = document.createElement('canvas'); out.width = bw + pad * 2; out.height = bh + pad * 2;
        out.getContext('2d').drawImage(c, minX, minY, bw, bh, pad, pad, bw, bh);
        return out.toDataURL('image/png');
      }
```

### 2.4 Vendosja ne rrumullak (rreshtat 4452-4470)

```javascript
      function placeAutoPhoto(dataUrl, name) {
        var items = scope.cfg.items || [];
        var idx = autoPhotoTarget != null ? autoPhotoTarget : Number(scope.selected) || 0;
        autoPhotoTarget = null;
        if (!items[idx] || items[idx].enabled === false) {
          idx = -1; for (var e3 = 0; e3 < items.length; e3++) if (items[e3].enabled !== false && !items[e3].image) { idx = e3; break; }
          if (idx < 0) idx = 0;
        }
        if (!items[idx]) { autoMsg('✗ S\'ka rrumullak'); return; }
        items[idx].enabled = true;
        items[idx].imageScale = 0.62; items[idx].imageOffsetX = 0; items[idx].imageOffsetY = 0; items[idx].imageRotation = 0;
        items[idx].image = dataUrl;
        if (name && (!items[idx].label || /^(car|makina|vehicle)?\s*\d*$/i.test(items[idx].label))) items[idx].label = String(name);
        if (scope.cfg.autoPhotoNext) {
          for (var s = 1; s <= items.length; s++) { var j2 = (idx + s) % items.length; if (!items[j2].image) { scope.selected = j2; scope.cfg.currentIndex = j2; break; } }
        }
        scope.persist();
        autoMsg('✓ Foto u vendos në rrumullakun ' + (idx + 1) + (name ? ' (' + name + ')' : ''));
      }
```

### 2.5 Degjuesi i eventit + `handleAutoPhoto` (rreshtat 4471-4515)

```javascript
      var unsubAutoPhoto = $rootScope.$on('creatorpack.autophoto', function(e, data) {
        if (data && !data.auto && autoPhotoWait) { $timeout.cancel(autoPhotoWait); autoPhotoWait = null; }
        scope.$applyAsync(function(){ handleAutoPhoto(data); });
      });
      function handleAutoPhoto(data) {
        {
          if (!data) return;
          if (data.auto && !scope.cfg.autoPhotoOnSwitch) return;
          var list = [];
          (data.paths || []).forEach(function(p){ if (p && list.indexOf(p) < 0) list.push(p); });
          if (data.path && list.indexOf(data.path) < 0) list.unshift(data.path);
          if (!list.length) { autoMsg('✗ S\'u gjet foto (' + (data.model || '?') + ')'); return; }
          // v5.9.4: provo edhe rrugen e plote local://local/... (disa versione te CEF)
          var cands2 = [];
          list.forEach(function(p){
            if (!p) return;
            var s = String(p); if (s.charAt(0) !== '/' && !/^[a-z]+:/i.test(s)) s = '/' + s;
            if (cands2.indexOf(s) < 0) cands2.push(s);
            if (s.charAt(0) === '/' && cands2.indexOf('local://local' + s) < 0) cands2.push('local://local' + s);
          });
          if (window.console && console.log) console.log('[HUD PRO] autofoto: ' + list.length + ' kandidate, model=' + (data.model || '?'));
          var tol = Math.max(8, Math.min(70, Number(scope.cfg.autoPhotoTolerance) || 28));
          (function tryAt(k) {
            if (k >= cands2.length) { autoMsg('✗ S\'u gjet foto (' + (data.model || '?') + ', ' + cands2.length + ' provuar)'); return; }
            var src = cands2[k];
            var img = new Image();
            img.onload = function() {
              if (!img.naturalWidth) return tryAt(k + 1);
              var url = null;
              try {
                var tries = [tol, tol * 0.6, tol * 0.35, 10];
                for (var t = 0; t < tries.length && !url; t++) url = removeBackground(img, tries[t]);
              } catch (err) { url = null; }
              if (!url) { try { var cc = document.createElement('canvas'); cc.width = img.naturalWidth; cc.height = img.naturalHeight; cc.getContext('2d').drawImage(img, 0, 0); url = cc.toDataURL('image/png'); } catch (e2) { url = src; } }
              scope.autoPhotoPreview = url; scope.autoPhotoSrc = src;
              scope.$applyAsync(function(){ placeAutoPhoto(url, data.name); });
            };
            img.onerror = function() {
              if (window.console && console.log) console.log('[HUD PRO] autofoto: imazhi nuk u ngarkua -> ' + src);
              tryAt(k + 1);
            };
            img.src = src;
          })(0);
        }
      }
```

### 2.6 Tasti Ctrl+1 (rreshtat 4580-4584)

```javascript
        // v5.9.0: Ctrl+1 = Auto foto
        if ((e.ctrlKey || e.metaKey) && (String(e.key || '') === '1' || e.code === 'Digit1')) {
          scope.$applyAsync(function(){ scope.autoPhoto(); });
          e.preventDefault(); e.stopPropagation(); return false;
        }
```

---

## 3) HTML — `ui/modules/apps/CreatorPack/app.html`

### 3.1 Blloku ne panel (rreshtat 1262-1272)

```html
        <button class="pf-btn green" ng-click="autoPhoto()" title="Ctrl+1">📸 Auto foto (Ctrl+1)</button>
        <button class="pf-btn" ng-click="clearAllImages()">Fshi fotot</button>
      </div>
      <div class="pf-row" style="margin-top:6px">
        <label class="pf-toggle"><input type="checkbox" ng-model="cfg.circlesClean" ng-change="persist()"> ⭕ Rrumullakat të pastër (pa emër, pa %, pa unazë)</label>
        <label class="pf-toggle"><input type="checkbox" ng-model="cfg.autoPhotoOnSwitch" ng-change="persist()"> 📸 Foto automatike kur ndërron makinën</label>
        <label class="pf-toggle"><input type="checkbox" ng-model="cfg.autoPhotoNext" ng-change="persist()"> pastaj kalo te rrumullaku i radhës bosh</label>
        <span class="mini">Fshirja e sfondit: {{cfg.autoPhotoTolerance}}</span>
        <input class="pf-range" style="width:120px" type="range" min="8" max="70" step="1" ng-model="cfg.autoPhotoTolerance" ng-change="persist()">
        <span class="mini" ng-if="autoPhotoMsg" style="color:#63ff9f">{{autoPhotoMsg}}</span>
        <img ng-if="autoPhotoPreview" ng-src="{{autoPhotoPreview}}" title="{{autoPhotoSrc}}" style="height:46px;max-width:110px;object-fit:contain;background:repeating-conic-gradient(#555 0 25%,#333 0 50%) 0/10px 10px;border-radius:6px">
```

### 3.2 Butoni per çdo rrumullak (rreshti 2040)

```html
                <button class="pf-btn slim green" ng-click="autoPhoto($index)">📸 Auto</button>
```

---

## 4) ÇKA ISHTE E PRISHUR (5.9.0 - 5.9.2) vs 5.9.4

```diff
  local function findVehiclePhoto()
    local vehId = be:getPlayerVehicleID(0)
-   local veh = vehId and be:getObject(vehId)      -- ID ne vend INDEKSI -> nil gjithmone
-   if not veh or not veh.JBeam then return {}, nil, nil end
-   local model = veh.JBeam
-   local configKey = basenameKey(veh.partConfig or '')
+   local veh = currentVehicle()                   -- be:getObjectByID(vehId) (+ getPlayerVehicle)
+   local model = vehField(veh, 'JBeam')           -- edhe me veh:getField('JBeam','')
+   if not model then return {}, nil, nil end
+   local configKey = basenameKey(vehField(veh, 'partConfig') or '')
    local cands, name = {}, model
```

Rezultati i testit mbi nje mjedis te simuluar GE Lua:

| Versioni | Rruga e perdorur | Rezultati |
|---|---|---|
| 5.9.0 / 5.9.1 / 5.9.2 / 5.9.3 (funksioni i vjeter) | `be:getObject(id)` | `paths=0` — asgje nuk gjendet |
| 5.9.4 | `be:getObjectByID(id)` | `paths=7` — fotoja gjendet |

---

## 5) Si ta shtosh ne nje mod tjeter

1. Ne `lua/ge/extensions/<modi>.lua` kopjo: `normKey`, `basenameKey`, `fileOk`,
   `currentVehicle`, `vehField`, `findVehiclePhoto`, `sendAutoPhoto`.
2. Ne `M.onVehicleSwitched` therrit `pcall(sendAutoPhoto, true)` (ose veç `sendAutoPhoto(false)`
   per nje buton) dhe eksportoji: `M.pfhudAutoPhoto = pfhudAutoPhoto`.
3. Ne JavaScript-in e UI-se kopjo: `AUTO_PHOTO_LUA`, `scope.autoPhoto`, `colDist`,
   `removeBackground`, `placeAutoPhoto`, degjuesin e `creatorpack.autophoto` dhe `handleAutoPhoto`
   (ndrro emrin e eventit).
4. Ku ruan imazhin: `item.image = dataUrl` + CSS `background-image` (shih `imageBgStyle`).

Kujdes: `be:getObject()` = **indeks**, `be:getObjectByID()` = **ID**. Mos i perziej.
