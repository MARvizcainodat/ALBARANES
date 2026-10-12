# Módulo de entregas / prealbaranes — Mármoles Hermanos Vizcaíno

Aplicación separada del panel de tesorería. Reutiliza el **mismo `Code.gs`** y la **misma hoja de Google Sheets**.

## Archivos

| Archivo | Uso |
|---------|-----|
| `../Code.gs` | Backend ampliado (colecciones + acciones de entregas + endpoints públicos por token) |
| `prealbaran.html` | Creación táctil de prealbaranes (basada en tu página original). Guarda en hoja + puede crear versión de entrega |
| `entregas.html` | Vista interna de gestión (Agente de Operaciones, modo manual) |
| `confirmar.html` | Página pública móvil del receptor (enlace con token) |
| `albaran_clientes_original.html` | Copia de seguridad de tu página original |
| `LEEME-modulo-entregas.md` | Esta documentación |

### Flujo completo actual

1. **prealbaran.html** — crear el prealbarán (táctil, sin valorar).
2. **Guardar prealbarán** → queda en la hoja (`prealbaranes`) y en el dispositivo.
3. **Guardar y preparar entrega** → guarda + crea versión inmutable + ofrece abrir `entregas.html`.
4. **entregas.html** — Agente de Operaciones prepara el SMS (texto + enlace), tú lo envías y registras.
5. **confirmar.html** — el receptor confirma desde el móvil.
6. Tú pasas el prealbarán a **MNSG** cuando corresponda para el albarán oficial y la factura.

## Colecciones nuevas en la hoja

Se crean solas al usarse:

- `prealbaranes` — datos del prealbarán (antes de pasar a MNSG)
- `deliveryVersions` — versión inmutable mostrada al receptor
- `deliveryTokens` — tokens de enlace (opacos, caducables, revocables)
- `deliveryEvents` — historial de eventos (no se sobrescribe)
- `smsOutbox` — cola / registro de SMS (proveedor `manual` al inicio)
- `deliveryAttachments` — escaneados de papel u otros adjuntos

## Estados de entrega

- `pendiente_envio`
- `sms_preparado`
- `sms_enviado`
- `aun_no_ha_llegado`
- `recibido_conforme`
- `recibido_con_reservas`
- `entrega_parcial`
- `no_recibido_rechazado`
- `pendiente_papel`
- `papel_incorporado`

**Nunca** se marca como conforme por falta de respuesta, apertura del enlace, SMS enviado o plazo vencido.

## Agente de Operaciones (modo manual)

Rol operativo bajo tu orden:

1. **Preparar SMS** — genera token + texto + enlace; muestra para copiar o abrir borrador.
2. **Registrar envío manual** — anota quién lo envió y cuándo (desde el móvil de empresa).
3. **Gestión en papel** — deja constancia de firma en papel y adjunta el escaneado (no se trata como firma electrónica).
4. **Revocar / regenerar enlace** — disponible vía acciones `revokeToken` / `regenerateToken` en el backend.

## Configuración para arrancar en modo manual

1. En `Code.gs`: poner el `SHEET_ID` real y una `CLAVE` larga (≥16 caracteres).
2. Desplegar Apps Script: **Implementar → Nueva implementación → Aplicación web**  
   - Ejecutar como: **yo**  
   - Acceso: **Cualquier persona**  
   - Copiar la URL `/exec`.
3. Subir a GitHub Pages (o el hosting que uses):
   - `entregas.html`
   - `confirmar.html`
4. Abrir `entregas.html`, indicar:
   - URL del servicio (`/exec`)
   - Clave
   - URL pública base del módulo (para construir el enlace del SMS), p. ej. `https://tuusuario.github.io/repo/modulo-entregas`
5. Crear al menos un registro de prueba en `prealbaranes` (vía `add` o desde la futura integración de la página de creación) y llamar a `createDeliveryVersion`.

### Ejemplo de prealbarán mínimo (JSON)

```json
{
  "ref": "00-000001/2026",
  "clienteNombre": "Cliente de prueba",
  "obraNombre": "Obra ejemplo",
  "direccionEntrega": "Calle Ejemplo 1, Málaga",
  "fechaPrevista": "2026-10-12",
  "contactoNombre": "Encargado",
  "contactoTelefono": "600000000",
  "lineas": [
    { "descripcion": "Mármol blanco 2 cm", "referencia": "MB-20", "unidad": "m²", "cantidadPrevista": 12.5 }
  ],
  "createdAt": "2026-10-11T18:00:00"
}
```

## Datos que faltan para el envío automático de SMS

El proveedor está desacoplado (`provider: "manual"`). Para conectar una API:

- Confirmar con **Orange** (u otro operador) si la línea de SMS ilimitados permite envío por API / pasarela, y qué credenciales, endpoint, formato y costes aplican.
- Implementar un adaptador que cumpla la misma interfaz interna (preparar → enviar → registrar estados: pendiente, preparado, enviado, entregado, fallido).
- Gestionar códigos OTP de un solo uso **sin** guardarlos en registros de aplicación ni mostrarlos a usuarios no autorizados.
- Programación por fecha prevista, cancelaciones y duplicados.

Hasta entonces el flujo es 100 % manual con registro de quién envió y cuándo.

## Seguridad y privacidad (aplicado)

- Acciones internas: clave del ERP.
- Acciones del receptor: solo token criptográfico aleatorio (sin datos del albarán en la URL).
- Tokens con caducidad configurable (`DELIVERY_CFG.tokenTtlHours`, por defecto 72 h), revocables y de un solo uso tras confirmación.
- Firma = nombre completo + aceptación expresa. **No** es firma cualificada. No se captura biometría del trazo.
- Historial de eventos append-only.
- Papel escaneado se marca explícitamente como `confirmationType: "paper"`, nunca como firma electrónica.
- Retención: documentada como configuración a definir por la empresa (no se inventan plazos legales).

## Limitaciones conocidas (esqueleto)

- Generación de PDF final firmado aún no implementada (se conserva el snapshot + eventos; el PDF se puede añadir en la siguiente iteración reutilizando el formato de impresión ya existente).
- La página de creación de prealbaranes (`albaran_clientes`) sigue en localStorage; la integración a `prealbaranes` es el siguiente paso.
- Adjuntos grandes no caben en celda de Sheets (~50k caracteres); para producción conviene Drive + enlace.
- La URL del servicio en `confirmar.html` se pide en la primera prueba (localStorage del dispositivo). En despliegue real conviene fijarla en el HTML del hosting o usar un proxy.
- No hay OTP por SMS todavía (canal no conectado).

## Pasos de despliegue recomendados

1. Actualizar `Code.gs` y crear nueva versión del despliegue web.
2. Publicar `entregas.html` + `confirmar.html`.
3. Probar con un prealbarán de prueba → crear versión → preparar SMS → copiar enlace → abrir en móvil → confirmar.
4. Verificar eventos en la hoja (`deliveryEvents`).
5. Probar flujo de papel.
6. Solo entonces conectar permisos reales de usuarios, secretos y política de retención.

**El módulo no se declara listo para producción** hasta configurar permisos, secretos, política de retención, canal de SMS y validar el flujo completo con entregas de prueba.
