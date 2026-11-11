# TP2 - Predicción de fumadores

El objetivo es entrenar un modelo de clasificación que prediga si una persona fuma smoking = 1 o no smoking = 0 a partir de datos de salud: edad, género, peso, presión, análisis de sangre, estado dental, etc.

Se entrenó con un dataset de 50.000 personas que ya tenía la respuesta, y con ese mismo proceso se predijo un segundo dataset de 5.692 personas sin la columna smoking. El resultado se guardó con una columna nueva llamada smoking_prediction. La métrica que se evalúa es el F1 de la clase 1 (fumadores).

## Cómo reproducirlo

1. Clonar el repositorio.
2. Instalar las librerías:
   ```
   pip install -r requirements.txt
   ```
3. Ejecutar los notebooks de la carpeta notebooks en orden, del 01 al 06. Cada uno usa archivos que deja guardados el anterior, así que no se pueden saltar.

Los archivos originales en Excel tienen que estar en data / raw.

## Estructura

```
data/
  raw/          smoking_prediction.xlsx (entrenamiento)
                smoking_prediction_entrega.xlsx (para predecir)
  processed/    datos corregidos, X_train, X_test, y_train, y_test
                smoking_prediction_entrega_resultado.csv (resultado final)
  external/     (vacía)
models/         limites_outliers, ohe, scaler, modelo_xgb, umbral (.joblib)
notebooks/
  01_lectura_y_discovery.ipynb
  02_eda.ipynb
  03_preprocesamiento.ipynb
  04_entrenamiento_y_optimizacion.ipynb
  05_validacion.ipynb
  06_prediccion.ipynb
requirements.txt
```

## Lo que fui haciendo

**Lectura de datos.** Lo primero que encontré fue que las columnas con decimales venían dañadas. Al pasar por Google Sheets se guardaron como horas: por ejemplo, la hemoglobina 14.5 aparecía como 0.6 (es decir, 14 horas y 5 minutos). Lo corregí multiplicando por 24 y separando horas y minutos. Usé los .xlsx en vez de los .csv porque los csv tenían los valores redondeados. La columna oral tenía un solo valor, así que la eliminé junto con el ID.

**EDA.** El 36,7% de las personas fuma. Lo más fuerte fue el género: fuma el 55% de los hombres y solo el 4% de las mujeres. La hemoglobina, la altura y el peso parecían relacionados con fumar, pero al separar por género casi no había diferencia; en realidad reflejaban que los hombres tienen valores más altos. También vi casi ninguna mujer menor de 40 años, valores imposibles (LDL mayor que el colesterol total) y valores tope como 999 en algunos análisis.

**Preprocesamiento.** Separé 70% para entrenar y 30% para test, manteniendo la misma proporción de fumadores. En vez de borrar outliers, los recorté al percentil 99, porque en el archivo de predicción no se puede borrar ninguna fila. Convertí gender y tartar con OneHotEncoder. Los límites, el encoder y el scaler se calcularon solo con el train y se guardaron para usarlos igual en la predicción.

**Modelos probados (F1 de fumadores):**

| Modelo | Train | Test |
|---|---|---|
| Árbol profundidad 3 | 0.66 | 0.65 |
| Árbol profundidad 6 | 0.69 | 0.68 |
| Árbol sin límite | 1.00 | 0.64 |
| KNN (71 vecinos) | 0.68 | 0.66 |
| Random Forest | 0.74 | 0.70 |
| XGBoost | 0.74 | 0.70 |
| XGBoost optimizado (RandomizedSearchCV) | 0.79 | 0.70 |

El árbol sin límite memorizó el train (overfitting). El XGBoost optimizado no mejoró en test y se separó más del train, así que me quedé con el XGBoost normal (300 árboles, learning rate 0.02, profundidad 6).

**Umbral.** El modelo por defecto decide "fuma" si la probabilidad es mayor a 0.5. Probé otros umbrales con validación cruzada sobre el train y el mejor fue 0.38. En test el F1 subió de 0.70 a 0.72: encuentra muchos más fumadores (recall de 0.72 a 0.88), a cambio de algunos falsos positivos más.

**Predicción.** Al archivo de entrega le apliqué exactamente los mismos pasos: corrección de horas, recorte, encoder, modelo y umbral. El modelo predijo que fuma el 53%. Casi ninguna mujer sale como fumadora y la mayoría de hombres sí.

## Conclusiones

- El género es por lejos la variable más importante para el modelo. Varias variables que al inicio parecían útiles estaban explicadas por el género.
- Los modelos de varios árboles (Random Forest y XGBoost) dieron mejores resultados que un solo árbol o KNN.
- Optimizar hiperparámetros no siempre mejora; en este caso aumentó el overfitting.
- Bajar el umbral a 0.38 fue lo que más ayudó al F1.
- Una limitación es que hay muy pocas mujeres fumadoras en los datos, así que al modelo le cuesta reconocerlas. Además, con el umbral bajo predice como fumadores a más hombres de los que realmente fuman.