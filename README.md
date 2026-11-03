# Punto 11 Café · Portal cautivo

Demostración académica de un portal cautivo para una cafetería ficticia, realizada por el **Equipo 11** en la práctica de **Redes**. El montaje usa **pfSense** como firewall y portal, con la interfaz de clientes **OPT1** (`192.168.77.1`) y la red `192.168.77.0/24`.

El proyecto tiene dos piezas independientes:

| Pieza | Archivo | Dónde se publica | Para qué sirve |
| --- | --- | --- | --- |
| Sitio de bienvenida público | `index.html`, `styles.css`, `assets/` | GitHub Pages, desde la raíz de `main` | Se muestra **después** de aceptar el portal. Es informativo. |
| Portal cautivo | `pfsense/portal.html` | Se pega en la zona `punto11cafe` de pfSense | Presenta la condición y envía la aceptación a pfSense. |

> **Importante:** GitHub Pages no concede acceso a la red. El sitio público no comprueba si el visitante pasó por pfSense, no usa `localStorage`, parámetros de URL ni formularios para simular una autenticación. La única prueba válida del control de acceso es la sesión que registra pfSense.

## Contenido del repositorio

```text
.
├── index.html              # sitio de bienvenida público
├── styles.css              # estilos del sitio público
├── assets/
│   └── favicon.svg         # ícono propio, sin dependencias externas
├── pfsense/
│   └── portal.html         # portal autocontenido para pfSense
├── .nojekyll               # evita el procesamiento de Jekyll en GitHub Pages
└── README.md
```

No hay React, Vite, npm, proceso de compilación, fuentes externas, CDN ni imágenes de terceros. Las rutas del sitio son relativas para funcionar tanto en local como bajo `/Punto-11-Cafe/`.

## Ver el sitio en local

Desde la raíz del repositorio:

```bash
python3 -m http.server 8080
```

Abre `http://localhost:8080/`.

Para comprobar el comportamiento bajo el subdirectorio real de GitHub Pages, sirve una carpeta temporal que contenga el proyecto como `Punto-11-Cafe/`:

```bash
mkdir -p /tmp/punto11-preview
ln -s "$PWD" /tmp/punto11-preview/Punto-11-Cafe
cd /tmp/punto11-preview
python3 -m http.server 8080
```

Abre `http://localhost:8080/Punto-11-Cafe/`. Los enlaces, el CSS y el favicon deben cargar sin errores.

Para revisar el portal solo visualmente, abre `pfsense/portal.html` en el navegador. El formulario no funcionará fuera de pfSense porque `$PORTAL_ACTION$` todavía no ha sido reemplazado; eso es esperado.

## 1. Publicar el sitio en GitHub Pages

1. Sube los cambios a la rama `main` del repositorio `Angixs-zz/Punto-11-Cafe`.
2. En GitHub, entra al repositorio y abre **Settings → Pages**.
3. En **Build and deployment**, elige **Source: Deploy from a branch**.
4. Selecciona **Branch: `main`** y **Folder: `/(root)`**. Pulsa **Save**.
5. Espera a que GitHub cree la publicación. El progreso aparece en la pestaña **Actions** y en **Settings → Pages**.
6. Comprueba la URL indicada por GitHub:

   ```text
   https://angixs-zz.github.io/Punto-11-Cafe/
   ```

La ruta distingue mayúsculas y guiones: debe ser exactamente `Punto-11-Cafe`. Mientras Pages no termine de publicar, la URL puede responder **404**; eso no significa que el portal de pfSense esté mal.

## 2. Configurar el portal en pfSense

Antes de empezar, confirma que la zona ya existe y que OPT1 tiene `192.168.77.1` con DHCP para `192.168.77.0/24`.

1. Entra a pfSense y ve a **Services → Captive Portal → Zones**.
2. Edita la zona **`punto11cafe`**.
3. Activa la zona si todavía no lo está.
4. Verifica que **Authentication Method** sea **`None, don't authenticate users`**. Este formulario está pensado para ese modo: no pide usuario ni contraseña.
5. Activa **Use custom captive portal page** si tu versión de pfSense muestra esa opción.
6. En **Portal page contents**, pega **todo** el contenido de `pfsense/portal.html`, desde `<!DOCTYPE html>` hasta `</html>`.
7. Guarda y aplica los cambios.

No modifiques ni escapes estas variables: pfSense las reemplaza al servir la página.

- `$PORTAL_ACTION$`
- `$PORTAL_REDIRURL$`
- `$PORTAL_ZONE$`

El formulario conserva la estructura necesaria:

```html
<form method="post" action="$PORTAL_ACTION$">
  <input type="hidden" name="redirurl" value="$PORTAL_REDIRURL$">
  <input type="hidden" name="zone" value="$PORTAL_ZONE$">
  <!-- casilla de condición y botón -->
  <button type="submit" name="accept" value="Conectarme a internet">...</button>
</form>
```

El botón `accept` envía la aceptación. La casilla es obligatoria (`required`) y, si JavaScript está disponible, el formulario también comprueba la condición antes de enviar.

No se incluyen páginas personalizadas de error ni de logout. Con este modo de autenticación no son necesarias para la demostración y pfSense puede usar sus páginas por defecto. Si más adelante se agregan, deben seguir siendo autocontenidas.

## 3. Configurar la redirección posterior

Cuando GitHub Pages ya esté publicado y la URL cargue, vuelve a la zona `punto11cafe` en pfSense y coloca en **After authentication Redirection URL**:

```text
https://angixs-zz.github.io/Punto-11-Cafe/
```

Guarda y aplica. Si el campo queda vacío, pfSense enviará al cliente a la URL que intentó abrir originalmente.

La URL debe configurarse **después** de comprobar que Pages responde. No uses el sitio público como método de autenticación: la autenticación ocurre en pfSense y la redirección solo se ejecuta después de aceptar el portal.

## 4. Probar desde un cliente conectado a OPT1

1. Desconéctate de otras redes y conéctate a la red de invitados servida por OPT1.
2. Comprueba que el cliente recibió una dirección `192.168.77.x`, con puerta de enlace `192.168.77.1`:
   - Linux: `ip addr` y `ip route`
   - Windows: `ipconfig`
   - macOS: `ifconfig` y `netstat -nr`
3. Antes de aceptar el portal, intenta abrir una dirección **HTTP**, por ejemplo `http://neverssl.com` o `http://example.com`. Debe aparecer el portal. Usa HTTP para la prueba: una página HTTPS puede no mostrarse igual antes de tener acceso.
4. Intenta pulsar **Conectarme a internet** sin marcar la casilla. El navegador debe impedir el envío y pedir que aceptes la condición.
5. Marca la casilla y pulsa **Conectarme a internet**. pfSense debe abrir la sesión y redirigir a `https://angixs-zz.github.io/Punto-11-Cafe/`.
6. Comprueba que ya puedes navegar por otras páginas.
7. Opcionalmente, cierra la sesión desde la página de logout de pfSense y verifica que el acceso vuelve a bloquearse.

### Comprobar la sesión en pfSense

Ve a **Status → Captive Portal** en pfSense. Debe aparecer una entrada para el cliente con, según la versión:

- dirección IP del cliente;
- dirección MAC;
- zona `punto11cafe`;
- hora de inicio de la sesión;
- tiempo restante (la sesión está configurada en 30 minutos);
- tráfico o estado de la sesión.

Esa entrada, junto con el bloqueo previo a la aceptación, es la prueba de que el portal funciona. **Abrir directamente la URL pública desde otra red solo muestra una página de bienvenida; no demuestra que el visitante haya pasado por pfSense ni que tenga acceso a la red.**

## Verificación realizada

Comprobado en este repositorio:

- Los documentos HTML se analizaron sin errores estructurales.
- El sitio público carga correctamente desde la raíz y desde un subdirectorio `/Punto-11-Cafe/`.
- `pfsense/portal.html` no contiene CSS, scripts, fuentes, imágenes ni otros recursos externos. Todos sus estilos y su ícono están incluidos en el propio archivo.
- El portal conserva las variables de pfSense sin escapar y el botón `name="accept"`.
- La condición es obligatoria: sin marcarla no se envía el formulario.
- El sitio público no tiene formularios, no usa `localStorage`, no lee parámetros de URL y no simula una verificación de sesión.
- Se revisó el diseño en anchos de celular y escritorio, la navegación por teclado y el contraste.
- Las rutas del sitio son relativas y no dependen de GitHub Pages para cargar el CSS o el favicon.

Pendiente de comprobar con un cliente real conectado a OPT1 y con pfSense en ejecución:

- Que pfSense sustituya correctamente `$PORTAL_ACTION$`, `$PORTAL_REDIRURL$` y `$PORTAL_ZONE$` al pegar la página en la zona.
- Que el portal aparezca al abrir una URL HTTP antes de aceptar.
- Que la redirección a GitHub Pages ocurra después de pulsar el botón.
- Que **Status → Captive Portal** registre la sesión y que el acceso se bloquee antes de aceptar.
- Que la duración de 30 minutos y los límites aproximados de 5000/2000 Kbit/s se apliquen como se espera.

## Privacidad y alcance

Esta primera demostración no recolecta datos personales. El portal solo envía a pfSense los campos que exige el portal cautivo y la aceptación de la condición; el sitio público no envía información a ningún servidor propio. GitHub Pages, como cualquier hosting, recibe las peticiones HTTP normales de sus visitantes según las políticas de GitHub.

No hay menú, precios, horarios, direcciones ni promociones reales. Punto 11 Café es el nombre del escenario académico del Equipo 11.
