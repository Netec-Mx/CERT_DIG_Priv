# Crear un certificado autofirmado para un servicio interno y validarlo con OpenSSL

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 20 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Crear (Create)                               |
| **Entorno**      | Linux / WSL2                                 |
| **Lab ID**       | 02-00-01                                     |

---

## Descripción General

En este laboratorio asumirás el rol de administrador de infraestructura de una empresa que necesita proteger un portal de monitoreo interno (`intranet.empresa.local`). Partiendo desde cero, generarás llaves privadas RSA 4096 y ECDSA P-256, crearás un archivo de configuración OpenSSL con extensiones X.509 v3 completas (SAN, BasicConstraints, KeyUsage), producirás un certificado autofirmado válido por 365 días y lo convertirás a los formatos PFX/PKCS#12 y DER. Finalizarás inspeccionando y validando el certificado con comandos OpenSSL para confirmar su estructura, vigencia y coherencia con la llave privada.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, podrás:

- [ ] Generar llaves privadas RSA 4096 y ECDSA P-256 con OpenSSL y comparar sus características.
- [ ] Crear un archivo de configuración `.cnf` con los campos Subject y extensiones X.509 v3 (SAN, BasicConstraints, KeyUsage, ExtendedKeyUsage).
- [ ] Generar un CSR y un certificado autofirmado a partir del archivo de configuración.
- [ ] Convertir el certificado entre los formatos PEM, DER y PKCS#12/PFX.
- [ ] Validar la estructura, vigencia y coherencia del certificado con `openssl x509`, `openssl verify` y comparación de módulos.

---

## Prerrequisitos

### Conocimiento previo

- Haber completado la Práctica 1 o tener conocimiento equivalente sobre la estructura de certificados X.509.
- Comprensión básica de par de llaves pública/privada y del propósito de un CSR.
- Manejo básico de la terminal Linux (navegación de directorios, edición de archivos con `nano` o `vim`).

### Acceso y herramientas

- Sistema Linux (nativo o WSL2 en Windows 10/11).
- OpenSSL 1.1.1 o superior instalado y funcional.
- Permisos de escritura en el directorio de trabajo (`~/labs/certs/`).

---

## Entorno de Laboratorio

### Requisitos de hardware

| Recurso       | Mínimo            | Recomendado       |
|---------------|-------------------|-------------------|
| CPU           | 2 núcleos x86-64  | 4 núcleos         |
| RAM           | 4 GB              | 8 GB              |
| Almacenamiento| 10 GB libres      | 20 GB libres      |

### Requisitos de software

| Herramienta  | Versión mínima | Verificación                        |
|--------------|----------------|-------------------------------------|
| OpenSSL      | 1.1.1          | `openssl version`                   |
| Bash         | 4.x            | `bash --version`                    |
| nano / vim   | cualquiera     | `nano --version` / `vim --version`  |

### Preparación del entorno

Ejecuta los siguientes comandos para crear el directorio de trabajo y verificar que OpenSSL esté disponible:

```bash
# Crear directorio de trabajo organizado
mkdir -p ~/labs/certs
cd ~/labs/certs

# Verificar versión de OpenSSL
openssl version -a

# Establecer umask restrictivo para esta sesión
umask 077
```

> **⚠️ Nota de seguridad:** El comando `umask 077` garantiza que todos los archivos creados durante la sesión tengan permisos `600` (solo lectura/escritura para el propietario). Esto es especialmente importante para las llaves privadas. Aplica este hábito en todos los entornos de producción.

---

## Procedimiento Paso a Paso

---

### Paso 1: Generar la llave privada RSA 4096

**Objetivo:** Crear una llave privada RSA de 4096 bits en formato PKCS#8, que servirá como base para el certificado del servicio interno.

#### Instrucciones

1. Asegúrate de estar en el directorio de trabajo con `umask 077` activo:

```bash
cd ~/labs/certs
umask 077
```

2. Genera la llave RSA 4096 usando el comando moderno `genpkey`:

```bash
openssl genpkey -algorithm RSA \
  -pkeyopt rsa_keygen_bits:4096 \
  -out intranet-rsa4096.key
```

> **Nota:** Para este laboratorio se genera la llave **sin contraseña** para simplificar el flujo. En producción, añade `-aes-256-cbc -pbkdf2` para cifrar la llave con una frase de contraseña robusta.

3. Verifica los permisos del archivo generado:

```bash
ls -l intranet-rsa4096.key
```

4. Inspecciona los parámetros de la llave:

```bash
openssl pkey -in intranet-rsa4096.key -text -noout
```

#### Salida esperada

```
# ls -l
-rw------- 1 usuario usuario 3272 jun 15 10:01 intranet-rsa4096.key

# openssl pkey -text -noout (fragmento)
RSA Private-Key: (4096 bit, 2 primes)
modulus:
    00:c3:4a:...
publicExponent: 65537 (0x10001)
...
```

#### Verificación

```bash
# Confirmar que el archivo comienza con el encabezado PKCS#8 correcto
head -1 intranet-rsa4096.key
# Salida esperada: -----BEGIN PRIVATE KEY-----
```

---

### Paso 2: Generar la llave privada ECDSA P-256 (comparativa)

**Objetivo:** Generar una llave ECDSA P-256 para comparar su tamaño y estructura con la llave RSA 4096, comprendiendo las diferencias en rendimiento y seguridad equivalente.

#### Instrucciones

1. Genera la llave ECDSA P-256 con `genpkey`:

```bash
openssl genpkey -algorithm EC \
  -pkeyopt ec_paramgen_curve:P-256 \
  -pkeyopt ec_param_enc:named_curve \
  -out intranet-ecdsa-p256.key
```

2. Inspecciona sus parámetros:

```bash
openssl pkey -in intranet-ecdsa-p256.key -text -noout
```

3. Compara el tamaño de ambos archivos:

```bash
ls -lh intranet-rsa4096.key intranet-ecdsa-p256.key
```

#### Salida esperada

```
# ls -lh (fragmento ilustrativo)
-rw------- 1 usuario usuario 3.2K jun 15 10:02 intranet-rsa4096.key
-rw------- 1 usuario usuario  227 jun 15 10:03 intranet-ecdsa-p256.key

# openssl pkey -text -noout (fragmento ECDSA)
Private-Key: (256 bit)
priv:
    4f:8a:...
pub:
    04:...
ASN1 OID: prime256v1
NIST CURVE: P-256
```

#### Verificación

```bash
# Verificar encabezado PKCS#8 para ECDSA
head -1 intranet-ecdsa-p256.key
# Salida esperada: -----BEGIN PRIVATE KEY-----
```

> **Punto de reflexión:** La llave RSA 4096 ocupa ~3 KB mientras que ECDSA P-256 ocupa ~227 bytes, ofreciendo seguridad equivalente a RSA 3072. Para el resto del laboratorio usaremos la llave RSA 4096 para el certificado principal, pero el mismo procedimiento aplica a ECDSA.

---

### Paso 3: Crear el archivo de configuración OpenSSL (`.cnf`)

**Objetivo:** Crear un archivo de configuración que defina los campos del Subject y las extensiones X.509 v3 necesarias para un certificado de servicio interno válido, incluyendo Subject Alternative Names (SAN).

#### Instrucciones

1. Crea el archivo de configuración con `nano` (o el editor de tu preferencia):

```bash
nano intranet-openssl.cnf
```

2. Escribe el siguiente contenido completo en el archivo:

```ini
# intranet-openssl.cnf
# Archivo de configuración para certificado del portal de monitoreo interno

[ req ]
default_bits        = 4096
default_md          = sha256
distinguished_name  = req_distinguished_name
req_extensions      = v3_req
prompt              = no

[ req_distinguished_name ]
# Campos del Subject del certificado
C  = MX
ST = Ciudad de Mexico
L  = Ciudad de Mexico
O  = Empresa Ejemplo S.A. de C.V.
OU = Infraestructura TI
CN = intranet.empresa.local

[ v3_req ]
# Extensiones incluidas en el CSR
subjectAltName = @alt_names
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ v3_ca ]
# Extensiones para el certificado autofirmado final
subjectAltName = @alt_names
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always, issuer

[ alt_names ]
# Subject Alternative Names: DNS e IPs del servicio interno
DNS.1 = intranet.empresa.local
DNS.2 = monitoreo.empresa.local
DNS.3 = localhost
IP.1  = 192.168.1.100
IP.2  = 127.0.0.1
```

3. Guarda el archivo (`Ctrl+O`, `Enter`, `Ctrl+X` en nano).

4. Verifica que el archivo se guardó correctamente:

```bash
cat intranet-openssl.cnf
```

#### Salida esperada

El contenido del archivo debe mostrarse completo sin errores de sintaxis. La sección `[ alt_names ]` es crítica: sin ella, los navegadores y clientes TLS modernos rechazarán el certificado incluso si el CN coincide.

#### Verificación

```bash
# Verificar que el archivo no tiene errores de sintaxis básicos
grep -c "\[" intranet-openssl.cnf
# Salida esperada: 5 (cinco secciones definidas)
```

> **Nota técnica:** A partir de Chrome 58 y Firefox 48, los navegadores requieren que el nombre del servidor aparezca en el campo `subjectAltName`. El campo `CN` solo ya no es suficiente. Por eso es imprescindible incluir `DNS.1` en la sección `alt_names`.

---

### Paso 4: Generar el CSR (Certificate Signing Request)

**Objetivo:** Crear la solicitud de firma de certificado usando la llave RSA 4096 y el archivo de configuración, verificando que los campos y extensiones se incorporaron correctamente.

#### Instrucciones

1. Genera el CSR usando la llave privada y el archivo de configuración:

```bash
openssl req -new \
  -key intranet-rsa4096.key \
  -config intranet-openssl.cnf \
  -out intranet.csr
```

2. Inspecciona el contenido del CSR para verificar que los campos son correctos:

```bash
openssl req -in intranet.csr -text -noout
```

3. Verifica específicamente que las extensiones SAN se incluyeron en el CSR:

```bash
openssl req -in intranet.csr -text -noout | grep -A 5 "Subject Alternative Name"
```

#### Salida esperada

```
# openssl req -text -noout (fragmento relevante)
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C=MX, ST=Ciudad de Mexico, L=Ciudad de Mexico,
                 O=Empresa Ejemplo S.A. de C.V., OU=Infraestructura TI,
                 CN=intranet.empresa.local
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
        Attributes:
        Requested Extensions:
            X509v3 Subject Alternative Name:
                DNS:intranet.empresa.local, DNS:monitoreo.empresa.local,
                DNS:localhost, IP Address:192.168.1.100, IP Address:127.0.0.1
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
    Signature Algorithm: sha256WithRSAEncryption
```

#### Verificación

```bash
# Verificar que el Subject contiene el CN correcto
openssl req -in intranet.csr -subject -noout
# Salida esperada: subject=C=MX, ST=Ciudad de Mexico, ..., CN=intranet.empresa.local
```

---

### Paso 5: Generar el certificado autofirmado X.509 v3

**Objetivo:** Firmar el CSR con la propia llave privada para producir un certificado autofirmado válido por 365 días, asegurando que las extensiones X.509 v3 se incluyan correctamente en el certificado final.

#### Instrucciones

1. Genera el certificado autofirmado a partir del CSR, aplicando las extensiones de la sección `v3_ca`:

```bash
openssl x509 -req \
  -in intranet.csr \
  -signkey intranet-rsa4096.key \
  -days 365 \
  -sha256 \
  -extfile intranet-openssl.cnf \
  -extensions v3_ca \
  -out intranet.crt
```

2. Verifica que el archivo de certificado se creó:

```bash
ls -lh intranet.crt
```

3. Inspecciona el certificado completo para confirmar todos los campos y extensiones:

```bash
openssl x509 -in intranet.crt -text -noout
```

4. Muestra solo las fechas de vigencia:

```bash
openssl x509 -in intranet.crt -noout -dates
```

5. Muestra el Subject y el Issuer (en un certificado autofirmado deben ser idénticos):

```bash
openssl x509 -in intranet.crt -noout -subject -issuer
```

#### Salida esperada

```
# openssl x509 -text -noout (fragmento relevante)
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7a:3f:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=MX, ST=Ciudad de Mexico, ..., CN=intranet.empresa.local
        Validity
            Not Before: Jun 15 10:05:00 2025 GMT
            Not After : Jun 15 10:05:00 2026 GMT
        Subject: C=MX, ST=Ciudad de Mexico, ..., CN=intranet.empresa.local
        ...
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:intranet.empresa.local, DNS:monitoreo.empresa.local,
                DNS:localhost, IP Address:192.168.1.100, IP Address:127.0.0.1
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Key Identifier:
                A1:B2:C3:...
            X509v3 Authority Key Identifier:
                keyid:A1:B2:C3:...

# openssl x509 -dates
notBefore=Jun 15 10:05:00 2025 GMT
notAfter=Jun 15 10:05:00 2026 GMT

# openssl x509 -subject -issuer
subject=C=MX, ST=Ciudad de Mexico, ..., CN=intranet.empresa.local
issuer=C=MX, ST=Ciudad de Mexico, ..., CN=intranet.empresa.local
```

#### Verificación

```bash
# Confirmar que es un certificado X.509 versión 3
openssl x509 -in intranet.crt -text -noout | grep "Version:"
# Salida esperada: Version: 3 (0x2)

# Confirmar que las SAN están presentes
openssl x509 -in intranet.crt -text -noout | grep -A 3 "Subject Alternative Name"
```

> **Punto clave:** El uso de `-extfile intranet-openssl.cnf -extensions v3_ca` es fundamental. Sin este parámetro, `openssl x509 -req` generaría un certificado v1 sin extensiones, que los clientes TLS modernos rechazarían.

---

### Paso 6: Convertir el certificado al formato PKCS#12 / PFX

**Objetivo:** Empaquetar la llave privada y el certificado en un archivo PKCS#12 (`.pfx`) para simular la importación en un servidor Windows o aplicación Java que requiera este formato.

#### Instrucciones

1. Crea el archivo PKCS#12 combinando la llave privada y el certificado:

```bash
openssl pkcs12 -export \
  -inkey intranet-rsa4096.key \
  -in intranet.crt \
  -name "intranet.empresa.local - Portal Monitoreo" \
  -out intranet.pfx
```

> Cuando se te solicite, ingresa una contraseña de exportación. Usa algo memorable para el laboratorio, por ejemplo: `Lab2024!`. **En producción, usa una contraseña fuerte y guárdala en un vault.**

2. Verifica el contenido del archivo PFX:

```bash
openssl pkcs12 -in intranet.pfx -info -noout
```

> Ingresa la contraseña de exportación cuando se solicite.

3. Lista los certificados y llaves incluidos en el PFX:

```bash
openssl pkcs12 -in intranet.pfx -nokeys -noout -info 2>&1 | grep -E "subject|issuer|Bag"
```

#### Salida esperada

```
# openssl pkcs12 -info -noout (fragmento)
MAC: sha256, Iteration 2048
MAC length: 32, salt length: 8
PKCS7 Encrypted data: pbeWithSHA1And40BitRC2-CBC, Iteration 2048
Certificate bag
PKCS7 Data
Shrouded Keybag: pbeWithSHA1And3-KeyTripleDES-CBC, Iteration 2048
```

#### Verificación

```bash
ls -lh intranet.pfx
# El archivo debe existir con permisos 600 y tamaño > 3 KB
```

> **Nota operativa:** El archivo `.pfx` contiene tanto la llave privada como el certificado. Debe tratarse con la misma confidencialidad que la llave privada. La contraseña de exportación es la única protección del contenido.

---

### Paso 7: Convertir el certificado al formato DER

**Objetivo:** Convertir el certificado PEM al formato DER (binario), requerido por algunos sistemas embebidos, aplicaciones Java (keystores) y ciertos servicios de directorio.

#### Instrucciones

1. Convierte el certificado de PEM a DER:

```bash
openssl x509 -in intranet.crt \
  -outform DER \
  -out intranet.der
```

2. Verifica que el archivo DER es binario (no legible como texto):

```bash
file intranet.der
xxd intranet.der | head -3
```

3. Confirma que ambos archivos contienen el mismo certificado convirtiéndolo de regreso a texto:

```bash
openssl x509 -in intranet.der -inform DER -noout -fingerprint -sha256
openssl x509 -in intranet.crt -inform PEM -noout -fingerprint -sha256
```

Ambas huellas digitales deben ser **idénticas**.

#### Salida esperada

```
# file intranet.der
intranet.der: data

# xxd intranet.der | head -3
00000000: 3082 0a2e 3082 0816 a003 0201 0202 1400  0...0...........
...

# Fingerprints (deben coincidir)
SHA256 Fingerprint=3A:7F:...:B2:C1
SHA256 Fingerprint=3A:7F:...:B2:C1
```

#### Verificación

```bash
# Confirmar que los fingerprints son idénticos
HASH_PEM=$(openssl x509 -in intranet.crt -inform PEM -noout -fingerprint -sha256 | cut -d= -f2)
HASH_DER=$(openssl x509 -in intranet.der -inform DER -noout -fingerprint -sha256 | cut -d= -f2)
[ "$HASH_PEM" = "$HASH_DER" ] && echo "✓ Hashes idénticos: conversión exitosa" || echo "✗ ERROR: Hashes no coinciden"
```

---

### Paso 8: Validar la coherencia entre la llave privada y el certificado

**Objetivo:** Verificar que la llave privada y el certificado corresponden al mismo par de llaves, comparando el módulo RSA de ambos. Este paso es crítico para evitar errores de configuración en servidores TLS.

#### Instrucciones

1. Extrae el módulo de la llave privada y calcula su hash MD5:

```bash
openssl rsa -in intranet-rsa4096.key -noout -modulus | openssl md5
```

2. Extrae el módulo del certificado y calcula su hash MD5:

```bash
openssl x509 -in intranet.crt -noout -modulus | openssl md5
```

3. Extrae el módulo del CSR y calcula su hash MD5:

```bash
openssl req -in intranet.csr -noout -modulus | openssl md5
```

Los tres hashes **deben ser idénticos**. Si difieren, la llave privada y el certificado no corresponden al mismo par.

4. Realiza también la verificación de integridad de la llave privada:

```bash
openssl rsa -in intranet-rsa4096.key -check -noout
```

#### Salida esperada

```
# Módulo de la llave privada
(stdin)= d41d8cd98f00b204e9800998ecf8427e

# Módulo del certificado (debe ser idéntico al anterior)
(stdin)= d41d8cd98f00b204e9800998ecf8427e

# Módulo del CSR (debe ser idéntico)
(stdin)= d41d8cd98f00b204e9800998ecf8427e

# Verificación de integridad
RSA key ok
```

> **Nota:** Los valores MD5 mostrados arriba son ilustrativos. En tu entorno, los tres hashes tendrán el mismo valor hexadecimal que corresponda a tu llave específica.

#### Verificación

```bash
# Script de verificación automática
KEY_MOD=$(openssl rsa -in intranet-rsa4096.key -noout -modulus | openssl md5)
CRT_MOD=$(openssl x509 -in intranet.crt -noout -modulus | openssl md5)
CSR_MOD=$(openssl req -in intranet.csr -noout -modulus | openssl md5)

if [ "$KEY_MOD" = "$CRT_MOD" ] && [ "$CRT_MOD" = "$CSR_MOD" ]; then
  echo "✓ Coherencia verificada: llave, CSR y certificado corresponden al mismo par"
else
  echo "✗ ERROR: Inconsistencia detectada entre llave, CSR y/o certificado"
fi
```

---

## Validación y Pruebas

Una vez completados todos los pasos, ejecuta las siguientes verificaciones finales para confirmar que el laboratorio fue completado exitosamente.

### Verificación 1: Inventario de archivos generados

```bash
cd ~/labs/certs
ls -lh intranet*
```

**Salida esperada:**

```
-rw------- 1 usuario usuario  227 jun 15 10:03 intranet-ecdsa-p256.key
-rw------- 1 usuario usuario 3.2K jun 15 10:01 intranet-rsa4096.key
-rw------- 1 usuario usuario 1.7K jun 15 10:04 intranet.cnf
-rw------- 1 usuario usuario 1.6K jun 15 10:05 intranet.csr
-rw------- 1 usuario usuario 2.1K jun 15 10:06 intranet.crt
-rw------- 1 usuario usuario 4.1K jun 15 10:07 intranet.pfx
-rw------- 1 usuario usuario 2.0K jun 15 10:08 intranet.der
```

Todos los archivos deben tener permisos `600` (`-rw-------`).

### Verificación 2: Validación del certificado con `openssl verify`

```bash
# Validar el certificado contra sí mismo (autofirmado actúa como su propia CA)
openssl verify -CAfile intranet.crt intranet.crt
```

**Salida esperada:**

```
intranet.crt: OK
```

> **Nota importante:** Esta verificación confirma que la firma del certificado es matemáticamente válida. Sin embargo, un cliente TLS real rechazará este certificado con una advertencia de "CA no confiable" porque el certificado no está en el almacén de confianza del sistema. Esto es el comportamiento **correcto y esperado** para certificados autofirmados.

### Verificación 3: Resumen de extensiones críticas

```bash
# Verificar que todas las extensiones X.509 v3 están presentes
echo "=== Extensiones del certificado ==="
openssl x509 -in intranet.crt -text -noout | grep -E \
  "Subject Alternative Name|Basic Constraints|Key Usage|Extended Key Usage" -A 2
```

**Salida esperada:**

```
=== Extensiones del certificado ===
            X509v3 Subject Alternative Name:
                DNS:intranet.empresa.local, DNS:monitoreo.empresa.local,
                DNS:localhost, IP Address:192.168.1.100, IP Address:127.0.0.1
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
```

### Verificación 4: Tabla de resumen de artefactos

```bash
echo "=== RESUMEN DEL LABORATORIO ==="
echo ""
echo "Llave RSA 4096:"
openssl pkey -in intranet-rsa4096.key -text -noout 2>&1 | grep "RSA Private-Key"
echo ""
echo "Llave ECDSA P-256:"
openssl pkey -in intranet-ecdsa-p256.key -text -noout 2>&1 | grep "NIST CURVE"
echo ""
echo "Certificado - Vigencia:"
openssl x509 -in intranet.crt -noout -dates
echo ""
echo "Certificado - Fingerprint SHA-256:"
openssl x509 -in intranet.crt -noout -fingerprint -sha256
echo ""
echo "Coherencia llave-certificado:"
KEY_MOD=$(openssl rsa -in intranet-rsa4096.key -noout -modulus | openssl md5)
CRT_MOD=$(openssl x509 -in intranet.crt -noout -modulus | openssl md5)
[ "$KEY_MOD" = "$CRT_MOD" ] && echo "✓ Coherentes" || echo "✗ No coherentes"
```

---

## Solución de Problemas

### Problema 1: El certificado no incluye las extensiones X.509 v3 (SAN, KeyUsage, etc.)

**Síntoma:**

Al ejecutar `openssl x509 -in intranet.crt -text -noout`, la sección de extensiones está vacía o muestra `Version: 1 (0x0)` en lugar de `Version: 3 (0x2)`. No aparecen los campos `Subject Alternative Name`, `Key Usage` ni `Basic Constraints`.

**Causa:**

El comando `openssl x509 -req` fue ejecutado **sin** los parámetros `-extfile` y `-extensions`. Sin estos parámetros, OpenSSL genera un certificado X.509 versión 1 sin ninguna extensión, independientemente de lo que esté definido en el archivo `.cnf`.

**Solución:**

Regenera el certificado incluyendo obligatoriamente los parámetros de extensiones:

```bash
# Verificar que el comando incluye -extfile y -extensions
openssl x509 -req \
  -in intranet.csr \
  -signkey intranet-rsa4096.key \
  -days 365 \
  -sha256 \
  -extfile intranet-openssl.cnf \
  -extensions v3_ca \
  -out intranet.crt

# Confirmar que ahora es versión 3 con extensiones
openssl x509 -in intranet.crt -text -noout | grep -E "Version:|Subject Alternative Name"
```

Asegúrate también de que la sección `[ v3_ca ]` existe en el archivo `intranet-openssl.cnf` con el nombre exacto referenciado en `-extensions v3_ca`.

---

### Problema 2: Los módulos de la llave privada y el certificado no coinciden

**Síntoma:**

Al ejecutar la comparación de módulos del Paso 8, los hashes MD5 son diferentes:

```
(stdin)= a1b2c3d4e5f6...   ← llave privada
(stdin)= f6e5d4c3b2a1...   ← certificado  (¡DIFERENTE!)
```

El servidor TLS devuelve errores como `SSL_CTX_use_PrivateKey_file: key values mismatch` al intentar iniciar.

**Causa:**

Existen dos causas comunes:
1. Se generó una **nueva llave privada** después de haber creado el CSR, y el certificado fue firmado con el CSR original (que corresponde a la llave anterior).
2. Se especificó un archivo de llave incorrecto al generar el certificado (por ejemplo, se usó `intranet-ecdsa-p256.key` en lugar de `intranet-rsa4096.key`).

**Solución:**

Identifica qué llave corresponde al certificado comparando módulos:

```bash
# Identificar cuál llave coincide con el certificado
CRT_MOD=$(openssl x509 -in intranet.crt -noout -modulus | openssl md5)
echo "Módulo del certificado: $CRT_MOD"

for keyfile in *.key; do
  # Solo aplica para llaves RSA (ECDSA no tiene módulo RSA)
  KEY_MOD=$(openssl rsa -in "$keyfile" -noout -modulus 2>/dev/null | openssl md5)
  if [ "$KEY_MOD" = "$CRT_MOD" ]; then
    echo "✓ Coincidencia encontrada: $keyfile"
  fi
done
```

Si ninguna llave coincide, debes regenerar el CSR con la llave correcta y volver a generar el certificado:

```bash
# Regenerar CSR con la llave correcta
openssl req -new \
  -key intranet-rsa4096.key \
  -config intranet-openssl.cnf \
  -out intranet.csr

# Regenerar certificado
openssl x509 -req \
  -in intranet.csr \
  -signkey intranet-rsa4096.key \
  -days 365 -sha256 \
  -extfile intranet-openssl.cnf \
  -extensions v3_ca \
  -out intranet.crt
```

---

## Limpieza del Entorno

Al finalizar el laboratorio, los archivos generados **no deben eliminarse** si planeas continuar con las Prácticas 4 y 5, que dependen de los artefactos creados aquí. Sin embargo, sí debes asegurarte de que todos los archivos sensibles tienen los permisos correctos.

### Verificar y corregir permisos

```bash
cd ~/labs/certs

# Asegurar permisos 600 en todos los archivos sensibles
chmod 600 intranet-rsa4096.key intranet-ecdsa-p256.key intranet.pfx

# Verificar permisos finales
ls -la ~/labs/certs/
```

### Si deseas limpiar completamente (solo al terminar el curso)

```bash
# ⚠️ EJECUTAR SOLO AL FINALIZAR EL CURSO COMPLETO
# Esto eliminará todos los artefactos del laboratorio
rm -rf ~/labs/certs/
```

### Restaurar umask por defecto

```bash
# Restaurar umask estándar (si cerraste la terminal, ya está restaurado)
umask 022
```

---

## Resumen

En este laboratorio completaste el ciclo completo de creación y gestión de un certificado autofirmado para un servicio interno:

| Tarea completada | Herramienta / Comando principal |
|---|---|
| Llave RSA 4096 generada | `openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096` |
| Llave ECDSA P-256 generada (comparativa) | `openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256` |
| Archivo `.cnf` con SAN y extensiones v3 | Editor de texto + secciones `[v3_ca]` y `[alt_names]` |
| CSR creado con campos Subject correctos | `openssl req -new -key ... -config ...` |
| Certificado autofirmado X.509 v3 generado | `openssl x509 -req ... -extfile ... -extensions v3_ca` |
| Convertido a PKCS#12/PFX | `openssl pkcs12 -export` |
| Convertido a DER | `openssl x509 -outform DER` |
| Coherencia llave-certificado verificada | Comparación de módulos MD5 |
| Certificado validado estructuralmente | `openssl verify`, `openssl x509 -text` |

### Conceptos clave reforzados

- **`umask 077` y `chmod 600`** son la primera línea de defensa para llaves privadas. Este hábito debe ser automático en cualquier entorno.
- **Las extensiones X.509 v3** (especialmente SAN) no son opcionales en entornos modernos; los clientes TLS actuales las exigen para validar la identidad del servidor.
- **La comparación de módulos** es el método más rápido para diagnosticar un desajuste entre llave privada y certificado, un error frecuente en configuraciones de servidores TLS.
- **`genpkey` vs. comandos legado:** `openssl genpkey` es el estándar moderno que produce formato PKCS#8; `genrsa` y `ecparam` producen formato tradicional. Ambos son funcionales, pero PKCS#8 es preferible por su soporte a cifrado con PBKDF2.

### Próximos pasos

- **Práctica 3:** Importar el archivo `.pfx` generado en el almacén de certificados de Windows y configurar IIS o un servicio Windows para usarlo.
- **Práctica 4:** Configurar nginx con el certificado autofirmado para habilitar HTTPS en el servicio interno, usando los archivos `intranet.crt` e `intranet-rsa4096.key` generados en este laboratorio.

### Recursos adicionales

- [OpenSSL Cookbook (Ivan Ristić) — Capítulo sobre generación de llaves y certificados](https://www.feistyduck.com/library/openssl-cookbook/)
- [OpenSSL man page: openssl-req(1)](https://www.openssl.org/docs/man3.0/man1/openssl-req.html)
- [OpenSSL man page: openssl-x509(1)](https://www.openssl.org/docs/man3.0/man1/openssl-x509.html)
- [OpenSSL man page: openssl-pkcs12(1)](https://www.openssl.org/docs/man3.0/man1/openssl-pkcs12.html)
- [RFC 5280: Internet X.509 PKI Certificate and CRL Profile](https://www.rfc-editor.org/rfc/rfc5280)
- [NIST SP 800-131A Rev.2: Transitioning the Use of Cryptographic Algorithms and Key Lengths](https://csrc.nist.gov/publications/detail/sp/800-131a/rev-2/final)

---
