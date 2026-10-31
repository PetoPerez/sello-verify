# Verificación independiente del Paquete de inspección

Este documento basta para comprobar un Paquete de inspección de Sello sin
usar Sello: con `sello-verify` o con un verificador propio en cualquier
lenguaje. Formatos del paquete: `sello-inspection-v1` (completo de la Empresa)
y `sello-inspection-v2` (acotado a un Trabajador). Versión del verificador:
`v1.1.1`.

## Uso de sello-verify

```
sello-verify <paquete.zip|directorio>           # ancla oficial de producción
sello-verify --keys ancla.json <paquete.zip>    # ancla externa (staging/pruebas)
sello-verify --version                           # imprime la versión y termina
```

No se conecta a la red ni a Sello. Imprime Eventos revisados, Cierres
válidos, sellos de tiempo válidos, la huella de cada llave usada
(`llave usada: <key_id> <huella>`) y cada falla con el archivo (y el `seq`)
donde ocurrió. Código de salida: 0 todo verifica, 1 hay fallas, 2 uso
incorrecto.

Con `--keys`, el binario usa el ancla del archivo en vez de la embebida y la
salida lo avisa: `AVISO: se verificó con llaves EXTERNAS …, no con las
oficiales de producción`. Es solo para paquetes de staging o de prueba; un
paquete real debe verificar sin `--keys`.

Binarios para Linux, macOS y Windows: `make verify-dist` (salida en `bin/`).

## Contenido del paquete

| Archivo | Contenido |
|---|---|
| `manifiesto.json` | `format`, datos de la Empresa y el periodo, y `archivos`: `ruta`, `sha256` (hex) y `bytes` de cada archivo menos él mismo |
| `json/eventos.json` | arreglo de Eventos desde `seq` 1 hasta el corte del periodo (ver abajo el formato según la versión) |
| `cierres/AAAA-MM-DD.json` | Cierre diario firmado |
| `cierres/AAAA-MM-DD.tsr` | respuesta RFC 3161 (TimeStampResp, DER) sobre ese Cierre |
| `llaves/publicas.json` | `llaves`: `key_id` y `public_key_base64` (Ed25519, 32 bytes). Cada llave del paquete solo se usa si coincide con una del **ancla** de Sello (ver el paso 0) |
| `bitacora/administracion.json` | `registros` de la bitácora de administración del periodo. **En `sello-inspection-v2` no va** (ver abajo) |
| `csv/checadas.csv`, `inspeccion.pdf`, `convenios/`, `verificacion.json`, `LEEME.txt` | lectura humana; se comprueban solo por su huella. `inspeccion.pdf` es la representación legible (portada, Checadas por Trabajador, Convenios y Cierres); la evidencia verificable es `json/eventos.json` y los Cierres |

Toda cadena es UTF-8. Los hashes son SHA-256 en hexadecimal minúsculo. Las
horas van en UTC con formato `2006-01-02T15:04:05.000000Z` (microsegundos).

### Paquete completo (v1) y paquete acotado (v2)

- **`sello-inspection-v1`** es el paquete completo de la Empresa: todos los
  Eventos van **completos** (con `tenant_id`, `seq`, `event_type`,
  `employee_id`, `actor_user_id`, `server_ts`, `payload`, `prev_hash`, `hash` y
  `canon`) y se incluye la bitácora.
- **`sello-inspection-v2`** es el paquete acotado a **un** Trabajador (H2 de la
  auditoría del 30 oct 2026): sus Eventos van completos y, de los Eventos de
  los demás, solo un **eslabón reducido** con `seq`, `prev_hash`, `inner` y
  `canon`, sin tipo, persona, hora ni ubicación. Con el eslabón se recomputa el
  hash ajeno (ver el paso 2) y se enlaza la cadena hasta la cabeza que firmó el
  Cierre, sin exponer datos de los demás. Como la bitácora completa no va en
  este paquete, `audit_day_hash` no se puede recomputar aquí: el verificador lo
  reporta como **no verificable** y no es una falla.
- Un paquete v2 solo puede traer eslabones (`inner`) de Eventos en formato
  canónico **v2**. Un Evento ajeno en formato v1 no se puede exportar sin
  revelar su `payload`, así que ese paquete acotado no se genera (Sello responde
  `409`); el paquete completo de la Empresa sigue disponible.

## Paso 0. Ancla de confianza de las llaves

`sello-verify` no confía en las llaves que vienen dentro del paquete: el
binario oficial embebe un **ancla** con las llaves públicas de producción de
Sello y usa una llave de `llaves/publicas.json` solo si coincide con una del
ancla. Si no coincide, falla `llave_no_anclada` y esa llave no verifica ningún
Cierre. Así, un paquete fabricado con la llave de un tercero —aunque traiga
tokens de una TSA de verdad, que sella cualquier huella— no verifica.

El archivo del ancla (formato `sello-seal-keys-anchor-v1`) es **de solo
agregar**:

```json
{
  "format": "sello-seal-keys-anchor-v1",
  "entorno": "produccion",
  "llaves": [
    {
      "key_id": "sello-prod-2026-a",
      "public_key_base64": "<llave Ed25519 de 32 bytes en base64>",
      "active_from": "2026-10-30T00:00:00.000000Z",
      "retired_at": "2027-01-01T00:00:00.000000Z",
      "revoked_at": "2027-02-01T00:00:00.000000Z",
      "notas": "opcional"
    }
  ]
}
```

- `active_from` y `retired_at` marcan la vigencia. Una llave **retirada** sigue
  verificando los Cierres que firmó (las llaves viejas no se borran).
- `revoked_at` se agrega **solo** si una llave privada se compromete: un Cierre
  firmado **después** de `revoked_at` falla; los anteriores solo se sostienen
  con los testigos fuera de Sello (el token RFC 3161 y el correo que el cliente
  ya tiene).
- El binario oficial lleva **solo las llaves de producción**; las de staging no
  se publican: si el binario confiara también en staging, un compromiso de
  staging —con menos custodia— fabricaría paquetes que se harían pasar por
  producción. El humo de staging se verifica con `--keys`.
- **Huella de la llave:** SHA-256 del `public_key` (32 bytes) en hexadecimal
  minúsculo, agrupado de 4 en 4, junto al `key_id` (p. ej. `9f2a 1c04 …`). Va en
  `LEEME.txt`, en la portada de `inspeccion.pdf` y en el correo diario del
  Cierre, para cotejarla por un canal fuera de Sello.

## Paso 1. Manifiesto

Para cada renglón de `archivos`, el SHA-256 y el tamaño del archivo deben
coincidir. Un archivo listado que falta, o un archivo presente que no está
listado (salvo `manifiesto.json`), es falla.

El manifiesto no está firmado: solo detecta alteraciones accidentales o
torpes. La integridad la dan los pasos 2 a 5, que se apoyan en la firma de
los Cierres y en el sello de tiempo de un tercero.

## Paso 2. Cadena de Eventos (formato canónico v1 o v2)

Cada Evento de `eventos.json` trae su versión en `canon` (1 = v1, 2 = v2). Un
Evento **completo** trae `tenant_id`, `seq`, `event_type`, `employee_id`,
`actor_user_id` (puede faltar: cadena vacía), `server_ts`, `payload`,
`prev_hash`, `hash` y `canon`. `payload` es el texto exacto que se hasheó; no
se debe reinterpretar ni reformatear.

Formato canónico **v1** (`canon=1`, Eventos anteriores a la migración `00034`):

```
canónico = "v1|" + tenant_id + "|" + seq + "|" + event_type + "|" + employee_id
         + "|" + actor_user_id + "|" + server_ts + "|" + prev_hash + "|" + payload
hash     = SHA-256(canónico)
```

Formato canónico **v2** (`canon=2`, de dos niveles):

```
inner = SHA-256(event_type + "|" + employee_id + "|" + actor_user_id + "|"
                + server_ts + "|" + payload)
hash  = SHA-256("v2|" + tenant_id + "|" + seq + "|" + prev_hash + "|" + inner)
```

donde `prev_hash` e `inner` van en hexadecimal minúsculo. Un **eslabón
reducido** (elemento sin `event_type`, con `seq`, `prev_hash`, `inner` y
`canon=2`) es un Evento ajeno de un paquete acotado v2: su `hash` se recalcula
igual que en v2 (segundo renglón, tomando el `tenant_id` del manifiesto), sin
necesitar su `payload`.

Recorriendo en orden, se exige: `seq` consecutivo desde 1; `prev_hash` del
primer Evento igual a 64 ceros y, en los demás, igual al `hash` del anterior; y
`hash` igual al recalculado (v1, v2 o eslabón).

## Paso 3. Firma de cada Cierre (seal-v1)

El JSON del Cierre tiene, en este orden y sin espacios: `format`,
`tenant_id`, `day`, `cutoff_utc`, `last_seq`, `last_hash`, `audit_last_id`,
`audit_day_hash`, `signed_at`, `key_id`, `signature` (hashes y firma en hex).

```
firmado = "seal-v1|" + tenant_id + "|" + day + "|" + cutoff_utc + "|" + last_seq
        + "|" + last_hash + "|" + audit_last_id + "|" + audit_day_hash + "|" + signed_at
```

La firma Ed25519 `signature` debe verificar `firmado` con la llave de
`key_id` en `llaves/publicas.json` (las llaves retiradas siguen ahí); esa llave
debe estar **anclada** (paso 0) y no estar revocada en el instante `signed_at`.

`day` es la fecha en UTC−6 fijo; el día D cierra a las 00:00 de D+1 en
UTC−6 (06:00 UTC).

## Paso 4. Enlace del Cierre con la cadena

`last_hash` debe ser el `hash` del Evento con `seq = last_seq` en
`eventos.json`. Si `last_seq` es 0 (no había Eventos antes del corte),
`last_hash` debe ser 64 ceros.

Esto ata la cadena completa hasta ese punto a una firma: alterar cualquier
Evento anterior cambia todos los hashes siguientes y rompe el enlace.

Se exige además que el paquete traiga **al menos un Cierre** (`cierre_ausente`
si no hay ninguno) y que el Cierre de **mayor `cutoff_utc`** cubra la **cabeza
exportada**: su `last_seq` y `last_hash` deben ser los del último Evento de
`eventos.json` (o la cabeza génesis, `last_seq = 0`, si el paquete no trae
Eventos). Si no, falla `cierre_no_cubre_periodo`: un paquete no puede terminar
con Eventos que ningún Cierre firmó.

## Paso 5. Sello de tiempo RFC 3161

El `.tsr` de cada Cierre debe:

1. tener estado concedido (granted);
2. usar SHA-256 y su huella (`messageImprint`) debe ser el SHA-256 de los
   bytes exactos de `cierres/AAAA-MM-DD.json`;
3. estar firmado por un certificado con uso extendido *timeStamping* que
   encadene a una raíz de confianza, vigente a la hora del sello.

`sello-verify` confía solo en las raíces de las TSA que usa Sello (FreeTSA y DigiCert,
de respaldo), incluidas en el binario. Con OpenSSL:

```
openssl ts -verify -in cierres/D.tsr -data cierres/D.json -CAfile raiz-tsa.pem
```

## Paso 6. Bitácora de administración (audit_day_hash)

En un paquete **v2** (acotado) `bitacora/administracion.json` no va: este paso
no se puede hacer y `audit_day_hash` queda como **no verificable** en ese
paquete (la firma del Cierre sí cubre su valor). En un paquete **v1** se
comprueba como sigue.

Para cada Cierre, tome los registros de `bitacora/administracion.json` con
`created_at` desde el inicio de `day` (00:00 UTC−6) hasta antes de
`cutoff_utc`, ordenados por `id`. Cada registro se vuelve una línea:

```
id|principal_kind|actor_user_id|action|target|details|created_at
```

`details` es el texto JSON tal cual viene en el archivo (una cadena; no se
reformatea) y `actor_user_id` vacío si falta. Las líneas se unen con `\n`
(sin salto final). `audit_day_hash` del Cierre debe ser el SHA-256 de ese
texto; sin registros, el SHA-256 de la cadena vacía.

## Archivo de llaves firmado del repositorio público

Además del ancla embebida en el binario, el repositorio público
`github.com/PetoPerez/sello-verify` publica el **mismo** ancla como archivo
firmado, para dejar una historia auditable y poder cotejar que el ancla
embebida es la esperada.

- **Formato:** el mismo `sello-seal-keys-anchor-v1`, publicado como
  `llaves-produccion.json`.
- **Firma:** firma SSH (`ssh-keygen -Y sign`, namespace `sello-verify`) en `llaves-produccion.json.sig`, calculada
  con la **llave de release** de Sello (distinta de la llave que firma los
  Cierres) sobre los bytes exactos del JSON canónico del ancla. `SHA256SUMS` de
  los binarios del release se firma con esa misma llave de release.

Procedimiento:

1. **Operador (producción):** `api seal-keys export --entorno produccion --out
   llaves-produccion.json` imprime las llaves de `seal_keys` en el formato del ancla
   (con `active_from`/`retired_at`; `revoked_at` se agrega a mano si una llave
   se comprometió) y lo entrega al mantenedor del verificador.
2. **Mantenedor:** revisa el archivo, lo firma con la llave de release de Sello
   y publica `llaves-produccion.json` y `llaves-produccion.json.sig` en la raíz del
   repositorio público.
3. **Mantenedor del binario:** copia el contenido a
   `internal/verify/llaves-ancla.json`, sube la versión de `sello-verify` y
   recompila con `make verify-dist`.
4. **Cualquiera:** descarga el binario, comprueba la firma y el `SHA256SUMS` y
   coteja la huella de la llave de un paquete contra el ancla firmada.

### Firma del release

La llave de release es Ed25519 en formato SSH (`ssh-keygen`, sin herramientas
extra), distinta de la que firma los Cierres. Su llave pública está en
`allowed_signers` del repositorio público; huella
`SHA256:FCcHrvexzpgDNBx2l4kWOpLTWTWZfg4XMsqrXLQ0yOA`. Se firman `SHA256SUMS`
(en cada release) y `llaves-produccion.json` (en la raíz del repositorio):

```
ssh-keygen -Y verify -f allowed_signers -I sello-release -n sello-verify \
  -s SHA256SUMS.sig < SHA256SUMS && sha256sum -c SHA256SUMS
ssh-keygen -Y verify -f allowed_signers -I sello-release -n sello-verify \
  -s llaves-produccion.json.sig < llaves-produccion.json
```

El mantenedor firma con `make verify-sign` (llave en
`~/.config/sello/release-key`, o `RELEASE_KEY=...`).

Hoy el ancla embebida (`internal/verify/llaves-ancla.json`) está **vacía** —no
se ha generado todavía la llave de producción de Sello—: hasta que el Operador
corra `api seal-keys export` y se actualice el ancla, el binario oficial no
verifica ningún paquete. Es el estado seguro por omisión.

## Qué acredita y qué no

Un paquete que pasa el ancla (paso 0) y los pasos 1 a 6 acredita que los
registros no cambiaron desde que se firmó y selló en tiempo cada Cierre. No
acredita la veracidad de la jornada ni la identidad de las personas: eso
depende de las credenciales usadas en cada registro.

En un paquete **v2** (acotado) el alcance es el de un Trabajador: prueba que
**sus** registros no cambiaron y que están anclados a la cabeza firmada del
Cierre, sin exponer datos de los demás; como los eslabones ajenos se recomputan
desde `inner`, alterar o borrar un Evento del Trabajador —o reenlazar el
eslabón siguiente— rompe la verificación. No prueba el contenido de los Eventos
ajenos (eso lo hace el paquete completo de la Empresa) ni que los registros
ajenos digan la verdad.
