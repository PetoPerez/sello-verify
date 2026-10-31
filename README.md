# sello-verify

Verificador independiente de los Paquetes de inspección de **Sello**, el
registro electrónico de jornada (art. 132 fr. XXXIV LFT). Comprueba un paquete
sin conectarse a la red ni a Sello.

## Descarga

En [Releases](../../releases/latest):

| Sistema | Archivo |
|---|---|
| Windows (x86-64) | `sello-verify-windows-amd64.exe` |
| macOS (Apple Silicon) | `sello-verify-darwin-arm64` |
| Linux (x86-64) | `sello-verify-linux-amd64` |

Desde v1.1.1, `SHA256SUMS` viene firmado (`SHA256SUMS.sig`) con la llave de
release de Sello, publicada en [allowed_signers](allowed_signers) (huella
`SHA256:FCcHrvexzpgDNBx2l4kWOpLTWTWZfg4XMsqrXLQ0yOA`, distinta de la llave que
firma los Cierres). Compruebe primero la firma (OpenSSH 8.2 o posterior, incluido
en Linux, macOS y Windows 10+):

```
ssh-keygen -Y verify -f allowed_signers -I sello-release -n sello-verify -s SHA256SUMS.sig < SHA256SUMS
```

Luego la huella del binario; debe coincidir con `SHA256SUMS`:

```
sha256sum -c SHA256SUMS --ignore-missing          # Linux
shasum -a 256 sello-verify-darwin-arm64           # macOS
certutil -hashfile sello-verify-windows-amd64.exe SHA256   # Windows
```

## Uso

```
sello-verify paquete.zip        # o el directorio ya descomprimido
sello-verify --version
```

## Llaves de Sello (desde v1.1.0)

`sello-verify` solo acepta Cierres firmados con las llaves de producción de
Sello que trae embebidas; un paquete firmado con cualquier otra llave falla, y
un paquete sin Cierres también. Las llaves vigentes están en
[llaves-produccion.json](llaves-produccion.json), firmado con la misma llave
de release en `llaves-produccion.json.sig`:

| key_id | Huella SHA-256 de la llave pública |
|---|---|
| `prod-1` | `2882 03ed 1e74 2c5f 72c3 7aa4 9cb2 c3c6 09d7 8745 d9ce 6732 50a9 af55 f52e 8cfc` |

La misma huella aparece en el `LEEME.txt` y en el PDF de cada paquete, y en el
correo diario del Cierre que recibe la empresa. Si no coincide, no confíe en el
paquete.

Para paquetes de pruebas o de staging existe `--keys archivo.json`; el
resultado avisa que se usaron llaves externas y no las oficiales.

v1.1.1 no sigue enlaces simbólicos que salgan del directorio del paquete.
v1.1.0 también verifica el paquete acotado de un solo Trabajador
(`sello-inspection-v2`) y el formato canónico v2 de los registros.

Imprime Eventos revisados, Cierres válidos, sellos de tiempo válidos y cada
falla con su ubicación. Código de salida: 0 verifica, 1 hay fallas, 2 uso
incorrecto. En macOS, la primera vez puede requerir
`xattr -d com.apple.quarantine sello-verify-darwin-arm64`.

## Reimplementarlo

[verificacion.md](verificacion.md) describe el formato del paquete y los seis
pasos de verificación con sus formatos exactos, suficiente para escribir un
verificador propio en cualquier lenguaje.
