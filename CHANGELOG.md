# Changelog

Todas las mudanzas notables de este proyecto serán documentadas en este archivo.

El formato está basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/),
y este proyecto se adhiere al [Versionamiento Semántico](https://semver.org/lang/es-ES/).

## [No lanzado]

### Cambiado

- **Menú lateral reorganizado (#109).** Bloque `.user-info` compactado
  (avatar 36px a la izquierda + nombre/email a la derecha, en lugar de los 135px
  fijos con avatar grande); pie de página `.legal` centrado; `.menu`
  ahora usa `flex: 1 1 auto` en vez de altura fija de 450px que
  sobraba o estallaba según la pantalla. Chevron del desplegable del
  usuario reposicionado en la esquina derecha de la línea (sin superponer el
  nombre) y elevado a `z-index: 9999` para abrir por encima de los elementos
  del menú. Todos los cambios viven en un bloque aislado al final de
  `djangosige/static/css/style.css`; el HTML de `base.html` continúa
  intacto.
- **Generación de PDF migrada de `geraldo` a [WeasyPrint](https://weasyprint.org/)
  (#142).** Reports de `geraldo` (`report_vendas.py`, `report_compras.py`,
  basados en `ReportBand`/`SubReport` posicionados en centímetros) fueron
  sustituidos por una única plantilla HTML/CSS reutilizable en
  `djangosige/apps/base/templates/base/pdf/relatorio_documento.html`. Las
  vistas `GerarPDFVenda`/`GerarPDFCompra` ahora renderizan la plantilla vía
  `render_to_string()` y generan el PDF con `weasyprint.HTML(...).write_pdf()`.
  Dependencias de sistema de WeasyPrint (`libpango-1.0-0`, `libpangoft2-1.0-0`,
  `libcairo2`, `libgdk-pixbuf-2.0-0`, `libharfbuzz0b`, `libfontconfig1`)
  añadidas al `Dockerfile`. Los 4 tests que estaban marcados como
  `@unittest.skip` a causa de geraldo fueron rehabilitados.
- Actualizaciones de dependencias:
  - `dj-database-url` 0.5.0 → 3.1.2 (cinco años de actualizaciones acumuladas).
  - `python-decouple` 3.1 → 3.8 (API del `Csv()` mantenida).
  - `flake8` (dev) 3.6.0 → 7.3.0, con `pyflakes`, `pycodestyle` y
    `mccabe` correspondientes.
  - `asgiref`, `sqlparse`, `pillow` ya estaban en el parche más reciente
    compatible.
- `requirements.txt` ahora es generado vía `uv export` (sincronizado con
  `uv.lock`).
- `pyproject.toml` reorganizado en tres grupos comentados: stack de
  NFe (anclada en versiones antiguas, ver nota abajo), PDF (WeasyPrint) y
  runtime general. Anclajes explícitos de `future`, `six`, `eight` y `pytz`
  eliminados — continúan disponibles como deps transitivas.
- `mock` eliminado de las deps de dev (no había uso en los tests — Python 3
  trae `unittest.mock` incorporado).
- Se aflojan constraints de `lxml` (`>=4.9,<5` — techo necesario porque
  `signxml==2.5.2` exige `lxml<5`) y `reportlab` (`>=4.5`). Sin efecto
  en versiones instaladas en el momento; permite parches futuros sin
  alteración en el `pyproject`.
- Constraint de `Django` aflojada a `>=5.2,<5.3` (línea LTS).
  Versión instalada continúa `5.2.14`. El salto a 6.x queda para una ronda
  separada tras validación.
- Python actualizado de 3.10 a 3.12 (`.python-version`, `pyproject`
  y `Dockerfile`). El salto a 3.13 quedó bloqueado por el conflicto entre
  `lxml<5` (exigido por `signxml==2.5.2`, parte del stack anclado de NFe)
  y el hecho de que `lxml 4.x` no compila con la API de CPython 3.13.
  Stack validado en contenedor Docker (reconstrucción de la imagen con
  `python:3.12-slim`, smoke test en `/login/`, `/` y `/static/*`
  devolviendo OK).
- README actualizado: sección "Dependencias" refleja Python 3.12, Django
  5.2 LTS, uv como gestor recomendado, PostgreSQL 18 (Docker),
  WeasyPrint como generador de PDF y nota sobre el anclaje del stack de NFe.
- Sustituido `locale.format()` (eliminado en Python 3.12) por
  `locale.format_string()` en 68 ocurrencias (apps de ventas, compras,
  stock, financiero y tests). Firma idéntica, sin cambio de
  comportamiento.
- `requirements.txt` pasa a ser generado vía `uv export`, sincronizado
  con `uv.lock`.
- `djangosige.__init__.__version__` pasa a `'2.0'` (estaba en `'0.0.1'`,
  desalineado con el lanzamiento v2.0.0). Plantilla `base.html` vuelve a usar
  `{{versao}}` (context processor `sige_version` ya existente).

### Corregido

- **Seguridad: 8 vistas AJAX `Info*` pasaban por Django sin verificación
  de permiso (#143, ALTA).** `InfoCliente`, `InfoFornecedor`,
  `InfoEmpresa`, `InfoTransportadora`, `InfoProduto`, `InfoVenda`,
  `InfoCompra` e `InfoCondicaoPagamento` heredaban de
  `django.views.generic.View` en lugar de `CustomView`, saltándose el
  `CheckPermissionMixin`. Cualquier usuario autenticado podía leer
  datos sensibles (CPF, CNPJ, RG, direcciones, precios, condiciones de
  pago) solo con el id del registro. Cada vista ahora hereda de
  `CustomView` y exige el permiso `view_<modelo>` correspondiente.
  Regresión cubierta por `djangosige/tests/test_security_ajax_views.py`
  (8 tests). Otros hallazgos del issue (state-changing via GET,
  bulk-delete sin verificación de propiedad) siguen abiertos.
- **Importación de NF-e v4.0 se rompía con `NOT NULL constraint failed:
  fiscal_notafiscal.dhemi` (#122).** Cuando el XML no traía `<dhEmi>`
  legible, `pysignfe` devolvía `None` para `nfe.infNFe.ide.dhEmi.valor`
  y el `save()` de `NotaFiscalSaida`/`NotaFiscalEntrada` violaba el NOT
  NULL. En `processador_nf.py`, tanto `importar_xml_cliente` como
  `importar_xml_fornecedor` ahora caen a `datetime.now()` cuando el
  valor del XML está ausente — el mismo fallback que ya se usaba en la
  ruta de creación manual de la nota. Tests de regresión en
  `djangosige/tests/fiscal/test_processador_nf.py`.
{% raw %}
- Las plantillas de listado usaban `{% ifequal %}` / `{% endifequal %}`,
  tags eliminadas en Django 4.0. Las 5 ocurrencias
  (`cliente_list_table.html`, `fornecedor_list_table.html`,
  ...
  fueron sustituidas por `{% if A == B %}…{% endif %}`.
{% endraw %}
- Login devolvía `403 Forbidden (Origin checking failed)` bajo proxy
  inverso (`nginx` en `8000:80` → `gunicorn`). Se añadió
  `CSRF_TRUSTED_ORIGINS` en `settings.py` (leído vía `decouple`) y la env
  `CSRF_TRUSTED_ORIGINS=http://localhost:8000,http://127.0.0.1:8000` en
  `docker-compose.yml`.
- Pie de página de la plantilla `base/base.html`: año `2017` → `2026` y versión
  (antes era el placeholder `{{versao}}` que renderizaba vacío por falta
  de context processor) → `2.0`.

### Cambiado

- README ganó sección "Comandos útiles" de Docker con ejemplos de
  `docker exec -it`, `docker compose logs -f --tail=N` y atajos
  (`ps`, `restart`, `down`). Incluye nota explícita sobre el orden de las
  flags `-it` (deben venir antes del nombre del contenedor).

### Añadido

- `database/` añadido al `.gitignore`. El volumen de postgres es creado
  como `root` por el contenedor y hacía que `git status` emitiera `Permiso
  denegado` en cada comando.

## [2.0.0] - 2026-05-23

Modernización de la infraestructura de construcción y ejecución. La aplicación en sí no
cambió — este lanzamiento se centra en cómo instalar, ejecutar y empaquetar el proyecto.

### Añadido

- `uv` como gestor de dependencias (`pyproject.toml`, `uv.lock`,
  `.python-version`), manteniendo `requirements.txt` en paralelo para quien
  prefiera `pip`.
- `.dockerignore` excluyendo `.git`, `.venv`, `database/` y configs locales
  de herramientas (`.claude/`, `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`).
- Red dedicada `sige_net` en `docker-compose.yml`.
- Healthcheck de Postgres con `depends_on: service_healthy` en el servicio
  `gunicorn`, garantizando orden correcta de inicialización.

### Cambiado

- Imagen Docker migrada de `alpine:3.7` a `python:3.10-slim`, con
  dependencias instaladas vía `uv` en `/opt/venv` (evita ser enmascarado
  por el bind mount de `docker-compose`).
- Postgres actualizado a `postgres:18-alpine`. Punto de montaje ajustado
  a `/var/lib/postgresql` (nuevo estándar con subdirectorios versionados).
- Servicio `gunicorn` ahora usa `build: .` en lugar de imagen preexistente.
- `ALLOWED_HOSTS` expandido para incluir `nginx`, `localhost` y
  `127.0.0.1`.
- `nginx` expuesto en `8000:80` (en lugar de `80:80`) para evitar conflicto
  con puertos privilegiados.
- `MAINTAINER` (deprecado) sustituido por `LABEL maintainer`.
- `.gitignore` pasa a ignorar configs locales de asistentes de código
  (`CLAUDE.md`, `.claude/`, `AGENTS.md`, `GEMINI.md`, `.cursor*`,
  `.aider*`, `docs/superpowers/`).

### Eliminado

- Atributos legados de `docker-compose.yml`: clave `version:` (obsoleta en
  Compose v2) y `links:` (sustituidos por red dedicada).

## [1.1.0] - 2026-05-22

Línea base pre-modernización. Esta entrada consolida el estado de djangoSIGE
acumulado en 164 commits anteriores hasta el ajuste de compatibilidad con
Django 5.x.

### Funcionalidades principales

- Registros: clientes, proveedores, transportistas, productos y
  empresas.
- Autenticación: login/logout, perfil por usuario, control de
  permisos.
- Presupuestos y pedidos de compra/venta con generación de PDF
  ([geraldo](https://github.com/thiagopena/geraldo)).
- Módulo financiero: plan de cuentas, flujo de caja y asientos.
- Módulo de stock.
- Módulo fiscal: emisión de NF-e/NFC-e versión 4.0, validación de XML,
  descarga, consulta, cancelación y manifestación del destinatario;
  comunicación con SEFAZ vía [PySIGNFe](https://github.com/thiagopena/PySIGNFe).
- Interfaz en portugués.

### Stack

- Python 3.10+
- Django 5.2.14
- Soporte a SQLite (dev) y Postgres (producción).

[No lanzado]: https://github.com/thiagopena/djangoSIGE/compare/v2.0.0...HEAD
[2.0.0]: https://github.com/thiagopena/djangoSIGE/compare/v1.1.0...v2.0.0
[1.1.0]: https://github.com/thiagopena/djangoSIGE/releases/tag/v1.1.0
