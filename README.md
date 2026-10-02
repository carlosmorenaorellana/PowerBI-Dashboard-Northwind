\# Análisis de Datos de Negocio: Estudio de Caso Northwind Traders



\## Executive Summary

Este proyecto implementa una solución integral de análisis de ventas y rendimiento de negocio para \*\*Northwind Traders\*\* utilizando \*\*Power BI\*\*. El objetivo principal es preparar, limpiar y modelar los datos transaccionales para proporcionar a la directiva una visión holística y accionable en las áreas de finanzas, operaciones y ventas.



\---



\## Tech Stack

\* \*\*Business Intelligence \& Modelado:\*\* Power BI (Power Query, DAX)

\* \*\*Fuentes de Datos:\*\* Archivos CSV transaccionales

\* \*\*Control de Versiones:\*\* Git \& GitHub



\---



\## Project Structure



```

northwind\_traders\_analysis

├── data                    # Archivos CSV originales de origen

├── executive\_summary       # (Ignorado mediante .gitignore) Informes ejecutivos y resúmenes

├── power\_bi                # Modelos y paneles de Power BI

├── outputs                 # Activos y exportaciones de informes

└── README.md               # Documentación completa del proyecto

```



1\. Calidad de Datos y Análisis de Valores Nulos

Encabezados de Columnas (customers): La primera fila de los datos de origen contenía los nombres reales de las columnas. Se aplicó la transformación "Use First Row as Headers" en Power Query para corregir la estructura y garantizar la coherencia de los campos.



Valores Nulos en Empleados (reportsTo): Se identificó un valor nulo en la columna reportsTo correspondiente al empleado con employeeID 2 (Andrew Fuller, Vice President Sales). Tras analizar la estructura de la jerarquía y los cargos (titles), se concluyó que no existe un cargo superior al de Vicepresidente de Ventas, por lo que el valor nulo se mantuvo intacto de forma intencionada para denotar la cúspide de la estructura organizativa.



Valores Nulos en Envíos (orders.shippedDate):



Se detectaron valores nulos en la columna shippedDate (\~3% de los registros), interpretados a nivel operativo como pedidos no enviados (non-shipped orders) que pueden generar costes adicionales o pérdidas. Se mantuvieron sin alteraciones.



Adición de Indicador (nonShippedOrders): Se creó una columna condicional personalizada para aislar este comportamiento:



Fragmento de código

= if \[shippedDate] = null then 1 else 0

Esta métrica permite evaluar de forma directa el impacto de los pedidos no enviados en la operativa del negocio.

