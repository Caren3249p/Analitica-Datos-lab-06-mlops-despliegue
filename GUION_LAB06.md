# Guion — Lab 06: MLOps y Despliegue

Duración: unos 5 minutos. Entre corchetes, lo que se muestra en pantalla.

---

## 1. Qué hice (30 s)

"Hola, soy Caren. En este laboratorio no entrené un modelo nuevo: conecté todo lo del semestre. Con un solo comando se entrenan y guardan dos modelos con los datos de Olist, y con otro se levantan como una API."

[Mostrar el diagrama: extract → quality → transform → train → evaluate → pipeline.py → api.py]

## 2. pipeline.py (1 min 30 s)

[Abrir `src/analytics/pipeline.py`]

"`ejecutar_pipeline` hace seis pasos: cargar, validar calidad, transformar, unir reseñas, entrenar y guardar. Tres decisiones importantes:"

- **Se detiene si los datos son inválidos.** "Valido con pandera antes de entrenar. Si hay errores, lanza `ValueError` y no guarda ningún modelo."
- **Reseñas sin calificación.** "Un pedido sin reseña no debe contar como reseña positiva. Uso `.where` para conservar el vacío y luego esas filas se descartan."
- **Trazabilidad.** "Guardo los modelos y un `metricas.json` con la fecha de entrenamiento y la versión de scikit-learn."

## 3. api.py (1 min)

[Abrir `src/analytics/api.py`]

"La API tiene tres rutas: `/salud`, `/predecir/entrega` y `/predecir/resena`. Son dos endpoints porque cada modelo predice en un momento distinto: uno al comprar y otro cuando el pedido ya fue entregado."

- "Los modelos se cargan una sola vez, al crear la app."
- "Pydantic valida las entradas: con `n_items = 0` responde 422 sin llegar al modelo."
- "Si `ticket_promedio_historico` llega vacío, el imputer del Pipeline lo resuelve."

## 4. Resultados y demo (1 min 30 s)

[Mostrar `uv run pytest`: 83 tests en verde, y `ruff` limpio]

[Correr el pipeline, levantar la API y probar los endpoints en `http://localhost:8000/docs`]

- Entrega: error promedio de unos 5 días (MAE ≈ 4.97).
- Reseña negativa: ROC-AUC ≈ 0.77 y F1 ≈ 0.47.

"La accuracy (0.80) queda por debajo del baseline (0.85), pero con clases desbalanceadas eso engaña: el baseline nunca detecta una reseña negativa y el modelo sí."

## 5. Reflexión y cierre (30 s)

"Sobre las preguntas del análisis, en pocas palabras:"

- **9.1** No se puede encadenar un modelo con otro: en producción recibiría predicciones, no los valores reales con los que se entrenó.
- **9.2** Detenerse si el problema es grande o estructural; filtrar si son pocas filas aisladas.
- **9.3** Guardar la versión de scikit-learn evita cargar un modelo con una versión incompatible.
- **9.4** Hay que vigilar si cambian los datos que llegan y si empeora el error real.

"En resumen: un comando entrena y guarda, otro sirve la API, y los tests garantizan que todo encaja. Gracias."

---

## Antes de entregar

- [ ] CI en verde en GitHub Actions
- [ ] Captura de `/docs` con una predicción exitosa, en el PR
- [ ] Reflexión (`LAB06_REFLEXION.md`) pegada en el PR
- [ ] Link del repo en el campus virtual
