# HTTP Lab 1: Virtual Hosts

**Autora:** Bea García Cogolludo
**Curso:** 2.º ASIR — IES Zaidín Vergeles

En esta práctica monto dos sitios web por nombre (*virtual hosts*) en un mismo servidor Apache: `apolo.olimpo.test` y `atenea.olimpo.test`. Para que los nombres funcionen uso un servidor DNS con BIND. Todo se levanta con Vagrant y VirtualBox.

---

## 1. Escenario

| Máquina | Nombre | IP | Función |
|---|---|---|---|
| `dns` | `ns1.olimpo.test` | 192.168.57.10 | Servidor DNS (BIND9) |
| `web` | `mercurio.olimpo.test` | 192.168.57.11 | Servidor web (Apache) |

- Red: `192.168.57.0/24`
- Dominio: `olimpo.test`
- El DNS tiene `dnssec-validation yes` y un forwarder a `1.1.1.1`.
- Las dos máquinas usan `ns1` como DNS y tienen `search olimpo.test` en el `resolv.conf`.

---

## 2. Estructura del proyecto

```
vagrant.student/
├── Vagrantfile
├── apache2-provision.sh
├── dns-provision.sh
└── files/
    ├── apolo.olimpo.test/
    │   ├── apolo.html
    │   ├── apolo1.txt
    │   ├── apolo2.txt
    │   └── messages/
    │       └── 404.html
    ├── atenea.olimpo.test/
    │   ├── atenea.html
    │   ├── atenea1.txt
    │   ├── atenea2.txt
    │   └── messages/
    │       └── 404.html
    ├── mercurio.olimpo.test/
    │   ├── apache2.conf
    │   ├── apolo.olimpo.test.conf
    │   ├── atenea.olimpo.test.conf
    │   └── resolv.conf
    └── ns1.olimpo.test/
        ├── 192.168.57.dns
        ├── named
        ├── named.conf.local
        ├── named.conf.options
        ├── olimpo.test.dns
        └── resolv.conf
```

Las carpetas `files/apolo.olimpo.test` y `files/atenea.olimpo.test` se sincronizan con la máquina `web` en `/var/www/apolo.olimpo.test` y `/var/www/atenea.olimpo.test` (lo hace el `Vagrantfile`).

---

## 3. Lo que he configurado

### 3.1. DNS: alias de apolo y atenea

En `files/ns1.olimpo.test/olimpo.test.dns`, debajo del comentario `; Poner alias de apolo`, he añadido los dos alias. Los dos apuntan a `mercurio`, que es la máquina con Apache:

```
; Poner alias de apolo
apolo     IN    CNAME   mercurio
atenea    IN    CNAME   mercurio
```

![Alias en olimpo.test.dns](img/01-dns-alias.png)

### 3.2. Páginas de error 404

Cada sitio tiene su página de error dentro de una carpeta `messages`:

`files/apolo.olimpo.test/messages/404.html`

```html
<h1>Resource not found on apolo.olimpo.test</h1>
```

`files/atenea.olimpo.test/messages/404.html`

```html
<h1>Resource not found on atenea.olimpo.test</h1>
```

Las he creado así:

```bash
mkdir files/apolo.olimpo.test/messages
echo "<h1>Resource not found on apolo.olimpo.test</h1>" > files/apolo.olimpo.test/messages/404.html
mkdir files/atenea.olimpo.test/messages
echo "<h1>Resource not found on atenea.olimpo.test</h1>" > files/atenea.olimpo.test/messages/404.html
```

![Páginas 404 creadas](img/02-paginas-404.png)

### 3.3. Virtual host de apolo

`files/mercurio.olimpo.test/apolo.olimpo.test.conf`

```apache
<VirtualHost *:80>
    ServerName apolo.olimpo.test
    DocumentRoot /var/www/apolo.olimpo.test

    <Directory /var/www/apolo.olimpo.test>
        Options +Indexes
    </Directory>

    ErrorDocument 403 "You do not have permission to access apolo.olimpo.test"
    ErrorDocument 404 /messages/404.html

    ErrorLog ${APACHE_LOG_DIR}/apolo.olimpo.test.error.log
    CustomLog ${APACHE_LOG_DIR}/apolo.olimpo.test.access.log combined
</VirtualHost>
```

- `ServerName`: el nombre al que responde el sitio.
- `DocumentRoot`: la carpeta con los ficheros del sitio.
- `Options +Indexes`: si se pide la raíz y no hay `index.html`, se muestra el listado de ficheros.
- `ErrorDocument 403`: mensaje personalizado para el error 403.
- `ErrorDocument 404`: página personalizada para el error 404.
- `ErrorLog` y `CustomLog`: logs propios del sitio.

![Virtual host de apolo](img/03-apolo-conf.png)

### 3.4. Virtual host de atenea

`files/mercurio.olimpo.test/atenea.olimpo.test.conf`

```apache
<VirtualHost *:80>
    ServerName atenea.olimpo.test
    DocumentRoot /var/www/atenea.olimpo.test

    <Directory /var/www/atenea.olimpo.test>
        Options -Indexes
    </Directory>

    ErrorDocument 403 "You do not have permission to access atenea.olimpo.test"
    ErrorDocument 404 /messages/404.html

    ErrorLog ${APACHE_LOG_DIR}/atenea.olimpo.test.error.log
    CustomLog ${APACHE_LOG_DIR}/atenea.olimpo.test.access.log combined
</VirtualHost>
```

Es igual que el de apolo, pero con `Options -Indexes`. Así el listado de ficheros está prohibido, y al pedir la raíz (o `/messages`) Apache devuelve el error 403 personalizado.

![Ficheros de configuración de mercurio](img/04-ficheros-conf.png)

### 3.5. Script de provisión de Apache

`apache2-provision.sh` venía con los comentarios y huecos `# tu código aquí`. Lo he completado siguiendo el mismo estilo que `dns-provision.sh`:

```bash
#
# Copiar ficheros de configuración de apache2 que incluyen:
#
# - apache2.conf,
# - apolo.olimpo.test.conf
# - atenea.olimpo.test.conf
#

cp -v /vagrant/files/mercurio.olimpo.test/apache2.conf /etc/apache2/
cp -v /vagrant/files/mercurio.olimpo.test/apolo.olimpo.test.conf /etc/apache2/sites-available/
cp -v /vagrant/files/mercurio.olimpo.test/atenea.olimpo.test.conf /etc/apache2/sites-available/

#
# Habilitar los servidores virtuales
#

a2ensite apolo.olimpo.test.conf
a2ensite atenea.olimpo.test.conf

#
# Reiniciar el servicio
#

systemctl restart apache2
systemctl status apache2
```

![Script de provisión de Apache](img/05-apache2-provision.png)

### 3.6. `apache2.conf`

El `apache2.conf` es el original de Debian, sin cambios. Lo saqué de la propia máquina `web` después del primer `vagrant up`:

```bash
vagrant ssh web -c "cp /etc/apache2/apache2.conf /vagrant/files/mercurio.olimpo.test/"
vagrant provision web
```

La primera vez, el `cp` de `apache2.conf` da error porque el fichero todavía no existe. Después de copiarlo y hacer `vagrant provision web` ya funciona bien. Como ahora está en el repositorio, al levantar el proyecto desde cero ya no pasa.

---

## 4. Puesta en marcha

Desde la carpeta del proyecto:

```bash
vagrant up
```

Crea las dos máquinas, instala BIND y Apache y ejecuta los scripts de provisión.

![vagrant up terminado](img/06-vagrant-up.png)

Después de `vagrant provision web`, Apache queda funcionando:

![Apache activo](img/07-apache-running.png)

---

## 5. Comprobaciones

### 5.1. DNS

```bash
vagrant ssh dns -c "dig apolo.olimpo.test"
vagrant ssh dns -c "dig atenea.olimpo.test"
```

Resultado de apolo (atenea sale igual):

```
;; ANSWER SECTION:
apolo.olimpo.test.     86400  IN  CNAME  mercurio.olimpo.test.
mercurio.olimpo.test.  86400  IN  A      192.168.57.11
```

![dig apolo.olimpo.test](img/08-dig-apolo.png)

### 5.2. Resolver los nombres desde mi ordenador

Mi ordenador no usa el DNS de las máquinas virtuales, así que añado los nombres a su fichero `hosts`:

```
192.168.57.11   apolo.olimpo.test atenea.olimpo.test
```

- **Linux:** `/etc/hosts`

  ```bash
  echo "192.168.57.11   apolo.olimpo.test atenea.olimpo.test" | sudo tee -a /etc/hosts
  ```

- **Windows:** `C:\Windows\System32\drivers\etc\hosts` (abrir el editor como administrador).

![dig atenea y /etc/hosts](img/09-dig-atenea-hosts.png)

### 5.3. Navegador

| Dirección | Resultado |
|---|---|
| `http://apolo.olimpo.test` | Listado de ficheros (apolo.html, apolo1.txt, apolo2.txt, messages/) |
| `http://apolo.olimpo.test/apolo.html` | Apolo site |
| `http://apolo.olimpo.test/index.html` | Resource not found on apolo.olimpo.test |
| `http://apolo.olimpo.test/messages/404.html` | Resource not found on apolo.olimpo.test |
| `http://atenea.olimpo.test/atenea.html` | Atenea site |
| `http://atenea.olimpo.test` | You do not have permission to access atenea.olimpo.test |
| `http://atenea.olimpo.test/index.html` | Resource not found on atenea.olimpo.test |
| `http://atenea.olimpo.test/messages` | You do not have permission to access atenea.olimpo.test |

Las 8 salen como pide el PDF.

Listado de ficheros de apolo:

![Listado de apolo](img/10-apolo-listado.png)

Error 404 personalizado de apolo:

![404 de apolo](img/11-apolo-404.png)

Error 403 personalizado de atenea:

![403 de atenea](img/12-atenea-403.png)

### 5.4. HURL

```bash
hurl --test --continue-on-error weblab-1-en.hurl
```

Se ejecutan 8 peticiones: pasan 6 y fallan 2.

| Test | Resultado | Motivo |
|---|---|---|
| `atenea.olimpo.test/soyuz.html` | Falla (404) | El fichero `soyuz.html` no aparece en el PDF ni en el `.tgz` |
| `atenea.olimpo.test` (403) | Falla | El `.hurl` busca el mensaje de **apolo**, pero el PDF pide el de **atenea**, que es el que devuelve el servidor |
| Los otros 6 | Pasan | |

He seguido lo que dice el PDF.

![Resultado de HURL](img/13-hurl.png)

---

## 6. Cómo lo he ido haciendo (Git)

He partido del proyecto original de la práctica y he hecho un commit por cada paso.

Proyecto inicial (los ficheros tal cual vienen en el `.tgz`):

![Proyecto inicial](img/00-proyecto-inicial.png)

Alias del DNS:

![Diff del DNS](img/01b-diff-dns.png)

Páginas 404:

![Páginas 404](img/02b-status-404.png)

Virtual hosts:

![Virtual hosts](img/03b-status-vhosts.png)

Script de provisión de Apache (en rojo lo que venía en la plantilla, en verde lo que he añadido):

![Diff del script de provisión](img/04b-diff-provision.png)

---

## 7. Entrega

Ficheros que pide el PDF (en la carpeta `entrega/`):

| Fichero entregado | Fichero en el proyecto |
|---|---|
| `named.conf.local` | `files/ns1.olimpo.test/named.conf.local` |
| `db.olimpo.test` | `files/ns1.olimpo.test/olimpo.test.dns` |
| `db.192.168.57` | `files/ns1.olimpo.test/192.168.57.dns` |
| `apolo.olimpo.test.conf` | `files/mercurio.olimpo.test/apolo.olimpo.test.conf` |
| `atenea.olimpo.test.conf` | `files/mercurio.olimpo.test/atenea.olimpo.test.conf` |
| `Vagrantfile` | `Vagrantfile` |
