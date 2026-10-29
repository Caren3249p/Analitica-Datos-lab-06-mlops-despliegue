# Guion — Lab 06: MLOps y Despliegue (Analítica de Datos)

Duración aproximada: 8–10 minutos. Entre corchetes va lo que se muestra en pantalla.

---

## 1. Apertura (30 s)

"Hola, soy Caren. Este es el Laboratorio 06 de Analítica de Datos. A diferencia de los anteriores, aquí no entrenamos un modelo nuevo: conectamos todo lo que hicimos en el semestre. El objetivo fue que, con un solo comando, el proyecto vaya de los CSV de Olist a dos modelos entrenados y guardados, y que esos modelos se puedan consultar como un servicio web."

[Mostrar el diagrama de la sección 2 de la guía: extract → quality → transform → train → evaluate → pipeline.py → models/ → api.py]

## 2. Cómo llegué: el orden de trabajo (1 min)

"Seguí el ciclo que recomienda la guía: empezar por lo que no depende de nada y avanzar hacia lo integrado."

1. Primero las funciones pequeñas de `pipeline.py`: `unir_reviews` y `preparar_datos`.
2. Luego `entrenar_y_evaluar` y `guardar_artefactos`.
3. Después `ejecutar_pipeline`, que encadena los módulos de los Labs 02, 03 y 04.
4. Al final `api.py`, porque necesita que existan los modelos guardados.
5. Con cada paso corrí los tests; al terminar, el pipeline sobre los datos reales.

[Mostrar `uv run pytest -v` en verde]

## 3. pipeline.py — las decisiones importantes (3 min)

[Abrir `src/analytics/pipeline.py`]

**unir_reviews.** "Un pedido puede tener varias reseñas. Si hago un merge directo, se multiplican las filas. Por eso me quedo con la reseña más reciente por `review_answer_timestamp` y uso `join_verificado` del Lab 02, que lanza error si el número de filas cambia."

**entrenar_y_evaluar.** "En clasificación uso `stratify=y`. `review_negativa` tiene cerca de 15 % de positivos; sin estratificar, el test podría quedar con muy pocos y las métricas serían poco confiables. Las métricas del baseline se combinan con el operador `|` de diccionarios."

**guardar_artefactos.** "Guardo cada pipeline como `.joblib` y un `metricas.json` con la fecha de entrenamiento en UTC y la versión de scikit-learn. Eso da trazabilidad: sé con qué se entrenó cada modelo. Uso `ensure_ascii=False` para que las tildes no se guarden como códigos."

**ejecutar_pipeline.** Tres detalles que quiero destacar:

- **Se detiene si los datos son inválidos.** "Valido `orders` e `items` con los schemas de pandera *antes* de entrenar. Si hay errores lanzo `ValueError` y no se guarda ningún modelo. Es mejor fallar que entrenar con datos que violan el contrato."
- **El error silencioso de las reseñas.** "Si `review_score` es NaN, la comparación `NaN <= 2` da `False`, no NaN. Un pedido sin reseña entraría al entrenamiento como 'reseña positiva'. Lo evito con `.where(df['review_score'].notna())`, que conserva el NaN, y luego `preparar_datos` descarta esas filas."
- **customer_unique_id, no customer_id.** "En Olist `customer_id` cambia en cada pedido; el cliente real es `customer_unique_id`. Con `customer_id`, el ticket promedio histórico habría sido NaN para todos."

"Además imprimo `[1/6]`, `[2/6]`… al empezar cada paso. Si el pipeline falla a las 3 a. m. en un servidor, esos mensajes dicen en qué paso quedó."

## 4. api.py — la API con FastAPI (2 min)

[Abrir `src/analytics/api.py`]

"Usé una función fábrica, `crear_app()`, en vez de una variable global `app`. Así, importar el módulo no falla aunque `models/` esté vacío, los tests apuntan a modelos de prueba en una carpeta temporal, y uvicorn la crea al arrancar con `--factory`."

- **Carga única.** "Los modelos se cargan una sola vez al crear la app, no en cada petición. Si falta algún archivo lanzo `FileNotFoundError` con un mensaje que indica qué ejecutar."
- **Validación gratis con pydantic.** "`PedidoCompra` y `PedidoEntregado` declaran rangos con `Field`. Si llega `n_items = 0`, FastAPI responde 422 sin que el modelo se entere."
- **Orden de columnas.** "Convierto el pedido con `model_dump()` y selecciono `[features]` con las listas `FEATURES_REGRESION` y `FEATURES_CLASIFICACION`; así el orden siempre es el del entrenamiento."
- **Valores nulos.** "`ticket_promedio_historico` puede ser `null` (primer pedido del cliente). No escribí ningún `if`: el `SimpleImputer` del Pipeline del Lab 04 se encarga."
- **predict_proba.** "Devuelve forma (1, 2); la probabilidad de reseña negativa es la columna 1."

**Dos endpoints, dos momentos.** "El modelo de entrega predice al momento de la compra; el de reseña, cuando el pedido ya fue entregado. No se pueden fusionar en uno: eso es lo que analizo en la pregunta 9.1."

## 5. Tests (1 min)

"La guía pide 16 tests nuevos (9 de pipeline y 7 de API) y 79 en total. En mi repositorio corren **83 tests y todos pasan**, y `ruff check` y `ruff format --check` salen limpios. Los tests de integración escriben CSV sintéticos con la estructura de Olist y ejecutan el pipeline completo; los de la API usan `TestClient`, sin levantar servidor."

[Mostrar la salida de pytest y ruff]

## 6. Demo con datos reales (1.5 min)

[Terminal 1]
```bash
uv run python -m analytics.pipeline --data-dir data/raw --models-dir models
```
"Se ven los seis pasos y al final se escribe `models/metricas.json`."

[Terminal 2]
```bash
uv run uvicorn analytics.api:crear_app --factory --port 8000
```
[Navegador: http://localhost:8000/docs → probar `/predecir/entrega` y `/predecir/resena`, y probar `n_items: 0` para mostrar el 422]

**Resultados (de `models/metricas.json`):**

- Regresión (tiempo de entrega): RMSE ≈ 8.18 días, MAE ≈ 4.97 días, R² ≈ 0.28.
- Clasificación (reseña negativa): ROC-AUC ≈ 0.77, F1 ≈ 0.47, accuracy ≈ 0.80.

"Un dato honesto: la accuracy del modelo (0.80) es menor que la del baseline de predecir siempre 'no negativa' (0.85). Pero con clases desbalanceadas la accuracy engaña: el baseline nunca detecta una reseña negativa, y el modelo sí detecta 1 766 de las 2 899 reales. Por eso miro F1 y ROC-AUC, que es lo que importa para que servicio al cliente contacte primero a quienes tienen más riesgo."

## 7. Reflexión (1.5 min)

Resumir las cuatro respuestas de `LAB06_REFLEXION.md`:

- **9.1 — Un solo endpoint.** "Pasarle al modelo de reseñas la *predicción* de tiempo de entrega, en vez del valor real con el que se entrenó, produce training-serving skew: la predicción es suavizada y casi nunca toma los valores extremos que justamente anticipan una reseña negativa."
- **9.2 — Detenerse o filtrar.** "Detenerse ante problemas estructurales o de muchas filas; filtrar si son pocas filas aisladas. El reporte de calidad (tasa de rechazo y lista de errores) ayuda a decidir."
- **9.3 — version_sklearn.** "Un `.joblib` solo es confiable con la misma versión de scikit-learn. Si no coincide, `crear_app` debería fallar al arrancar con un error claro, o como mínimo advertir y mostrar ambas versiones en `/salud`."
- **9.4 — Reentrenar.** "Data drift (cambian las features que llegan) se detecta de inmediato; performance drift (cambia el error real) llega con retraso, cuando se conoce la entrega o la reseña. Conviene vigilar ambos."

## 8. Extensión y cierre (30 s)

"También dejé un `Dockerfile` para contenerizar la API (extensión opcional). Copia `pyproject.toml` y `uv.lock` antes que `src/` para que Docker reutilice la capa de dependencias cuando solo cambia el código."

"En resumen: un comando entrena y guarda los modelos con trazabilidad, otro levanta la API que valida las entradas, y los tests garantizan que todo encaja. Gracias."

---

## Antes de entregar (checklist rápido)

- [ ] `uv run ruff check .` y `uv run ruff format --check .` sin errores
- [ ] `uv run pytest` en verde (83 tests)
- [ ] `git push` y confirmar el ✓ verde de CI en GitHub Actions
- [ ] Captura de pantalla de `/docs` con una predicción exitosa (la tienes que tomar tú; incluirla en el PR)
- [ ] Pegar `LAB06_REFLEXION.md` en el cuerpo del pull request
- [ ] Compartir el link del repositorio en el campus virtual
