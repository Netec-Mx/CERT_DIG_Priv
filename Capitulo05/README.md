# Diagnosticar un fallo TLS simulado y proponer una ruta de corrección en nube u on-prem.

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 30 minutos                                   |
| **Complejidad**  | Alta                                         |
| **Nivel Bloom**  | Crear (Síntesis y evaluación aplicada)       |
| **Modalidad**    | Individual / Parejas                         |
| **Entorno**      | Linux (o WSL2) + referencias conceptuales cloud |

---

## Descripción General

En esta práctica el estudiante configurará deliberadamente tres escenarios de fallo TLS en un servidor HTTPS local, los diagnosticará con herramientas de línea de comandos (`openssl s_client`, `curl`, `openssl x509`) y completará una **ficha de diagnóstico** para cada escenario. A continuación, propondrá rutas de corrección concretas tanto para entornos on-premise como para Azure Key Vault y AWS Certificate Manager, aplicando los conceptos de gestión de ciclo de vida vistos en la lección 5.1. La práctica culmina con una revisión de estrategias de monitoreo de expiración y buenas prácticas de protección de llaves privadas.

---

## Objetivos de Aprendizaje

- [ ] Identificar y diagnosticar los tres errores TLS más comunes: certificado expirado, hostname mismatch y cadena de confianza incompleta, utilizando `openssl s_client` y `curl -v`.
- [ ] Completar una ficha de diagnóstico estructurada (síntoma → comando → causa raíz → ruta de corrección) para cada escenario de fallo.
- [ ] Proponer rutas de corrección diferenciadas entre entornos on-premise (OpenSSL/nginx) y cloud (Azure Key Vault con políticas de rotación y AWS ACM con renovación automática).
- [ ] Aplicar buenas prácticas de protección de llaves privadas (`chmod 600`) y describir estrategias de monitoreo de expiración con scripts, Azure Monitor y AWS CloudWatch.

---

## Prerrequisitos

### Conocimientos previos
- Haber completado las Prácticas 1, 2 y 4 del curso, o tener experiencia equivalente con OpenSSL y certificados X.509.
- Comprensión de la estructura de un certificado X.509: Subject, SAN, fechas de validez, cadena de confianza.
- Familiaridad con el handshake TLS y el concepto de cadena de certificados (root → intermedio → entidad final).
- Lectura básica de la documentación de Azure Key Vault Certificates y AWS ACM (suficiente para las secciones conceptuales).

### Acceso y herramientas requeridas
- OpenSSL 1.1.1 o superior (verificar con `openssl version`).
- Python 3.8 o superior con módulo `ssl` disponible.
- `curl` 7.x o superior.
- Terminal de Linux (o WSL2 en Windows).
- Editor de texto (nano, vim, VS Code).
- Directorio de trabajo de prácticas anteriores: `~/labs/certs/` con la CA local y llave privada generadas en la Práctica 2.

> **Nota:** Si no completaste la Práctica 2, el Paso 1 de esta práctica incluye comandos para generar una CA local mínima desde cero.

---

## Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso       | Mínimo          | Recomendado     |
|---------------|-----------------|-----------------|
| CPU           | 2 núcleos x86-64| 4 núcleos       |
| RAM           | 4 GB disponibles| 8 GB            |
| Almacenamiento| 500 MB libres   | 1 GB libres     |
| Red           | Loopback (127.0.0.1) — no requiere Internet para los 3 escenarios principales | Acceso a Internet para sección cloud conceptual |

### Software

| Herramienta   | Versión mínima | Verificación                      |
|---------------|----------------|-----------------------------------|
| OpenSSL       | 1.1.1          | `openssl version`                 |
| Python 3      | 3.8            | `python3 --version`               |
| curl          | 7.x            | `curl --version`                  |
| bash          | 5.x (o 4.x)    | `bash --version`                  |

### Preparación del directorio de trabajo

```bash
# Crear directorio de trabajo para esta práctica
mkdir -p ~/labs/certs/practica5
cd ~/labs/certs/practica5

# Verificar herramientas disponibles
openssl version
python3 --version
curl --version | head -1
```

> **Importante sobre seguridad:** Todas las llaves privadas generadas en esta práctica deben tener permisos `chmod 600`. Este hábito es obligatorio en entornos productivos y se verificará en cada paso.

---

## Procedimiento Paso a Paso

---

### Paso 0 — Preparar la Infraestructura PKI Local (CA Mínima)

**Objetivo:** Crear una CA local autofirmada que firme los certificados de los tres escenarios de fallo. Si ya cuentas con la CA de la Práctica 2, puedes reutilizarla; de lo contrario, ejecuta los comandos a continuación.

#### Instrucciones

```bash
cd ~/labs/certs/practica5

# 0.1 Generar la llave privada de la CA
openssl genrsa -out ca.key 4096
chmod 600 ca.key

# 0.2 Generar el certificado autofirmado de la CA (válido 10 años para el laboratorio)
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt \
  -subj "/C=MX/ST=CDMX/O=LabCA/CN=Lab Root CA"

# 0.3 Verificar el certificado de la CA
openssl x509 -noout -subject -issuer -dates -in ca.crt
```

**Salida esperada:**
```
subject=C = MX, ST = CDMX, O = LabCA, CN = Lab Root CA
issuer=C = MX, ST = CDMX, O = LabCA, CN = Lab Root CA
notBefore=<fecha actual>
notAfter=<fecha actual + 10 años>
```

**Verificación:**
```bash
ls -la ca.key ca.crt
# ca.key debe mostrar -rw------- (600)
```

---

### Paso 1 — Escenario 1: Certificado Expirado

**Objetivo:** Configurar un servidor HTTPS con un certificado cuya fecha de expiración ya pasó, diagnosticar el error `certificate has expired` y proponer la ruta de corrección.

#### 1.1 Generar el certificado expirado

```bash
cd ~/labs/certs/practica5

# Generar llave privada del servidor
openssl genrsa -out server-expired.key 2048
chmod 600 server-expired.key

# Generar CSR
openssl req -new \
  -key server-expired.key \
  -out server-expired.csr \
  -subj "/C=MX/ST=CDMX/O=EmpresaLab/CN=servicio.empresa.local"

# Firmar con la CA usando -days -1 para que el certificado nazca ya expirado
# NOTA: algunos sistemas requieren ajustar la fecha; usa -days 0 si -days -1 falla
openssl x509 -req \
  -in server-expired.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server-expired.crt \
  -days -1 \
  -sha256

# Verificar que el certificado esté expirado
openssl x509 -noout -dates -in server-expired.crt
```

> **Nota alternativa:** Si tu versión de OpenSSL no acepta `-days -1`, usa el siguiente método con fechas explícitas:
> ```bash
> openssl x509 -req \
>   -in server-expired.csr \
>   -CA ca.crt \
>   -CAkey ca.key \
>   -CAcreateserial \
>   -out server-expired.crt \
>   -startdate 20200101000000Z \
>   -enddate 20200102000000Z \
>   -sha256
> ```

**Salida esperada de `-noout -dates`:**
```
notBefore=<fecha en el pasado>
notAfter=<fecha en el pasado o igual a notBefore>
```

#### 1.2 Levantar el servidor HTTPS con el certificado expirado

Crea el archivo `server_tls.py` con el siguiente contenido:

```python
# server_tls.py — Servidor HTTPS mínimo con Python ssl
import ssl
import http.server
import sys

CERT_FILE = sys.argv[1]   # Certificado
KEY_FILE  = sys.argv[2]   # Llave privada
PORT      = int(sys.argv[3]) if len(sys.argv) > 3 else 4443

context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.load_cert_chain(certfile=CERT_FILE, keyfile=KEY_FILE)

httpd = http.server.HTTPServer(('127.0.0.1', PORT), http.server.SimpleHTTPRequestHandler)
httpd.socket = context.wrap_socket(httpd.socket, server_side=True)
print(f"[+] Servidor HTTPS iniciado en https://127.0.0.1:{PORT}")
print("[+] Presiona Ctrl+C para detener.")
httpd.serve_forever()
```

```bash
# Iniciar el servidor con el certificado expirado en puerto 4443
# (ejecutar en una terminal separada o en background)
python3 server_tls.py server-expired.crt server-expired.key 4443 &
SERVER_PID=$!
echo "PID del servidor: $SERVER_PID"
sleep 1
```

#### 1.3 Diagnosticar el error con openssl s_client y curl

```bash
# Diagnóstico con openssl s_client
echo "Q" | openssl s_client \
  -connect 127.0.0.1:4443 \
  -CAfile ca.crt \
  2>&1 | grep -E "Verify|error|notAfter|notBefore|subject"

# Diagnóstico con curl (verbose)
curl -v --cacert ca.crt https://127.0.0.1:4443/ 2>&1 | grep -E "SSL|expire|certif|error"

# Ver detalles completos del certificado recibido
echo "Q" | openssl s_client \
  -connect 127.0.0.1:4443 \
  -CAfile ca.crt \
  2>&1 | openssl x509 -noout -text 2>/dev/null | grep -A2 "Validity"
```

**Salida esperada (fragmentos clave):**
```
verify error:num=10:certificate has expired
notAfter=<fecha pasada>
Verify return code: 10 (certificate has expired)
```

```
curl: (60) SSL certificate problem: certificate has expired
```

#### 1.4 Completar la Ficha de Diagnóstico — Escenario 1

Crea el archivo `ficha-escenario1.md` con el siguiente contenido (completa los campos en blanco según lo que observaste):

```bash
cat > ficha-escenario1.md << 'EOF'
# Ficha de Diagnóstico — Escenario 1: Certificado Expirado

## Síntoma observado
- Error reportado por curl: `SSL certificate problem: certificate has expired`
- Error reportado por openssl s_client: `verify error:num=10:certificate has expired`
- Código de verificación: 10

## Comando de diagnóstico usado
```
openssl s_client -connect 127.0.0.1:4443 -CAfile ca.crt
curl -v --cacert ca.crt https://127.0.0.1:4443/
openssl x509 -noout -dates -in server-expired.crt
```

## Causa raíz identificada
El certificado fue emitido con una fecha `notAfter` en el pasado.
El cliente TLS rechaza el certificado porque su período de validez ha concluido.

## Ruta de corrección — On-Premise
1. Generar nueva llave privada (o reutilizar la existente si no fue comprometida):
   `openssl genrsa -out server-nuevo.key 2048 && chmod 600 server-nuevo.key`
2. Generar nuevo CSR con los mismos atributos (o actualizados):
   `openssl req -new -key server-nuevo.key -out server-nuevo.csr -subj "..."`
3. Enviar el CSR a la CA interna (o externa) para obtener nuevo certificado con
   fecha de expiración válida (mínimo 90 días, recomendado 1 año).
4. Reemplazar el certificado en el servidor (nginx, Apache, etc.) y recargar el servicio.
5. Verificar con: `openssl x509 -noout -dates -in server-nuevo.crt`

## Ruta de corrección — Cloud (Azure Key Vault)
- En Azure Key Vault, configurar una "lifetime action" de tipo `AutoRenew` o
  `EmailContacts` con un umbral de días antes del vencimiento (e.g., 30 días).
- Si el emisor es integrado (DigiCert, GlobalSign), AKV puede renovar automáticamente.
- Para certificados importados: automatizar con Azure Automation o Function App
  que detecte la expiración via evento de Event Grid y reemplace el certificado.
- Comando para revisar vigencia en AKV:
  `az keyvault certificate show --vault-name <KV> --name <cert> --query "attributes"`

## Ruta de corrección — Cloud (AWS ACM)
- Si el certificado fue emitido por ACM y está asociado a un ALB/CloudFront:
  ACM renueva automáticamente (sin intervención) mientras la validación DNS esté activa.
- Si el certificado fue importado a ACM: ACM NO renueva automáticamente.
  Se debe importar un nuevo certificado antes del vencimiento.
- Monitoreo: configurar AWS CloudWatch Alarm sobre la métrica
  `DaysToExpiry` del certificado en ACM (umbral: 30 días).
EOF
```

```bash
# Detener el servidor del Escenario 1
kill $SERVER_PID 2>/dev/null
```

---

### Paso 2 — Escenario 2: Hostname Mismatch

**Objetivo:** Configurar un servidor con un certificado emitido para `servicio.empresa.local` pero accedido como `app.empresa.local`, diagnosticar el error de nombre de host no coincidente y proponer la corrección.

#### 2.1 Generar el certificado con SAN incorrecto

```bash
cd ~/labs/certs/practica5

# Generar llave privada
openssl genrsa -out server-mismatch.key 2048
chmod 600 server-mismatch.key

# Crear archivo de extensiones con SAN = servicio.empresa.local (NO app.empresa.local)
cat > san-mismatch.cnf << 'EOF'
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = servicio.empresa.local
EOF

# Generar CSR con SAN
openssl req -new \
  -key server-mismatch.key \
  -out server-mismatch.csr \
  -subj "/C=MX/ST=CDMX/O=EmpresaLab/CN=servicio.empresa.local" \
  -config san-mismatch.cnf

# Crear archivo de extensiones para la firma
cat > ext-mismatch.cnf << 'EOF'
subjectAltName = DNS:servicio.empresa.local
EOF

# Firmar el certificado (válido 365 días — NO expirado)
openssl x509 -req \
  -in server-mismatch.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server-mismatch.crt \
  -days 365 \
  -sha256 \
  -extfile ext-mismatch.cnf

# Verificar los SAN del certificado
openssl x509 -noout -text -in server-mismatch.crt | grep -A3 "Subject Alternative"
```

**Salida esperada:**
```
X509v3 Subject Alternative Name:
    DNS:servicio.empresa.local
```

#### 2.2 Levantar el servidor y diagnosticar

```bash
# Iniciar servidor con el certificado mismatch en puerto 4444
python3 server_tls.py server-mismatch.crt server-mismatch.key 4444 &
SERVER_PID2=$!
sleep 1

# Añadir entrada en /etc/hosts para simular el acceso por nombre incorrecto
# (requiere sudo; si no tienes sudo, usa el flag --resolve de curl)
# Opción A: con /etc/hosts
echo "127.0.0.1  app.empresa.local" | sudo tee -a /etc/hosts

# Opción B: sin sudo, usando --resolve de curl
# (usaremos esta opción en los comandos de diagnóstico)

# Diagnóstico con openssl s_client — accediendo al nombre INCORRECTO
echo "Q" | openssl s_client \
  -connect 127.0.0.1:4444 \
  -servername app.empresa.local \
  -CAfile ca.crt \
  2>&1 | grep -E "Verify|error|subject|altname|hostname"

# Diagnóstico con curl
curl -v \
  --cacert ca.crt \
  --resolve app.empresa.local:4444:127.0.0.1 \
  https://app.empresa.local:4444/ \
  2>&1 | grep -E "SSL|hostname|certif|error|subject"

# Inspeccionar los SAN del certificado presentado por el servidor
echo "Q" | openssl s_client \
  -connect 127.0.0.1:4444 \
  -servername app.empresa.local \
  -CAfile ca.crt \
  2>&1 | openssl x509 -noout -text 2>/dev/null | grep -A5 "Subject Alternative"
```

**Salida esperada:**
```
# openssl s_client:
verify error:num=62:hostname mismatch
Verify return code: 62 (hostname mismatch)

# curl:
SSL: certificate subject name 'servicio.empresa.local' does not match target host name 'app.empresa.local'
curl: (60) SSL: certificate subject name ...
```

#### 2.3 Completar la Ficha de Diagnóstico — Escenario 2

```bash
cat > ficha-escenario2.md << 'EOF'
# Ficha de Diagnóstico — Escenario 2: Hostname Mismatch

## Síntoma observado
- curl: `SSL: certificate subject name 'servicio.empresa.local' does not match target host name 'app.empresa.local'`
- openssl s_client: `verify error:num=62:hostname mismatch`
- El certificado es válido en fecha y está firmado por una CA de confianza,
  pero el nombre al que se accede no está en el CN ni en los SAN.

## Comando de diagnóstico usado
```
openssl s_client -connect 127.0.0.1:4444 -servername app.empresa.local -CAfile ca.crt
curl -v --cacert ca.crt --resolve app.empresa.local:4444:127.0.0.1 https://app.empresa.local:4444/
openssl x509 -noout -text -in server-mismatch.crt | grep -A3 "Subject Alternative"
```

## Causa raíz identificada
El certificado fue emitido con SAN = `DNS:servicio.empresa.local`.
El cliente accede a `app.empresa.local`, que no aparece en ningún SAN del certificado.
Nota: el CN ya no es suficiente para validación de hostname en navegadores modernos
(RFC 2818); los SAN son el campo autoritativo.

## Ruta de corrección — Opción A: Reemitir con SAN correcto
1. Crear nuevo archivo de extensiones con ambos nombres:
   ```
   subjectAltName = DNS:servicio.empresa.local, DNS:app.empresa.local
   ```
2. Generar nuevo CSR y firmar con la CA interna.
3. Reemplazar el certificado en el servidor.

## Ruta de corrección — Opción B: Certificado Wildcard
1. Emitir un certificado con SAN = `DNS:*.empresa.local`.
2. Cubre todos los subdominios de primer nivel de `empresa.local`.
3. Consideración de seguridad: un wildcard comprometido afecta todos los subdominios.

## Ruta de corrección — Cloud (Azure Key Vault)
- Al crear la política del certificado en AKV, especificar todos los SANs requeridos
  en el campo `dnsNames` de la política:
  ```json
  "x509CertificateProperties": {
    "subject": "CN=empresa.local",
    "subjectAlternativeNames": {
      "dnsNames": ["servicio.empresa.local", "app.empresa.local"]
    }
  }
  ```
- Si ya existe el certificado, crear una nueva versión con la política actualizada.

## Ruta de corrección — Cloud (AWS ACM)
- Solicitar un nuevo certificado incluyendo todos los nombres:
  ```bash
  aws acm request-certificate \
    --domain-name servicio.empresa.local \
    --subject-alternative-names app.empresa.local \
    --validation-method DNS
  ```
- Asociar el nuevo ARN al listener del ALB/CloudFront.
- El certificado anterior puede eliminarse una vez migrado el tráfico.
EOF
```

```bash
# Detener el servidor del Escenario 2
kill $SERVER_PID2 2>/dev/null
```

---

### Paso 3 — Escenario 3: Cadena de Confianza Incompleta

**Objetivo:** Configurar un servidor que sólo envía el certificado de entidad final sin los certificados intermedios, diagnosticar la cadena incompleta y demostrar cómo corregirla concatenando el bundle completo.

#### 3.1 Crear una CA intermedia y un certificado de entidad final

```bash
cd ~/labs/certs/practica5

# --- Crear CA Intermedia ---
openssl genrsa -out intermediate-ca.key 2048
chmod 600 intermediate-ca.key

openssl req -new \
  -key intermediate-ca.key \
  -out intermediate-ca.csr \
  -subj "/C=MX/ST=CDMX/O=EmpresaLab/CN=Lab Intermediate CA"

# Firmar la CA intermedia con la CA raíz
cat > ext-intermediate.cnf << 'EOF'
basicConstraints = CA:TRUE, pathlen:0
keyUsage = keyCertSign, cRLSign
EOF

openssl x509 -req \
  -in intermediate-ca.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out intermediate-ca.crt \
  -days 1825 \
  -sha256 \
  -extfile ext-intermediate.cnf

# --- Crear certificado de entidad final firmado por la CA INTERMEDIA ---
openssl genrsa -out server-chain.key 2048
chmod 600 server-chain.key

cat > ext-server-chain.cnf << 'EOF'
subjectAltName = DNS:app.empresa.local
EOF

openssl req -new \
  -key server-chain.key \
  -out server-chain.csr \
  -subj "/C=MX/ST=CDMX/O=EmpresaLab/CN=app.empresa.local"

openssl x509 -req \
  -in server-chain.csr \
  -CA intermediate-ca.crt \
  -CAkey intermediate-ca.key \
  -CAcreateserial \
  -out server-chain.crt \
  -days 365 \
  -sha256 \
  -extfile ext-server-chain.cnf

# Verificar la jerarquía
echo "=== Certificado entidad final ==="
openssl x509 -noout -subject -issuer -in server-chain.crt
echo "=== Certificado CA intermedia ==="
openssl x509 -noout -subject -issuer -in intermediate-ca.crt
echo "=== Certificado CA raíz ==="
openssl x509 -noout -subject -issuer -in ca.crt
```

**Salida esperada:**
```
=== Certificado entidad final ===
subject=C=MX, ST=CDMX, O=EmpresaLab, CN=app.empresa.local
issuer=C=MX, ST=CDMX, O=EmpresaLab, CN=Lab Intermediate CA
=== Certificado CA intermedia ===
subject=C=MX, ST=CDMX, O=EmpresaLab, CN=Lab Intermediate CA
issuer=C=MX, ST=CDMX, O=LabCA, CN=Lab Root CA
=== Certificado CA raíz ===
subject=C=MX, ST=CDMX, O=LabCA, CN=Lab Root CA
issuer=C=MX, ST=CDMX, O=LabCA, CN=Lab Root CA
```

#### 3.2 Iniciar servidor con cadena INCOMPLETA (sólo entidad final)

```bash
# El servidor sólo envía server-chain.crt (sin el intermedio) — cadena incompleta
python3 server_tls.py server-chain.crt server-chain.key 4445 &
SERVER_PID3=$!
sleep 1

# Diagnóstico — la CA raíz está en el trust store, pero el intermedio NO se envía
echo "Q" | openssl s_client \
  -connect 127.0.0.1:4445 \
  -CAfile ca.crt \
  2>&1 | grep -E "Verify|error|depth|chain|unable"

# Ver cuántos certificados envía el servidor con -showcerts
echo "Q" | openssl s_client \
  -connect 127.0.0.1:4445 \
  -CAfile ca.crt \
  -showcerts \
  2>&1 | grep -E "BEGIN CERTIFICATE|subject|issuer"

# curl también falla
curl -v \
  --cacert ca.crt \
  --resolve app.empresa.local:4445:127.0.0.1 \
  https://app.empresa.local:4445/ \
  2>&1 | grep -E "SSL|unable|certif|error|chain"
```

**Salida esperada (cadena incompleta):**
```
# openssl s_client:
verify error:num=20:unable to get local issuer certificate
depth=0 CN = app.empresa.local
Verify return code: 20 (unable to get local issuer certificate)

# -showcerts muestra sólo 1 certificado (el de entidad final):
-----BEGIN CERTIFICATE-----
subject=CN=app.empresa.local
issuer=CN=Lab Intermediate CA

# curl:
SSL certificate problem: unable to get local issuer certificate
```

#### 3.3 Crear el bundle completo y verificar la corrección

```bash
# Crear el bundle: entidad final + CA intermedia (orden correcto)
cat server-chain.crt intermediate-ca.crt > server-chain-bundle.crt

# Iniciar servidor con el bundle completo en puerto 4446
python3 server_tls.py server-chain-bundle.crt server-chain.key 4446 &
SERVER_PID4=$!
sleep 1

# Ahora el diagnóstico debe ser exitoso
echo "Q" | openssl s_client \
  -connect 127.0.0.1:4446 \
  -CAfile ca.crt \
  -showcerts \
  2>&1 | grep -E "Verify|error|depth|subject|BEGIN CERTIFICATE"

curl -v \
  --cacert ca.crt \
  --resolve app.empresa.local:4446:127.0.0.1 \
  https://app.empresa.local:4446/ \
  2>&1 | grep -E "SSL|certif|issuer|200|Verify"
```

**Salida esperada (cadena completa — éxito):**
```
# -showcerts muestra 2 certificados:
-----BEGIN CERTIFICATE-----   ← entidad final
subject=CN=app.empresa.local
issuer=CN=Lab Intermediate CA
-----BEGIN CERTIFICATE-----   ← CA intermedia
subject=CN=Lab Intermediate CA
issuer=CN=Lab Root CA

Verify return code: 0 (ok)
```

#### 3.4 Completar la Ficha de Diagnóstico — Escenario 3

```bash
cat > ficha-escenario3.md << 'EOF'
# Ficha de Diagnóstico — Escenario 3: Cadena de Confianza Incompleta

## Síntoma observado
- curl: `SSL certificate problem: unable to get local issuer certificate`
- openssl s_client: `verify error:num=20:unable to get local issuer certificate`
- Con `-showcerts`, el servidor sólo envía 1 certificado (entidad final).
  El cliente no puede construir la cadena hasta la CA raíz.

## Comando de diagnóstico usado
```
openssl s_client -connect 127.0.0.1:4445 -CAfile ca.crt -showcerts
curl -v --cacert ca.crt --resolve app.empresa.local:4445:127.0.0.1 https://app.empresa.local:4445/
```

## Causa raíz identificada
El servidor está configurado para presentar únicamente el certificado de entidad final.
El certificado intermedio (`Lab Intermediate CA`) no se incluye en la cadena enviada
durante el handshake TLS. El cliente no tiene el intermedio en su trust store local
y no puede completar la cadena de confianza hasta la CA raíz.

## Ruta de corrección — On-Premise
1. Concatenar el certificado de entidad final con todos los intermedios en orden:
   ```bash
   cat server.crt intermediate-ca.crt > server-bundle.crt
   ```
   (Si hay múltiples intermedios: entidad_final → intermedio_1 → intermedio_2)
2. Actualizar la configuración del servidor web para usar el bundle:
   - nginx: `ssl_certificate /path/to/server-bundle.crt;`
   - Apache: `SSLCertificateFile` + `SSLCertificateChainFile` (o bundle en `SSLCertificateFile`)
3. Recargar el servicio y verificar con `openssl s_client -showcerts`.

## Ruta de corrección — Cloud (Azure Key Vault)
- Al importar un certificado en AKV, incluir la cadena completa en el PFX:
  ```bash
  openssl pkcs12 -export \
    -in server.crt \
    -inkey server.key \
    -certfile intermediate-ca.crt \
    -out server-complete.pfx
  az keyvault certificate import --vault-name <KV> --name <cert> --file server-complete.pfx
  ```
- AKV almacena la cadena completa y la presenta correctamente cuando se vincula
  a App Service o Application Gateway.

## Ruta de corrección — Cloud (AWS ACM)
- Al importar certificado en ACM, siempre proporcionar `--certificate-chain`:
  ```bash
  aws acm import-certificate \
    --certificate fileb://server.crt \
    --private-key fileb://server.key \
    --certificate-chain fileb://intermediate-ca.crt
  ```
- Para certificados emitidos por ACM: ACM gestiona la cadena automáticamente;
  ALB/CloudFront reciben la cadena completa sin intervención manual.
EOF
```

```bash
# Detener los servidores del Escenario 3
kill $SERVER_PID3 $SERVER_PID4 2>/dev/null
```

---

### Paso 4 — Monitoreo de Expiración y Buenas Prácticas

**Objetivo:** Revisar estrategias de monitoreo de expiración de certificados a nivel on-premise (scripts + cron) y cloud (Azure Monitor, AWS CloudWatch), y consolidar buenas prácticas de protección de llaves privadas.

#### 4.1 Script de monitoreo de expiración (on-premise)

```bash
# Crear script de monitoreo de expiración
cat > check_cert_expiry.sh << 'SCRIPT'
#!/bin/bash
# check_cert_expiry.sh — Verifica días restantes para expiración de un certificado
# Uso: ./check_cert_expiry.sh <archivo.crt> [umbral_dias]
# Retorna: 0 si OK, 1 si expira pronto o ya expiró

CERT_FILE="${1:-server-expired.crt}"
THRESHOLD="${2:-30}"

if [ ! -f "$CERT_FILE" ]; then
  echo "ERROR: No se encontró el archivo $CERT_FILE"
  exit 2
fi

# Obtener fecha de expiración en segundos epoch
EXPIRY_DATE=$(openssl x509 -noout -enddate -in "$CERT_FILE" | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY_DATE" +%s 2>/dev/null || date -j -f "%b %d %T %Y %Z" "$EXPIRY_DATE" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))

SUBJECT=$(openssl x509 -noout -subject -in "$CERT_FILE" | sed 's/subject=//')

echo "Certificado : $CERT_FILE"
echo "Subject     : $SUBJECT"
echo "Expira      : $EXPIRY_DATE"
echo "Días restantes: $DAYS_LEFT"

if [ $DAYS_LEFT -le 0 ]; then
  echo "ESTADO: [CRÍTICO] El certificado YA EXPIRÓ hace $(( -DAYS_LEFT )) días."
  exit 1
elif [ $DAYS_LEFT -le $THRESHOLD ]; then
  echo "ESTADO: [ADVERTENCIA] El certificado expira en menos de $THRESHOLD días."
  exit 1
else
  echo "ESTADO: [OK] El certificado es válido."
  exit 0
fi
SCRIPT

chmod +x check_cert_expiry.sh

# Probar el script con los certificados del laboratorio
echo "=== Certificado expirado ==="
./check_cert_expiry.sh server-expired.crt 30

echo ""
echo "=== Certificado válido ==="
./check_cert_expiry.sh server-chain.crt 30
```

**Salida esperada:**
```
=== Certificado expirado ===
Certificado : server-expired.crt
Días restantes: -XXXX
ESTADO: [CRÍTICO] El certificado YA EXPIRÓ hace XXXX días.

=== Certificado válido ===
Certificado : server-chain.crt
Días restantes: 364
ESTADO: [OK] El certificado es válido.
```

#### 4.2 Ejemplo de entrada cron para monitoreo automatizado

```bash
# Ver cómo se vería una entrada cron (sólo visualización, no instalar en el lab)
cat << 'EOF'
# Entrada cron para verificar expiración diariamente a las 08:00
# Añadir con: crontab -e
0 8 * * * /home/usuario/labs/certs/practica5/check_cert_expiry.sh \
  /etc/ssl/certs/servidor-produccion.crt 30 \
  | mail -s "Alerta Certificado TLS" seguridad@empresa.com
EOF
```

#### 4.3 Referencias a monitoreo cloud (conceptual)

```bash
# Crear documento de referencia cloud
cat > monitoreo-cloud.md << 'EOF'
# Monitoreo de Expiración de Certificados en Cloud

## Azure Monitor — Azure Key Vault

### Alerta por evento de ciclo de vida (EventGrid)
Azure Key Vault emite eventos automáticamente cuando un certificado está próximo a expirar.
Configuración recomendada:
- Fuente: Azure Key Vault → Event Grid Topic
- Tipo de evento: `Microsoft.KeyVault.CertificateNearExpiry`
- Acción: Azure Function / Logic App → notificación + ticket automático

### Comando para revisar vigencia desde CLI
```bash
az keyvault certificate show \
  --vault-name <nombre-keyvault> \
  --name <nombre-certificado> \
  --query "{notBefore:attributes.notBefore, expires:attributes.expires, enabled:attributes.enabled}"
```

### Lifetime Actions (autorrenovación)
```bash
# Ver política actual del certificado
az keyvault certificate policy show \
  --vault-name <nombre-keyvault> \
  --name <nombre-certificado>

# La política incluye lifetimeActions con trigger (daysBeforeExpiry o lifetimePercentage)
# y action (AutoRenew o EmailContacts)
```

## AWS CloudWatch — ACM

### Métrica nativa: DaysToExpiry
- ACM publica automáticamente la métrica `DaysToExpiry` en CloudWatch.
- Namespace: `AWS/CertificateManager`
- Dimensión: `CertificateArn`

### Crear alarma CloudWatch (CLI)
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "CertificadoProximoExpirar" \
  --metric-name DaysToExpiry \
  --namespace AWS/CertificateManager \
  --dimensions Name=CertificateArn,Value=<CERT_ARN> \
  --statistic Minimum \
  --period 86400 \
  --threshold 30 \
  --comparison-operator LessThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions <SNS_TOPIC_ARN>
```

### Nota sobre renovación automática ACM
- Certificados PÚBLICOS emitidos por ACM: renovación automática si el registro DNS
  de validación sigue activo. No se requiere intervención.
- Certificados IMPORTADOS en ACM: NO se renuevan automáticamente.
  Se debe importar el nuevo certificado antes del vencimiento.
  Configurar la alarma CloudWatch con umbral de 45-60 días para dar tiempo al proceso.
EOF
```

#### 4.4 Verificar permisos de todas las llaves privadas generadas

```bash
echo "=== Verificación de permisos de llaves privadas ==="
ls -la *.key 2>/dev/null
echo ""
echo "Todos los archivos .key deben mostrar: -rw------- (600)"
echo ""

# Corregir permisos si alguno está mal
find . -name "*.key" -exec chmod 600 {} \;
echo "Permisos corregidos a 600 en todos los archivos .key"
ls -la *.key
```

**Salida esperada:**
```
-rw------- 1 usuario usuario 1679 <fecha> ca.key
-rw------- 1 usuario usuario 1679 <fecha> server-expired.key
-rw------- 1 usuario usuario 1679 <fecha> server-mismatch.key
-rw------- 1 usuario usuario 1679 <fecha> server-chain.key
-rw------- 1 usuario usuario 1679 <fecha> intermediate-ca.key
```

---

## Validación y Pruebas

### Lista de verificación final

Ejecuta los siguientes comandos para confirmar que completaste todos los objetivos:

```bash
cd ~/labs/certs/practica5

echo "=========================================="
echo "VALIDACIÓN FINAL — Lab 05-00-01"
echo "=========================================="

# V1: Verificar que los 3 certificados de escenario existen
echo ""
echo "[V1] Certificados de escenario:"
for cert in server-expired.crt server-mismatch.crt server-chain.crt; do
  if [ -f "$cert" ]; then
    EXPIRY=$(openssl x509 -noout -enddate -in "$cert" 2>/dev/null | cut -d= -f2)
    echo "  ✓ $cert — expira: $EXPIRY"
  else
    echo "  ✗ FALTA: $cert"
  fi
done

# V2: Verificar fichas de diagnóstico
echo ""
echo "[V2] Fichas de diagnóstico:"
for ficha in ficha-escenario1.md ficha-escenario2.md ficha-escenario3.md; do
  if [ -f "$ficha" ]; then
    LINES=$(wc -l < "$ficha")
    echo "  ✓ $ficha ($LINES líneas)"
  else
    echo "  ✗ FALTA: $ficha"
  fi
done

# V3: Verificar permisos de llaves privadas
echo ""
echo "[V3] Permisos de llaves privadas (deben ser 600):"
for key in *.key; do
  PERMS=$(stat -c "%a" "$key" 2>/dev/null || stat -f "%OLp" "$key" 2>/dev/null)
  if [ "$PERMS" = "600" ]; then
    echo "  ✓ $key — permisos: $PERMS"
  else
    echo "  ✗ $key — permisos INCORRECTOS: $PERMS (debe ser 600)"
  fi
done

# V4: Verificar bundle de cadena completa
echo ""
echo "[V4] Bundle de cadena completa:"
if [ -f "server-chain-bundle.crt" ]; then
  CERT_COUNT=$(grep -c "BEGIN CERTIFICATE" server-chain-bundle.crt)
  echo "  ✓ server-chain-bundle.crt contiene $CERT_COUNT certificados (esperado: 2)"
else
  echo "  ✗ FALTA: server-chain-bundle.crt"
fi

# V5: Verificar script de monitoreo
echo ""
echo "[V5] Script de monitoreo:"
if [ -x "check_cert_expiry.sh" ]; then
  echo "  ✓ check_cert_expiry.sh existe y es ejecutable"
  ./check_cert_expiry.sh server-expired.crt 30 > /dev/null 2>&1
  if [ $? -eq 1 ]; then
    echo "  ✓ Detecta correctamente el certificado expirado"
  fi
else
  echo "  ✗ FALTA o no ejecutable: check_cert_expiry.sh"
fi

echo ""
echo "=========================================="
echo "Validación completada."
echo "=========================================="
```

### Prueba diferencial con `--insecure`

```bash
# Demostrar la diferencia entre diagnóstico con y sin verificación TLS
# (--insecure bypasea la validación pero muestra el error en verbose)

# Reiniciar servidor expirado brevemente para la prueba
python3 server_tls.py server-expired.crt server-expired.key 4443 &
TEMP_PID=$!
sleep 1

echo "=== Con validación (falla correctamente) ==="
curl --cacert ca.crt https://127.0.0.1:4443/ 2>&1 | grep -E "SSL|error|curl:"

echo ""
echo "=== Sin validación --insecure (sólo para diagnóstico, NUNCA en producción) ==="
curl --insecure https://127.0.0.1:4443/ 2>&1 | head -5

kill $TEMP_PID 2>/dev/null

echo ""
echo "IMPORTANTE: --insecure y -k de curl NUNCA deben usarse en scripts de producción."
echo "Son herramientas de diagnóstico diferencial para confirmar que el problema es TLS."
```

---

## Solución de Problemas

### Problema 1: `openssl x509 -req ... -days -1` falla con error de fecha

**Síntoma:**
```
Error: notBefore/notAfter dates out of range
```
o el certificado se genera con fechas en el futuro en lugar del pasado.

**Causa raíz:**
Versiones recientes de OpenSSL (3.x) aplican validaciones más estrictas sobre rangos de fechas. El parámetro `-days -1` puede interpretarse de forma diferente o rechazarse.

**Solución:**
Usar fechas explícitas con `-startdate` y `-enddate` en formato UTC:
```bash
openssl x509 -req \
  -in server-expired.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server-expired.crt \
  -startdate 20200101000000Z \
  -enddate   20200102000000Z \
  -sha256

# Verificar que las fechas están en el pasado
openssl x509 -noout -dates -in server-expired.crt
```
Si el sistema aún lo rechaza, usar `faketime` (si está disponible) o modificar temporalmente la fecha del sistema (restaurar inmediatamente después):
```bash
# Con faketime (instalar con: sudo apt install faketime)
faketime '2020-01-01 00:00:00' openssl x509 -req \
  -in server-expired.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server-expired.crt -days 1 -sha256
```

---

### Problema 2: El servidor Python falla con `[SSL: NO_SHARED_CIPHER]` o `[SSL: WRONG_VERSION_NUMBER]`

**Síntoma:**
```
ssl.SSLError: [SSL: NO_SHARED_CIPHER] no shared cipher
```
o el cliente reporta:
```
SSL routines:ssl3_get_record:wrong version number
```

**Causa raíz:**
La versión de Python/ssl puede requerir configuración explícita del protocolo TLS mínimo. En Python 3.10+, TLS 1.0 y 1.1 están deshabilitados por defecto. También puede ocurrir si el archivo de certificado o llave está corrupto o tiene permisos incorrectos que impiden su lectura.

**Solución:**
Modificar `server_tls.py` para especificar el contexto TLS explícitamente:
```python
context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.minimum_version = ssl.TLSVersion.TLSv1_2
context.load_cert_chain(certfile=CERT_FILE, keyfile=KEY_FILE)
```
Verificar también que los archivos son legibles:
```bash
# Verificar que la llave privada es legible por el usuario actual
ls -la server-chain.key
# Si el problema persiste, verificar que el certificado y la llave corresponden
openssl x509 -noout -modulus -in server-chain.crt | md5sum
openssl rsa  -noout -modulus -in server-chain.key | md5sum
# Ambos md5sum deben ser idénticos
```

---

## Limpieza

```bash
cd ~/labs/certs/practica5

# Asegurarse de que no quedan servidores Python corriendo
pkill -f "server_tls.py" 2>/dev/null
echo "Servidores HTTPS de prueba detenidos."

# Eliminar entrada de /etc/hosts si fue añadida (Escenario 2)
sudo sed -i '/app\.empresa\.local/d' /etc/hosts 2>/dev/null
echo "Entrada /etc/hosts limpiada (si aplica)."

# Listar artefactos generados (NO eliminar aún — pueden ser útiles para referencia)
echo ""
echo "=== Artefactos generados en esta práctica ==="
ls -lh ~/labs/certs/practica5/

echo ""
echo "NOTA: Los archivos .key contienen llaves privadas de laboratorio."
echo "Puedes eliminar todo el directorio al finalizar el curso con:"
echo "  rm -rf ~/labs/certs/practica5/"
echo ""
echo "NUNCA subas archivos .key a repositorios públicos (Git, etc.)."
```

> **Retención recomendada:** Conserva los archivos `ficha-escenario*.md` y `monitoreo-cloud.md` como referencia de estudio. Los archivos `.key`, `.crt` y `.csr` pueden eliminarse una vez concluido el curso.

---

## Resumen

En esta práctica construiste y diagnosticaste tres escenarios de fallo TLS que representan los errores más frecuentes en entornos productivos:

| Escenario | Error TLS | Código OpenSSL | Corrección Principal |
|-----------|-----------|----------------|----------------------|
| Certificado expirado | `certificate has expired` | `num=10` | Renovar certificado; en cloud: renovación automática ACM o lifetime action AKV |
| Hostname mismatch | `hostname mismatch` | `num=62` | Reemitir con SAN correcto o wildcard; actualizar ARN en ALB/AKV |
| Cadena incompleta | `unable to get local issuer certificate` | `num=20` | Concatenar bundle PEM; importar PFX completo a AKV/ACM |

### Conceptos clave consolidados

- **`openssl s_client -showcerts`** es la herramienta más informativa para diagnosticar cadenas de confianza; muestra exactamente cuántos y cuáles certificados envía el servidor.
- **`curl -v --cacert`** permite diagnóstico diferencial: si falla con `--cacert` pero funciona con `--insecure`, el problema es de confianza o validación, no de conectividad.
- **Permisos `chmod 600`** en llaves privadas son obligatorios; cualquier llave con permisos más abiertos debe considerarse comprometida.
- **ACM renueva automáticamente** sólo certificados públicos emitidos por ACM asociados a recursos; los importados requieren gestión manual.
- **Azure Key Vault** centraliza el ciclo de vida con eventos de EventGrid para notificación y `lifetimeActions` para autorrenovación con emisores integrados.
- Los scripts de monitoreo con `openssl x509 -enddate` + cron son la solución on-premise más simple y efectiva para evitar expiración silenciosa.

### Recursos adicionales

- [RFC 5280 — Internet X.509 PKI Certificate Profile](https://www.rfc-editor.org/rfc/rfc5280)
- [OpenSSL s_client documentation](https://www.openssl.org/docs/man3.0/man1/openssl-s_client.html)
- [Azure Key Vault — About certificates](https://learn.microsoft.com/azure/key-vault/certificates/about-certificates)
- [AWS ACM — Certificate renewal](https://docs.aws.amazon.com/acm/latest/userguide/managed-renewal.html)
- [AWS CloudWatch — ACM metrics](https://docs.aws.amazon.com/acm/latest/userguide/cloudwatch-metrics.html)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — referencia para configuración TLS en servidores

---
