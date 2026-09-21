# Despliegue en Dokploy — notas de este fork

Este fork existe para desplegar Breeze **detrás del Traefik de Dokploy**, en
lugar de dejar que su Caddy ocupe el borde. Todo lo que sigue documenta
decisiones que parecen errores si se leen sin contexto.

Rama de despliegue: **`dokploy`**, partiendo de la etiqueta **v0.114.0**.

---

## Los tres cambios sobre el original

### 1. Caddy no publica los puertos 80 y 443

En `docker-compose.yml` se eliminó el bloque `ports:` del servicio `caddy`.
Esos puertos ya son del Traefik de Dokploy; publicarlos hace fallar el arranque
por conflicto.

Caddy **se conserva** a propósito: reparte entre API, web y portal (`/portal`),
y replicar esas reglas en Traefik sería frágil. Traefik termina el TLS y reenvía
a Caddy por la red interna.

### 2. `CADDY_SITE_ADDRESS` ya no deriva de `BREEZE_DOMAIN`

La aplicación necesita el dominio público para construir URLs, pero Caddy debe
escuchar en texto plano hacia dentro y **no** intentar emitir su propio
certificado. El certificado lo gestiona Traefik.

### 3. Registro de accesos en `docker/Caddyfile.prod`

Añadido un bloque `log`. Sin él, cuando la API no reconoce una ruta registra
`route=unmatched` **sin decir qué ruta se pidió**, y un 308 se vuelve
indiagnosticable. Con el registro de Caddy queda la URL completa.

---

## `FORCE_HTTPS=false` — NO es un descuido

**Esta es la variable que alguien querrá "arreglar". No lo hagas sin leer esto.**

### El problema

Traefik descifra y habla con Caddy por HTTP. Avisa del origen cifrado con
`X-Forwarded-Proto: https`, y Caddy la recibe correctamente — pero **al reenviar
a la API reescribe esa cabecera con su propio esquema**, que es `http`.

Con `FORCE_HTTPS=true`, la API ve una conexión insegura y responde **308
Permanent Redirect** hacia HTTPS.

### Por qué el síntoma despista

| tipo de petición | comportamiento ante el 308 |
|---|---|
| GET y POST normales | la siguen: el navegador reintenta y **funciona** |
| **WebSocket** | **no sigue redirecciones**: la conexión muere |

Resultado: se puede iniciar sesión, el menú carga y el nombre de usuario
aparece, pero **todos los recuadros del panel muestran «Connection error»**,
porque dependen del canal en tiempo real.

### Comprobación que lo demuestra

Contra la API, con la cabecera `Host` del dominio real:

```
X-Forwarded-Proto: https  → HTTP/1.1 101 Switching Protocols   ✓
X-Forwarded-Proto: http   → HTTP/1.1 308 Permanent Redirect    ✗
```

### Por qué es seguro

El cifrado lo impone **Traefik**, que sólo sirve HTTPS y redirige el puerto 80
al 443. `FORCE_HTTPS` en la API es una segunda barrera que, en esta topología,
no puede saber que la conexión ya venía cifrada.

### Si algún día se quiere volver a poner en `true`

Habría que hacer que Caddy **preserve** la cabecera en lugar de reescribirla,
con un `header_up X-Forwarded-Proto https` en los bloques `reverse_proxy` que
alcanzan la API. No basta con `CADDY_TRUSTED_PROXIES` (ver abajo).

---

## `CADDY_TRUSTED_PROXIES=10.0.1.0/24` — sí hace falta

Es la subred de `dokploy-network`, donde vive Traefik.

**Lo que arregla:** que las IP reales de los clientes lleguen a la aplicación.
Sin esto, la API ve siempre la IP del proxy, y entonces los límites de intentos
de acceso cuentan en bloque para todos los usuarios y las listas de IP
permitidas no pueden funcionar.

**Lo que NO arregla:** el esquema (`http`/`https`). Esa confianza gobierna la
derivación de la IP del cliente, no la cabecera de protocolo. Asumir lo
contrario costó varias horas de diagnóstico.

Se usa la subred `/24` en lugar de la IP exacta de Traefik porque éste corre
bajo Swarm y su dirección puede cambiar al recrearlo, lo que rompería el acceso
sin avisar.

---

## Variables que hay que rellenar a mano

El instalador guiado del proyecto original las pregunta. Al montar el `.env`
desde la plantilla es fácil dejarlas mal — ocurrió con las cuatro:

| variable | error cometido | valor correcto |
|---|---|---|
| `PUBLIC_APP_URL` | quedó `https://breeze.yourdomain.com` | el dominio real |
| `DASHBOARD_URL` | ídem | el dominio real |
| `CORS_ALLOWED_ORIGINS` | quedó `https://app.yourdomain.com` | el dominio real |
| `BREEZE_BOOTSTRAP_ADMIN_*` | quedaron vacías | correo y contraseña **de 16+ caracteres** |

`BREEZE_DOMAIN` **no** propaga a las demás: son independientes.

`DATABASE_URL` sí puede quedarse con su valor de plantilla: el compose la
reconstruye apuntando a `postgres:5432`, y así lo indica el propio `.env.example`.

### Credenciales de arranque

`BREEZE_BOOTSTRAP_ADMIN_EMAIL` y `BREEZE_BOOTSTRAP_ADMIN_PASSWORD` sólo se usan
para crear el primer usuario. **Vaciarlas después de completar el asistente** y
redesplegar: no deben quedarse guardadas en el panel de Dokploy.

---

## Referencias de imagen

El compose exige imágenes fijadas por digest (`BREEZE_*_IMAGE_REF`), y la
plantilla trae marcadores. El instalador original los resuelve; al desplegar
desde Dokploy hay que resolverlos a mano contra el registro:

```sh
for img in api web portal binaries; do
  TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:lanternops/breeze/$img:pull&service=ghcr.io" \
    | python3 -c 'import json,sys; print(json.load(sys.stdin)["token"])')
  curl -sI -H "Authorization: Bearer $TOKEN" \
    -H "Accept: application/vnd.oci.image.index.v1+json" \
    "https://ghcr.io/v2/lanternops/breeze/$img/manifests/0.114.0" \
    | grep -i '^docker-content-digest'
done
```

Al subir de versión: cambiar la etiqueta en ese comando, actualizar los cuatro
digests y `BREEZE_VERSION`.

---

## Configuración en Dokploy

- **Tipo:** Compose (no Stack — Swarm ignora `container_name`, las condiciones
  de salud en `depends_on` y los secretos por fichero)
- **Repositorio:** este fork · **rama:** `dokploy` · **compose:** `./docker-compose.yml`
- **Dominio:** servicio `caddy`, puerto `80`, HTTPS con Let's Encrypt

Debe desplegarse **desde git**, no pegando el compose: hay ficheros montados por
ruta relativa (`docker/Caddyfile.prod` y `docker/secrets/.empty-jwk`) que sólo
existen si se clona el repositorio.

---

## Pendiente

**Escritorio remoto.** Necesita el servidor TURN, que va por **UDP 3478** y no
puede pasar por Traefik: hay que publicarlo directamente y abrir el puerto en
`ufw`, con `TURN_HOST` apuntando a la IP pública.
