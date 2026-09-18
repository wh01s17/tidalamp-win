# next.md — lo que entra en la próxima versión

Cola de trabajo comprometido, no una lista de deseos. Cada entrada trae lo que hay que
hacer, por qué, las trampas que ya se conocen y cómo se comprueba, para que se pueda
retomar sin releer el hilo en el que se decidió.

**Cómo se usa este fichero.** Cuando algo de aquí queda hecho, **se borra de aquí** y
pasa a dos sitios: la línea de usuario a `CHANGELOG.md`, bajo `[Unreleased]`, y el
detalle de diseño y las decisiones a la sección que le corresponda de `plan.md`. Un
fichero que acumula entradas tachadas deja de decir qué falta, que es lo único para lo
que existe. Si algo se descarta, no se borra sin más: baja a «Descartado por ahora» con
el motivo.

**En este fork, la cola es la migración.** Mientras las fases F0–F7 de
[`windows.md`](./windows.md) no estén cerradas no hay funcionalidad nueva que planear:
lo que hay es un cliente de TIDAL para Linux que todavía no arranca en Windows. Este
fichero no duplica esas fases —viven en `windows.md` §4, con su criterio de hecho— sino
que recoge lo que **no cabe** allí: las decisiones abiertas, lo que hay que comprobar a
mano y lo que se descartó por el camino.

---

## Para la próxima versión

**Nada publicable todavía.** La versión `0.13.0` del `pyproject.toml` es la heredada de
upstream y describe un árbol Linux. No se etiqueta ni se sube nada hasta cerrar F2
(suite verde en Windows) como mínimo, y en la práctica hasta F4.

### Comprobaciones que solo se hacen a mano

La suite no cubre ninguna de estas, y ninguna se puede automatizar desde aquí. Van
contra la lista de `windows.md` §6, que es la versión larga; aquí queda lo que
**necesita criterio humano**, no solo ejecutar un comando.

- **El TUI en conhost.** Todo el desarrollo va a pasar en Windows Terminal, que es lo
  razonable, y por eso mismo conhost se va a quedar sin mirar hasta que lo rompa un
  usuario. Colores, redimensionado y velocidad de repintado se comportan distinto.
  *Qué mirar:* las cuatro disposiciones (`quattro`, `nova`, `retro`, `ascii`), que la
  fila del transporte no se salga ni se parta, y que la carátula en medios bloques no
  quede una losa gris —ya mordió una vez en Linux, ver `plan.md` §7.

- **La disposición compacta.** Heredada sin comprobar de upstream: por debajo de 80×26
  la interfaz quita la carátula y la fila de balance, y solo la cubren tests que miden
  celdas y no colores (`plan.md` §9.5). En Windows hay un motivo nuevo para mirarla: la
  fuente por defecto de la consola no es la de kitty, y los anchos pueden no cuadrar.
  *Comprobación:* capturas entre 60×18 y 79×25 en los cuatro temas.

- **Un DAC USB real a 24/96 en modo exclusivo.** Es la comprobación que da sentido a F5
  y la única que prueba que el proyecto sigue siendo lo que dice ser. En Linux se hizo
  con un FiiO BTR15 mostrando `PCM 176.4K` en su pantalla; aquí hace falta el
  equivalente. Sin un DAC que informe del rate recibido, F5 se queda en «el código
  parece correcto», que no es lo mismo.

- **Una pista hi-res de principio a fin, sin cortes.** El paso de una pista a otra usa
  `--prefetch-playlist=yes` y la cola de mpv; el transporte nuevo de named pipe es
  justo lo que se mete en medio de ese camino. Escuchar un álbum entero es la prueba,
  no un test.

### Decisiones abiertas

Estas hay que tomarlas, no comprobarlas. Están aquí para que no se decidan por
omisión.

- **SMTC: hacerlo o no hacerlo.** `windows.md` §5.2 deja el servicio como un stub que
  conserva la API de `MprisService`, y eso es suficiente para que la app funcione. Pero
  la integración de escritorio es una funcionalidad real de upstream —teclas de
  multimedia, Waybar, widgets externos— y aquí queda en nada. Implementarla exige una
  ventana oculta con su propio bucle de mensajes para colgar SMTC de un `HWND`. Es
  trabajo de verdad y es hostil a los tests. **Decidir antes de anunciar la primera
  versión**, porque cambia lo que promete el README.

- **Modo exclusivo por defecto: no.** Decidido en `windows.md` §5.5, y anotado aquí
  para no volver a discutirlo: en WASAPI exclusivo ningún otro programa suena. Una app
  de música que enmudece las notificaciones del sistema sin avisar genera un reporte de
  bug que no es un bug. Va como ajuste, apagado, y el README explica qué gana quien lo
  encienda.

- **El nombre del comando.** La distribución es `tidalamp-win` pero el ejecutable
  se sigue llamando `tidalamp` (`windows.md` §5.14). La alternativa —renombrarlo— pide
  perseguir la cadena literal `tidalamp` por `cli.py`, `desktop.py`, `about.py` y el
  catálogo entero de `i18n.py`. Nadie va a instalar los dos paquetes en la misma
  máquina, así que la colisión es teórica. **Si alguien la encuentra de verdad, esto
  se reabre.**

- **Qué hacer con `packaging/aur/`.** `windows.md` §5.15 dice borrarlo. Está sin hacer
  a propósito: mientras el port no arranque, el PKGBUILD es la única receta de
  empaquetado que existe en el árbol y sirve de referencia de qué declara el proyecto
  como dependencia. Se borra en F6, cuando haya algo que lo sustituya.

## Sin fecha

Heredado de upstream, y sigue siendo válido aquí porque toca código que no cambia con
el sistema operativo:

- **«más…» en las categorías de Inicio.** Una categoría de la página de inicio trae los
  diez primeros que TIDAL pone en la página, y la lista completa está detrás de su
  «view all». `tidalapi` 0.8.11 lo tiene roto: `PageCategoryV2.view_all` llama a un
  `session.view_all` que no existe. Habría que pedir la ruta de `_more.api_path` a
  mano, como ya se hace con los enlaces de Explorar (`library._page_at`).

- **Sacar objetos de verdad de `TidalAmp`.** 3232 líneas en `app.py`; `_setting_changed`
  es de lo más enredado. No en mixins (ver «Descartado»), sino objetos con su propio
  diseño: reproducción, carátula, presentación de la cola y aplicación de ajustes,
  dejando `TidalAmp` como raíz de composición. Es un rediseño grande que no arregla
  ningún fallo. **Y aquí hay un motivo extra para no tocarlo:** `app.py` es el fichero
  que la regla de oro de `windows.md` §1 manda dejar intacto para que los cambios de
  upstream sigan aplicando. Reordenarlo cierra ese canal.

- **Una guarda común para los workers al cerrar.** Un worker de hilo que termina después
  de salir llama a `call_from_thread` contra un loop que se cierra: en la app cuesta una
  excepción dentro de ese hilo, sin efecto visible, y en los tests es la carrera por la
  que `app_helpers.isolate_runtime` desactiva el worker del sink. Razonable y pequeño,
  pero sin un fallo que lo pida. **Vigilar si reaparece en Windows**, donde el bucle por
  defecto es Proactor y no Selector: es el tipo de carrera que cambia de forma al
  cambiar de bucle.

- **Seleccionar varias pistas en la cola** para quitarlas o moverlas juntas; hoy se hace
  de a una. Interesa, pero sin versión decidida.

- **Espectro FFT de verdad.** `windows.md` §5.6 deja el visualizador en el vúmetro RMS
  porque cava no tiene build de Windows. La vía es captura loopback de WASAPI más FFT
  con `numpy`, dentro del proceso, y **arreglaría de paso** la pega que cava tiene en
  Linux: escucha el sink, así que muestra lo que suene en la máquina y no lo que suena
  en mpv (`plan.md` §9.4). Es la única funcionalidad donde el port puede quedar por
  encima del original.

## Descartado por ahora

No se borran: quedan escritos con el motivo para no volver a discutirlos desde cero.

- **Una capa multiplataforma en vez de un fork** (2026-09-18). Era la alternativa obvia:
  un `platform/` con implementaciones Linux y Windows detrás de una interfaz común, y un
  solo repositorio. Se descartó porque duplica el coste de cada cambio futuro y porque
  ninguno de los dos caminos queda bien probado —el CI de una máquina no ejerce las
  ramas de la otra, y la que no se ejerce se pudre. La decisión del mantenedor fue un
  proyecto en paralelo. Lo que sí se conserva de aquella idea es lo único que costaba
  poco: **las firmas públicas no cambian** (`windows.md` §1), que es lo que mantiene los
  diffs de upstream aplicables sin pagar por una abstracción.

- **`pywin32` por adelantado** (2026-09-18). Resolvería de golpe los timeouts del named
  pipe (E/S solapada), las ACL del fichero de sesión y la creación del acceso directo,
  las tres cosas mejor que los apaños que propone `windows.md`. Pero es una dependencia
  binaria pesada en un proyecto cuya decisión fundacional fue «cero dependencias
  nativas» (`plan.md` §2, la fila del IPC). Se queda fuera **hasta que un problema
  medido lo pida**; si entra, entra como extra opcional, no como dependencia dura.

- **Empaquetar mpv dentro del instalador** (2026-09-18). Tentador: quita el único paso
  manual de la instalación. Pero mpv es GPL y el proyecto también, así que redistribuir
  su binario obliga a ofrecer su fuente correspondiente, y eso es una carga de
  cumplimiento real para un mantenedor individual. Además contradice lo que dice la
  descripción del paquete desde la primera versión («requires mpv»). `distro.py` da la
  orden de instalación; con eso basta.

- **Los descartes heredados de upstream siguen en pie** y no se reabren aquí: radio de
  artista o de playlist, temporizador de apagado, `app.py` en mixins, y enrutar mpv a un
  sink propio. Los motivos están en el `next.md` del repositorio original y ninguno
  depende del sistema operativo.
