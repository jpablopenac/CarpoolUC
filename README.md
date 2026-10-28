# CarpoolUC

Aplicacion web para coordinar carpools universitarios. Recoge las preferencias de horario de cada persona y calcula automaticamente que conductores deben manejar cada dia y cuantos pasajeros lleva cada auto, minimizando los viajes necesarios.

Stack: **Python · Flask · pandas · PuLP (programacion lineal entera)**

## Que hace

1. **Formulario de preferencias** – cada persona indica, de lunes a jueves, su hora de ida y de vuelta, si puede manejar, si es flexible para moverse un bloque, si es voluntaria para un segundo viaje y, opcionalmente, a quien cubre (ver [Cobertura entre usuarios](#cobertura-entre-usuarios)).
2. **Panel de administracion** (`/admin`) – ver, editar y eliminar respuestas, y descargar el CSV.
3. **Optimizador en dos etapas** (`/optimizar`):
   - **Etapa 1 – Consolidacion de demanda:** mueve a las personas flexibles hacia bloques horarios con mayor densidad (ida: hasta un bloque antes; vuelta: hasta un bloque despues) para llenar mejor los autos.
   - **Etapa 2 – Asignacion de conductores:** modelo de programacion lineal entera (PuLP) que decide quien maneja en cada bloque respetando la capacidad del vehiculo, la disponibilidad declarada, el limite de viajes por persona y las coberturas entre usuarios.
4. **Resultados** – cobertura y deficit por dia y bloque, conductores asignados y usuarios reubicados, en HTML y en JSON (`/api/optimizar`).

## Estructura

```
CarpoolUC/
├── app.py                  # Aplicacion Flask: formulario, admin, rutas de optimizacion y API
├── optimizador.py          # Logica de optimizacion (ConsolidadorDemanda + OptimizadorConductores)
├── templates/              # Vistas Jinja2 (formulario, admin, edicion, resultados)
├── tests/                  # Tests con pytest para el optimizador
├── data/datos_ejemplo.csv  # Datos ficticios para probar la app
├── requirements.txt        # Dependencias de ejecucion
└── requirements-dev.txt    # Dependencias de desarrollo (pytest)
```

## Instalacion y uso

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows  (Linux/Mac: source .venv/bin/activate)
pip install -r requirements.txt

# Opcional: arrancar con datos de ejemplo
cp data/datos_ejemplo.csv datos_usuarios.csv

python app.py
```

Abrir `http://localhost:5000`. Si no existe `datos_usuarios.csv`, se crea vacio al iniciar.

| Ruta | Descripcion |
|---|---|
| `GET /` | Formulario de preferencias |
| `POST /submit` | Guarda una respuesta |
| `GET /admin` | Panel de administracion |
| `GET /admin/editar/<i>` · `POST /admin/eliminar/<i>` | Editar / eliminar un registro |
| `GET /download-csv` | Descargar los datos |
| `GET /optimizar?capacidad=4` | Ejecutar la optimizacion y ver resultados |
| `GET /api/optimizar?capacidad=4` | Resultados en JSON |
| `GET /api/estadisticas` | Conteos rapidos sin optimizar |

`capacidad` es el numero de pasajeros por auto sin contar al conductor (por defecto 4).

## Tests

```bash
pip install -r requirements-dev.txt
pytest
```

## Formato de datos

Un CSV con una fila por persona: `Nombre`, `Timestamp`, y para cada dia (`Lunes`…`Jueves`) las columnas `_Ida`, `_Vuelta`, `_Conductor`, `_Flex_Ida`, `_Flex_Vuelta`; mas `Voluntario_Segundo_Viaje` y `Cubre_A` (opcional). Ver [data/datos_ejemplo.csv](data/datos_ejemplo.csv). Los datos reales (`datos_usuarios.csv`) estan excluidos del repositorio por contener nombres.

### Cobertura entre usuarios

Una persona puede cubrir a otra: quien cubre maneja 1 dia extra por cada persona cubierta (2 dias en total si cubre a una) y la persona cubierta puede viajar sin manejar. La cobertura se declara en la columna `Cubre_A` de quien cubre, con los nombres separados por `;` (por ejemplo `Ana;Luis`), y se elige desde el formulario o desde la edicion en `/admin`.

Reglas:
- Quien cubre no puede ser voluntario de segundo viaje (el formulario lo desactiva).
- Una persona solo puede ser cubierta por una persona, y quien esta cubierto no puede cubrir a otros.
- Si la cobertura no se puede cumplir (quien cubre no tiene disponibilidad, o no tiene suficientes dias con ida y vuelta disponibles para manejar), el optimizador la ignora y avisa en el log: la persona cubierta maneja como cualquier otra.

## Posibles mejoras

- **Persistencia real:** reemplazar el CSV por SQLite/PostgreSQL (hoy hay riesgo de escrituras concurrentes y los registros se editan por indice).
- **Autenticacion:** `/admin` y las rutas de edicion estan abiertas; agregar login y proteccion CSRF.
- **Calidad del codigo:** dividir `optimizador.py` (~1200 lineas) en modulos, sacar el HTML embebido de `/success` a una plantilla, usar el logger en lugar de `logging.basicConfig` a nivel de modulo.
- **Mas tests:** cubrir las rutas Flask (formularios y validacion de `Cubre_A`).
- **Despliegue:** Dockerfile, `gunicorn` y CI con GitHub Actions; desactivar `debug=True` en produccion.
- **Configurabilidad:** dias, bloques horarios y capacidad parametrizables desde la interfaz.
- **Visualizacion:** graficos de ocupacion por dia y bloque en la pagina de resultados.
