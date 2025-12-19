# 📉🗓️💰 Predicción de churn — Clientes Paquete VIP (Banca) | LightGBM + Validación temporal + Optimización de Ganancia

Proyecto end-to-end para **predecir churn** de clientes asociados a un **paquete VIP** en un banco, con foco en **maximizar retorno económico** de una campaña de retención.

En lugar de optimizar métricas genéricas (AUC/F1), el sistema prioriza una **función de ganancia de negocio**, y produce como salida un **ranking** de clientes junto con un **corte óptimo (top-N)** de clientes a contactar.

---

## 🎯 Qué resuelve (visión de negocio)

**Problema:** dado un universo mensual de clientes VIP, identificar a quiénes conviene incentivar para evitar baja.  
**Decisión:** contactar a un subconjunto limitado (capacidad de campaña).  
**Objetivo:** maximizar ganancia esperada considerando:
- ✅ beneficio por retener un cliente de alto riesgo,
- 💸 costo por contactar a un cliente que no iba a darse de baja.

**Salida final:** archivo de scoring para campaña con los `N` clientes priorizados.

---

## 🧠 Enfoque de modelado

1. **Modelo de ranking**: LightGBM predice probabilidad de churn y ordena clientes.
2. **Optimización orientada a ganancia**: se evalúa el retorno económico al contactar a los top-N.
3. **Selección robusta de N**: se elige un `N` que maximiza ganancia de forma **estable** (evitando decisiones “finas” que cambian mucho ante pequeñas variaciones).
4. **Validación temporal**: splits por mes para simular condiciones reales (entrenar en pasado, evaluar en meses posteriores), evitando leakage.

---

## 🧩 Features y calidad de datos

Feature engineering diseñado para series temporales por cliente (mensual):

- **Detección de datos anómalos por mes**: identifica variables “rotas” (por ejemplo, todo en cero para todos los clientes) y las marca como faltantes.
- **Normalización robusta de variables monetarias por mes**: corrige cambios de escala / drift usando transformaciones robustas por período.
- **Features temporales**:
  - lags,
  - deltas (cambios vs meses anteriores),
  - tendencia (pendiente) en ventanas.
- **Ratios** entre variables relevantes (ej. consumos vs límites) para capturar comportamiento relativo.

---

## 🧪 Reproducibilidad y experimentación

- **Gestión de experimentos**: cada corrida genera un identificador único y guarda:
  - logs,
  - base de Optuna,
  - modelos entrenados,
  - predicciones finales.
- **Reutilización de modelos**: si un modelo ya existe para una semilla/configuración, se carga sin reentrenar.
- **Ensembles multi-semilla**: reduce varianza y mejora estabilidad del ranking.

---

## 🗂️ Estructura del repositorio

```text
churn_bancario_machine_learning/
├─ config/                     # parámetros, paths, splits temporales, seeds, ganancia
├─ src/                        # carga/splits, optimización, entrenamiento, predicción
├─ target_feature_engineering/ # creación de target + feature engineering
├─ main.py                     # pipeline completo (train → validaciones → test → CSV final)
├─ requirements.txt
└─ crear_venv.sh
```


### 🧱 Módulos principales

- `target_feature_engineering/crear_target.ipynb`  
  Construye el target (clases de churn) a partir de la historia temporal del cliente.

- `target_feature_engineering/run_fe.py` + `features.py`  
  Ejecuta feature engineering y genera el dataset final en Parquet.

- `src/data_load_preparation.py`  
  Carga eficiente, crea targets/pesos y arma splits por meses.

- `src/optuna_optimization.py`  
  Optimiza hiperparámetros maximizando ganancia en validación temporal.

- `src/training_predict.py`  
  Entrena ensemble final, rankea y aplica top-N para generar el output.

- `main.py`  
  Orquesta todo el pipeline y genera el CSV final de campaña.

---

## 🚀 Cómo correr el proyecto

Seguir estos pasos en orden para configurar el entorno y ejecutar el pipeline de datos.

### 1) Configuración del Entorno

Primero, crear y activar el entorno virtual, e instalar las dependencias necesarias:

```bash
bash crear_venv.sh
source venv/bin/activate
pip install -r requirements.txt
```

### 2) Generar dataset con target (si no existe)

Ejecutar el notebook:

- `target_feature_engineering/crear_target.ipynb`

Este paso genera el dataset con la columna `clase_ternaria` y lo guarda en el path configurado en `DATASET_TARGETS_CREADOS_PATH` (ver `config/config.py`).


### 3) Generar features (Parquet)

```bash
python -m target_feature_engineering.run_fe
```
Este paso lee el dataset con target y produce el dataset final con feature engineering en formato Parquet (guardado en `FE_PATH`).

### 4) Entrenar + seleccionar top-N + exportar archivo final

```bash
python main.py
```

Outputs principales (se crean dentro del directorio del experimento):

- **Logs**: `.../logs/`
- **Modelos**: `.../modelos_*`
- **Predicción final (archivo de campaña/envío)**: `.../resultados_prediccion/envio_<experimento>_N<k>.csv`
