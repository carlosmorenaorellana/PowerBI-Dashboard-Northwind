# Análisis de Datos de Negocio: Estudio de Caso Northwind Traders

## Executive Summary
Este proyecto implementa una solución integral de análisis de ventas y rendimiento de negocio para **Northwind Traders** utilizando **Power BI**. El objetivo principal es preparar, limpiar y modelar los datos transaccionales para proporcionar a la directiva una visión holística y accionable en las áreas de finanzas, operaciones y ventas.

---

## Tech Stack
* **Business Intelligence & Modelado:** Power BI (Power Query, DAX)
* **Fuentes de Datos:** Archivos CSV transaccionales
* **Control de Versiones:** Git & GitHub

---

## Project Structure

```
northwind_traders_analysis
├── data                    # Archivos CSV originales de origen
├── executive_summary       # (Ignorado mediante .gitignore) Informes ejecutivos y resúmenes
├── power_bi                # Modelos y paneles de Power BI
├── outputs                 # Activos y exportaciones de informes
└── README.md               # Documentación completa del proyecto
```
---

## 1. Calidad de Datos y Análisis de Valores Nulos

* **Encabezados de Columnas (customers):** La primera fila de los datos de origen contenía los nombres reales de las columnas. Se aplicó la transformación "Use First Row as Headers" en Power Query para corregir la estructura y garantizar la coherencia de los campos.

* **Valores Nulos en Empleados (reportsTo):** Se identificó un valor nulo en la columna `reportsTo` correspondiente al empleado con `employeeID 2` (Andrew Fuller, Vice President Sales). Tras analizar la estructura de la jerarquía y los cargos (`titles`), se concluyó que no existe un cargo superior al de Vicepresidente de Ventas, por lo que el valor nulo se mantuvo intacto de forma intencionada para denotar la cúspide de la estructura organizativa.

* **Valores Nulos en Envíos (orders.shippedDate):**
  * Se detectaron valores nulos en la columna `shippedDate` (~3% de los registros), interpretados a nivel operativo como pedidos no enviados (`non-shipped orders`) que pueden generar costes adicionales o pérdidas. Se mantuvieron sin alteraciones.
  * **Adición de Indicador (`nonShippedOrders`):** Se creó una columna condicional personalizada para aislar este comportamiento:
```
    = if [shippedDate] = null then 1 else 0```

    Esta métrica permite evaluar de forma directa el impacto de los pedidos no enviados en la operativa del negocio.

---

## 2. Enriquecimiento de Datos y Transformaciones

* **Formateo de Descuentos (`order_details.discount`):** Los valores de descuento se convirtieron de números enteros a porcentajes dividiendo la columna entre 100 y aplicando el tipo de dato correspondiente.

* **Estandarización de Precios (`unitPrice`):** Los campos de precios unitarios en las tablas `order_details` y `products` presentaban puntos como separadores decimales debido al formato de origen `.csv`. Se sustituyeron los puntos por comas y se transformaron a formato de número decimal fijo.

* **Tipado de Fechas y Numéricos:** Las columnas de fecha (`orderDate`, `requiredDate`, `shippedDate`) se tiparon correctamente para habilitar la inteligencia de tiempo en Power BI. Asimismo, los identificadores y claves foráneas se estandarizaron como números enteros.

* **Mapeo Geográfico por Regiones (`country_regions`):** Se creó una tabla auxiliar duplicando el campo `country` de los clientes y añadiendo una columna condicional para agrupar los países en tres grandes regiones estratégicas:
```
  = if [country] = "USA" or [country] = "Canada" or [country] = "Mexico" then "Americas"
  else if [country] = "Argentina" or [country] = "Brazil" or [country] = "Venezuela" then "LATAM"
  else "EMEA"
```

* **Gestión de Productos Descontinuados (`products.discontinued`):** Se conservaron los productos con estado discontinuado (1) por su relevancia histórica en los estados financieros e ingresos pasados. Adicionalmente, se creó una columna calculada `productStatus` para etiquetar intuitivamente los productos como "Discontinued" o "In Production".

---

## 3. Modelado de Datos y Relaciones

El modelo se estructuró bajo un esquema de estrella optimizado para el análisis multidimensional:

* **Relaciones Automáticas:** Validadas e integradas de uno a muchos (1:*) desde las tablas de dimensiones (`categories`, `customers`, `shippers`, `employees`) hacia la tabla de hechos principal (`orders`).

* **Relaciones Añadidas:** Se configuraron manualmente las relaciones clave entre `products` y `order_details`, así como entre `orders` y `order_details` a través de `productID` y `orderID` respectivamente. También se vinculó la tabla de mapeo `country_regions` con `customers`.

* **Tabla de Fechas Dinámica (`date_table`):** Se generó mediante DAX utilizando `CALENDARAUTO()` para garantizar una cobertura temporal completa y dinámica. Se marcaron formalmente las propiedades de tabla de fechas y se desglosaron columnas de año, trimestre, número/nombre de mes y día de la semana. Las jerarquías temporales se ordenaron numéricamente para evitar fallos de orden alfabético en las visualizaciones.

* **Relaciones Temporales Activas e Inactivas:** La relación principal con `orders` se estableció mediante la fecha de pedido (`orderDate`), mientras que las fechas de requerimiento y envío se conectaron mediante relaciones inactivas para permitir análisis avanzados mediante DAX (`USERRELATIONSHIP`).

* **Relaciones Recursivas:** Se duplicó la tabla de empleados bajo el nombre `team_leader` para filtrar jerárquicamente a los equipos de ventas en función de sus responsables directos.

---

## 4. Análisis de Datos y Medidas (DAX)

Se estructuraron columnas calculadas y medidas optimizadas para garantizar un rendimiento fluido del modelo:

* **`revenues` (Columna Calculada):** Calcula los ingresos netos por línea de detalle aplicando la cantidad, el precio unitario y el descuento:
```
  revenues = (order_details[unitPrice] * order_details[quantity]) * (1 - order_details[discount])
```

* **`region` (Columna Calculada):** Enriquecimiento de clientes mediante la función `LOOKUPVALUE` contra la tabla auxiliar de regiones geográficas.

* **`operatingTime` (Columna Calculada):** Mide el tiempo de preparación de los pedidos en días utilizando `DATEDIFF` entre la fecha de pedido y la de envío.

* **Medidas Financieras Principales:**
  * **Total Revenues:** Suma optimizada de los ingresos netos.
  * **Profit y Profit Margin:** Cálculo dinámico de la rentabilidad considerando los costos de flete (`freight`):
```
    Profit = SUMX('order_details', 'order_details'[revenues]) - RELATED('orders'[freight])
```
```
    Profit Margin = [Profit] / [Total Revenues]
```
  * **Operating Time KPI:** Promedio del tiempo de procesamiento operativo.
  * **Total Freight on Non-Shipped Orders:** Cuantificación del impacto financiero de los envíos no realizados mediante `CALCULATE` y `ISBLANK`.
  * Promedios de precios y cantidades mediante `AVERAGEX` para evaluar el comportamiento de compra.

* **Organización del Modelo:** Todas las medidas se centralizaron en una tabla auxiliar de gestión denominada `kpi`.

---

## 5. Paneles de Informe y Visualización

El informe ejecutivo se divide en tres paneles estratégicos diseñados para cubrir las áreas clave de finanzas, operaciones y ventas:

### 5.1 Panel "Executive Dashboard"
* **Propósito:** Proporcionar a la alta dirección una visión ejecutiva del rendimiento financiero y las fuentes de ingresos clave.
* **Hallazgos Clave:**
  * El margen de beneficio se mantiene sólidamente por encima del 84% en todo el periodo.
  * La región EMEA lidera la aportación a los ingresos totales, destacando las categorías de productos Dairy Products y Beverages.
  * La categoría Beverages es la de mayor facturación global, mientras que Meat & Poultry registra el margen de beneficio más elevado (89%).

### 5.2 Panel "Operating Efficiency"
* **Propósito:** Evaluar la eficiencia de la cadena de suministro, los tiempos de procesamiento y los costos logísticos para gerentes de operaciones.
* **Hallazgos Clave:**
  * El indicador de tiempo operativo (Operating Time KPI) se sitúa en una media de 8 días, experimentando una reducción destacada a 6,4 días en el último trimestre.
  * Los pedidos no enviados (Non-Shipped Orders) se concentran temporalmente a partir del segundo trimestre de 2015, vinculándose principalmente a la operativa de la empresa transportista United Package.
  * El equipo liderado por Laura Callahan destaca por un alto volumen de procesamiento y una alta eficiencia operativa temporal.

### 5.3 Panel "Products Dashboard"
* **Propósito:** Ofrecer a los equipos de ventas y marketing un análisis detallado del rendimiento de productos y el comportamiento del cliente.
* **Hallazgos Clave:**
  * El producto "Côte de Blaye" representa un hito fundamental, acaparando el 53% de los ingresos de su categoría (Beverages), más del 11% de los ingresos totales de la compañía y un margen de beneficio del 95%.
  * Se identificó que ciertos productos descontinuados de la categoría Meat & Poultry (como Thüringer Rostbratwurst) poseían alta rentabilidad y márgenes superiores al 90%, sugiriendo que su retirada pudo mermar el crecimiento potencial de los ingresos.