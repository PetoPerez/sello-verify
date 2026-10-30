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

Compruebe la huella antes de usarlo; debe coincidir con `SHA256SUMS`:

```
sha256sum -c SHA256SUMS --ignore-missing          # Linux
shasum -a 256 sello-verify-darwin-arm64           # macOS
certutil -hashfile sello-verify-windows-amd64.exe SHA256   # Windows
```

## Uso

```
sello-verify paquete.zip        # o el directorio ya descomprimido
```

Imprime Eventos revisados, Cierres válidos, sellos de tiempo válidos y cada
falla con su ubicación. Código de salida: 0 verifica, 1 hay fallas, 2 uso
incorrecto. En macOS, la primera vez puede requerir
`xattr -d com.apple.quarantine sello-verify-darwin-arm64`.

## Reimplementarlo

[verificacion.md](verificacion.md) describe el formato del paquete y los seis
pasos de verificación con sus formatos exactos, suficiente para escribir un
verificador propio en cualquier lenguaje.
