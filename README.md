# DjangoSIGE [![Build Status](https://travis-ci.org/thiagopena/djangoSIGE.svg?branch=master)](https://travis-ci.org/thiagopena/djangoSIGE)

Sistema Integrado de Gestión Empresarial basado en Django

Proyecto independiente open-source desarrollado en Python 3 en Windows, probado en GNU/Linux y Windows.

## Dependencias

- [Python](https://www.python.org/downloads/) — 3.12 (definido en `.python-version` y `pyproject.toml`)
- [Django](http://www.djangoproject.com) — 5.2.x (línea LTS, `>=5.2,<5.3`)
- [PostgreSQL](https://www.postgresql.org/) — 18 (vía Docker) o compatible
- [uv](https://docs.astral.sh/uv/) (recomendado) — gestiona el entorno y las dependencias a partir de `pyproject.toml` / `uv.lock`
- [WeasyPrint](https://weasyprint.org/) — generación de PDF a partir de plantillas HTML/CSS (sustituye a `geraldo`, ver issue [#142](https://github.com/thiagopena/djangoSIGE/issues/142)). En Linux exige las librerías `libpango-1.0-0`, `libcairo2`, `libgdk-pixbuf-2.0-0`, `libharfbuzz0b` y `libfontconfig1` — el `Dockerfile` ya las instala.
- [PySIGNFe](https://github.com/thiagopena/PySIGNFe) (opcional) — generación de NF-e/NFC-e, comunicación con SEFAZ, DANFE. Mantiene ancladas las versiones antiguas de `cryptography==2.9.2`, `pyOpenSSL==17.5.0` y `signxml==2.5.2`, sin las cuales la emisión se rompe.
- [apache2](https://www.apache.org/) + [mod_wsgi](https://modwsgi.readthedocs.io/en/develop/) (opcional, alternativo a Docker)


## Capturas de pantalla

### Login

![](img/login1.png)

### Dashboard

![](img/dashboard.png)


## Instalación

1. Clone el repositorio:

```bash
git clone https://github.com/thiagopena/djangoSIGE.git
cd djangoSIGE
```

### Opción A — uv (recomendado)

[uv](https://docs.astral.sh/uv/) crea el entorno virtual e instala las
dependencias a partir del `pyproject.toml`/`uv.lock` en un solo paso,
fijando la versión de Python definida en `.python-version`. En **Linux** y
**Windows** no necesita instalar nada previamente (ni el Python) —
el propio `uv` descarga el intérprete y resuelve las dependencias a partir
de ruedas precompiladas.

Instale `uv` (ver [documentación oficial](https://docs.astral.sh/uv/getting-started/installation/)):

```bash
# Linux / macOS
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```powershell
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Sincronice las dependencias (crea `.venv` automáticamente):

```bash
uv sync
```

A partir de aquí, anteceda los comandos `manage.py` com `uv run`:

```bash
uv run python contrib/env_gen.py
uv run python manage.py migrate
uv run python manage.py createsuperuser
uv run python manage.py runserver
```

### Opción B — pip + venv (alternativa)

**Requisitos previos en Linux** (Debian/Ubuntu) — necesarios para compilar
extensiones nativas (`lxml`, `cryptography`):

```bash
sudo apt install -y libxml2 gcc python3-dev libxml2-dev libxslt1-dev zlib1g-dev git
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.12 python3.12-venv python3.12-dev
```

**Requisitos previos en Windows:**

- Instale [Python 3.12](https://www.python.org/downloads/) (marque
  *Add Python to PATH* durante el instalador).
- Instale [Git para Windows](https://git-scm.com/download/win).
- Algunas dependencias nativas (`lxml`, `cryptography==2.9.2`) pueden
  exigir [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)
  en caso de que `pip` no encuentre una rueda lista. Si es posible, prefiera la
  Opción A (uv), que evita este paso.

**Crear el entorno:**

```bash
# Linux / macOS
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

```powershell
# Windows (PowerShell)
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

A continuación, con el `.venv` activado:

```bash
python contrib/env_gen.py
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

### Post-instalación (ambas opciones)

2. Edite el contenido del archivo **djangosige/configs/configs.py**.

3. Acceda a `http://localhost:8000` en el navegador.

### Poblar la base de datos con datos de ejemplo (opcional)

El comando `create_data` popula la base de datos con datos realistas en portugués
(locale `pt_BR`, com CPF/CNPJ válidos generados por
[Faker](https://faker.readthedocs.io/)):

```bash
# uv
uv run python manage.py create_data

# pip + venv (com o .venv ativado)
python manage.py create_data

# Docker
docker compose exec gunicorn python manage.py create_data
```

#### Que se puebla

| App        | Modelo / tabela            | Default | Observações                                                          |
|------------|----------------------------|--------:|----------------------------------------------------------------------|
| auth       | `User`                     |       3 | Usuarios comunes (contraseña `senha123`). Superusers/staff se preservan.|
| login      | `Usuario`                  |       3 | Perfil 1:1 ligado a cada `User` recién creado.                        |
| cadastro   | `Empresa` (+ `PessoaJuridica`) | 2   | CNPJ, nome fantasia, CNAE, regime tributário.                        |
| cadastro   | `Cliente` (+ `PessoaFisica`/`PessoaJuridica`) | 15 | Mezcla PF/PJ aleatoria, con `limite_de_credito`.       |
| cadastro   | `Fornecedor` (+ `PessoaJuridica`) | 8 | Siempre PJ, con ramo de actividad.                                    |
| cadastro   | `Transportadora` (+ `PessoaJuridica`) | 3 | Siempre PJ.                                                       |
| cadastro   | `Endereco`, `Telefone`, `Email`, `Banco` | 1 por pessoa | Creados y vinculados como `_padrao` de cada persona.       |
| cadastro   | `Produto`                  |      25 | Código secuencial `PRD00001…`, EAN13, NCM, costo/venta coherentes.    |
| cadastro   | `Categoria`, `Marca`, `Unidade` | fixo (8/8/6) | Conjunto fijo vía `get_or_create` — nunca duplica.           |

Los módulos `vendas`, `compras`, `estoque`, `financeiro` e `fiscal` **no
se puebla** con el comando (implican reglas tributarias y flujos de trabajo
más complejos).

#### Flags

- `--clear` — borra todos los datos de ejemplo antes de recrear
(preserva superusers y staff creados manualmente).
- `--seed N` — fija la semilla del Faker para resultados reproducibles.
- `--clientes N`, `--fornecedores N`, `--produtos N`, `--empresas N`,
  `--transportadoras N`, `--usuarios N` — ajusta cada cantidad
individualmente (ver `--help` para los valores por defecto).

Ejemplo vaciando la base de datos y generando un conjunto mayor:

```bash
docker compose exec gunicorn python manage.py create_data \
    --clear --clientes 50 --produtos 100
```

### Docker (opcional)

También hay un `docker-compose.yml` con Postgres 18, Gunicorn y Nginx:

```bash
docker compose up -d
docker compose exec gunicorn python manage.py migrate
docker compose exec gunicorn python manage.py createsuperuser
```

La aplicación estará disponible en `http://localhost:8000`.

#### Comandos útiles

Abrir un shell interactivo dentro del contenedor de la aplicación (atención: las
flags `-it` deben ir antes del nombre del contenedor):

```bash
docker exec -it djangosige-gunicorn-1 bash
# equivalente via compose:
docker compose exec gunicorn bash
```

Seguir los logs en tiempo real (`-f` = follow, `--tail=N` limita el
historial inicial):

```bash
# todos os servicos
docker compose logs -f

# apenas o gunicorn (com as ultimas 100 linhas)
docker compose logs -f --tail=100 gunicorn

# equivalente sem compose
docker logs -f djangosige-gunicorn-1
```

Otros atajos útiles:

```bash
docker compose ps              # status dos containers
docker compose restart gunicorn
docker compose down            # derruba o stack (preserva volumes)
```

## Implementaciones

- Registro de productos, clientes, empresas, proveedores y transportistas
- Login/Logout
- Creación de perfil para cada usuario.
- Definición de permisos para usuarios.
- Creación y generación de PDF para presupuestos y pedidos de compra/venta
- Módulo financiero (Plan de Cuentas, Flujo de Caja y Asientos)
- Módulo para control de stock
- Módulo fiscal:
    - Generación y almacenamiento de notas fiscales
    - Validación del XML de NF-e/NFC-e
    - Emisión, descarga, consulta y cancelación de NF-e/NFC-e **(Probar en entorno de homologación)**
    - Comunicación con SEFAZ (Consulta de registro, inutilización de notas, manifestación del destinatario)
- Interfaz simple y en portugués

## Créditos

- [AdminBSBMaterialDesign](https://github.com/gurayyarar/AdminBSBMaterialDesign)
- [WeasyPrint](https://weasyprint.org/)
- [jQuery-Mask-Plugin](https://igorescobar.github.io/jQuery-Mask-Plugin/)
- [DataTables](https://datatables.net/)
- [JQuery multiselect](http://loudev.com/)

## Ayuda

Para reportar bugs o hacer preguntas utilice el [Issues](https://github.com/thiagopena/djangoSIGE/issues) o vía email thiagopena01@gmail.com

Como este es un proyecto en desarrollo, cualquier comentario será bienvenido.
