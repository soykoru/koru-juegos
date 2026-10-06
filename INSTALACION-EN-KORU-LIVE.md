# Juegos propios de KORU (EL HUERTO y siguientes): instalar desde KORU Live

Encargo para la sesión de KORU Live (maru-desktop). Los juegos propios en Godot no se
instalan como un mod de Steam: son un `.zip` con un `.exe` que KORU Live descarga,
verifica, descomprime y abre. Después hablan con la app por **KORU Link v2** como
cualquier mod (EL HUERTO: id `huerto`, puerto `47812`, pasa `probador.py test 47812 --live --all` 47/47).

## 1. Dónde se publica (una vez por versión)

- Repositorio **público** `soykoru/koru-juegos`, que solo tiene los descargables (el código
  fuente NO se sube; vive en `PROYECTOS/KORU JUEGOS/<juego>`).
- Una release por versión: etiqueta `huerto-v0.1.0` con estos archivos:
  - `EL_HUERTO-0.1.0-windows.zip`
  - `logo.png`
  - `manifest.json`
- **Habrá varios juegos (un paquete).** Cada juego tiene su propia release (`huerto-v0.1.0`,
  `<otro>-v0.1.0`…) y hay **un solo manifiesto con TODOS** en la rama `main`, en una URL fija:
  `https://raw.githubusercontent.com/soykoru/koru-juegos/main/manifest.json`
  (o servido desde soykoru.net si se prefiere no depender de GitHub). Al publicar un juego
  nuevo o una versión nueva solo se añade/actualiza su entrada en `games[]`.

### `manifest.json`
```json
{
  "schema": 1,
  "updated": "2026-10-05",
  "games": [{
    "id": "huerto",
    "name": "El Huerto",
    "version": "0.1.0",
    "description": "Llena el huerto mientras el chat manda animales, tornados y estampidas.",
    "koruLink": { "id": "huerto", "port": 47812 },
    "windows": {
      "url": "https://github.com/soykoru/koru-juegos/releases/download/huerto-v0.1.0/EL_HUERTO-0.1.0-windows.zip",
      "sha256": "<hash del zip>",
      "size": 133378801,
      "exe": "EL_HUERTO/EL_HUERTO.exe"
    },
    "cover": "https://github.com/soykoru/koru-juegos/releases/download/huerto-v0.1.0/logo.png",
    "minKoruLive": "1.20.0"
  }]
}
```
El `manifest.json` real (con hash y tamaño) se genera en `huerto/build/` en cada build.

## 2. Qué tiene que hacer KORU Live

1. **Registrar el juego** en `koru_link_games.py`: `KoruLinkGameDef("huerto", "El Huerto", 47812)`.
   Relajar el `assert set(SPECS) == set(KORU_LINK_GAMES)` de `mod_installer.py:371` a `<=`
   (los juegos propios no tienen Spec de Steam). Añadir `huerto: 47812` al `GAME_PORTS` del probador.
2. **Leer el manifiesto** (una lista: mostrar todos los juegos como un catálogo, cada uno con su
   portada, descripción, versión y botón propio) al abrir la pestaña de juegos (caché de 1 h; si falla la red, usar la
   última copia guardada y decirlo, nunca mostrar la lista vacía sin motivo).
3. **Botón «Descargar»**, solo cuando el streamer lo pulsa (nada automático):
   - Descargar a `%LOCALAPPDATA%\KORU Live\juegos\huerto\.descarga\` con barra de progreso
     y reanudación (cabecera `Range`).
   - Comprobar **tamaño y SHA-256**. Si no coinciden: borrar y avisar «La descarga llegó dañada».
   - Descomprimir en `...\juegos\huerto\0.1.0\` (carpeta temporal + renombrar al final, así una
     instalación a medias nunca queda como válida). Rechazar rutas con `..` dentro del zip.
   - Guardar `instalado.json` con `{version, exe, sha256, fecha}`.
4. **Botón «Jugar»**: lanzar `exe` con el directorio del juego como carpeta de trabajo. El
   juego abre KORU Link en `127.0.0.1:47812` en unos 2-3 s; la app ya lo detecta con `/koru/hello`.
5. **Actualizar**: si `manifest.version` > `instalado.version`, mostrar «Actualizar». Descarga la
   nueva versión al lado, cambia `instalado.json` y borra la vieja solo si el juego no está abierto.
   Los ajustes y wins del juego no se pierden (viven en `%APPDATA%\Godot\app_userdata\EL HUERTO\`).
6. **Desinstalar**: cerrar el juego si está abierto y borrar `...\juegos\huerto\`, previa confirmación.
7. **Fotos**: además de las acciones, mandar la foto del espectador por
   `POST /koru/avatar {"id": user.id, "png": base64}` (PNG ≤ 256×256, ≤ 200 KB) **antes** o justo
   después de la primera acción de ese usuario. El juego la pone sobre todo lo que ese usuario mande.
   En `POST /koru/actions` enviar siempre `user.id` (además de `name`/`nick`).

## 3. Lo que el juego ya garantiza
- Responde en 2 ms; cada acción empieza en ≤ 10 ms; nada se ejecuta dos veces (idempotencia por `id`).
- Si llega un regalo durante una cinemática, **espera** y sale cuando termina; `action.done` se
  manda cuando de verdad sale en pantalla.
- Ráfagas: probado con 134 pedidos en 0,5 s durante un tornado → 134/134 terminados, 0 duplicados.
- `POST /koru/panic` vacía la cola y quita los animales.
- Funciona en PCs modestos: calidad automática (BAJA en gráficas integradas) y 60 fps fijos.
