# Empaquetado

> [!WARNING]
> **Nada de esto está en marcha todavía.** El port a Windows no ha arrancado (ver
> [`../windows.md`](../windows.md)) y no hay ninguna versión de `tidalamp-win`
> publicada en ningún canal. Este fichero describe **el plan**, no un procedimiento
> ejecutado. El contenido de `aur/` es la receta heredada de upstream y se borra en F6.

Tres canales, que cubren públicos distintos:

| Canal | Para quién | Estado |
| --- | --- | --- |
| **PyPI** | quien ya tiene Python y usa `pipx` | pendiente de alta |
| **winget** | el usuario de Windows que no quiere saber de Python | pendiente del instalador |
| **Instalador `.exe`** | el que llega desde una Release de GitHub | pendiente de F6 |

La guía completa, paso a paso, para preparar la versión, configurar PyPI, crear el
GitHub Release y publicar el manifiesto está en [`../publish.md`](../publish.md). Este
archivo conserva el resumen y las decisiones específicas del empaquetado.

## Publicar una versión

1. Cerrar la sección `[Unreleased]` de `CHANGELOG.md` con el número y la fecha.
2. Actualizar la versión en los **cuatro** sitios: `pyproject.toml`,
   `tidalamp/__init__.py`, la entrada de `releases()` en `tidalamp/about.py` y el `?v=`
   de las URLs de imagen del `README.md`. El detalle está en `publish.md` §3.2.
   *Upstream tenía cinco; el quinto era `pkgver` del PKGBUILD, que aquí no existe.*
3. Crear el commit, un tag anotado con `git tag -a vX.Y.Z -m "tidalamp-win X.Y.Z"` y
   subir únicamente ese tag con `git push origin vX.Y.Z`.
4. El tag dispara `.github/workflows/release.yml`, que comprueba que el tag coincide con
   la versión del `pyproject.toml`, construye sdist y wheel, pasa `twine check` y publica
   en PyPI.
5. Construir el instalador, adjuntarlo a la GitHub Release y **entonces** calcular su
   `sha256` para el manifiesto de winget. En ese orden: winget necesita una URL de
   descarga estable y un hash del fichero que está detrás de ella.

## PyPI (Trusted Publishing)

**Sin configurar.** El Trusted Publishing de upstream está atado a
`wh01s17/tidalamp`, así que no sirve aquí: hay que dar de alta el proyecto nuevo. Si no
se hace, el job `publish` falla en el primer tag con un error de OIDC, no con uno de
credenciales.

- En PyPI → *Your projects* → *Publishing* → *Add a new pending publisher*:
  - PyPI Project Name: `tidalamp-win`
  - Owner: `wh01s17`
  - Repository name: `tidalamp-win`
  - Workflow name: `release.yml`
  - Environment name: `pypi`
- En GitHub → *Settings* → *Environments* → crear el entorno `pypi`, con al menos una
  persona en **Required reviewers** para aprobar manualmente cada publicación.

El nombre en PyPI es `tidalamp-win`; **el módulo importable se sigue llamando
`tidalamp`**, y eso es deliberado: es lo que mantiene aplicables los diffs de upstream
(`windows.md` §1 y §5.14). Nadie va a instalar los dos paquetes en la misma máquina, así
que la colisión de nombres de comando es teórica.

## El instalador

Dos piezas, en este orden:

**1. PyInstaller** produce el ejecutable único para quien no quiere instalar Python. El
`.spec` vive en `windows/`. Dos cosas que hay que declarar a mano o el binario sale
roto: los ficheros `.tcss` de `tidalamp/styles/` y el `.ico`, ambos como `datas` —
PyInstaller no los encuentra solo, y Textual falla en el arranque sin su hoja de
estilos.

**2. Inno Setup** envuelve ese ejecutable en un `.exe` de instalación que además crea el
acceso directo del menú Inicio. Eso deja `desktop.py` como respaldo para quien haya
instalado por pip, que es justo el reparto que tenía upstream entre el paquete del AUR y
`pipx`.

## winget

El manifiesto son tres YAML (`version`, `installer`, `locale`) bajo
`manifests/w/wh01s17/tidalamp-win/<version>/`, y se envían por pull request al
repositorio `microsoft/winget-pkgs`. Requisitos que conviene saber antes de empezar:

- La URL del instalador tiene que ser **estable y pública**: el enlace de la GitHub
  Release vale.
- El `InstallerSha256` es el del fichero exacto que está detrás de esa URL. Si se
  reconstruye el instalador, cambia el hash y hay que rehacer el manifiesto.
- Cada versión nueva es un pull request nuevo. `wingetcreate update` automatiza la mayor
  parte.
- La validación automática instala y desinstala el paquete en una máquina limpia, así
  que el instalador tiene que ser silencioso (`/VERYSILENT` en Inno Setup) y desinstalar
  sin dejar nada.

## mpv no se instala con pip — ni se empaqueta aquí

Quien haga `pip install tidalamp-win` sin `mpv` en el sistema se encuentra un
`MpvNotFound` al arrancar. Por eso la descripción del paquete lo dice en la primera
línea y el README lo repite en la sección de instalación. `distro.py` detecta si hay
`winget`, `scoop` o `choco` y nombra la orden concreta, para que el error sea accionable
en vez de solo cierto.

**Y no se empaqueta mpv dentro del instalador**, aunque quitaría el único paso manual de
la instalación. mpv es GPL y este proyecto también, así que redistribuir su binario
obliga a ofrecer su fuente correspondiente: es una carga de cumplimiento real para un
mantenedor individual, y contradice lo que la descripción del paquete promete desde la
primera versión. La decisión está anotada en `next.md`, «Descartado por ahora», para no
volver a discutirla.

## `aur/` — heredado, a borrar

`aur/PKGBUILD` es la receta de Arch de upstream. No aplica a Windows y **se borra en
F6**, junto con `tidalamp.desktop`. Sigue aquí a propósito hasta entonces: mientras no
exista el empaquetado nuevo, es la única declaración completa en el árbol de qué
considera el proyecto una dependencia real (`mpv`) y qué opcional (`cava`), y eso es la
referencia contra la que se escribe lo que la sustituya.

Los tarballs `tidalamp-*.tar.gz` de ese directorio están en `.gitignore` y no viajan por
git; ocupan espacio en el árbol de trabajo y se pueden borrar en cualquier momento.
