# Certificados Digitales: Uso, Gestión y Firma

Este curso ofrece una visión integral sobre el uso, administración y firma de certificados digitales en entornos Windows, Linux y nube. A lo largo del curso, los participantes comprenderán el funcionamiento de la PKI, la gestión de llaves y certificados X.509, el uso de herramientas como OpenSSL y PowerShell, así como su integración en aplicaciones, APIs y servicios cloud.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

### Entorno canónico del curso

Todos los capítulos usan el entorno `cert-digital-lab` y conservan esta estructura:

```text
cert-digital-lab/
├── certs/
├── private/
├── csr/
├── exports/
├── signed/
├── scripts/
├── config/
└── evidence/
```

En Bash, `<LAB_ROOT>` equivale a `$HOME/cert-digital-lab`. En PowerShell equivale a
`$HOME\cert-digital-lab`. Los nombres compartidos entre capítulos son `server.key`,
`server.csr`, `server.crt`, `certificate.pfx`, `document.txt` y `document.sig`.
Los dominios reservados para el laboratorio son `service.local` y `api.service.local`.

### Seguridad y compatibilidad obligatorias

- Nunca versiones `private/`, `exports/`, `.key`, `.pfx`, `.p12`, `.env` ni secretos.
- Usa permisos `600` para llaves en Linux/macOS. En Windows restringe la ACL al usuario actual; no confíes en atributos de archivo como sustituto de una ACL.
- No instales certificados leaf en almacenes raíz y no desactives la validación TLS.
- Bash usa `/` y continuación `\`; PowerShell usa `Join-Path` y continuación con acento grave. No copies comandos entre shells sin adaptar comillas y escape.
- `LocalMachine`, MMC y stores globales requieren elevación controlada; `CurrentUser` no debe ejecutarse elevado por defecto.
- OpenSSL de macOS puede diferir del de Linux. Verifica `openssl version` y utiliza los comandos alternativos documentados cuando `grep -P`, `stat -c` o `date -d` no existan.
- Azure/AWS se trabajan con cuentas de laboratorio, mínimo privilegio y recursos no productivos. Key Vault y ACM no son servicios equivalentes.

#### Equivalencias operativas

| Operación | Bash (Linux/macOS) | PowerShell (Windows) |
|---|---|---|
| Definir raíz | `export LAB_ROOT="$HOME/cert-digital-lab"` | `$LabRoot = Join-Path $HOME "cert-digital-lab"` |
| Crear carpeta | `mkdir -p "$LAB_ROOT/evidence"` | `New-Item -ItemType Directory -Path (Join-Path $LabRoot "evidence") -Force` |
| Unir ruta | `"$LAB_ROOT/certs/server.crt"` | `Join-Path $LabRoot "certs\server.crt"` |
| Comprobar archivo | `test -f "$CERTIFICATE_PATH"` | `Test-Path -LiteralPath $CertificatePath` |
| Ver SHA-256 | `openssl sha256 < archivo` | `Get-FileHash -Algorithm SHA256 -LiteralPath archivo` |
| Restringir llave | `chmod 600 "$PRIVATE_KEY_PATH"` | `icacls $PrivateKeyPath /inheritance:r /grant:r "$($env:USERNAME):(R,W)"` |
| Eliminar archivo exacto | `rm -- "$OUTPUT_PATH"` | `Remove-Item -LiteralPath $OutputPath` |

Antes de eliminar, resuelve y comprueba que la ruta permanezca dentro de `<LAB_ROOT>`. No uses globbing ni eliminación recursiva para la limpieza normal.

#### Diferencias por plataforma

- Linux: `stat -c`, `date -d` y `update-ca-certificates` no son universales fuera de GNU/Linux.
- macOS: usa `stat -f`, `date -j -f` y el Keychain cuando corresponda; el OpenSSL del curso debe instalarse y verificarse explícitamente.
- Windows: `certmgr.msc` administra `CurrentUser`; MMC/certlm y `LocalMachine` requieren privilegios. Las llaves del Certificate Store se protegen mediante ACL/CNG/CSP, no con `chmod`.

## Lista de laboratorios

### Capítulo 1

- [Analizar un certificado HTTPS real e identificar emisor, vigencia, algoritmos y cadena de confianza.](Capitulo01/README.md#analizar-un-certificado-https-real-e-identificar-emisor-vigencia-algoritmos-y-cadena-de-confianza)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración esencial: 25 min

### Capítulo 2

- [Crear un certificado autofirmado para un servicio interno y validarlo con OpenSSL.](Capitulo02/README.md#crear-un-certificado-autofirmado-para-un-servicio-interno-y-validarlo-con-openssl)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración esencial: 20 min

### Capítulo 3

- [Crear, instalar y exportar un certificado desde Windows para uso local.](Capitulo03/README.md#crear-instalar-y-exportar-un-certificado-desde-windows-para-uso-local)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración esencial: 20 min

### Capítulo 4

- [Firmar un archivo y validar su integridad; después habilitar HTTPS en un servicio de prueba.](Capitulo04/README.md#firmar-un-archivo-y-validar-su-integridad-después-habilitar-https-en-un-servicio-de-prueba)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración esencial: 20 min

### Capítulo 5

- [Diagnosticar un fallo TLS simulado y proponer una ruta de corrección en nube u on-prem.](Capitulo05/README.md#diagnosticar-un-fallo-tls-simulado-y-proponer-una-ruta-de-corrección-en-nube-u-on-prem)
  - Descripción: Actividad práctica guiada basada estrictamente en el contenido del módulo.
  - Duración esencial: 30 min


