# FTP con Vagrant y Ansible como nodo controlador

**Alumno:** Raúl Calvo Santos  
**Dominio de la práctica:** `raul.test`  
**Repositorio:** `https://github.com/raulcalvosantos2007-lang/FTP-RaulCalvoSantos`  
**Fecha:** noviembre de 2026

Este README presenta el proyecto y explica cómo ponerlo en marcha. El desarrollo
detallado, los resultados de las comprobaciones y las capturas reales están en la
[memoria de la práctica](docs/MEMORIA.md). El índice de imágenes y cómo-obtenerlas
está en la [guía de capturas](docs/capturas/README.md).

## Objetivo del proyecto

Desplegar de forma automatizada un entorno de prácticas con tres máquinas virtuales
Debian Bookworm gestionadas por Vagrant:

| Máquina | Dirección | Función |
|---|---|---|
| `dns.raul.test` | `192.168.57.5` | Servidor DNS con BIND9 (zonas directa e inversa). |
| `ansible.raul.test` | `192.168.57.6` | Nodo de control de Ansible. |
| `negocio.raul.test` | `192.168.57.7` | Apache2 y servidor FTP seguro `vsftpd` con TLS. |

Los alias `www.raul.test` y `ftp.raul.test` son CNAME de `negocio.raul.test`.
Toda la instalación y configuración se realiza mediante playbooks de Ansible.

## Herramientas utilizadas

- **GitHub**: control de versiones, issues y GitHub Projects.
- **Vagrant** y **Oracle VirtualBox**: infraestructura virtual reproducible.
- **Ansible**: aprovisionamiento de los servicios.
- **Ansible Vault**: cifrado de los hashes de contraseñas.
- **BIND9**: DNS directo e inverso.
- **Apache2**: servidor web.
- **vsftpd** + **TLS**: servidor FTP seguro.
- **community.crypto**: generación del certificado autofirmado.
- **FileZilla/WinSCP o `ftplib` con FTP_TLS**: pruebas del servicio FTP.

## Requisitos

- Windows 10/11 con virtualización activada.
- Vagrant 2.4.x y Oracle VirtualBox 7.x.
- Git y OpenSSH disponibles en el anfitrión.
- Conexión a Internet para descargar paquetes y la colección `community.crypto`.
- Al menos 3 GB de RAM libres para las tres máquinas virtuales.

## Puesta en marcha básica

1. Desde la raíz del repositorio:

   ```powershell
   vagrant up
   ```

   Vagrant crea las tres máquinas y ejecuta `script/ansible.sh` en el nodo
   `ansible`: instala Ansible y copia las claves privadas de `dns` y `negocio`
   dentro de la VM (nunca se suben al repositorio).

2. En el nodo de control:

   ```powershell
   vagrant ssh ansible
   ```

   Desde allí se pueden ejecutar, en orden:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ansible-inventory --graph
   ansible all -m ping
   ansible all -b -a "uptime"
   ansible-playbook dns/dns.yml
   ansible-playbook web/web.yml
   ansible-playbook ftp/ftp.yml --vault-password-file ./.vault_pass.txt
   ```

3. Comprobar los servicios con los scripts de prueba incluidos:

   ```bash
   ./dns/test-dns.sh
   ./web/test-web.sh
   ```

   Las comprobaciones FTP y el resultado esperado están documentados en
   [`ftp/PRUEBAS.md`](ftp/PRUEBAS.md).

4. Para que el navegador del anfitrión resuelva los nombres `*.raul.test` hay que
   configurar el DNS primario con `192.168.57.5` (o añadir entradas al archivo
   `hosts`). El procedimiento y las capturas reales están en
   [`docs/MEMORIA.md`](docs/MEMORIA.md).

## Documentación y capturas

- [`docs/MEMORIA.md`](docs/MEMORIA.md) — memoria del trabajo realizado, con las
  capturas incrustadas en el texto.
- [`docs/capturas/README.md`](docs/capturas/README.md) — guía e índice de las
  capturas, enlazada con cada apartado de la memoria.
- [`docs/diagrama-arquitectura.svg`](docs/diagrama-arquitectura.svg) y
  [`docs/diagrama-publicacion.svg`](docs/diagrama-publicacion.svg) — esquemas del
  despliegue y del flujo de publicación.

Los archivos de instalación, pruebas y configuración están en `dns/`, `web/` y
`ftp/`, cada uno en su carpeta correspondiente. No se incluyen registros de
ejecución ni scripts auxiliares: solo los archivos pedidos en el enunciado y la
documentación con sus capturas.

## Seguridad de esta práctica

`ansible.cfg` desactiva la comprobación de claves de host SSH y usa el usuario
`vagrant` con una clave conocida. Ello simplifica el laboratorio, pero **no es
apto para producción**. Las contraseñas de los usuarios FTP se guardan como
hashes SHA-512-crypt dentro de `ftp/vars/vault.yml`, cifrado con Ansible Vault.
La contraseña maestra no se sube al repositorio: se mantiene en `vault_pass.txt`,
ignorado por Git. La plantilla sin cifrar es `ftp/vars/vault.yml.example`.

## Autor y licencia

Raul Calvo Santos — 2.º DAW. El proyecto se distribuye bajo licencia MIT
([`LICENSE`](LICENSE)).
