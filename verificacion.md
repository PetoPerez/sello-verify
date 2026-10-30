# Verificación independiente del Paquete de inspección

Este documento basta para comprobar un Paquete de inspección de Sello sin
usar Sello: con `sello-verify` o con un verificador propio en cualquier
lenguaje. Formato del paquete: `sello-inspection-v1`.

## Uso de sello-verify

```
sello-verify paquete.zip        # o el directorio ya descomprimido
```

No se conecta a la red ni a Sello. Imprime Eventos revisados, Cierres
válidos, sellos de tiempo válidos y cada falla con el archivo (y el `seq`)
donde ocurrió. Código de salida: 0 todo verifica, 1 hay fallas, 2 uso
incorrecto.

Binarios para Linux, macOS y Windows: `make verify-dist` (salida en `bin/`).

## Contenido del paquete

| Archivo | Contenido |
|---|---|
| `manifiesto.json` | `format`, datos de la Empresa y el periodo, y `archivos`: `ruta`, `sha256` (hex) y `bytes` de cada archivo menos él mismo |
| `json/eventos.json` | arreglo de Eventos desde `seq` 1 hasta el corte del periodo |
| `cierres/AAAA-MM-DD.json` | Cierre diario firmado |
| `cierres/AAAA-MM-DD.tsr` | respuesta RFC 3161 (TimeStampResp, DER) sobre ese Cierre |
| `llaves/publicas.json` | `llaves`: `key_id` y `public_key_base64` (Ed25519, 32 bytes) |
| `bitacora/administracion.json` | `registros` de la bitácora de administración del periodo |
| `csv/checadas.csv`, `convenios/`, `verificacion.json`, `LEEME.txt` | lectura humana; se comprueban solo por su huella |

Toda cadena es UTF-8. Los hashes son SHA-256 en hexadecimal minúsculo. Las
horas van en UTC con formato `2006-01-02T15:04:05.000000Z` (microsegundos).

## Paso 1. Manifiesto

Para cada renglón de `archivos`, el SHA-256 y el tamaño del archivo deben
coincidir. Un archivo listado que falta, o un archivo presente que no está
listado (salvo `manifiesto.json`), es falla.

El manifiesto no está firmado: solo detecta alteraciones accidentales o
torpes. La integridad la dan los pasos 2 a 5, que se apoyan en la firma de
los Cierres y en el sello de tiempo de un tercero.

## Paso 2. Cadena de Eventos (formato canónico v1)

Cada Evento de `eventos.json` trae `tenant_id`, `seq`, `event_type`,
`employee_id`, `actor_user_id` (puede faltar: cadena vacía), `server_ts`,
`payload`, `prev_hash` y `hash`. `payload` es el texto exacto que se hasheó;
no se debe reinterpretar ni reformatear.

```
canónico = "v1|" + tenant_id + "|" + seq + "|" + event_type + "|" + employee_id
         + "|" + actor_user_id + "|" + server_ts + "|" + prev_hash + "|" + payload
hash     = SHA-256(canónico)
```

Recorriendo en orden, se exige: `seq` consecutivo desde 1; `prev_hash` del
primer Evento igual a 64 ceros y, en los demás, igual al `hash` del anterior;
y `hash` igual al recalculado.

## Paso 3. Firma de cada Cierre (seal-v1)

El JSON del Cierre tiene, en este orden y sin espacios: `format`,
`tenant_id`, `day`, `cutoff_utc`, `last_seq`, `last_hash`, `audit_last_id`,
`audit_day_hash`, `signed_at`, `key_id`, `signature` (hashes y firma en hex).

```
firmado = "seal-v1|" + tenant_id + "|" + day + "|" + cutoff_utc + "|" + last_seq
        + "|" + last_hash + "|" + audit_last_id + "|" + audit_day_hash + "|" + signed_at
```

La firma Ed25519 `signature` debe verificar `firmado` con la llave de
`key_id` en `llaves/publicas.json` (las llaves retiradas siguen ahí).

`day` es la fecha en UTC−6 fijo; el día D cierra a las 00:00 de D+1 en
UTC−6 (06:00 UTC).

## Paso 4. Enlace del Cierre con la cadena

`last_hash` debe ser el `hash` del Evento con `seq = last_seq` en
`eventos.json`. Si `last_seq` es 0 (no había Eventos antes del corte),
`last_hash` debe ser 64 ceros.

Esto ata la cadena completa hasta ese punto a una firma: alterar cualquier
Evento anterior cambia todos los hashes siguientes y rompe el enlace.

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

## Qué acredita y qué no

Un paquete que pasa los seis pasos acredita que los registros no cambiaron
desde que se firmó y selló en tiempo cada Cierre. No acredita la veracidad de
la jornada ni la identidad de las personas: eso depende de las credenciales
usadas en cada registro.
