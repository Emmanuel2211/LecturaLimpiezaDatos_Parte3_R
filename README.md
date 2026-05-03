# Ejercicio: Lectura y Limpieza de Datos de Café — R

**Archivo principal:** `calidad_cafe.Rmd`  
**Dataset:** `coffee_ratings.csv` (1,339 filas × 43 columnas)  
**Lenguaje:** R con R Markdown  

## Descripción del proyecto

Este ejercicio forma parte del Módulo 2 del Diplomado de Ciencia de Datos. El objetivo es practicar la lectura, exploración, limpieza y transformación de datos usando el ecosistema `tidyverse`, así como la creación de visualizaciones de calidad de publicación con `ggplot2`. El dataset contiene evaluaciones de calidad de café de múltiples países, con variables que van desde características del grano hasta puntajes sensoriales.

---

## Configuración del documento R Markdown

```r
knitr::opts_chunk$set(
  echo    = TRUE,    # Muestra el código fuente en el documento
  message = FALSE,   # Oculta mensajes de las funciones (ej. de readr al leer CSV)
  warning = FALSE,   # Oculta advertencias (warnings)
  fig.width  = 9,    # Ancho por defecto de las figuras (en pulgadas)
  fig.height = 6,    # Alto por defecto de las figuras (en pulgadas)
  dpi = 150          # Resolución de las figuras (puntos por pulgada)
)
```
- **`knitr::opts_chunk$set()`**: establece opciones globales para todos los chunks del documento. Solo se aplica a chunks que no las sobreescriban individualmente.

---

## Librerías utilizadas

```r
library(tidyverse)
library(lubridate)
```

- **`tidyverse`**: meta-paquete que carga simultáneamente: `dplyr` (manipulación), `ggplot2` (gráficos), `stringr` (texto), `tidyr` (restructuración), `purrr` (programación funcional), `readr` (lectura de archivos) y `forcats` (factores).
- **`lubridate`**: facilita el manejo de fechas y tiempos con funciones como `ymd()`, `dmy()`, `mdy()`.

---

## Carga de datos

```r
datos <- readr::read_csv("coffee_ratings.csv")
```
- **`readr::read_csv(ruta)`**: lee un CSV y retorna un `tibble` (versión moderna del `data.frame`). Detecta automáticamente los tipos de columna e imprime un mensaje con los tipos inferidos (silenciado por `message=FALSE`). A diferencia de `read.csv()` base, no convierte cadenas a factores y es más rápido.

```r
cat(sprintf("Dimensiones del dataset: %d filas × %d columnas\n", nrow(datos), ncol(datos)))
```
- **`nrow(datos)`**: número de filas; **`ncol(datos)`**: número de columnas.
- **`sprintf(formato, ...)`**: formatea una cadena de texto. `%d` es el especificador para enteros.
- **`cat()`**: imprime texto en la consola sin comillas ni índices (a diferencia de `print()`).

---

## Actividad 0: Análisis de valores faltantes

```r
na_counts <- datos |> is.na() |> colSums()
```
- **`datos |> is.na()`**: el operador pipe nativo de R (`|>`, disponible desde R 4.1) pasa `datos` como primer argumento de `is.na()`. Retorna una matriz lógica del mismo tamaño que `datos`, con `TRUE` donde hay `NA`.
- **`|> colSums()`**: suma cada columna de la matriz lógica. Como `TRUE=1` y `FALSE=0`, el resultado es el conteo de `NA` por columna. El resultado es un vector numérico nombrado.

```r
resumen_na <- tibble(
  columna           = na_counts |> names(),
  valores_faltantes = na_counts |> as.integer(),
  porcentaje        = (na_counts / nrow(datos) * 100) |> round(2)
) |>
```
- **`tibble(...)`**: crea un `tibble` (equivalente moderno del `data.frame`). Cada argumento nombrado define una columna.
- **`na_counts |> names()`**: extrae los nombres del vector `na_counts` (que son los nombres de las columnas del dataset original).
- **`na_counts |> as.integer()`**: convierte los valores de tipo `numeric` a `integer`.
- **`(na_counts / nrow(datos) * 100) |> round(2)`**: calcula el porcentaje y lo redondea a 2 decimales. Los paréntesis son necesarios para que `|>` tome la expresión completa.

```r
  filter(valores_faltantes > 0) |>
```
- **`dplyr::filter(condicion)`**: conserva solo las filas donde la condición es `TRUE`. Aquí, elimina las columnas sin ningún valor faltante.

```r
  arrange(desc(valores_faltantes))
```
- **`dplyr::arrange()`**: ordena el tibble por la columna indicada. **`desc()`** invierte el orden (mayor a menor).

```r
resumen_na |> print(row.names = FALSE)
```
- **`print(row.names = FALSE)`**: imprime el tibble sin mostrar los números de fila.

---

## Actividad 1: Crear la columna `color2`

```r
datos <- datos |>
  mutate(
    color2 = case_when(
      is.na(color)            ~ NA_character_,
      color == "Green"        ~ "#00FF66",
      color == "Bluish-Green" ~ "#CCEBC5",
      color == "Blue-Green"   ~ "#BFFFFF",
      TRUE                    ~ NA_character_
    )
  )
```
- **`dplyr::mutate(...)`**: añade nuevas columnas o modifica las existentes. Las expresiones se evalúan vectorizadamente sobre todas las filas. El resultado se asigna de vuelta a `datos`.
- **`dplyr::case_when(...)`**: evaluación condicional vectorizada. Cada línea tiene la forma `condición ~ valor`. Se evalúan en orden y se usa el valor de la **primera** condición verdadera.
  - `is.na(color) ~ NA_character_`: si `color` es `NA`, asignar `NA` de tipo character. Debe ir primero para que no se evalúe `color == "Green"` sobre `NA` (lo que retornaría `NA`, no `FALSE`).
  - `TRUE ~ NA_character_`: cláusula por defecto (como `else`). Cualquier valor no reconocido recibe `NA`.

```r
cat("Distribución de color vs color2:\n")
print(table(datos$color, datos$color2, useNA = "always"))
```
- **`datos$color`**: accede a la columna `color` usando el operador `$`. Equivalente a `datos[["color"]]`.
- **`table(x, y, useNA)`**: tabla de contingencia (frecuencias cruzadas). `useNA = "always"` incluye los `NA` como una categoría en las dimensiones de la tabla.

---

## Actividad 2: Crear la columna `bag_weight2`

```r
ambiguos_bw <- datos |>
  filter(str_detect(bag_weight, ","))
```
- **`stringr::str_detect(string, pattern)`**: retorna un vector lógico `TRUE`/`FALSE` indicando si cada elemento de `string` contiene el patrón. Aquí detecta la presencia de una coma en `bag_weight`.
- El resultado `ambiguos_bw` es un tibble con solo las filas ambiguas.

```r
datos <- datos |>
  mutate(
    bag_weight2 = if_else(
      str_detect(bag_weight, ","),
      NA_real_,
      bag_weight |> str_extract("\\d+(\\.\\d+)?") |> as.numeric()
    )
  )
```
- **`dplyr::if_else(condicion, verdadero, falso)`**: versión estricta de `ifelse()` que exige que `verdadero` y `falso` tengan el mismo tipo. `NA_real_` es el `NA` de tipo `double`.
- **`bag_weight |> str_extract("\\d+(\\.\\d+)?")`**: para cada elemento de la columna `bag_weight`, extrae la primera subcadena que coincide con el patrón.
  - **`stringr::str_extract(string, pattern)`**: extrae la **primera** coincidencia del patrón regex. Retorna `NA` si no hay coincidencia.
  - **Patrón `\\d+(\\.\\d+)?`**: `\\d+` = uno o más dígitos; `(\\.\\d+)?` = opcionalmente un punto y más dígitos (para decimales). En R, `\\` representa un backslash literal en regex.
- **`|> as.numeric()`**: convierte la cadena extraída a tipo numérico.

```r
vals_ambiguos_bw <- ambiguos_bw$bag_weight |> unique()
```
- **`unique()`**: retorna los valores únicos de un vector, eliminando duplicados.

---

## Actividad 3: Crear `method1` y `method2`

```r
pm_match <- str_match(datos$processing_method, "^(.+?)\\s*/\\s*(.+)$")
```
- **`stringr::str_match(string, pattern)`**: aplica el regex y retorna una **matriz** donde:
  - Columna 1 (`[, 1]`): la coincidencia completa.
  - Columna 2 (`[, 2]`): el primer grupo de captura `(...)`.
  - Columna 3 (`[, 3]`): el segundo grupo de captura.
  - Las filas sin coincidencia quedan como `NA` en todas las columnas.
- **Patrón `^(.+?)\\s*/\\s*(.+)$`**:
  - `^` = inicio de cadena.
  - `(.+?)` = grupo 1, captura uno o más caracteres de forma **no-greedy** (lo mínimo posible).
  - `\\s*/\\s*` = barra `/` con cero o más espacios a cada lado.
  - `(.+)$` = grupo 2, captura todo lo que sigue hasta el final de cadena.

```r
datos <- datos |>
  mutate(
    method1 = pm_match[, 2] |> str_trim(),
    method2 = pm_match[, 3] |> str_trim()
  )
```
- **`pm_match[, 2]`**: extrae la segunda columna de la matriz (grupo de captura 1 = texto antes de `/`).
- **`stringr::str_trim()`**: elimina espacios en blanco al inicio y al final de cada cadena. Sin argumentos, limpia ambos lados (equivalente a `side = "both"`).

```r
n_ambiguos_pm  <- pm_match[, 1] |> is.na() |> sum()
```
- `pm_match[, 1]` contiene la coincidencia completa; es `NA` donde el regex no coincidió.
- **`is.na()`**: retorna un vector lógico.
- **`sum()`**: suma los `TRUE` (cada `TRUE` vale 1).

```r
vals_ambiguos_pm <- datos$processing_method[pm_match[, 1] |> is.na()] |> unique()
```
- **`vector[mascara_logica]`**: indexación lógica en R, selecciona los elementos donde la máscara es `TRUE`.
- La expresión obtiene los valores únicos de `processing_method` en las filas sin coincidencia.

---

## Actividad 4: Crear `expiration_day`, `expiration_month`, `expiration_year`

```r
exp_match <- str_match(
  datos$expiration,
  "^(\\w+)\\s+(\\d{1,2})(?:st|nd|rd|th),\\s+(\\d{4})$"
)
```
- **Patrón con 3 grupos de captura**:
  - `(\\w+)` — grupo 1: nombre del mes. `\\w` = carácter alfanumérico o guión bajo.
  - `\\s+` — uno o más espacios.
  - `(\\d{1,2})` — grupo 2: día de 1 o 2 dígitos.
  - `(?:st|nd|rd|th)` — sufijo ordinal. `(?:...)` es un **grupo no-capturante** (no genera columna en la matriz).
  - `,\\s+` — coma seguida de espacios.
  - `(\\d{4})$` — grupo 3: año de exactamente 4 dígitos al final.

```r
datos <- datos |>
  mutate(
    expiration_month = exp_match[, 2],
    expiration_day   = exp_match[, 3] |> as.integer(),
    expiration_year  = exp_match[, 4] |> as.integer()
  )
```
- Las columnas 2, 3 y 4 de `exp_match` corresponden a los tres grupos de captura.
- **`as.integer()`**: convierte la cadena del día/año a entero. Los `NA` (regex no coincidió) se preservan como `NA_integer_`.

```r
n_ambiguos_exp <- (!is.na(datos$expiration) & is.na(exp_match[, 2])) |> sum()
```
- Condición compuesta: valor original presente (`!is.na`) Y mes extraído ausente (`is.na`). Identifica filas con formato inesperado.
- **`&`**: AND lógico vectorizado en R.

```r
datos |>
  select(expiration, expiration_day, expiration_month, expiration_year) |>
  head(8) |>
  print()
```
- **`dplyr::select(col1, col2, ...)`**: selecciona columnas por nombre.
- **`head(n)`**: retorna las primeras `n` filas.
- **`print()`**: imprime el tibble.

---

## Actividad 5: Crear `harvest_mes` y `harvest_anio`

```r
datos <- datos |>
  mutate(
    partes_hy    = str_split(harvest_year, "\\s*/\\s*|\\s*-\\s*(?=[0-9])", n = 2),
```
- **`stringr::str_split(string, pattern, n)`**: divide cada cadena usando el patrón como separador, en máximo `n` partes. El resultado es una **lista** (list-column en el tibble), donde cada elemento es un vector de character.
- **Patrón `\\s*/\\s*|\\s*-\\s*(?=[0-9])`**:
  - `\\s*/\\s*` = barra con espacios opcionales.
  - `|` = alternancia (OR).
  - `\\s*-\\s*(?=[0-9])` = guión con espacios opcionales, seguido de un dígito (lookahead positivo `(?=...)` verifica sin consumir el carácter).

```r
    harvest_mes  = map_chr(partes_hy, ~ str_trim(.x[1])),
```
- **`purrr::map_chr(lista, funcion)`**: aplica la función a cada elemento de la lista y retorna un **vector de character**. Es como `sapply` pero con tipo garantizado.
- **`~ str_trim(.x[1])`**: fórmula lambda de purrr. El `~` inicia la fórmula; `.x` representa el elemento actual de la lista. `.x[1]` accede al primer elemento del vector (la primera parte del split).

```r
    harvest_anio = map_chr(partes_hy, ~ if (length(.x) >= 2) str_trim(.x[2]) else NA_character_)
```
- Si el vector tiene 2 o más elementos (`length(.x) >= 2`), retorna el segundo elemento (la segunda parte). Si solo tiene 1 (sin separador) o es `NA`, retorna `NA_character_`.

```r
  ) |>
  select(-partes_hy)
```
- **`select(-columna)`**: el signo `-` en `select()` **elimina** la columna indicada. Se usa para limpiar la columna auxiliar `partes_hy` que ya no se necesita.

```r
n_ambiguos_hy   <- datos$harvest_anio |> is.na() |> sum()
n_na_originales <- datos$harvest_year |> is.na() |> sum()
n_sin_separador <- (is.na(datos$harvest_anio) & !is.na(datos$harvest_year)) |> sum()
```
- Tres conteos:
  1. Total de ambiguos (incluyendo originalmente NA).
  2. Solo los NA originales en `harvest_year`.
  3. Valores presentes en `harvest_year` pero que no pudieron dividirse (sin separador).

```r
datos$harvest_year[is.na(datos$harvest_anio) & !is.na(datos$harvest_year)] |>
  unique() |>
  sort() |>
  print()
```
- **`sort()`**: ordena un vector en orden creciente (alfabético para character).
- La cadena de pipes extrae, deduplica, ordena e imprime los valores problemáticos de `harvest_year`.

---

## Actividad 6: Scatter `total_cup_points` vs `acidity` por `color2`

```r
df6 <- datos |> drop_na(color2, total_cup_points, acidity)
```
- **`tidyr::drop_na(col1, col2, ...)`**: elimina filas con `NA` en cualquiera de las columnas indicadas. Sin argumentos, eliminaría filas con `NA` en cualquier columna.

```r
ggplot(df6, aes(x = acidity, y = total_cup_points, color = color2)) +
```
- **`ggplot2::ggplot(data, aes(...))`**: inicializa el gráfico. `data` es el DataFrame fuente. `aes()` define los **mapeos estéticos** globales: qué columna va en cada eje y qué define el color.

```r
  geom_point(alpha = 0.75, size = 2.5, shape = 16) +
```
- **`geom_point()`**: capa de puntos (diagrama de dispersión).
  - `alpha`: transparencia (0 = invisible, 1 = opaco).
  - `size`: tamaño de los puntos.
  - `shape = 16`: círculo relleno (sin borde). Los shapes de R van del 0 al 25.

```r
  scale_color_identity(
    guide  = "legend",
    breaks = c("#00FF66", "#CCEBC5", "#BFFFFF"),
    labels = c("Green", "Bluish-Green", "Blue-Green")
  ) +
```
- **`scale_color_identity()`**: le indica a ggplot2 que los valores de la estética `color` son **literales** (códigos hex directos), no niveles de una variable a mapear a una paleta. Sin esta escala, ggplot intentaría mapear las cadenas hex a colores de una paleta por defecto.
  - `guide = "legend"`: activa la leyenda (por defecto está desactivada con `identity`).
  - `breaks`: qué valores de `color2` mostrar en la leyenda.
  - `labels`: etiquetas legibles que corresponden a cada `break`.

```r
  theme_bw(base_size = 13) +
```
- **`theme_bw()`**: tema de fondo blanco con bordes negros en los paneles. `base_size` establece el tamaño de fuente base del que derivan todos los demás textos del gráfico.

```r
  theme(
    plot.title      = element_text(face = "bold", size = 14),
    legend.position = "right"
  )
```
- **`theme(...)`**: personaliza elementos del tema. Llama a funciones `element_*()`:
  - **`element_text(face, size)`**: define la apariencia del texto. `face = "bold"` pone el título en negrita.
  - `legend.position = "right"`: posiciona la leyenda a la derecha del gráfico.

---

## Actividad 7: Densidad de `bag_weight2` por `species`

```r
df7 <- datos |>
  drop_na(bag_weight2, species) |>
  filter(bag_weight2 < 2000)
```
- Encadena dos pasos: primero elimina NA, luego filtra outliers extremos (> 2000 kg).

```r
ggplot(df7, aes(x = bag_weight2, fill = species, color = species)) +
```
- Mapea `species` tanto a `fill` (relleno del área) como a `color` (línea de contorno). Esto es necesario para que ambas escalas (`scale_fill_brewer` y `scale_color_brewer`) se unifiquen en una sola leyenda.

```r
  geom_density(alpha = 0.35, linewidth = 1.2) +
```
- **`geom_density()`**: calcula y dibuja la estimación de densidad por kernel (KDE) para la variable `x`.
  - `alpha`: transparencia del relleno.
  - `linewidth`: grosor de la línea de contorno de la curva.

```r
  scale_fill_brewer(palette = "Set1") +
  scale_color_brewer(palette = "Set1") +
```
- **`scale_fill_brewer()` / `scale_color_brewer()`**: usan paletas de [ColorBrewer](https://colorbrewer2.org/), diseñadas para ser perceptualmente distinguibles. `"Set1"` es una paleta cualitativa con colores saturados. Ambas escalas deben especificarse para que la leyenda sea única y los colores coincidan.

---

## Actividad 8: Fecha de expiración vs `total_cup_points` (4 países)

```r
paises <- c("Mexico", "Brazil", "Colombia", "Guatemala")
```
- **`c(...)`**: función de concatenación que crea un vector. Los nombres están en inglés tal como aparecen en el dataset.

```r
month_levels <- c("January","February","March","April","May","June",
                  "July","August","September","October","November","December")
month_num <- setNames(1:12, month_levels)
```
- **`1:12`**: genera la secuencia entera del 1 al 12.
- **`setNames(objeto, nombres)`**: asigna nombres a los elementos de un vector. El resultado es un vector nombrado donde `month_num["January"]` retorna `1`, `month_num["February"]` retorna `2`, etc. Se usa como diccionario para convertir nombres de mes a número.

```r
df8 <- datos |>
  filter(country_of_origin %in% paises) |>
```
- **`%in%`**: operador de pertenencia. `x %in% y` retorna `TRUE` para cada elemento de `x` que esté en el vector `y`. Equivale a `x == y[1] | x == y[2] | ...`.

```r
  mutate(
    mes_num   = month_num[expiration_month],
    fecha_exp = paste(expiration_year, mes_num, "01", sep = "-") |> ymd()
  ) |>
```
- **`month_num[expiration_month]`**: indexación con un vector de caracteres. R busca en el vector nombrado `month_num` los valores correspondientes a cada nombre del mes en `expiration_month`.
- **`paste(a, b, c, sep = "-")`**: concatena los vectores elemento a elemento, separando con `"-"`. Produce cadenas como `"2016-4-01"`.
- **`lubridate::ymd(cadena)`**: convierte cadenas en formato "Año-Mes-Día" a objetos `Date`. Más robusto que `as.Date()` para este formato.

```r
  group_by(fecha_exp, country_of_origin) |>
  summarise(total_cup_points = mean(total_cup_points, na.rm = TRUE), .groups = "drop") |>
```
- **`dplyr::group_by(col1, col2)`**: marca el tibble para operaciones por grupo. No modifica los datos; solo agrega metadatos de agrupación.
- **`dplyr::summarise(...)`**: colapsa cada grupo en una fila, calculando las estadísticas indicadas.
  - `mean(..., na.rm = TRUE)`: media aritmética ignorando `NA`.
  - `.groups = "drop"`: elimina la agrupación del resultado (equivalente a hacer `ungroup()` al final).

```r
ggplot(df8, aes(x = fecha_exp, y = total_cup_points,
                color = country_of_origin, group = country_of_origin)) +
```
- `group = country_of_origin`: necesario para que `geom_line()` conecte los puntos del mismo país. Sin esto, si los puntos de distintos países están intercalados temporalmente, las líneas se conectarían incorrectamente.

```r
  scale_color_manual(values = palette_paises) +
```
- **`scale_color_manual(values = vector_nombrado)`**: asigna colores específicos a cada categoría. El vector debe tener nombres que coincidan con los niveles de la variable `color`.

```r
  scale_x_date(date_labels = "%b %Y", date_breaks = "3 months") +
```
- **`scale_x_date()`**: escala específica para ejes de tipo `Date`.
  - `date_labels = "%b %Y"`: formato de etiquetas. `%b` = nombre del mes abreviado en el locale del sistema; `%Y` = año de 4 dígitos.
  - `date_breaks = "3 months"`: intervalo entre marcas del eje.

```r
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```
- `axis.text.x`: estética del texto de las marcas del eje X.
- `angle = 45`: rota las etiquetas 45°.
- `hjust = 1`: alineación horizontal = 1 (derecha), necesaria junto con la rotación para que las etiquetas queden debajo de las marcas.

---

## Actividad 9: Mes de expiración vs altitudes (4 países, 2016–2017)

```r
df9 <- datos |>
  filter(
    country_of_origin %in% paises,
    expiration_year %in% c(2016L, 2017L)
  ) |>
```
- `c(2016L, 2017L)`: la `L` al final indica que son literales de tipo `integer` en R, lo que asegura la comparación correcta con la columna `expiration_year` de tipo entero.

```r
  mutate(
    expiration_month = factor(expiration_month, levels = month_levels)
  ) |>
```
- **`factor(x, levels)`**: convierte el vector `x` en un factor. `levels` define el orden de las categorías. Esto es crucial para que ggplot2 ordene el eje X de enero a diciembre en lugar de hacerlo alfabéticamente.

```r
  pivot_longer(
    cols      = c(altitude_low_meters, altitude_mean_meters, altitude_high_meters),
    names_to  = "tipo_altitud",
    values_to = "altitud_metros"
  ) |>
```
- **`tidyr::pivot_longer(cols, names_to, values_to)`**: transforma el tibble de formato **ancho** a **largo**.
  - `cols`: columnas que se van a "derretir" (convertir de columnas a filas).
  - `names_to`: nombre de la nueva columna que contendrá los nombres de las columnas originales.
  - `values_to`: nombre de la nueva columna que contendrá los valores.
  - Cada fila original se convierte en 3 filas (una por cada variable de altitud).

```r
  mutate(
    tipo_altitud = case_when(
      tipo_altitud == "altitude_low_meters"  ~ "Altitud Mínima",
      tipo_altitud == "altitude_mean_meters" ~ "Altitud Media",
      tipo_altitud == "altitude_high_meters" ~ "Altitud Máxima"
    )
  )
```
- Renombra los valores de `tipo_altitud` al español para que aparezcan así en la leyenda del gráfico.

```r
ggplot(df9, aes(x = expiration_month, y = altitud_metros,
                color = tipo_altitud, group = tipo_altitud)) +
  stat_summary(fun = mean, geom = "line",  linewidth = 1.2) +
  stat_summary(fun = mean, geom = "point", size = 2.5) +
```
- **`stat_summary(fun, geom)`**: calcula una estadística (`fun`) por cada valor del eje X y la grafica con el `geom` indicado. Aquí calcula la media de `altitud_metros` por mes y la dibuja como línea y como punto.
  - `fun = mean`: función a aplicar a los valores Y por grupo X.
  - `geom = "line"` / `"point"`: geometría con la que se dibuja el resultado.
  - Ventaja sobre `geom_line()` + `geom_point()`: no requiere agregar previamente, maneja múltiples observaciones por punto X automáticamente.

```r
  facet_wrap(~ expiration_year, nrow = 1) +
```
- **`facet_wrap(~ variable, nrow)`**: divide el gráfico en subpaneles (facetas), uno por cada valor único de `variable`. `nrow = 1` coloca todos los paneles en una sola fila.
- La sintaxis `~ variable` es una fórmula R; el `~` separa el lado izquierdo (vacío aquí) del derecho (la variable de facetado).

```r
  theme(strip.text = element_text(face = "bold", size = 12))
```
- `strip.text`: texto del encabezado de las facetas (en este caso "2016" y "2017").

---

## Actividad 10: Violin + Boxplot de `aftertaste`, `acidity`, `body` por `species`

```r
df10 <- datos |>
  select(aftertaste, acidity, body, species) |>
  drop_na() |>
  pivot_longer(
    cols      = c(aftertaste, acidity, body),
    names_to  = "variable",
    values_to = "valor"
  ) |>
  mutate(
    variable = case_when(
      variable == "aftertaste" ~ "Aftertaste",
      variable == "acidity"    ~ "Acidez",
      variable == "body"       ~ "Cuerpo"
    )
  )
```
- `select()` conserva solo las 4 columnas necesarias.
- `drop_na()` sin argumentos elimina filas con `NA` en **cualquier** columna del tibble actual (las 4 seleccionadas).
- `pivot_longer()` convierte las 3 métricas en una columna `variable` y sus valores en `valor`, produciendo 3 filas por observación original.
- `case_when()` traduce los nombres de columna originales al español para las etiquetas del gráfico.

```r
ggplot(df10, aes(x = species, y = valor, fill = species)) +
  geom_violin(alpha = 0.45, trim = FALSE, linewidth = 0.6) +
```
- **`geom_violin()`**: dibuja un diagrama de violín — una estimación de densidad espejada verticalmente que muestra la distribución completa de los datos.
  - `alpha`: transparencia del relleno.
  - `trim = FALSE`: extiende el violín hasta los valores mínimo y máximo reales (en lugar de truncar en los cuartiles extremos).
  - `linewidth`: grosor del contorno del violín.

```r
  geom_boxplot(width = 0.15, alpha = 0.85, outlier.size = 0.8,
               outlier.alpha = 0.5, linewidth = 0.5) +
```
- **`geom_boxplot()`**: superpone un boxplot sobre el violín. Muestra mediana (línea central), IQR (caja), bigotes (1.5×IQR) y puntos atípicos.
  - `width = 0.15`: ancho estrecho para que no tape el violín.
  - `outlier.size` / `outlier.alpha`: tamaño y transparencia de los puntos atípicos.

```r
  facet_wrap(~ variable, scales = "free_y", nrow = 1) +
```
- `scales = "free_y"`: permite que cada faceta tenga su propio rango en el eje Y. Útil cuando las variables tienen escalas diferentes (aunque aquí son similares, permite ajuste óptimo por panel).

```r
  theme(
    strip.text      = element_text(face = "bold", size = 12),
    axis.text.x     = element_text(angle = 15, hjust = 1),
    legend.position = "none"
  )
```
- `legend.position = "none"`: oculta la leyenda. Como el eje X ya muestra las especies directamente, la leyenda sería redundante.
- `axis.text.x = element_text(angle = 15, hjust = 1)`: rota ligeramente las etiquetas del eje X para mejorar la legibilidad.
