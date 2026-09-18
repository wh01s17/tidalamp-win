# windows.md — migración de `tidalamp` a Windows

Documento de traspaso para el agente que ejecute la migración. Describe el punto de
partida, qué es exactamente lo que ata el proyecto a Linux, en qué orden desatarlo y
cómo comprobar cada paso. Está escrito para que se pueda ejecutar sin haber visto
nunca el repo de origen y sin releer el hilo en el que se decidió nada.

**Punto de partida:** copia literal de `wh01s17/tidalamp` v0.13.0 (commit `02ecbb3`),
con el historial reiniciado y `origin` apuntando a
`https://github.com/wh01s17/tidalamp-win.git`. Ni una línea de código migrada
todavía: al abrir este documento, el árbol es Linux puro.

---

## 1. La regla de oro

**Fork duro, pero diffeable.** El objetivo es una aplicación que corra en Windows,
no una que corra en los dos sitios. No hay que introducir una capa de abstracción
multiplataforma ni `if sys.platform == ...` repartidos por el código: eso duplica el
coste de cada cambio futuro y ninguno de los dos caminos queda bien probado.

Pero sí hay que **conservar los nombres de fichero, los nombres de clase y las firmas
públicas** de los módulos que se reescriban. `player.Mpv` sigue llamándose
`player.Mpv` y sigue teniendo `.volume`, `.position`, `.append()`, `.restart()`,
`.close()`. `mpris.MprisService` conserva `start()/stop()/publish()/publish_tracks()`
aunque por dentro no hable D-Bus. La razón es concreta: upstream (`tidalamp`) va a
seguir moviéndose, y con las firmas intactas un `git diff` o un cherry-pick de un
arreglo en `library.py`, `queue.py` o `screens/` sigue aplicando aquí sin conflictos.
Si se renombran los módulos, ese canal se cierra para siempre.

**Corolario:** cambia el *cuerpo* de las funciones, no su *contrato*. Cuando una
función no tiene equivalente en Windows, se deja con la misma firma devolviendo el
valor neutro (`""`, `None`, `False`, lista vacía) y se documenta en su docstring por
qué. El llamante ya sabe tratar ese caso —todo el código de origen está escrito para
que la falta de `cava`, de carátula o de MPRIS no pare la música.

---

## 2. Qué NO hay que tocar

De los ~17 100 líneas de `tidalamp/`, unas 13 000 son independientes del sistema
operativo. Ficheros que **no** requieren ningún cambio y en los que abrir un diff es
perder el tiempo y arriesgarse a romper la funcionalidad:

| Fichero | Qué es |
|---|---|
| `library.py` (1420) | navegación del catálogo TIDAL, caché de niveles |
| `queue.py` (409) | cola de reproducción y su persistencia JSON |
| `i18n.py` (1284) | catálogo es/en — *salvo* `_language()`, ver §5.11 |
| `screens/*` (~3000) | todas las ventanas de Textual |
| `widgets.py`, `layouts.py`, `scrolling.py`, `columns.py` | presentación |
| `net.py` | reintentos sobre `requests` |
| `about.py`, `lyrics.py`, `analyzer.py`, `settings.py` | lógica pura |
| `theme.py` | paletas — *salvo* dos funciones, ver §5.10 |

`app.py` (3232 líneas) es el caso intermedio: es neutro **excepto** por los métodos
`mpris_*` y la llamada a `MprisService`. Si se respeta la regla de oro (§1), `app.py`
no se toca en absoluto.

---

## 3. Inventario: de qué depende hoy el proyecto

| Dependencia del sistema | Dónde | Estado en Windows |
|---|---|---|
| `mpv` (binario) | `player.py` | **Existe.** Distinto mecanismo de IPC |
| socket Unix (`AF_UNIX`) | `player.py`, `tests/fake_mpv.py` | **No existe.** Named pipe |
| D-Bus + MPRIS | `mpris.py`, `dbus-fast` | **No existe.** SMTC, con pegas |
| PipeWire / PulseAudio (`pactl`) | `audio.py` | **No existe.** WASAPI vía mpv |
| `/proc/asound` | `audio.py` | **No existe.** Registro / mpv |
| `cava` | `spectrum.py` | **No existe.** Sin build Windows |
| `/etc/os-release` | `distro.py` | **No existe.** winget/scoop/choco |
| freedesktop `.desktop` + XDG_DATA_DIRS | `desktop.py` | **No existe.** Acceso directo `.lnk` |
| Omarchy | `theme.py`, `desktop.py` | **No existe.** Es una distro Linux |
| XDG Base Directories | `config.py` | **No aplica.** `%APPDATA%` / `%LOCALAPPDATA%` |
| Permisos POSIX (`chmod 0600`) | `auth.py`, `config.py` | **No aplica.** ACL / herencia |
| kitty graphics protocol | `artwork.py` | Solo WezTerm |
| sixel | `artwork.py` | Windows Terminal ≥ 1.22 |

---

## 4. Fases

Cada fase deja el repo en un estado comprobable. No pasar a la siguiente sin cumplir
el «criterio de hecho». El orden importa: F1 desbloquea el arranque, F2 desbloquea la
suite, y sin suite verde el resto se hace a ciegas.

| Fase | Qué entrega | Criterio de hecho |
|---|---|---|
| **F0** | Metadatos y limpieza | `pip install -e .` funciona en Windows |
| **F1** | Rutas + IPC + MPRIS neutralizado | `tidalamp tui` abre y reproduce una pista |
| **F2** | Suite adaptada | `pytest -q` verde en Windows |
| **F3** | Carátulas y terminal | Portada visible en Windows Terminal |
| **F4** | Paquetes, lanzador, permisos | Acceso directo en el menú Inicio |
| **F5** | Audio hi-res (WASAPI) | 24/96 confirmado bit-perfect |
| **F6** | CI y empaquetado | Workflow verde; instalador generado |
| **F7** | Documentación | README/plan/CHANGELOG coherentes |

---

## 5. Mapa de trabajo, fichero a fichero

### 5.1 `tidalamp/player.py` — el bloqueante principal · F1

Hoy: un `mpv --idle` de vida larga, hablado por **socket Unix** JSON IPC.

```python
self._sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
sock.connect(str(IPC_SOCKET))
```

`socket.AF_UNIX` no existe en Windows. mpv en Windows expone el mismo protocolo JSON
sobre un **named pipe**: `--input-ipc-server=\\.\pipe\tidalamp-mpv`.

**Qué hacer:**

1. **Extraer el transporte.** Antes de nada, sacar de `Mpv` las cuatro operaciones de
   transporte a una clase pequeña con este contrato exacto:

   ```python
   class _Transport:
       def connect(self, target: str, timeout: float) -> None: ...
       def send(self, payload: bytes) -> None: ...
       def readline(self, timeout: float) -> bytes | None: ...   # b"" = timeout, None = muerto
       def close(self) -> None: ...
   ```

   La distinción `b""` (mpv lento) vs `None` (mpv muerto) es la que hoy vive en
   `_TIMED_OUT` y en la que se apoyan `_stalled_since`, `STALL_LIMIT` y `restart()`.
   **Perderla rompe la recuperación de un mpv colgado**, que es una funcionalidad
   deliberada del proyecto (ver `plan.md §7`). Preservarla literalmente.

   Este paso es el de mayor apalancamiento de toda la migración: con el transporte
   inyectable, los tests pueden hablar TCP sobre `127.0.0.1` y `tests/fake_mpv.py`
   no necesita reescribirse como servidor de named pipes (§5.13).

2. **Implementar el transporte de named pipe sin dependencias nativas.** La vía
   simple, y la recomendada:

   ```python
   handle = open(r"\\.\pipe\tidalamp-mpv", "r+b", buffering=0)
   ```

   Un `open()` sobre un pipe funciona, pero **sus lecturas bloquean sin timeout**, y
   `TIMEOUT = 1.0` existe precisamente porque los ticks de la UI llaman desde el hilo
   del interfaz: una lectura sin techo congela la pantalla para siempre en vez de
   una vez. Solución: un hilo demonio que hace lecturas bloqueantes y empuja líneas a
   una `queue.Queue`; `readline(timeout)` pasa a ser `queue.get(timeout=...)`, que
   levanta `Empty` → se devuelve `b""`. Cuando el hilo ve EOF, encola un centinela y
   `readline` devuelve `None`.

   La alternativa es `pywin32` con E/S solapada (`win32file.ReadFile` +
   `win32event.WaitForSingleObject`). Da timeouts reales sin hilo extra, pero añade
   una dependencia binaria pesada. **Usar `pywin32` solo si el enfoque del hilo
   demuestra problemas medidos**, no por adelantado.

3. **Reintento de conexión.** Hoy `_connect()` espera a que `IPC_SOCKET.exists()`.
   Un named pipe no es un fichero del sistema de ficheros: `Path.exists()` sobre
   `\\.\pipe\...` no es fiable. Sustituir el sondeo por un bucle de intentos de
   apertura hasta `deadline`, tragando:
   - `FileNotFoundError` — mpv todavía no ha creado el pipe;
   - `PermissionError` / `OSError` con `winerror == 231` (`ERROR_PIPE_BUSY`) — hay
     que esperar y reintentar.

4. **Borrar `IPC_SOCKET.unlink(missing_ok=True)`** del `_spawn()`. Los named pipes los
   destruye el sistema al cerrarse el último handle; no hay fichero que borrar y la
   llamada lanzará o no hará nada según la ruta.

5. **Borrar `SOCKET_PATH_MAX = 107`** y la comprobación de longitud que lanza
   `MpvNotFound`. El límite de 108 bytes es de `sun_path`. Los nombres de pipe llegan
   a 256 caracteres y `\\.\pipe\tidalamp-mpv` mide 21: la comprobación ya no protege
   de nada. Borrar también la cadena traducida asociada en `i18n.py` («la ruta del
   socket de mpv es demasiado larga…»).

6. **Localizar el binario.** `shutil.which("mpv")` falla a menudo en Windows porque
   mpv rara vez está en el `PATH`. Encadenar respaldos, en este orden:
   `shutil.which("mpv")` → `%LOCALAPPDATA%\Microsoft\WinGet\Links\mpv.exe` →
   shim de scoop (`%USERPROFILE%\scoop\shims\mpv.exe`) → `%ProgramFiles%\mpv\mpv.exe`
   → una ruta explícita en el fichero de configuración (**añadir el ajuste
   `mpv_path`**, con su `TIDALAMP_MPV_PATH`, siguiendo el patrón de `ENV_VARS` en
   `config.py`). Solo cuando todo falla, `MpvNotFound` con el mensaje de §5.7.

   Usar **`mpv.exe`, no `mpv.com`**: el `.com` es el envoltorio de consola y abrirá
   una ventana negra encima del TUI.

7. **Sin ventana de consola.** Añadir a la llamada `subprocess.Popen` del `_spawn()`:

   ```python
   creationflags=subprocess.CREATE_NO_WINDOW
   ```

   Sin esto parpadea una consola en cada arranque y en cada `restart()`.

8. **Argumentos de mpv que se quedan como están.** No tocar `--af=@astats:...`,
   `--demuxer-lavf-o=protocol_whitelist=%NN%...`, `--cache=yes`,
   `--demuxer-readahead-secs`, `--prefetch-playlist=yes`. Cada uno documenta en el
   propio fuente el problema que resuelve y ninguno es específico de Linux. En
   particular, el escape `%<longitud>%` de la lista de protocolos **sigue siendo
   obligatorio** en Windows: es una peculiaridad del parser de mpv, no del SO.

   `--load-scripts=no` se puede mantener; su motivo original (evitar que un
   `mpv-mpris` del sistema publique un reproductor duplicado) desaparece, pero sigue
   siendo la opción correcta para un mpv embebido.

**Criterio de hecho:** `tidalamp tui` arranca, `x` reproduce, la posición avanza en
el display, `z`/`c`/`v` responden, y matar `mpv.exe` desde el Administrador de tareas
hace que la app lo relance conservando el volumen.

---

### 5.2 `tidalamp/mpris.py` — integración de escritorio · F1

Hoy: publica `org.mpris.MediaPlayer2.tidalamp` en D-Bus con `dbus-fast`, 419 líneas
que exponen tres interfaces (`Root`, `Player`, `TrackList`).

D-Bus no existe en Windows. El equivalente funcional son los **System Media Transport
Controls** (SMTC): lo que pinta el panel de multimedia al pulsar las teclas de
reproducción y lo que aparece en la superposición de volumen.

**Hay un obstáculo real y hay que saberlo antes de empezar:** SMTC se obtiene con
`SystemMediaTransportControls.GetForCurrentView()`, que exige un `CoreWindow`, o con
`ISystemMediaTransportControlsInterop.GetForWindow(hwnd)`, que exige un `HWND`. Una
aplicación de terminal no tiene ninguna de las dos cosas. Se puede crear una ventana
oculta con `ctypes`/`win32gui` y colgar SMTC de ella, pero es trabajo delicado con
bucle de mensajes propio y hostil a los tests.

**Decisión recomendada: neutralizar en F1, implementar en F5 o más tarde.**

1. Reescribir `mpris.py` conservando **el nombre del fichero, la clase
   `MprisService` y su API completa**: `start() -> str`, `stop()`, `publish()`,
   `publish_tracks()`, `bus_name`. La implementación no hace nada: `start()` devuelve
   `""`, el resto son `return None`. Con eso `app.py` no se toca ni una línea —
   `_start_mpris()`, `_mpris_ready` y los 30 métodos `mpris_*` siguen compilando y
   siguen siendo el punto de enganche cuando llegue SMTC.
2. Conservar el `Protocol` `PlayerBackend` tal cual. Es la documentación ejecutable
   de qué expone el reproductor, y SMTC va a leer exactamente lo mismo.
3. Quitar `dbus-fast` de las dependencias duras (§5.14).
4. Comprobar que `app.py:901` (`if name != "org.mpris.MediaPlayer2.tidalamp"`) no
   muestra un aviso espurio al usuario cuando `start()` devuelve `""`. Si lo hace,
   ese es el **único** cambio permitido en `app.py` en esta fase.

**Si se implementa SMTC (fase posterior):** usar el paquete `winsdk` (el `winrt`
original está abandonado). Mapear `Playing`/`Paused`/`Stopped` a
`MediaPlaybackStatus`, y `DisplayUpdater.MusicProperties` a título/artista/álbum. Ojo
a las unidades, que es donde se cae este tipo de puente: MPRIS usaba microsegundos y
volumen 0.0–1.0 mientras mpv usa porcentaje; SMTC usa `TimeSpan` (centenas de
nanosegundos). Documentar la conversión en el propio fuente como la documentaba
`mpris.py`.

---

### 5.3 `tidalamp/config.py` — rutas · F1

Hoy:

```python
def _xdg(var: str, default: str) -> Path:
    return Path(os.environ.get(var) or Path.home() / default)

CONFIG_DIR = _xdg("XDG_CONFIG_HOME", ".config") / "tidalamp"
CACHE_DIR  = _xdg("XDG_CACHE_HOME",  ".cache")  / "tidalamp"
STATE_DIR  = _xdg("XDG_STATE_HOME",  ".local/state") / "tidalamp"
```

**Qué hacer:**

1. Reescribir `_xdg()` conservando el nombre y la firma (lo importan `audio.py`,
   `desktop.py` y `theme.py`). Destinos:

   | Constante | Windows |
   |---|---|
   | `CONFIG_DIR` | `%APPDATA%\tidalamp` |
   | `CACHE_DIR` | `%LOCALAPPDATA%\tidalamp\cache` |
   | `STATE_DIR` | `%LOCALAPPDATA%\tidalamp\state` |

   Seguir honrando `XDG_CONFIG_HOME` y compañía **si están definidas** —quien viene
   de WSL o de dotfiles portables las tiene— pero sin usarlas de respaldo. Respaldo
   real: `Path(os.environ["APPDATA"])`, y si no existe (servicio, sesión rara),
   `Path.home() / "AppData/Roaming"`.

2. `IPC_SOCKET = CACHE_DIR / "mpv.sock"` **deja de ser una ruta**. Sustituir por:

   ```python
   IPC_PIPE = r"\\.\pipe\tidalamp-mpv"
   ```

   Si conviven dos instancias, el segundo mpv no puede crear el mismo pipe. Añadir el
   PID: `rf"\\.\pipe\tidalamp-mpv-{os.getpid()}"`. Es el mismo razonamiento que ya
   hace `mpris.py` al pedir un nombre de bus con sufijo de instancia.

   Buscar todos los usos de `IPC_SOCKET` (`player.py`, `tests/test_stream.py:134`)
   y actualizarlos.

3. `write_atomically()` — dos problemas, ambos silenciosos:
   - `os.replace()` en Windows **falla con `PermissionError`** si el destino está
     abierto por cualquier proceso, incluido un antivirus escaneando. Envolver en un
     reintento corto (3 intentos, 50 ms) antes de propagar.
   - `mode = path.stat().st_mode & 0o777` y `os.chmod(temporary, mode)` no significan
     nada: en Windows `chmod` solo conmuta el bit de solo lectura. No hace daño;
     simplificar a conservar únicamente ese bit y ajustar el docstring, que hoy
     promete «el fichero conserva su modo, y uno nuevo es 0644» — una mentira en
     cuanto se ejecute aquí.

4. `ensure_dirs()` y `setup_logging()` funcionan tal cual.

---

### 5.4 `tidalamp/auth.py` — permisos del fichero de sesión · F4

Hoy, y con muy buen motivo (guarda el token de acceso y el de refresco):

```python
os.chmod(path.parent, 0o700)
os.chmod(path, 0o600)
os.close(os.open(path, os.O_WRONLY | os.O_CREAT, 0o600))
```

En Windows los tres son **efectivamente no-ops**. `%APPDATA%` ya hereda una ACL que
solo concede al propio usuario y a SYSTEM/Administradores, así que el resultado por
defecto no es alarmante, pero la promesa del docstring («para su dueño y nadie más»)
deja de cumplirse tal como está escrita.

**Qué hacer:**

1. Reescribir `_tighten()` y `_private()` conservando nombre y firma.
2. Aplicar una ACL explícita, una sola vez, sobre `CONFIG_DIR`:
   ```
   icacls "<CONFIG_DIR>" /inheritance:r /grant:r "%USERNAME%":(OI)(CI)F
   ```
   vía `subprocess.run` con `CREATE_NO_WINDOW`. Alternativa sin depender del binario:
   `win32security` de `pywin32` — más correcto, otra dependencia.
3. **Que no pare el arranque.** El `_save()` actual ya traga `OSError` para que un
   directorio de solo lectura no impida reproducir; mantener esa garantía: si
   `icacls` falla, se registra en el log y se sigue. Una sesión que no se puede
   endurecer vale más que una app que no abre.
4. Actualizar los docstrings para que digan la verdad sobre el mecanismo de Windows.
   Son de los mejores del proyecto; que sigan siendo exactos.

---

### 5.5 `tidalamp/audio.py` — la pila de audio · F5

261 líneas enteramente PipeWire/PulseAudio: `pactl get-default-sink`,
`pactl list sinks`, `pactl list sink-inputs`, lectura de
`/proc/asound/cardN/stream0` para las tasas del DAC USB, y escritura de
`~/.config/pipewire/pipewire.conf.d` para forzar la tasa del grafo.

Ninguna línea sobrevive. Pero **el problema que resuelve sí sobrevive**: que la pista
de 24/96 llegue al DAC a 24/96 y no remuestreada. En Windows eso se llama otra cosa.

**El equivalente:**

- En **modo compartido** (el normal), el motor de audio de Windows remuestrea todo a
  la frecuencia del «Formato predeterminado» del dispositivo (Panel de control de
  sonido → Propiedades → Opciones avanzadas). Una pista de 96 kHz en un dispositivo
  configurado a 48 kHz se remuestrea, y nada dentro de la app puede evitarlo.
- En **modo exclusivo** WASAPI, mpv toma el dispositivo para sí y lo abre a la
  frecuencia de la fuente. Eso es bit-perfect y es el equivalente exacto de lo que
  hace hoy el módulo con PipeWire. Se activa con `--audio-exclusive=yes`.

**Qué hacer:**

1. Reescribir `audio.py` conservando los nombres de las funciones públicas que
   consumen `app.py` y `screens/config_window.py` (localizarlos con
   `grep -rn 'from .audio import\|audio\.' tidalamp/`). Implementaciones nuevas:
   - enumerar dispositivos con `mpv --audio-device=help` (parsear la salida, que es
     estable y tiene el prefijo `wasapi/`);
   - tasa efectiva: la propiedad `audio-out-params` que ya lee `player.py`; no hace
     falta nada más;
   - tasas soportadas: leer el formato predeterminado del dispositivo en
     `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio\Render\<guid>\Properties`
     con el módulo `winreg` (de la stdlib). Es un `WAVEFORMATEX` empaquetado; si
     resulta frágil, **es aceptable devolver una lista vacía** y que la ventana
     muestre «no disponible».
2. Añadir el ajuste `exclusive` (con `TIDALAMP_EXCLUSIVE`) al bloque `ENV_VARS` de
   `config.py`, y pasar `--audio-exclusive=yes` en `player.py::_spawn()` cuando esté
   activo. **Por defecto apagado:** en exclusivo ningún otro programa suena, y una
   app de música que enmudece las notificaciones sin avisar es un mal reporte de bug
   esperando a ocurrir.
3. La función que reinicia PipeWire (`audio.py:249`, «esto corta el audio un momento
   y mpv pierde su salida») **se borra**. En Windows el cambio de dispositivo lo
   gestiona mpv solo.
4. Revisar `screens/config_window.py` y `i18n.py`: hay cadenas visibles que hablan de
   PipeWire, del grafo y del resampling (p. ej. la entrada «PipeWire restart disabled
   by the environment» en `i18n.py:470`). Reescribirlas en términos de WASAPI o
   borrarlas junto con su par del catálogo.

**Criterio de hecho:** con `exclusive` activo y una pista de 24/96, la propiedad
`audio-out-params` de mpv reporta 96 000 Hz y el panel de sonido de Windows marca el
dispositivo como en uso exclusivo.

---

### 5.6 `tidalamp/spectrum.py` — el espectro de cava · F3

`cava` no tiene build de Windows y no la va a tener. El módulo lanza `cava -p` con
`method = pulse` y lee bytes crudos de su stdout.

**Qué hacer (lo barato y correcto):**

1. Dejar el fichero, la clase `Cava` y la excepción `SpectrumUnavailable` en su sitio.
   Hacer que `__init__` lance `SpectrumUnavailable` siempre, con un mensaje que diga
   que el espectro FFT es solo de Linux. El llamante **ya sabe degradar** al medidor
   RMS integrado: el propio docstring del módulo lo promete («si cava falta, muere, o
   el sink no se puede abrir, el llamante conserva el medidor RMS»). No hay que tocar
   `analyzer.py` ni `app.py`.
2. En `screens/config_window.py`, marcar la opción `visualizer = "spectrum"` como no
   disponible en lugar de ofrecerla y que falle.
3. Borrar `tidalamp/../tests/fake_cava.py` y `tests/test_spectrum.py`, o marcarlos con
   `pytest.mark.skipif(os.name == "nt", reason="cava es solo Linux")`. Preferible
   `skipif`: si algún día aparece una implementación, el test vuelve gratis.

**Ruta futura, si se quiere el espectro de verdad:** captura loopback de WASAPI
(`soundcard` o `pyaudiowpatch`) + FFT con `numpy`, todo dentro del proceso. Eso
además **arregla** la pega que el propio módulo documenta como conocida —cava escucha
el sink y no a mpv, así que muestra lo que suene en la máquina. Un loopback filtrado
por sesión escucharía solo a mpv. No es trabajo de esta migración.

---

### 5.7 `tidalamp/distro.py` — cómo se instala un paquete · F1

Hoy lee `/etc/os-release`, mira `ID` e `ID_LIKE` y devuelve
`sudo pacman -S mpv` o equivalente. Existe por una razón que sigue siendo válida:
`mpv` no es una dependencia de Python, pip no puede instalarlo, y el mensaje de error
tiene que decir cómo conseguirlo.

**Qué hacer:** conservar el nombre del fichero y las dos funciones públicas
`install_command(package) -> str` y `missing(package) -> str`. Cambiar la detección:

```python
_COMMANDS = {
    "winget": "winget install <ID>",
    "scoop":  "scoop install mpv",
    "choco":  "choco install mpv",
}
```

Detectar con `shutil.which("winget"|"scoop"|"choco")`, en ese orden de preferencia
(winget viene de serie en Windows 11 y en Windows 10 actualizado). Si no hay ninguno,
devolver `""` y que `missing()` caiga en la forma sin paréntesis, exactamente como
hace hoy en una distro no reconocida.

⚠️ **Verificar el identificador real de winget antes de escribirlo.** Ejecutar
`winget search mpv` y usar lo que salga: los IDs del repositorio cambian y `mpv.net`
es otra aplicación distinta. No escribir de memoria un ID que no se haya comprobado.

`cava` desaparece como paquete consultable (§5.6): si `missing("cava")` deja de
llamarse desde ningún sitio, borrar esa ruta.

---

### 5.8 `tidalamp/desktop.py` — el lanzador · F4

228 líneas que escriben un `.desktop` en `~/.local/share/applications`, rastrean
`XDG_DATA_DIRS` para no duplicar uno existente, y tratan Omarchy como caso especial
(`xdg-terminal-exec --app-id=TUI.tile`).

Todo el mecanismo es freedesktop. En Windows el equivalente es un **acceso directo
`.lnk`** en el menú Inicio.

**Qué hacer,** conservando `offer()`, `create()`, `existing()`, `command()`, `MARKER`
y sobre todo **la política**: se pregunta una vez, un «no» no se vuelve a preguntar,
un lanzador borrado a mano no se reescribe, y ningún fallo aquí puede impedir que se
abra el reproductor.

1. `data_dirs()` → devolver
   `%APPDATA%\Microsoft\Windows\Start Menu\Programs` (el del usuario) y
   `%ProgramData%\Microsoft\Windows\Start Menu\Programs` (el del sistema).
2. `existing()` → buscar `TidalAmp.lnk` en esos directorios. El escaneo por regex de
   `_EXEC` sobre el contenido no aplica: un `.lnk` es binario. Basta con el nombre.
3. `entry()` → ya no genera texto, sino que crea el acceso directo. Sin dependencias
   nuevas, vía COM con PowerShell:
   ```powershell
   $s = (New-Object -ComObject WScript.Shell).CreateShortcut($ruta)
   $s.TargetPath = "wt.exe"; $s.Arguments = "-- tidalamp tui"
   $s.IconLocation = $ico; $s.Save()
   ```
   Lanzar con `CREATE_NO_WINDOW`. Alternativa más limpia si ya se metió `pywin32`:
   `win32com.client.Dispatch("WScript.Shell")` directamente.
4. **El objetivo del acceso directo.** `tidalamp.exe` a secas se abre en `conhost`,
   donde el TUI se ve mal. Preferir Windows Terminal si existe
   (`shutil.which("wt.exe")`): `wt.exe -- tidalamp tui`. Si no, `tidalamp.exe tui` y
   asumir conhost. Este es el análogo directo de la rama Omarchy de hoy, y por el
   mismo motivo: el escritorio abre las apps de terminal a su manera.
5. `omarchy()` → **borrar la función y sus dos llamadas.** Omarchy es una distro de
   Linux; aquí no significa nada.
6. `_enabled()` → cambiar `sys.platform.startswith("linux")` por `== "win32"`.
   Conservar el respeto a `TIDALAMP_NO_DESKTOP_ENTRY`: `tests/conftest.py:20` lo pone
   para no escribir en el menú del desarrollador, y esa protección debe seguir
   funcionando aquí.
7. **Icono.** `_install_icon()` escribe un SVG en `icons/hicolor/scalable/apps`. Los
   `.lnk` no leen SVG. Generar `tidalamp.ico` multi-resolución (16, 32, 48, 256) a
   partir del SVG del paquete, guardarlo en `packaging/windows/` **y** dentro del
   paquete para que `importlib.resources` lo alcance, y apuntar `IconLocation` ahí.

---

### 5.9 `tidalamp/artwork.py` — carátulas en el terminal · F3

837 líneas con tres caminos: kitty graphics, sixel y medios bloques. La detección es
por `$TERM`, `$TERM_PROGRAM` y `$KITTY_WINDOW_ID`.

**El estado real en Windows:**

| Protocolo | Soporte |
|---|---|
| kitty graphics | Solo WezTerm. Ni Windows Terminal ni conhost |
| sixel | Windows Terminal ≥ 1.22. ConEmu parcial |
| medios bloques | Siempre |

Y **`$TERM` normalmente no está definida** en Windows, así que la detección actual
cae por defecto en medios bloques — que es lo correcto, pero por accidente.

**Qué hacer:**

1. Reescribir `detect_protocol()` conservando firma y el respeto a `configured`
   (el ajuste `TIDALAMP_ART`, que debe seguir pudiendo forzar cualquier protocolo).
   Señales nuevas: `WT_SESSION` (Windows Terminal), `WEZTERM_EXECUTABLE` /
   `TERM_PROGRAM == "WezTerm"`, `ConEmuANSI`.
2. Por defecto: medios bloques. Sixel solo con `WT_SESSION` presente **y** tras
   confirmar la versión de Windows Terminal — y aun así, dejarlo tras una opción
   explícita hasta haberlo visto funcionar en una máquina real. Un sixel emitido a un
   terminal que no lo entiende vomita basura sobre la pantalla; los medios bloques
   nunca fallan.
3. `_SEXTANT_TERMS` y la detección de sextantes (línea ~381): los caracteres
   U+1FB00–U+1FBFF no están cubiertos por Cascadia Code ni por las fuentes de consola
   habituales. **Desactivar sextantes por defecto en Windows** y quedarse en medios
   bloques; si no, el usuario ve una rejilla de cuadros vacíos donde iba la portada.
4. Pillow sigue siendo el decodificador y sigue siendo opcional
   (`pip install "tidalamp-win[art]"`). Sin cambios.

---

### 5.10 `tidalamp/theme.py` — dos funciones · F3

493 líneas de paletas puras, que se quedan. Solo dos funciones son de Linux:

- `omarchy_colors_path()` → devuelve
  `$XDG_STATE_HOME/omarchy/current/theme/colors.toml`. **Borrar**, junto con quien la
  llame y con `tests/test_theme.py:150`.
- `palette_dir()` → «donde viven las paletas portables del usuario». Reapuntar a
  `%APPDATA%\tidalamp\palettes` reutilizando el `_xdg()` nuevo de §5.3.

El formato TOML de paletas se conserva tal cual: es un formato de datos, y mantenerlo
compatible deja que un usuario se traiga sus paletas de una máquina Linux.

---

### 5.11 `tidalamp/i18n.py` — detección de idioma · F2

Una sola función, `_language()` (línea 1150). Hoy:

```python
locale_code = locale.getlocale(locale.LC_MESSAGES)[0]
```

**`locale.LC_MESSAGES` no existe en Windows.** El `except (ValueError, AttributeError)`
ya lo captura, así que no rompe nada — pero el resultado es que **un usuario de
Windows con el sistema en inglés ve la interfaz en español**, porque se cae al
`return "es"` final. Es un fallo silencioso y de cara al usuario.

**Qué hacer:** en la rama de respaldo, obtener el LCID de la interfaz de usuario y
traducirlo con la tabla que la propia stdlib trae:

```python
import ctypes, locale
lcid = ctypes.windll.kernel32.GetUserDefaultUILanguage()
code = locale.windows_locale.get(lcid)      # p. ej. "es_ES"
```

No usar `locale.getlocale()[0]` a secas: en Windows devuelve nombres como
`'Spanish_Spain'`, no `'es_ES'`, y el `split("_")[0].lower()` de la línea siguiente
daría `'spanish'`.

Conservar intacta la precedencia de `LANGUAGE`/`LC_ALL`/`LC_MESSAGES`/`LANG` sobre
todo lo demás: quien las define lo hace a propósito, y `tests/conftest.py` depende de
que `TIDALAMP_LANG` siga mandando.

---

### 5.12 `tidalamp/stream.py` y `tidalamp/cli.py` — arreglos menores · F1

**`stream.py:137` — un fd filtrado que en Windows sí duele:**

```python
path = Path(tempfile.mkstemp(dir=CACHE_DIR, prefix=..., suffix=".m3u8")[1])
```

`mkstemp` devuelve `(fd, ruta)` y aquí se descarta el `fd` **sin cerrarlo**. En Linux
pasa desapercibido. En Windows, `cleanup()` (línea 231) hará `unlink` sobre ficheros
que el propio proceso tiene abiertos y fallará con `PermissionError`, dejando los
`.m3u8` acumulándose en la caché para siempre. Arreglar:

```python
fd, name = tempfile.mkstemp(dir=CACHE_DIR, prefix=f"track-{track_id}-", suffix=".m3u8")
os.close(fd)
path = Path(name)
```

Y envolver el `stale.unlink(missing_ok=True)` del bucle de limpieza en un
`contextlib.suppress(OSError)`: si mpv todavía tiene abierta la playlist de la pista
en curso, Windows no deja borrarla y eso no es motivo para romper nada.

**`cli.py` — codificación de la consola.** Antes de que Textual tome la pantalla,
`typer.echo`/`typer.secho` escriben mensajes con `«»`, `ñ` y `—` (los errores de login
y de mpv, precisamente los que el usuario va a leer cuando algo falla). En consolas
Windows con la página de códigos heredada eso sale como basura. Al principio de
`main()`:

```python
for stream in (sys.stdout, sys.stderr):
    with contextlib.suppress(AttributeError, OSError):
        stream.reconfigure(encoding="utf-8")
```

Y documentar en el README que `PYTHONUTF8=1` es la solución global.
La entrada de consola (`[project.scripts]`) genera `tidalamp.exe` y funciona sin
cambios; **no** pasarla a `gui-scripts`, que le quitaría la consola a un TUI.

---

### 5.13 `tests/` — la suite · F2

47 ficheros de test. La mayoría son neutros. Estos no:

| Fichero | Problema | Arreglo |
|---|---|---|
| `fake_mpv.py` | servidor `AF_UNIX` | ver abajo |
| `test_player.py:117,126` | `_proc.send_signal(signal.SIGKILL)` | `signal.SIGKILL` no existe; usar `_proc.kill()` |
| `test_mpris.py` | bus D-Bus real | borrar, o reescribir contra el stub de §5.2 |
| `test_settings.py:156,162` | `locked.chmod(0o500)` para simular directorio no escribible | no-op en Windows: el test pasa sin probar nada. Sustituir por un `monkeypatch` que haga fallar la escritura, o `skipif` |
| `test_app_logout.py:35,72` | ídem | ídem |
| `test_desktop.py:115,117` | asume `/usr/bin/tidalamp` y `xdg-terminal-exec` | reescribir contra el `.lnk` de §5.8 |
| `test_distro.py` | fixtures de `/etc/os-release` | reescribir contra winget/scoop/choco |
| `test_audio.py` | fixtures de `pactl` | reescribir con §5.5 |
| `test_spectrum.py`, `fake_cava.py` | cava | `skipif` |
| `test_stream.py:134` | espera `IPC_SOCKET` como fichero en la caché | ajustar a `IPC_PIPE` |
| `conftest.py:~final` | `monkeypatch.setenv("XDG_DATA_HOME", ...)` | apuntar a la variable nueva |

**`fake_mpv.py` es el caso importante.** Es un doble excelente: responde el subconjunto
del JSON IPC que usa `player.Mpv`, emite eventos asíncronos por el mismo canal (la
parte del protocolo que ya ha causado bugs, porque la respuesta que se espera no es
necesariamente la primera línea que vuelve) y tiene tres variables de entorno para
portarse mal a propósito: `FAKE_MPV_HANG_ON`, `FAKE_MPV_EOF_ON`, `FAKE_MPV_SPLIT`.
Todo eso hay que conservarlo — es lo que prueba la recuperación de un mpv colgado.

Con el transporte ya extraído (§5.1, paso 1), la salida barata es **hacer que el fake
hable TCP sobre `127.0.0.1`**: el protocolo es idéntico, el fake cambia tres líneas
(`AF_UNIX` → `AF_INET`, `bind` a puerto 0, imprimir el puerto elegido) y no hace falta
implementar un servidor de named pipes en los tests. Los tests inyectan el transporte
TCP; producción usa el de pipe. El único riesgo, y hay que asumirlo conscientemente,
es que el transporte de pipe queda sin cobertura automática: **compensarlo con un
test de integración marcado, que lance un `mpv.exe` real y sea `skipif` cuando no
esté instalado.**

**Rutas largas.** `tmp_path` de pytest + `track-<id>-XXXXXX.m3u8` puede rozar el
límite de 260 caracteres de `MAX_PATH`. Si aparecen `FileNotFoundError` inexplicables,
habilitar rutas largas (`HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1`)
y documentarlo en `CONTRIBUTING.md`.

---

### 5.14 `pyproject.toml` · F0

```toml
name = "tidalamp-win"
description = "Windows terminal TIDAL client with hi-res playback and a retro TUI (requires mpv)"
keywords = [..., "windows", "wasapi"]    # quitar "mpris"
```

**Clasificadores:** quitar `"Operating System :: POSIX :: Linux"`. Añadir
`"Operating System :: Microsoft :: Windows"` y `"Environment :: Win32 (MS Windows)"`.

**Dependencias:** quitar `dbus-fast>=5.0.22`. Si se prefiere dejar la puerta abierta,
marcarla `; sys_platform == "linux"`, pero lo limpio en un fork de Windows es
borrarla. Añadir, solo si se llegan a usar: `winsdk` (extra `smtc`), `pywin32`
(extra `win32`).

**URLs:** las cuatro apuntan a `wh01s17/tidalamp`. Cambiarlas a `tidalamp-win`.

**Nombre del paquete importable:** `[tool.hatch.build.targets.wheel] packages =
["tidalamp"]` **se queda como está**. La distribución se llama `tidalamp-win`, el
módulo se sigue llamando `tidalamp`. Esto es lo que mantiene los diffs contra upstream
aplicables (§1) y no molesta a nadie: nadie va a instalar los dos en la misma máquina.

**Entrada de consola:** `tidalamp = "tidalamp.cli:main"` se puede dejar. Si preocupa
la colisión teórica con el paquete Linux, `tidalamp-win = ...` — pero entonces hay
que perseguir todos los sitios que escriben literalmente `tidalamp` en un mensaje al
usuario (`cli.py`, `desktop.py::command()`, `about.py`, el catálogo de `i18n.py`).
**Recomendación: dejar `tidalamp`** y ahorrarse esa persecución.

**Versión:** mantener `0.13.0` y seguir el minor de upstream. No usar un segmento
local tipo `0.13.0+win.0`: PyPI no acepta versiones locales.

**`[tool.hatch.build.targets.sdist]`:** quitar `packaging/` del `include` si se
elimina el AUR, o dejarlo apuntando a `packaging/windows/`.

**mypy:** añadir `platform = "win32"` a la configuración para que los tipos se
comprueben contra el SO correcto (sin esto, mypy en un runner Linux valida ramas que
aquí no existen).

---

### 5.15 `packaging/` · F6

**Borrar:** `packaging/aur/` entero (PKGBUILD + los cuatro tarballs, ~ el grueso de
los 31 MB del repo) y `packaging/tidalamp.desktop`. Reescribir
`packaging/README.md`.

**Crear `packaging/windows/`:**

1. `tidalamp.ico` — multi-resolución, generado del SVG del paquete (§5.8).
2. **PyInstaller** — un `tidalamp.spec` que produzca un ejecutable único para quien
   no quiere instalar Python. Ojo: PyInstaller y Textual conviven mal si no se
   declaran los datos del paquete; incluir `tidalamp/*.tcss` y el `.ico` como
   `datas`. mpv **no** se empaqueta: es un binario externo, es lo que dice la
   descripción del proyecto y es lo que espera `distro.py`.
3. **Manifiesto winget** (`manifests/w/wh01s17/tidalamp-win/<version>/`) — tres YAML
   (version, installer, locale). Solo cuando haya un instalador publicado con un
   `sha256` estable.
4. Opcional: script de **Inno Setup** para un `.exe` de instalación que además cree el
   acceso directo del menú Inicio, lo que dejaría `desktop.py` como respaldo para
   quien instale por pip.

---

### 5.16 `.github/workflows/` · F6

**`ci.yml`:**

1. Los tres jobs (`pytest`, `bare`, `lint`) → `runs-on: windows-latest`.
2. **Borrar el paso «Install dbus-daemon»** (`sudo apt-get install -y dbus`).
3. Añadir mpv al job `pytest`: `choco install mpv -y`, y **verificar que el paquete
   existe y que el runner lo resuelve** antes de dar el workflow por bueno. Si no,
   descargar un build oficial y añadirlo al `PATH` con `$env:GITHUB_PATH`.
4. ⚠️ **El job `bare` se rompe tal cual está.** Usa un heredoc de bash:
   ```yaml
   run: |
     python - <<'PY'
     ...
     PY
   ```
   En `windows-latest` la shell por defecto es `pwsh` y el heredoc no existe. Dos
   salidas: `shell: bash` en ese paso (Git Bash está instalado en los runners), o —
   mejor— mover el script a `tools/import_all.py` y llamarlo. La segunda además lo
   hace ejecutable a mano.
   Este job es valioso y hay que conservarlo: existe porque una dependencia dura sobre
   una opcional rompió la instalación documentada sin que ningún otro job se enterara
   (así desaparecieron las carátulas en un clon limpio). Los skips de la suite son la
   señal, no el problema.
5. `lint` puede quedarse en `ubuntu-latest` (ruff y mypy son estáticos y es más
   rápido), **pero mypy necesita `--platform win32`** (§5.14) o validará el SO
   equivocado.
6. Matriz de Python: mantener 3.11–3.14.

**`release.yml`:** el paso que compara el tag con la versión de `pyproject.toml` usa
sintaxis de shell POSIX (`${GITHUB_REF_NAME#v}`, `if [ ... ]`). Dejar ese job en
`ubuntu-latest` —solo construye un wheel puro, no necesita Windows— y ahorrarse la
reescritura. Publicación por Trusted Publishing: **hay que dar de alta el nuevo
proyecto `tidalamp-win` en PyPI y configurar su publisher OIDC**, porque el existente
está atado a `wh01s17/tidalamp`; si no, el `publish` fallará en el primer tag.

---

### 5.17 Documentación · F7

| Fichero | Qué cambiar |
|---|---|
| `README.md` | «Quick start» (`sudo apt install mpv` → `winget install ...`), sección «Arch Linux (AUR)» fuera, «Requirements» reescrito, el título y la descripción |
| `plan.md` | Es el documento de traspaso del proyecto. Reescribir su §7 (mpv IPC) y la sección de audio. **Poner la fecha y el estado reales de este fork**, no los heredados |
| `next.md` | Vaciar la cola heredada; la cola de este proyecto es §4 de este documento |
| `publish.md` | Rehacer contra PyPI `tidalamp-win` + winget, sin AUR |
| `CHANGELOG.md` | Entrada `[Unreleased]` nueva que abra con la migración. **Conservar el histórico de upstream**: explica por qué el código es como es |
| `CONTRIBUTING.md` | Preparar entorno en Windows (venv, mpv, Windows Terminal, rutas largas) |
| `img/*.webp` | Capturas de un terminal Linux. Rehacerlas en Windows Terminal. **No bloquea nada** — dejar para el final |

**Y lo que más importa, aunque no se versione:** `.agents/skills/` contiene
`tidalamp-mpv-ipc/SKILL.md` («`Mpv` es dueño de un proceso `mpv --idle` y su socket
IPC Unix») y `tidalamp-mpris-contract/SKILL.md` («MPRIS expone el estado de
reproducción a clientes de escritorio»). Están en `.gitignore`, así que no viajan por
git, **pero están en el disco y un agente los va a leer y los va a creer**. Reescribir
los dos junto con §5.1 y §5.2, o borrarlos. Un skill que describe la arquitectura
anterior es peor que no tener skill.

---

## 6. Checklist de verificación final

Manual, en una máquina Windows real. La suite verde no prueba ninguno de estos puntos.

- [ ] `pip install -e ".[dev]"` termina sin errores en Python 3.11 y 3.14
- [ ] `tidalamp --version` responde en un entorno sin sesión y sin mpv
- [ ] `tidalamp login` imprime la URL legible (acentos y `«»` correctos) y guarda sesión
- [ ] `tidalamp config` crea `%APPDATA%\tidalamp\config.toml` y lo lista
- [ ] `tidalamp tui` abre sin parpadeo de consola negra
- [ ] Reproduce una pista; la posición avanza; `z` `x` `c` `v` responden
- [ ] Cambio de pista sin corte (gapless) — es la razón de `--prefetch-playlist=yes`
- [ ] Matar `mpv.exe` desde el Administrador de tareas → la app lo relanza con el mismo volumen
- [ ] Una pista de 24/96 llega a 96 kHz con `exclusive` activo
- [ ] La portada se ve (medios bloques como mínimo)
- [ ] El visualizador cae al medidor RMS sin error visible
- [ ] Se ofrece el acceso directo una vez; un «no» no se repite en el siguiente arranque
- [ ] Interfaz en inglés en un Windows en inglés, en español en uno en español
- [ ] «Cerrar sesión» con la casilla borra config, cola, ecualizador, caché y acceso directo, y **nada fuera de `%APPDATA%`/`%LOCALAPPDATA%`**
- [ ] `%LOCALAPPDATA%\tidalamp\cache` no acumula `.m3u8` tras varias sesiones (§5.12)
- [ ] Todo lo anterior en Windows Terminal **y** en conhost

---

## 7. Trampas conocidas

Cosas que van a costar una tarde si no se saben de antemano.

1. **`b""` no es `None`.** En `player.py`, timeout y socket muerto son estados
   distintos y el segundo dispara `restart()`. Un transporte nuevo que los confunda
   hace que la app reinicie mpv cada vez que tarda un segundo en contestar, o que se
   quede colgada contra un mpv muerto. Es el primer sitio donde mirar si «va raro».
2. **La lista blanca de protocolos necesita el escape `%NN%`.** mpv parte las opciones
   clave-valor por comas; sin `%<longitud>%` la lista nunca llega a ffmpeg y todos los
   segmentos de una pista hi-res fallan con «Protocol 'https' not on whitelist». No es
   cosa de Linux y no se puede simplificar.
3. **`--cache=yes` es obligatorio.** Una pista hi-res llega como un `.m3u8` *local*
   cuyos segmentos son https: mpv mira la playlist, la considera local y desactiva la
   caché. Medido en el proyecto original: 1,02 s de buffer contra 6 Mbit/s. Con la
   caché explícita, 29,8 s.
4. **`os.replace()` falla si el destino está abierto.** Windows, no POSIX. Afecta a
   `write_atomically()` y por tanto a la config, la cola y el ecualizador. Un
   antivirus escaneando el fichero basta para provocarlo.
5. **`chmod` es una ilusión.** Cualquier test que use `chmod(0o500)` para simular un
   directorio no escribible **pasa sin probar nada** en Windows. Hay dos así hoy.
6. **`locale.LC_MESSAGES` no existe** y el `except AttributeError` lo esconde: el
   síntoma no es un error, es una interfaz en el idioma equivocado (§5.11).
7. **SMTC exige un HWND.** No hay atajo. Si el plan es «MPRIS pero para Windows»,
   presupuestarlo como trabajo propio, no como un ajuste.
8. **`MAX_PATH` = 260.** Aparece como `FileNotFoundError` sin sentido en los tests.
9. **`mpv.com` vs `mpv.exe`.** El `.com` abre una ventana de consola encima del TUI.
10. **conhost no es Windows Terminal.** Colores, redimensionado y sixel se comportan
    distinto. Probar en los dos antes de cerrar F3.

---

## 8. Estado

- [x] Copia literal de `tidalamp` v0.13.0 (`02ecbb3`)
- [x] `.venv`, `.mypy_cache`, `.pytest_cache`, `.ruff_cache`, `.coverage` y
      `__pycache__` eliminados de la copia
- [x] Historial git reiniciado; `origin` → `https://github.com/wh01s17/tidalamp-win.git`
- [ ] F0 · metadatos — [ ] F1 · arranque — [ ] F2 · suite — [ ] F3 · terminal
- [ ] F4 · lanzador — [ ] F5 · audio — [ ] F6 · CI — [ ] F7 · documentación
