# Pruebas del servidor FTP

Estas pruebas se realizan desde el equipo anfitrión con un cliente FTP gráfico
(FileZilla o WinSCP) y con `openssl`. Antes de empezar, comprueba que las
máquinas están arrancadas y que el playbook FTP se ha ejecutado sin errores:

```bash
ansible-playbook ftp/ftp.yml --ask-vault-pass
```

## 1. Comprobar el certificado TLS

```bash
openssl s_client -connect ftp.raul.test:21 -starttls ftp -showcerts
```

En la salida se debe ver:

- El certificado presentado por el servidor.
- `subject=CN = ftp.raul.test` y `X509v3 Subject Alternative Name: DNS:ftp.raul.test`.
- Una cadena de confianza con el certificado autofirmado (advertencia normal).

## 2. Acceso anónimo rechazado

En FileZilla, crea un sitio con:

- Protocolo: FTP - File Transfer Protocol.
- Servidor: `ftp.raul.test`.
- Puerto: `21`.
- Usuario: `anonymous`.
- Contraseña: cualquiera.

El servidor debe rechazar la autenticación. En los logs del servidor:

```bash
sudo tail -n 30 /var/log/vsftpd.log
```

## 3. Acceso de `juan` y chroot

Crea un sitio con:

- Servidor: `ftp.raul.test`.
- Cifrado: **FTP explícito sobre TLS**.
- Usuario: `juan`.
- Contraseña: `juan`.
- Modo pasivo.

Comprobaciones:

1. La conexión se establece y el certificado se muestra con la advertencia
   esperada.
2. `pwd` muestra `/` (su directorio personal enjaulado).
3. Al intentar subir un nivel (`cd ..`) no se puede salir del directorio.
4. Se puede subir un archivo de prueba dentro de `/home/juan`.

## 4. Acceso de `ana` y publicación web

Crea un sitio con:

- Servidor: `ftp.raul.test`.
- Cifrado: **FTP explícito sobre TLS**.
- Usuario: `ana`.
- Contraseña: `ana`.
- Modo pasivo.

Comprobaciones:

1. `ana` no está enjaulada y su directorio inicial es `/var/www/html`.
2. Sube un archivo, por ejemplo `prueba-ana.html`, a `/var/www/html`.
3. El archivo debe quedar legible por Apache y con el grupo `webpublish`:

   ```bash
   ls -l /var/www/html/prueba-ana.html
   ```

   El resultado esperado es similar a:

   ```text
   -rw-rw-r-- 1 ana webpublish ... prueba-ana.html
   ```

4. Desde el navegador, abre `http://www.raul.test/prueba-ana.html` y comprueba
   que se sirve correctamente.

## 5. Transferencia segura comprobada

En FileZilla, el mensaje de estado debe indicar que la conexión usa TLS, por
ejemplo:

```text
Status: Connection established, waiting for welcome message...
Status: Initializing TLS...
Status: TLS connection established.
```

Si el cliente muestra un aviso de certificado desconocido, es normal: el
certificado es autofirmado.

## 6. Comprobaciones desde línea de comandos

También se puede probar la subida con `curl`:

```bash
# Listar el directorio de ana
curl --ssl-reqd -k -u ana:ana ftp://ftp.raul.test/

# Subir un archivo de prueba
echo '<h1>Prueba FTP</h1>' > /tmp/prueba-ana.html
curl --ssl-reqd -k -u ana:ana -T /tmp/prueba-ana.html ftp://ftp.raul.test/
```

Y comprobar que Apache lo sirve:

```bash
curl -fsS http://www.raul.test/prueba-ana.html
```

## 7. Resultado esperado

| Prueba                                   | Resultado esperado              |
|------------------------------------------|---------------------------------|
| Acceso anónimo                           | Rechazado                       |
| `juan` con contraseña correcta           | Acceso correcto y chroot        |
| `juan` intentando salir de su home       | Bloqueado                       |
| `ana` con contraseña correcta            | Acceso correcto en `/var/www/html` |
| Subida de `ana` a `/var/www/html`        | Correcta y visible por Apache   |
| Conexión sin TLS                         | Rechazada                       |
| Certificado presentado                   | `CN=ftp.raul.test`, autofirmado |
