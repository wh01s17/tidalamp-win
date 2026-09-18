# Publicar tidalamp-win

Esta es la guía completa para crear una versión de tidalamp-win y publicarla en los tres
lugares que usa el proyecto:

1. **PyPI**, para instalarla con `pipx` o `pip` en cualquier máquina que ya tenga Python.
2. **GitHub Releases**, como página pública de la versión, sus notas y el instalador.
3. **winget**, para instalarla en Windows sin saber que Python existe.

El orden importa. Primero se prepara y valida el código, después se crea el tag —que
publica automáticamente en PyPI—, luego se crea el GitHub Release con el instalador
adjunto y finalmente se calcula su checksum y se envía el manifiesto a winget. Ese
último paso va al final por una razón mecánica: winget necesita una URL de descarga
estable y el `sha256` del fichero que está detrás de ella, así que no se puede preparar
antes de que el instalador exista y esté publicado.

> [!WARNING]
> **Nada de esto se ha ejecutado todavía.** tidalamp-win es un port en curso y no tiene
> ninguna versión publicada en ningún canal. Este documento describe **el procedimiento
> previsto**, adaptado del de upstream, no uno que se haya recorrido.
>
> Concretamente, y para que nadie lo dé por hecho:
>
> - **El Trusted Publishing de PyPI está sin configurar.** El de upstream está atado a
>   `wh01s17/tidalamp` y no sirve aquí: hay que dar de alta el proyecto `tidalamp-win`
>   (§1.3). Si no se hace, el job `publish` falla en el primer tag con un error de OIDC,
>   no con uno de credenciales, que es un error mucho más confuso de diagnosticar.
> - **No existe el instalador**, ni el `.spec` de PyInstaller que lo produce. Es F6 de
>   [`windows.md`](./windows.md) §4.
> - **`release.yml` viene de upstream** y todavía no se ha comprobado en este
>   repositorio.
>
> Y lo más importante: **no hay nada que publicar hasta cerrar F2 como mínimo**, y en la
> práctica hasta F4. La versión `0.13.0` del `pyproject.toml` es la heredada y describe
> un árbol Linux.

Los ejemplos usan `0.1.0`. Sustituir por la versión que toque.

```powershell
$env:TIDALAMP_VERSION = "0.1.0"
$env:TIDALAMP_TAG = "v$env:TIDALAMP_VERSION"
```

Mantén esas variables en la misma terminal durante todo el proceso. Los bloques de esta
guía van en PowerShell salvo donde se diga otra cosa; los pasos que corren dentro de
GitHub Actions siguen siendo shell POSIX, porque `release.yml` se queda en un runner
Linux (§6) — sólo construye un wheel puro y no gana nada con Windows.

## 1. Configuración inicial — sólo antes de la primera publicación

### 1.1. Permisos necesarios

La persona que publica necesita:

- permisos de escritura en `https://github.com/wh01s17/tidalamp-win`;
- una cuenta verificada en PyPI;
- una cuenta de GitHub para el pull request a `microsoft/winget-pkgs`;
- **una máquina Windows**, para construir el instalador, correr la suite y mirar el TUI.
  Los pasos §1–§7 salen adelante desde cualquier sistema; §8 y la comprobación final de
  §9, no.

No se necesita guardar un token de PyPI en GitHub. El workflow usa Trusted Publishing
con OIDC.

### 1.2. Crear el environment `pypi` en GitHub

**Completado el 2026-09-09.** El environment existe, exige la revisión manual de
`wh01s17`, no permite saltarse las reglas de protección y no contiene secretos. Los
pasos siguientes conservan la configuración exacta para poder auditarla o recrearla.

1. Abre `https://github.com/wh01s17/tidalamp-win`.
2. Entra en **Settings → Environments**.
3. Pulsa **New environment**.
4. Escribe exactamente `pypi` y pulsa **Configure environment**.
5. En las reglas de protección, añade al menos una persona en **Required reviewers**
   para que cada publicación necesite aprobación manual.
6. No añadas ningún secreto de PyPI.

La aprobación manual es una protección obligatoria para este flujo de publicación.
Si además se restringen las ramas y tags permitidos, hay que permitir tags con el
patrón `v*`; de lo contrario el job quedará bloqueado.

El nombre debe coincidir con `environment: pypi` en
`.github/workflows/release.yml`.

### 1.3. Registrar el pending publisher en PyPI

**Sin hacer.** El publisher de upstream está atado a `wh01s17/tidalamp` y no cubre este
repositorio: PyPI empareja el token OIDC con *owner + repo + workflow + environment*, y
tres de esos cuatro cambian. Hay que registrar uno nuevo.

1. Inicia sesión en `https://pypi.org/`.
2. Abre **Account settings → Publishing**.
3. En la sección de GitHub, pulsa **Add a new pending publisher**.
4. Completa los campos exactamente así:

   | Campo de PyPI | Valor |
   |---|---|
   | PyPI project name | `tidalamp-win` |
   | Owner | `wh01s17` |
   | Repository name | `tidalamp-win` |
   | Workflow name | `release.yml` |
   | Environment name | `pypi` |

5. Guarda el publisher.

Un pending publisher no crea el proyecto ni reserva su nombre. Conviene hacer este paso
inmediatamente antes de la primera publicación. Cuando el workflow publique por primera
vez, PyPI creará el proyecto y convertirá el publisher pendiente en uno normal.

> [!NOTE]
> El nombre en PyPI es `tidalamp-win`; **el módulo importable y el comando se siguen
> llamando `tidalamp`**. No es un descuido: es lo que mantiene aplicables los diffs de
> upstream (`windows.md` §5.14). Lo único que hay que vigilar es que
> `[tool.hatch.build.targets.wheel] packages = ["tidalamp"]` no se «corrija» a
> `tidalamp_win` en algún momento, porque entonces el wheel deja de instalar el paquete
> que el código importa.

### 1.4. Preparar el instalador y la cuenta de winget

A diferencia del AUR, winget no pide cuenta propia: los manifiestos se envían por pull
request a `microsoft/winget-pkgs` con la cuenta de GitHub que ya tienes. Lo que sí hace
falta es tener algo que instalar.

1. **Construir el instalador.** Dos piezas, en este orden, y ninguna existe todavía:
   - PyInstaller produce el ejecutable único, desde `packaging/windows/tidalamp.spec`.
     Hay que declarar como `datas` los `.tcss` de `tidalamp/styles/` y el `.ico`:
     PyInstaller no los encuentra solo y Textual falla en el arranque sin su hoja de
     estilos.
   - Inno Setup lo envuelve en un `.exe` de instalación que además crea el acceso
     directo del menú Inicio.
2. **Que desinstale limpio.** La validación automática de winget instala y desinstala
   el paquete en una máquina limpia, así que el instalador tiene que ser silencioso
   (`/VERYSILENT`) y no dejar nada detrás.
3. **Instalar `wingetcreate`**, que genera y actualiza los manifiestos sin escribir los
   tres YAML a mano:

   ```powershell
   winget install Microsoft.WingetCreate
   ```

4. **Tener el repositorio bifurcado.** `wingetcreate submit` abre el pull request contra
   tu fork de `microsoft/winget-pkgs`; la primera vez pedirá autorizarse con GitHub.

### 1.5. Herramientas locales

El entorno Python de desarrollo necesita el proyecto y sus extras:

```powershell
python -m venv .venv
.venv\Scripts\python -m pip install --upgrade pip
.venv\Scripts\python -m pip install -e ".[dev]" build twine
```

Y las herramientas de empaquetado:

```powershell
winget install Microsoft.WingetCreate
winget install JRSoftware.InnoSetup
.venv\Scripts\python -m pip install pyinstaller
```

`gh`, la CLI de GitHub, es opcional; todos sus pasos tienen una alternativa desde la web.

## 2. Elegir la versión

tidalamp sigue versionado semántico `MAJOR.MINOR.PATCH`:

- incrementa `PATCH` para correcciones compatibles, por ejemplo `0.1.0 → 0.1.1`;
- incrementa `MINOR` para funcionalidad nueva compatible, por ejemplo `0.1.1 → 0.2.0`;
- incrementa `MAJOR` cuando haya cambios incompatibles en una versión estable.

Antes de usar una versión, confirma que no existe como tag ni en PyPI:

```powershell
git fetch origin --tags
git tag --list $env:TIDALAMP_TAG
```

El segundo comando no debe imprimir nada. Comprueba también
`https://pypi.org/project/tidalamp-win/`. PyPI no permite reemplazar los archivos de
una versión que ya fue publicada.

## 3. Preparar el repositorio

### 3.1. Partir de `main` actualizado y limpio

```powershell
git switch main
git pull --ff-only origin main
git status --short
```

`git status --short` debe quedar vacío. No mezcles una publicación con cambios que no
pertenezcan a esa versión.

### 3.2. Actualizar los números de versión

La versión vive en **cuatro** sitios. `release.yml` sólo compara el tag con el primero,
y `tests/test_about.py::test_every_copy_of_the_version_agrees` ata los otros dos que
son de Python: si alguno se queda atrás, la suite falla antes de llegar al tag. El
último —el `?v=` del README— no lo comprueba nadie más que esta sección.

*Upstream tenía cinco; el quinto era `pkgver` del PKGBUILD del AUR, que aquí no existe.*

1. En `pyproject.toml`:

   ```toml
   version = "0.1.0"
   ```

2. En `tidalamp/__init__.py`:

   ```python
   __version__ = "0.1.0"
   ```

   Es el respaldo de `about.version()` cuando no hay metadata instalada, y lo que ve
   quien ejecuta desde un checkout.

3. En `tidalamp/about.py`, la entrada correspondiente de `releases()`: el número, la
   fecha —que hasta la publicación dice «sin publicar»— y las notas de la versión.
   Es lo que muestra «Cambios por versión» en la pantalla de ayuda (`?`).

   ```python
   def releases() -> tuple[Release, ...]:
       return (
           Release(
               "0.1.0",
               "2026-09-15",
               (
                   _("…"),
                   _("…"),
               ),
           ),
       )
   ```

   Va aquí y no leído de `CHANGELOG.md` a propósito: ese fichero no viaja dentro del
   wheel, así que la pantalla saldría vacía para todo el que instale el paquete.

4. En `README.md`, el `?v=` de las URLs de `img/`:

   ```powershell
   $v = $env:TIDALAMP_VERSION
   (Get-Content README.md -Raw) -replace '\?v=[0-9.]+(["\)])', "?v=$v`$1" |
     Set-Content README.md -NoNewline
   (Select-String -Path README.md -Pattern '\?v=' -AllMatches).Matches.Count
   ```

   El último comando cuenta las URLs con token de versión; compáralo con las que
   quedaron sin tocar (`Select-String '\?v=' README.md` las lista).

   Las dos clases de enlace tienen que entrar: las de `<img src="...">`, que
   acaban en comilla, y las de `![alt](...)` de Markdown, que acaban en
   paréntesis. Una expresión que solo contemple la comilla deja el banner de la
   cabecera anclado a la versión anterior, que es justo la imagen que más se ve.

   Las imágenes se enlazan por URL absoluta porque PyPI no resuelve rutas
   relativas, y esa URL no cambia cuando el fichero sí: el proxy de imágenes de
   GitHub sigue sirviendo la copia anterior durante horas. El token de versión es
   lo que la invalida. Si en esta versión no cambió ninguna imagen, da igual
   actualizarlo.

El manifiesto de winget **no** se actualiza aquí: lleva su propia versión y su propio
`sha256`, y ninguno de los dos se puede calcular hasta que el instalador esté publicado.
Va en §8.

### 3.3. Cerrar el changelog

El `CHANGELOG.md` está **en inglés**, como el README y las notas de los releases: es lo
que lee alguien de fuera. Conserva una sección vacía para cambios futuros y mueve las
notas actuales bajo la versión y la fecha de publicación:

```markdown
## [Unreleased]

## [0.1.0] - YYYY-MM-DD

### Added

...
```

Los encabezados son los de Keep a Changelog en inglés: `Added`, `Changed`, `Deprecated`,
`Removed`, `Fixed`, `Security`.

Revisa que las notas describan sólo cambios incluidos en el commit que se etiquetará, y
que el resumen de `about.releases()` (paso 3.2) diga lo mismo en corto: son dos textos
para dos públicos —el changelog completo y las cuatro líneas que caben en la pantalla
de ayuda— y no deben contradecirse.

### 3.4. Comprobar que no queda nada del AUR

Upstream tenía aquí un paso para regenerar `.SRCINFO` con `makepkg`. En este proyecto no
aplica, y lo que queda en su lugar es una comprobación: que `packaging/aur/` ya no
exista, o que si existe no lo referencie nadie.

```powershell
Test-Path packaging\aur
Select-String -Path pyproject.toml,packaging\README.md -Pattern 'aur|PKGBUILD|SRCINFO'
```

Si `pyproject.toml` sigue incluyendo `packaging/` en el `sdist`, revisa que apunte a
`packaging/windows/` y no arrastre la receta de Arch dentro del tarball que se sube a
PyPI.

## 4. Validar la versión antes de etiquetarla

### 4.1. Calidad y pruebas

Ejecuta todas las comprobaciones desde la raíz del repositorio:

```powershell
.venv\Scripts\ruff check tidalamp tests
.venv\Scripts\ruff format --check tidalamp tests
.venv\Scripts\mypy tidalamp
.venv\Scripts\python -m pytest -q
git diff --check
```

`mypy` necesita `platform = "win32"` en su configuración. Sin eso valida las ramas del
sistema equivocado y se salta las que sí existen aquí (`windows.md` §5.14).

No continúes si alguna falla.

Si la única que falla es `test_the_version_is_the_one_the_package_declares`, no es la
versión sino la instalación: la editable guarda la metadata de cuando se instaló y
sigue diciendo el número anterior hasta reinstalarla con
`.venv\Scripts\python -m pip install -e ".[dev]"`. Pasó al preparar la `0.6.0` en
upstream, y la causa no cambia con el sistema.

### 4.2. Construir y revisar los artefactos

Usa un directorio temporal nuevo para no validar por accidente archivos de una versión
anterior:

```powershell
$env:TIDALAMP_BUILD_DIR = (New-Item -ItemType Directory -Path (Join-Path $env:TEMP (New-Guid))).FullName
.venv\Scripts\python -m build --outdir $env:TIDALAMP_BUILD_DIR
.venv\Scripts\python -m twine check "$env:TIDALAMP_BUILD_DIR\*"
Get-ChildItem $env:TIDALAMP_BUILD_DIR
```

Deben existir exactamente un `sdist` (`.tar.gz`) y un wheel (`.whl`) de la versión
elegida, y ambos deben pasar `twine check`.

Comprueba además que el wheel se instala en un entorno vacío:

```powershell
$venv = (New-Item -ItemType Directory -Path (Join-Path $env:TEMP (New-Guid))).FullName
python -m venv $venv
& "$venv\Scripts\pip" install (Get-ChildItem "$env:TIDALAMP_BUILD_DIR\tidalamp*.whl").FullName
& "$venv\Scripts\tidalamp" --help
```

Esta comprobación no instala `mpv`; `tidalamp --help` no lo necesita — es eager a
propósito, para que funcione en la instalación que está rota. La reproducción real sí
requiere `mpv` en el sistema.

El wheel se llama `tidalamp_win-X.Y.Z-...whl` aunque el paquete que instala se importe
como `tidalamp`: PyPI normaliza el guion del nombre de distribución a guion bajo.

## 5. Crear y subir el commit de la versión

Revisa todas las diferencias antes de confirmar:

```powershell
git diff
git diff --check
git status --short
```

Añade los archivos de la versión, crea el commit y sube `main`:

```powershell
git add CHANGELOG.md README.md pyproject.toml tidalamp/__init__.py tidalamp/about.py
git commit -m "chore(release): prepare $env:TIDALAMP_TAG"
git push origin main
```

Esos cinco nombres cubren todas las copias de la versión, la fecha y las notas. Si la
versión también contiene otros archivos ya revisados, deben incluirse en el commit
correspondiente antes de crear el tag.

En GitHub, abre **Actions → CI** y espera a que el commit de `main` termine en verde.
No crees el tag sobre un commit cuyo CI no haya finalizado correctamente.

## 6. Crear el tag y publicar en PyPI

### 6.1. Comprobación final

Antes de crear el tag, confirma:

- que el environment `pypi` existe en GitHub;
- que el pending publisher de PyPI usa exactamente los cinco valores de la sección
  1.3, **y que dicen `tidalamp-win` y no `tidalamp`**;
- que `main` está sincronizado y limpio;
- que la versión de `pyproject.toml` es idéntica al tag sin la `v`.

```powershell
git status --short --branch
git log -1 --oneline
Select-String -Path pyproject.toml -Pattern '^version = '
```

### 6.2. Crear y subir sólo el tag deseado

```powershell
git tag -a $env:TIDALAMP_TAG -m "tidalamp-win $env:TIDALAMP_VERSION"
git show --no-patch $env:TIDALAMP_TAG
git push origin $env:TIDALAMP_TAG
```

Evita `git push --tags`: podría subir tags locales que no pretendías publicar.

### 6.3. Vigilar el workflow

El push dispara `.github/workflows/release.yml`. Ese workflow:

1. comprueba que `vX.Y.Z` coincide con `project.version`;
2. construye el sdist y el wheel;
3. ejecuta `twine check`;
4. guarda los paquetes como artefacto de GitHub Actions;
5. obtiene una identidad OIDC y publica en PyPI sin token permanente.

Desde la web, abre **Actions → release → ejecución de `vX.Y.Z`**. Si el environment
requiere aprobación, aprueba el job `publish` cuando aparezca como pendiente.

Con GitHub CLI se puede consultar y seguir la ejecución:

```powershell
gh run list --workflow release.yml --limit 5
gh run watch ID_DE_LA_EJECUCION
```

No continúes con el anuncio ni con el instalador hasta que los jobs `build` y `publish`
estén en verde.

### 6.4. Verificar PyPI desde cero

Abre `https://pypi.org/project/tidalamp-win/` y confirma la versión, descripción, README,
licencia, versión mínima de Python y enlaces del proyecto.

Prueba lo que recibirá un usuario, descargándolo desde PyPI en otro entorno vacío:

```powershell
$venv = (New-Item -ItemType Directory -Path (Join-Path $env:TEMP (New-Guid))).FullName
python -m venv $venv
& "$venv\Scripts\pip" install "tidalamp-win[art]==$env:TIDALAMP_VERSION"
& "$venv\Scripts\tidalamp" --help
```

## 7. Crear el GitHub Release

El tag y el Release no son lo mismo. El tag ya existe y publicó PyPI; ahora se crea una
página pública con las notas de la versión. GitHub añadirá automáticamente descargas
`.zip` y `.tar.gz` del código correspondiente al tag.

### 7.1. Desde la web

1. Abre `https://github.com/wh01s17/tidalamp-win/releases`.
2. Pulsa **Draft a new release**.
3. En **Choose a tag**, selecciona el tag que ya se publicó: `vX.Y.Z`.
4. Usa como título `tidalamp X.Y.Z`.
5. Copia las notas de esa versión desde `CHANGELOG.md`, que ya está en inglés, o
   pulsa **Generate release notes** y revísalas manualmente. No dejes las notas en
   español: todos los releases publicados están en inglés y el enlace «Full changelog»
   de cada uno lleva a un fichero en inglés.
6. Marca **Set as the latest release**.
7. Marca **This is a pre-release** sólo si quieres presentar explícitamente esa versión
   como preliminar.
8. Pulsa **Publish release**.

No crees un tag diferente desde el formulario: el Release debe apuntar exactamente al
tag que activó la publicación de PyPI.

### 7.2. Con GitHub CLI

Para generar notas desde los commits:

```powershell
gh release create $env:TIDALAMP_TAG `
  --verify-tag `
  --title "tidalamp-win $env:TIDALAMP_VERSION" `
  --generate-notes
```

Si preparaste las notas en un archivo, sustituye `--generate-notes` por
`--notes-file RUTA_AL_ARCHIVO`.

### 7.3. Adjuntar wheel y sdist — opcional

PyPI ya hospeda los paquetes y GitHub ya ofrece el código fuente. Si también quieres
que el wheel y el sdist aparezcan como assets del Release:

```powershell
gh run download ID_DE_LA_EJECUCION `
  --name dist `
  --dir "dist\release-$env:TIDALAMP_VERSION"
gh release upload $env:TIDALAMP_TAG (Get-ChildItem "dist\release-$env:TIDALAMP_VERSION").FullName
```

Lo que **sí** tiene que estar adjunto al Release es el instalador: winget descarga desde
esa URL y resume ese fichero (§8.3). El wheel y el sdist son opcionales.

## 8. Construir el instalador y publicar en winget

Este es el único tramo que **no** se puede preparar por adelantado. winget empareja una
URL de descarga con el `sha256` del fichero que hay detrás, así que el orden es rígido:
construir, adjuntar al Release, calcular el hash de lo adjuntado, y sólo entonces
escribir el manifiesto. Invertir cualquier par de pasos produce un manifiesto que la
validación rechaza.

### 8.1. Construir el ejecutable y el instalador

```powershell
.venv\Scripts\pyinstaller packaging\windows\tidalamp.spec --clean --noconfirm
& "${env:ProgramFiles(x86)}\Inno Setup 6\ISCC.exe" packaging\windows\tidalamp.iss
```

Dos cosas que hay que verificar a ojo antes de seguir, porque ninguna da error:

- **Que el ejecutable arranca en una máquina sin Python.** Un PyInstaller que olvidó los
  `.tcss` de `tidalamp/styles/` construye sin quejarse y falla al abrir la interfaz, que
  es el peor momento para enterarse.
- **Que el `.ico` está dentro.** Sin él el acceso directo sale con el icono genérico.

### 8.2. Probar la instalación y la desinstalación

La validación de winget instala y desinstala en una máquina limpia. Hacerlo antes, en
una máquina virtual o en un usuario nuevo, cuesta menos que un pull request rechazado:

```powershell
.\Output\tidalamp-win-$env:TIDALAMP_VERSION-setup.exe /VERYSILENT
tidalamp --version
# y después, desde Aplicaciones instaladas o:
# "%ProgramFiles%\tidalamp-win\unins000.exe" /VERYSILENT
```

Comprueba que la desinstalación se lleva el acceso directo del menú Inicio y **no** se
lleva `%APPDATA%\tidalamp` — la sesión y la configuración del usuario no son del
instalador. Eso lo borra «Cerrar sesión» desde la propia aplicación, con su casilla.

### 8.3. Adjuntar el instalador al GitHub Release

Desde la web, o con la CLI:

```powershell
gh release upload $env:TIDALAMP_TAG `
  ".\Output\tidalamp-win-$env:TIDALAMP_VERSION-setup.exe"
```

**Este fichero no se vuelve a tocar.** Si hay que reconstruirlo, cambia el hash y hay
que rehacer el manifiesto de winget aunque la versión sea la misma.

### 8.4. Calcular el checksum de lo publicado

Del fichero que está detrás de la URL pública, no del que quedó en `Output\`. Descarga
y resume:

```powershell
$url = "https://github.com/wh01s17/tidalamp-win/releases/download/$env:TIDALAMP_TAG/tidalamp-win-$env:TIDALAMP_VERSION-setup.exe"
Invoke-WebRequest $url -OutFile verify.exe
(Get-FileHash verify.exe -Algorithm SHA256).Hash
```

Comparar ese hash con el del fichero local es además la comprobación de que la subida no
se corrompió.

### 8.5. Generar y enviar el manifiesto

La primera versión crea el paquete; las siguientes lo actualizan.

```powershell
# primera vez
wingetcreate new $url

# versiones posteriores
wingetcreate update wh01s17.tidalamp-win --version $env:TIDALAMP_VERSION --urls $url
```

`wingetcreate` rellena los tres YAML (`version`, `installer`, `locale`), calcula el
hash por su cuenta y abre el pull request contra `microsoft/winget-pkgs`. Guarda una
copia de los manifiestos en `packaging/windows/manifests/` para tener en el repositorio
lo que se envió.

Antes de enviar, revisa a mano el `PackageIdentifier` (`wh01s17.tidalamp-win`), la
licencia (`GPL-3.0-or-later`) y la descripción corta, que es lo que ve quien hace
`winget search`. Y que el `ShortDescription` mencione mpv: es la única dependencia que
el instalador no trae y el sitio donde más se va a leer.

### 8.6. Verificar winget

El pull request pasa por validación automática y, a veces, por revisión humana; puede
tardar de horas a días. Cuando se fusione:

```powershell
winget search tidalamp
winget install wh01s17.tidalamp-win
tidalamp --version
```

Si la validación falla, el bot comenta en el pull request con el motivo. Los dos motivos
habituales son un instalador que no desinstala limpio (§8.2) y un hash que no coincide
porque se reconstruyó el binario después de calcularlo (§8.4).

## 9. Lista final antes de anunciar la versión

- [ ] `CHANGELOG.md` contiene `X.Y.Z` y la fecha correcta.
- [ ] `pyproject.toml`, `tidalamp/__init__.py`, `tidalamp/about.py` y el `?v=` del
      `README.md` muestran la misma versión (§3.2). La suite lo comprueba para los tres
      primeros; el `?v=` no lo cubre nadie más que esta casilla.
- [ ] La pantalla de ayuda (`?`) muestra la versión y sus notas, y la fecha ya no dice
      «sin publicar».
- [ ] Las pruebas, Ruff, mypy, build y `twine check` pasan **en Windows**.
- [ ] El CI del commit de versión está en verde, en las cuatro versiones de Python.
- [ ] Existe el tag anotado `vX.Y.Z` y apunta al commit correcto.
- [ ] El workflow `release` terminó con `build` y `publish` en verde.
- [ ] PyPI muestra la versión y una instalación nueva funciona.
- [ ] Existe un GitHub Release publicado sobre el mismo tag, con el instalador adjunto.
- [ ] El instalador se instala y se desinstala limpio en una máquina sin Python (§8.2).
- [ ] El `sha256` del manifiesto es el del fichero que sirve la URL pública, no el del
      binario local (§8.4).
- [ ] El pull request de `winget-pkgs` está fusionado y `winget install` funciona.

Y una que no es de publicación pero decide si la versión merece anunciarse: **el TUI
abierto a ojo en Windows Terminal y en conhost**, con una pista sonando. La suite mide
celdas, no colores (`plan.md` §7 y §9).

## 10. Problemas frecuentes y recuperación

### El workflow dice que el tag no coincide con `pyproject.toml`

El tag `vX.Y.Z` y `project.version = "X.Y.Z"` deben ser idénticos. No muevas ni
reutilices un tag que ya haya sido publicado o consumido. La opción segura es corregir
los archivos y publicar una versión nueva.

### PyPI responde `invalid-pending-publisher` o `invalid-publisher`

Compara carácter por carácter owner, repositorio, nombre del workflow y environment.
Los errores más comunes son escribir `.github/workflows/release.yml` donde PyPI pide
sólo `release.yml`, y —el propio de este fork— **haber dejado el publisher apuntando a
`tidalamp` en vez de a `tidalamp-win`**, que es lo que pasa si se copió la configuración
de upstream sin releerla (§1.3).

### El job `publish` queda esperando

Abre la ejecución en GitHub Actions. Si el environment exige revisión, aprueba el
deployment. Revisa también que sus reglas permitan tags `v*`.

### PyPI dice que el archivo o la versión ya existe

Una versión de PyPI es inmutable: no se puede sobrescribir. Incrementa la versión,
actualiza el changelog, crea un commit y tag nuevos y repite el proceso.

### El CI falla sólo en el job `bare`

Ese job corre un script de comprobación que en upstream iba como *heredoc* de bash. En
un runner `windows-latest` la shell por defecto es PowerShell y el heredoc no existe.
La solución está en `windows.md` §5.16: o `shell: bash` en ese paso, o —mejor— mover el
script a un fichero `.py` y llamarlo.

El job existe por un motivo que sigue vigente: comprueba la instalación **sin extras**,
que es lo que teclea un usuario. Una dependencia dura sobre una opcional rompe la
instalación documentada sin que ningún otro job se entere; así desaparecieron las
carátulas en un clon limpio. No lo desactives para que el CI pase.

### El instalador construye pero la aplicación no abre

Casi siempre son los datos que PyInstaller no encuentra solo. Comprueba que el `.spec`
declare como `datas` los `.tcss` de `tidalamp/styles/` y el `.ico`. Textual falla en el
arranque sin su hoja de estilos, y el error que da no menciona el fichero que falta.

### La validación de winget rechaza el pull request

Los dos motivos habituales:

- **El hash no coincide.** Se reconstruyó el instalador después de calcularlo. Vuelve a
  §8.4 con el fichero que realmente sirve la URL.
- **No desinstala limpio.** La validación instala y desinstala en una máquina virgen.
  Reprodúcelo tú primero (§8.2) antes de volver a enviar.

### Se descubre un fallo grave después de publicar

No reemplaces el tag ni intentes sobrescribir PyPI. Corrige el fallo, incrementa la
versión PATCH y publica una nueva versión. Si el paquete defectuoso no debe instalarse,
márcalo como *yanked* en PyPI y explica la sustitución en el GitHub Release y en el
manifiesto de winget.

## 11. Publicaciones posteriores

Para la segunda versión y las siguientes no se repite la configuración de PyPI ni el
GitHub environment, ni el alta en `winget-pkgs`. El ciclo normal es:

1. elegir una versión nueva;
2. actualizar el changelog y los cuatro sitios de la versión (§3.2: `pyproject.toml`,
   `tidalamp/__init__.py`, `releases()` en `tidalamp/about.py`, y el `?v=` del README);
3. validar y construir;
4. confirmar el commit y esperar el CI;
5. crear y subir el tag;
6. verificar PyPI;
7. crear el GitHub Release;
8. construir el instalador, adjuntarlo, recalcular el hash y actualizar winget con
   `wingetcreate update`.

Los pasos 1–7 se pueden hacer sin tocar Windows: el wheel es puro y el workflow corre en
un runner Linux. **El paso 8 no**, y tampoco la comprobación final de §9.

## Referencias oficiales

- [PyPI: crear un proyecto mediante un Trusted Publisher](https://docs.pypi.org/trusted-publishers/creating-a-project-through-oidc/)
- [PyPI: publicar con Trusted Publishing](https://docs.pypi.org/trusted-publishers/using-a-publisher/)
- [PyPI: solución de problemas de Trusted Publishing](https://docs.pypi.org/trusted-publishers/troubleshooting/)
- [GitHub: administrar Releases](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)
- [GitHub: administrar environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
- [winget-pkgs: cómo contribuir un manifiesto](https://github.com/microsoft/winget-pkgs/blob/master/CONTRIBUTING.md)
- [winget: esquema de los manifiestos](https://learn.microsoft.com/en-us/windows/package-manager/package/manifest)
- [wingetcreate](https://github.com/microsoft/winget-create)
- [PyInstaller: ficheros de datos en el .spec](https://pyinstaller.org/en/stable/spec-files.html)
