# CarpoolUC

Aplicación web para coordinar carpools universitarios. Recoge las preferencias de horario de cada persona y calcula automáticamente qué conductores deben manejar cada día y cuántos pasajeros lleva cada auto, minimizando los viajes necesarios.

Stack: **Python · Flask · pandas · PuLP (programación lineal entera)**

## Qué hace

1. **Formulario de preferencias** – cada persona indica, de lunes a jueves, su hora de ida y de vuelta, si puede manejar, si es flexible para moverse un bloque, y si es voluntaria para un segundo viaje.
2. **Panel de administración** (`/admin`) – ver, editar y eliminar respuestas, y descargar el CSV.
3. **Optimizador en dos etapas** (`/optimizar`):
   - **Etapa 1 – Consolidación de demanda:** mueve a las personas flexibles hacia bloques horarios con mayor densidad (ida: hasta un bloque antes; vuelta: hasta un bloque después) para llenar mejor los autos.
   - **Etapa 2 – Asignación de conductores:** modelo de programación lineal entera (PuLP) que decide quién maneja en cada bloque respetando la capacidad del vehículo, la disponibilidad declarada, el límite de viajes por persona y las excepciones de compensación.
4. **Resultados** – cobertura y déficit por día y bloque, conductores asignados y usuarios reubicados, en HTML y en JSON (`/api/optimizar`).

## Estructura

```
CarpoolUC/
├── app.py                  # Aplicación Flask: formulario, admin, rutas de optimización y API
├── optimizador.py          # Lógica de optimización (ConsolidadorDemanda + OptimizadorConductores)
├── templates/              # Vistas Jinja2 (formulario, admin, edición, resultados)
├── tests/                  # Tests con pytest para el optimizador
├── data/datos_ejemplo.csv  # Datos ficticios para probar la app
├── requirements.txt        # Dependencias de ejecución
└── requirements-dev.txt    # Dependencias de desarrollo (pytest)
```

## Instalación y uso

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows  (Linux/Mac: source .venv/bin/activate)
pip install -r requirements.txt

# Opcional: arrancar con datos de ejemplo
cp data/datos_ejemplo.csv datos_usuarios.csv

python app.py
```

Abrir `http://localhost:5000`. Si no existe `datos_usuarios.csv`, se crea vacío al iniciar.

| Ruta | Descripción |
|---|---|
| `GET /` | Formulario de preferencias |
| `POST /submit` | Guarda una respuesta |
| `GET /admin` | Panel de administración |
| `GET /admin/editar/<i>` · `POST /admin/eliminar/<i>` | Editar / eliminar un registro |
| `GET /download-csv` | Descargar los datos |
| `GET /optimizar?capacidad=4` | Ejecutar la optimización y ver resultados |
| `GET /api/optimizar?capacidad=4` | Resultados en JSON |
| `GET /api/estadisticas` | Conteos rápidos sin optimizar |

`capacidad` es el número de pasajeros por auto sin contar al conductor (por defecto 4).

## Tests

```bash
pip install -r requirements-dev.txt
pytest
```

> Estado actual: 29 de 30 tests pasan. `test_detecta_voluntarios_segundo_viaje` falla y está pendiente de revisión.

## Formato de datos

Un CSV con una fila por persona: `Nombre`, `Timestamp`, y para cada día (`Lunes`…`Jueves`) las columnas `_Ida`, `_Vuelta`, `_Conductor`, `_Flex_Ida`, `_Flex_Vuelta`; más `Voluntario_Segundo_Viaje`. Ver [data/datos_ejemplo.csv](data/datos_ejemplo.csv). Los datos reales (`datos_usuarios.csv`) están excluidos del repositorio por contener nombres.

## Posibles mejoras

- **Persistencia real:** reemplazar el CSV por SQLite/PostgreSQL (hoy hay riesgo de escrituras concurrentes y los registros se editan por índice).
- **Configuración externa:** las excepciones de compensación y los nombres bloqueados están escritos en el código (`optimizador.py`, `app.py`, `index.html`); moverlos a un archivo de configuración o a la base de datos.
- **Autenticación:** `/admin` y las rutas de edición están abiertas; agregar login y protección CSRF.
- **Calidad del código:** dividir `optimizador.py` (~900 líneas) en módulos, sacar el HTML embebido de `/success` a una plantilla, usar el logger en lugar de `logging.basicConfig` a nivel de módulo.
- **Más tests:** cubrir las rutas Flask y arreglar el test fallido.
- **Despliegue:** Dockerfile, `gunicorn` y CI con GitHub Actions; desactivar `debug=True` en producción.
- **Configurabilidad:** días, bloques horarios y capacidad parametrizables desde la interfaz.
- **Visualización:** gráficos de ocupación por día y bloque en la página de resultados.
