<img src="images/neteclogo (2).png" alt="logo" width="300"/>

# Certificados Digitales: Uso, Gestión y Firma

## Plataforma de laboratorios

Te damos la bienvenida a la **plataforma de laboratorios** del curso **Certificados Digitales: Uso, Gestión y Firma**. Aquí podrás explorar diferentes tecnologías a través de prácticas guiadas. ¡Desarrolla tus habilidades y lleva tus conocimientos al siguiente nivel!

Este curso ofrece una visión integral sobre el uso, administración y firma de certificados digitales en entornos Windows, Linux y nube. A lo largo del curso, los participantes comprenderán el funcionamiento de la PKI, la gestión de llaves y certificados X.509, el uso de herramientas como OpenSSL y PowerShell, así como su integración en aplicaciones, APIs y servicios cloud.

## Lista de laboratorios

Cada uno de estos laboratorios está diseñado para ofrecerte una experiencia práctica. Haz clic en los enlaces para comenzar.

### Capítulo 1

- [Analizar un certificado HTTPS real e identificar emisor, vigencia, algoritmos y cadena de confianza.](Capitulo01/README.md#analizar-un-certificado-https-real-e-identificar-emisor-vigencia-algoritmos-y-cadena-de-confianza)
  
- **Descripción**: En esta práctica el estudiante inspeccionará certificados digitales X.509 de sitios HTTPS públicos utilizando el navegador web y la herramienta de línea de comandos openssl s_client. Se identificarán los campos principales del certificado (Subject, Issuer, Validity, Public Key, Extensions), se trazará la cadena de confianza desde el certificado de entidad final hasta la CA raíz, y se relacionarán los algoritmos criptográficos encontrados (RSA, ECDSA, SHA-256, etc.) con los conceptos de hash, cifrado asimétrico y firma digital vistos en la lección 1.1. Al finalizar, el estudiante completará una tabla comparativa de tres certificados reales que le servirá de referencia para las prácticas posteriores.
  
- ⏱️ **Duración estimada**: 25 min

### Capítulo 2

- [Crear un certificado autofirmado para un servicio interno y validarlo con OpenSSL.](Capitulo02/README.md#crear-un-certificado-autofirmado-para-un-servicio-interno-y-validarlo-con-openssl)
  
 - **Descripción**: En este laboratorio asumirás el rol de administrador de infraestructura de una empresa que necesita proteger un portal de monitoreo interno (intranet.empresa.local). Partiendo desde cero, generarás llaves privadas RSA 4096 y ECDSA P-256, crearás un archivo de configuración OpenSSL con extensiones X.509 v3 completas (SAN, BasicConstraints, KeyUsage), producirás un certificado autofirmado válido por 365 días y lo convertirás a los formatos PFX/PKCS#12 y DER. Finalizarás inspeccionando y validando el certificado con comandos OpenSSL para confirmar su estructura, vigencia y coherencia con la llave privada.
   
- ⏱️ **Duración estimada**: 20 min

### Capítulo 3

- [Crear, instalar y exportar un certificado desde Windows para uso local.](Capitulo03/README.md#crear-instalar-y-exportar-un-certificado-desde-windows-para-uso-local)

- **Descripción**: En esta práctica explorarás el almacén de certificados de Windows en sus dos ámbitos principales —usuario actual (CurrentUser) y equipo local (LocalMachine)— utilizando certmgr.msc, certlm.msc y la consola MMC. Generarás un certificado autofirmado mediante el cmdlet New-SelfSignedCertificate de PowerShell, lo instalarás en los almacenes apropiados y lo exportarás en los formatos .cer (solo clave pública) y .pfx (con clave privada protegida por contraseña). Finalmente, simularás la recepción e importación del certificado en otro almacén, verificando su correcta instalación y comprendiendo las diferencias de seguridad entre ambos formatos de exportación.
  
- ⏱️ **Duración estimada**: 20 min

### Capítulo 4

- [Firmar un archivo y validar su integridad; después habilitar HTTPS en un servicio de prueba.](Capitulo04/README.md#firmar-un-archivo-y-validar-su-integridad-después-habilitar-https-en-un-servicio-de-prueba)
  
- **Descripción**: En esta práctica aplicarás directamente los conceptos de firma digital y HTTPS trabajados en el curso. En la Parte A usarás la llave privada y el certificado autofirmado generados en la Práctica 2 para firmar digitalmente un archivo de texto, verificarás la firma con la clave pública y simularás una manipulación del archivo para comprobar que la verificación falla. En la Parte B levantarás un servidor HTTPS local usando Python 3 con ssl.SSLContext, accederás a él con curl y desde el navegador, interpretarás los errores por certificado no confiable y aprenderás a añadir el certificado autofirmado como CA local de confianza. Al final revisarás conceptualmente la configuración de mTLS.
  
- ⏱️ **Duración estimada**: 20 min

### Capítulo 5

- [Diagnosticar un fallo TLS simulado y proponer una ruta de corrección en nube u on-prem.](Capitulo05/README.md#diagnosticar-un-fallo-tls-simulado-y-proponer-una-ruta-de-corrección-en-nube-u-on-prem)
  
- **Descripción**: En esta práctica el estudiante configurará deliberadamente tres escenarios de fallo TLS en un servidor HTTPS local, los diagnosticará con herramientas de línea de comandos (openssl s_client, curl, openssl x509) y completará una ficha de diagnóstico para cada escenario. A continuación, propondrá rutas de corrección concretas tanto para entornos on-premise como para Azure Key Vault y AWS Certificate Manager, aplicando los conceptos de gestión de ciclo de vida vistos en la lección 5.1. La práctica culmina con una revisión de estrategias de monitoreo de expiración y buenas prácticas de protección de llaves privadas.
  
- ⏱️ **Duración estimada**: 30 min


## 📬 **Contacto y más información**


Si tienes alguna pregunta o necesitas más detalles, no dudes en [contactarnos](mailto:soporte@netec.com). También puedes encontrar más recursos en nuestra [página](https://netec.com).

---

¡Gracias por visitar nuestra plataforma! No olvides revisar todos los laboratorios y comenzar tu viaje de aprendizaje hoy mismo.
