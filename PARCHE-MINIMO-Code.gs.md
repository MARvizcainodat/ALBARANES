# Parche mínimo para tu Code.gs actual

**Objetivo:** añadir el módulo de entregas **sin sustituir** tu script de tesorería.

**Antes de empezar**
1. En Apps Script: Archivo → Gestionar versiones (o copia todo el código a un Bloc de notas).
2. No cambies `SHEET_ID` ni `CLAVE`.
3. Haz los 4 pasos en orden. Al terminar: **Implementar → Administrar implementaciones → Editar → Nueva versión**.

---

## PASO 1 — Ampliar la lista COLS

Busca esta línea (o muy parecida):

```javascript
const COLS = ['clients','suppliers','works','invoicesOut','invoicesIn','fixedExpenses','fixedExpensePayments','workBudgetItems','priceListItems','invoiceOutPayments','invoiceInPayments','bankAccounts','bankBalances','oneOffExpenses','bankTransfers','materialGlossary'];
```

**Sustitúyela solo por esta** (mismas colecciones de siempre + 6 nuevas al final):

```javascript
const COLS = [
  'clients','suppliers','works','invoicesOut','invoicesIn',
  'fixedExpenses','fixedExpensePayments','workBudgetItems','priceListItems',
  'invoiceOutPayments','invoiceInPayments','bankAccounts','bankBalances',
  'oneOffExpenses','bankTransfers','materialGlossary',
  /* módulo entregas */
  'prealbaranes','deliveryVersions','deliveryTokens',
  'deliveryEvents','smsOutbox','deliveryAttachments'
];

const DELIVERY_CFG = {
  tokenTtlHours: 72,
  maxTokenAttempts: 8,
  retentionNote: 'Definir política de retención. No se inventan plazos legales.'
};
```

El panel de tesorería sigue funcionando: solo usa las colecciones que ya conoce.

---

## PASO 2 — Sustituir solo la función doGet

Busca:

```javascript
function doGet() { return out_({ ok: true, mensaje: 'Servicio del panel activo' }); }
```

**Sustitúyela por:**

```javascript
function doGet(e) {
  if (e && e.parameter && e.parameter.t) {
    return handlePublicGetDelivery_(e.parameter.t);
  }
  return out_({ ok: true, mensaje: 'Servicio del panel y módulo de entregas activo' });
}
```

---

## PASO 3 — Ajustar doPost (tres cambios pequeños)

### 3a. Acciones públicas ANTES de la comprobación de clave

Justo después de:

```javascript
const p = JSON.parse(e.postData.contents);
```

**Inserta estas líneas** (antes del `if (CLAVE...`):

```javascript
    /* Acciones públicas del receptor (sin clave; protegidas por token) */
    if (p.action === 'publicGetDelivery') return handlePublicGetDelivery_(p.token);
    if (p.action === 'publicSubmitConfirmation') return handlePublicSubmitConfirmation_(p);
    if (p.action === 'publicMarkNotArrived') return handlePublicMarkNotArrived_(p);
```

### 3b. Dentro del bloque con candado (lock)

Ahora el código hace:

```javascript
      if (p.action === 'add') {
        ...
      }
      else if (p.action === 'update') {
        ...
      } else if (p.action === 'delete') { ... }
      else throw new Error('Acción no válida');
```

**Sustituye el `else throw...` por:**

```javascript
      else if (p.action === 'delete') {
        const r = rowOf_(s, p.id);
        if (r > 0) s.deleteRow(r);
      }
      /* --- acciones del módulo de entregas --- */
      else if (p.action === 'createDeliveryVersion') return createDeliveryVersion_(p);
      else if (p.action === 'prepareSmsManual') return prepareSmsManual_(p);
      else if (p.action === 'registerManualSmsSent') return registerManualSmsSent_(p);
      else if (p.action === 'revokeToken') return revokeToken_(p);
      else if (p.action === 'regenerateToken') return regenerateToken_(p);
      else if (p.action === 'attachPaperScan') return attachPaperScan_(p);
      else if (p.action === 'addDeliveryEvent') return addDeliveryEvent_(p);
      else throw new Error('Acción no válida');
```

**Importante:** las acciones de entregas no usan `p.c` (no necesitan la hoja genérica).  
Si tu `doPost` actual hace siempre `const s = sheet_(p.c);` **antes** del `if (p.action === 'add')`, mueve esa línea **dentro** de cada rama add/update/delete, o deja el `sheet_(p.c)` solo cuando `p.c` exista:

Ejemplo seguro:

```javascript
    const lock = LockService.getScriptLock(); lock.waitLock(20000);
    try {
      if (p.action === 'add' || p.action === 'update' || p.action === 'delete') {
        const s = sheet_(p.c);
        if (p.action === 'add') {
          if (typeof p.json !== 'string' || p.json.length > 45000) throw new Error('Registro no válido o demasiado grande');
          JSON.parse(p.json);
          if (rowOf_(s, p.id) < 0) s.appendRow([p.id, p.json]);
        } else if (p.action === 'update') {
          const r = rowOf_(s, p.id);
          if (r > 0) s.getRange(r, 2).setValue(JSON.stringify(Object.assign(JSON.parse(s.getRange(r, 2).getValue() || '{}'), JSON.parse(p.patch))));
        } else if (p.action === 'delete') {
          const r = rowOf_(s, p.id);
          if (r > 0) s.deleteRow(r);
        }
        return out_({ ok: true });
      }
      if (p.action === 'createDeliveryVersion') return createDeliveryVersion_(p);
      if (p.action === 'prepareSmsManual') return prepareSmsManual_(p);
      if (p.action === 'registerManualSmsSent') return registerManualSmsSent_(p);
      if (p.action === 'revokeToken') return revokeToken_(p);
      if (p.action === 'regenerateToken') return regenerateToken_(p);
      if (p.action === 'attachPaperScan') return attachPaperScan_(p);
      if (p.action === 'addDeliveryEvent') return addDeliveryEvent_(p);
      throw new Error('Acción no válida');
    } finally { lock.releaseLock(); }
```

(Puedes usar este bloque completo si te resulta más claro; sustituye solo el interior del `try` del candado.)

---

## PASO 4 — Pegar al FINAL del archivo (después de doPost)

Copia **todo** el bloque siguiente y pégalo al final de tu `Code.gs`, después de la última llave de `doPost`.

```javascript
/* ========== MÓDULO ENTREGAS / PREALBARANES (añadido) ========== */
function nowIso_() {
  return Utilities.formatDate(new Date(), 'Europe/Madrid', "yyyy-MM-dd'T'HH:mm:ss");
}
function uid_(prefix) {
  return (prefix || '') + Utilities.getUuid().replace(/-/g, '');
}
function randomToken_() {
  const bytes = Utilities.getUuid() + Utilities.getUuid() + Utilities.getUuid();
  return Utilities.base64EncodeWebSafe(
    Utilities.computeDigest(Utilities.DigestAlgorithm.SHA_256, bytes)
  ).replace(/=+$/, '').substring(0, 48);
}
function getJson_(col, id) {
  const s = sheet_(col);
  const r = rowOf_(s, id);
  if (r < 0) return null;
  try { return JSON.parse(s.getRange(r, 2).getValue() || '{}'); } catch (e) { return null; }
}
function setJson_(col, id, obj) {
  const s = sheet_(col);
  const r = rowOf_(s, id);
  const js = JSON.stringify(obj);
  if (r > 0) s.getRange(r, 2).setValue(js);
  else s.appendRow([id, js]);
}
function appendEvent_(ev) {
  const id = uid_('ev_');
  ev.id = id;
  ev.at = ev.at || nowIso_();
  setJson_('deliveryEvents', id, ev);
  return id;
}

function createDeliveryVersion_(p) {
  const pre = getJson_('prealbaranes', p.prealbaranId);
  if (!pre) throw new Error('Prealbarán no encontrado');
  const versionId = uid_('ver_');
  const versionNumber = (pre.versionCount || 0) + 1;
  const snapshot = {
    prealbaranId: p.prealbaranId,
    ref: pre.ref || pre.numeroFormateado || '',
    pedidoRef: pre.pedidoRef || '',
    cliente: pre.clienteNombre || pre.cliente || '',
    clienteCodigo: pre.clienteCodigo || '',
    obra: pre.obraNombre || pre.obra || '',
    direccion: pre.direccionEntrega || pre.direccion || '',
    fechaPrevista: pre.fechaPrevista || '',
    contactoNombre: pre.contactoNombre || '',
    contactoTelefono: pre.contactoTelefono || '',
    lineas: pre.lineas || [],
    observaciones: pre.observaciones || '',
    createdAt: nowIso_()
  };
  const version = {
    id: versionId,
    prealbaranId: p.prealbaranId,
    versionNumber: versionNumber,
    snapshot: snapshot,
    status: 'pendiente_envio',
    isSigned: false,
    confirmationType: null,
    createdAt: nowIso_(),
    createdBy: p.actor || 'usuario'
  };
  setJson_('deliveryVersions', versionId, version);
  pre.versionCount = versionNumber;
  pre.currentVersionId = versionId;
  pre.updatedAt = nowIso_();
  setJson_('prealbaranes', p.prealbaranId, pre);
  appendEvent_({
    prealbaranId: p.prealbaranId,
    deliveryVersionId: versionId,
    type: 'version_created',
    actor: p.actor || 'usuario',
    payload: { versionNumber: versionNumber }
  });
  return out_({ ok: true, versionId: versionId, version: version });
}

function prepareSmsManual_(p) {
  const version = getJson_('deliveryVersions', p.deliveryVersionId);
  if (!version) throw new Error('Versión no encontrada');
  if (version.isSigned) throw new Error('La versión ya está firmada');
  const token = randomToken_();
  const expiresAt = new Date();
  expiresAt.setHours(expiresAt.getHours() + (p.ttlHours || DELIVERY_CFG.tokenTtlHours));
  const expiresIso = Utilities.formatDate(expiresAt, 'Europe/Madrid', "yyyy-MM-dd'T'HH:mm:ss");
  const tokenRec = {
    id: token,
    deliveryVersionId: p.deliveryVersionId,
    prealbaranId: version.prealbaranId,
    contactPhone: version.snapshot.contactoTelefono || p.phone || '',
    contactName: version.snapshot.contactoNombre || '',
    expiresAt: expiresIso,
    revokedAt: null,
    usedAt: null,
    createdAt: nowIso_(),
    createdBy: p.actor || 'Agente de Operaciones'
  };
  setJson_('deliveryTokens', token, tokenRec);
  const baseUrl = (p.publicBaseUrl || '').replace(/\/$/, '');
  const link = baseUrl ? (baseUrl + '/confirmar.html?t=' + token) : ('confirmar.html?t=' + token);
  const mensaje = 'Mármoles Hermanos Vizcaíno: material entregado en obra. Confirme la recepción cuando lo haya revisado: ' +
    link + ' (enlace personal, caduca). Si no ha llegado, indíquelo en el enlace.';
  const outboxId = uid_('sms_');
  const outbox = {
    id: outboxId,
    prealbaranId: version.prealbaranId,
    deliveryVersionId: p.deliveryVersionId,
    tokenId: token,
    phone: tokenRec.contactPhone,
    messageBody: mensaje,
    link: link,
    status: 'prepared',
    provider: 'manual',
    preparedAt: nowIso_(),
    preparedBy: p.actor || 'Agente de Operaciones',
    sentAt: null,
    sentBy: null
  };
  setJson_('smsOutbox', outboxId, outbox);
  version.status = 'sms_preparado';
  setJson_('deliveryVersions', p.deliveryVersionId, version);
  appendEvent_({
    prealbaranId: version.prealbaranId,
    deliveryVersionId: p.deliveryVersionId,
    type: 'sms_prepared',
    actor: p.actor || 'Agente de Operaciones',
    payload: { outboxId: outboxId, phone: tokenRec.contactPhone, expiresAt: expiresIso }
  });
  return out_({
    ok: true,
    outboxId: outboxId,
    token: token,
    link: link,
    messageBody: mensaje,
    expiresAt: expiresIso,
    phone: tokenRec.contactPhone
  });
}

function registerManualSmsSent_(p) {
  const outbox = getJson_('smsOutbox', p.outboxId);
  if (!outbox) throw new Error('Registro de SMS no encontrado');
  outbox.status = 'sent_manual';
  outbox.sentAt = nowIso_();
  outbox.sentBy = p.actor || 'Agente de Operaciones';
  setJson_('smsOutbox', p.outboxId, outbox);
  const version = getJson_('deliveryVersions', outbox.deliveryVersionId);
  if (version) {
    version.status = 'sms_enviado';
    setJson_('deliveryVersions', outbox.deliveryVersionId, version);
  }
  appendEvent_({
    prealbaranId: outbox.prealbaranId,
    deliveryVersionId: outbox.deliveryVersionId,
    type: 'sms_sent_manual',
    actor: p.actor || 'Agente de Operaciones',
    payload: { outboxId: p.outboxId, phone: outbox.phone }
  });
  return out_({ ok: true });
}

function revokeToken_(p) {
  const tok = getJson_('deliveryTokens', p.token);
  if (!tok) throw new Error('Token no encontrado');
  tok.revokedAt = nowIso_();
  setJson_('deliveryTokens', p.token, tok);
  appendEvent_({
    prealbaranId: tok.prealbaranId,
    deliveryVersionId: tok.deliveryVersionId,
    type: 'token_revoked',
    actor: p.actor || 'Agente de Operaciones',
    payload: { token: p.token }
  });
  return out_({ ok: true });
}

function regenerateToken_(p) {
  if (p.oldToken) {
    const old = getJson_('deliveryTokens', p.oldToken);
    if (old && !old.revokedAt) {
      old.revokedAt = nowIso_();
      setJson_('deliveryTokens', p.oldToken, old);
    }
  }
  return prepareSmsManual_(p);
}

function attachPaperScan_(p) {
  const version = getJson_('deliveryVersions', p.deliveryVersionId);
  if (!version) throw new Error('Versión no encontrada');
  const attId = uid_('att_');
  const att = {
    id: attId,
    prealbaranId: version.prealbaranId,
    deliveryVersionId: p.deliveryVersionId,
    type: 'paper_scan',
    fileName: p.fileName || 'escaneado.pdf',
    contentBase64: p.contentBase64 || null,
    notes: p.notes || '',
    attachedAt: nowIso_(),
    attachedBy: p.actor || 'Agente de Operaciones'
  };
  setJson_('deliveryAttachments', attId, att);
  version.status = 'papel_incorporado';
  version.confirmationType = 'paper';
  version.isSigned = false;
  version.paperAttachedAt = nowIso_();
  version.paperAttachedBy = p.actor || 'Agente de Operaciones';
  setJson_('deliveryVersions', p.deliveryVersionId, version);
  appendEvent_({
    prealbaranId: version.prealbaranId,
    deliveryVersionId: p.deliveryVersionId,
    type: 'paper_attached',
    actor: p.actor || 'Agente de Operaciones',
    payload: { attachmentId: attId, fileName: att.fileName }
  });
  return out_({ ok: true, attachmentId: attId });
}

function addDeliveryEvent_(p) {
  const id = appendEvent_({
    prealbaranId: p.prealbaranId,
    deliveryVersionId: p.deliveryVersionId || null,
    type: p.type,
    actor: p.actor || 'usuario',
    payload: p.payload || {}
  });
  return out_({ ok: true, eventId: id });
}

function handlePublicGetDelivery_(token) {
  if (!token || String(token).length < 20) return out_({ error: 'Enlace no válido' });
  const tok = getJson_('deliveryTokens', token);
  if (!tok) return out_({ error: 'Enlace no válido o caducado' });
  if (tok.revokedAt) return out_({ error: 'Este enlace ha sido revocado' });
  if (tok.expiresAt && tok.expiresAt < nowIso_()) return out_({ error: 'Este enlace ha caducado' });
  const version = getJson_('deliveryVersions', tok.deliveryVersionId);
  if (!version) return out_({ error: 'Documento no disponible' });
  if (version.isSigned) {
    return out_({
      ok: true,
      alreadySigned: true,
      status: version.status,
      signedAt: version.signedAt || null,
      message: 'Esta entrega ya fue confirmada.'
    });
  }
  appendEvent_({
    prealbaranId: tok.prealbaranId,
    deliveryVersionId: tok.deliveryVersionId,
    type: 'link_opened',
    actor: 'receptor',
    payload: { token: String(token).substring(0, 8) + '…' }
  });
  return out_({
    ok: true,
    delivery: {
      ref: version.snapshot.ref,
      cliente: version.snapshot.cliente,
      obra: version.snapshot.obra,
      direccion: version.snapshot.direccion,
      fechaPrevista: version.snapshot.fechaPrevista,
      contactoNombre: version.snapshot.contactoNombre,
      lineas: version.snapshot.lineas,
      observaciones: version.snapshot.observaciones || ''
    },
    status: version.status,
    expiresAt: tok.expiresAt
  });
}

function handlePublicMarkNotArrived_(p) {
  const tok = getJson_('deliveryTokens', p.token);
  if (!tok || tok.revokedAt || (tok.expiresAt && tok.expiresAt < nowIso_())) {
    return out_({ error: 'Enlace no válido o caducado' });
  }
  const version = getJson_('deliveryVersions', tok.deliveryVersionId);
  if (!version || version.isSigned) return out_({ error: 'No se puede modificar' });
  version.status = 'aun_no_ha_llegado';
  setJson_('deliveryVersions', tok.deliveryVersionId, version);
  appendEvent_({
    prealbaranId: tok.prealbaranId,
    deliveryVersionId: tok.deliveryVersionId,
    type: 'not_arrived',
    actor: 'receptor',
    payload: { notes: p.notes || '' }
  });
  return out_({ ok: true, status: 'aun_no_ha_llegado' });
}

function handlePublicSubmitConfirmation_(p) {
  const tok = getJson_('deliveryTokens', p.token);
  if (!tok || tok.revokedAt || (tok.expiresAt && tok.expiresAt < nowIso_())) {
    return out_({ error: 'Enlace no válido o caducado' });
  }
  if (tok.usedAt) return out_({ error: 'Este enlace ya fue utilizado' });
  const version = getJson_('deliveryVersions', tok.deliveryVersionId);
  if (!version || version.isSigned) return out_({ error: 'Esta entrega ya fue confirmada' });
  const allowed = ['recibido_conforme', 'recibido_con_reservas', 'entrega_parcial', 'no_recibido_rechazado'];
  if (allowed.indexOf(p.status) < 0) return out_({ error: 'Estado no válido' });
  const signerName = String(p.signerName || '').trim();
  if (!signerName || signerName.length < 3) return out_({ error: 'Indique el nombre completo de quien confirma' });
  version.status = p.status;
  version.isSigned = true;
  version.confirmationType = 'electronic';
  version.signedAt = nowIso_();
  version.signerName = signerName;
  version.signerRole = p.signerRole || '';
  version.observations = p.observations || '';
  version.lineIssues = p.lineIssues || [];
  version.acceptanceText = p.acceptanceText || 'He revisado el material y confirmo la recepción según el estado indicado.';
  setJson_('deliveryVersions', tok.deliveryVersionId, version);
  tok.usedAt = nowIso_();
  setJson_('deliveryTokens', p.token, tok);
  appendEvent_({
    prealbaranId: tok.prealbaranId,
    deliveryVersionId: tok.deliveryVersionId,
    type: 'confirmed',
    actor: 'receptor',
    payload: { status: p.status, signerName: signerName, observations: version.observations }
  });
  return out_({
    ok: true,
    status: p.status,
    signedAt: version.signedAt,
    message: 'Confirmación registrada. Gracias.'
  });
}
```

---

## Después del parche

1. Guardar el script.
2. **Implementar → Administrar implementaciones → lápiz → Nueva versión → Implementar**.
3. Probar el **panel de tesorería** (abrir, cargar datos, guardar algo menor). Si falla, vuelve a la versión anterior del despliegue.
4. Si el panel va bien, probar `prealbaran.html` → conectar → guardar un prealbarán de prueba.

## Qué NO toca este parche

- Tu `SHEET_ID` y `CLAVE`
- La lógica de add / update / delete de tesorería
- Ninguna pestaña existente de la hoja
- El `index.html` del panel

Solo añade pestañas nuevas cuando se usen por primera vez y las funciones del módulo de entregas.
