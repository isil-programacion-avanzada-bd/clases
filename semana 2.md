---
title: "Guía del Estudiante - S02 Tipos de consultas (Parte 1)"
lang: es
geometry: margin=22mm
header-includes:
  - \usepackage{fancyhdr}
  - \usepackage{longtable}
  - \usepackage{array}
  - \pagestyle{fancy}
  - \fancyhf{}
  - \fancyhead[L]{Lideratec Academy}
  - \fancyhead[R]{ISIL-PABD-30627 - S02}
  - \fancyfoot[C]{\thepage}
  - \setlength{\headheight}{15pt}
---

# Guía del Estudiante

**Curso:** Programación Avanzada de Base de Datos  
**Sesión:** S02 - Tipos de consultas (Parte 1)  
**Laboratorio asociado:** `FASE_15`  
**Base autocontenida:** `Northwind_S02_Lab`

## 1. Propósito de la sesión

En esta sesión aprenderás a transformar una necesidad de información en una consulta T-SQL verificable. Trabajarás con selección de columnas, filtros, rangos, conjuntos, patrones de texto, agregaciones, agrupaciones, filtros de grupos y funciones integradas. El objetivo no es memorizar sentencias aisladas: debes poder anticipar qué filas o valores debería devolver una consulta, ejecutarla, interpretar el resultado y corregirla cuando la salida no coincide con lo esperado.

El laboratorio usa un conjunto reducido con estructura inspirada en Northwind para que puedas trabajar sin depender de rutas físicas de archivos `.mdf/.ldf`. Si tu institución ya tiene la Northwind completa, puedes reutilizarla; las consultas se mantienen dentro de los conceptos de la sesión.

## 2. Resultado observable

### A. Lectura sugerida del docente

Al finalizar la práctica deberás ser capaz de abrir una consulta en SQL Server Management Studio, seleccionar la base correcta y construir sentencias que respondan preguntas concretas sobre los datos. No bastará con que una consulta “corra”: tendrás que explicar qué conjunto de filas estás consultando, qué condición elimina o conserva registros, qué cambia cuando modificas un límite y por qué una agregación devuelve una fila o varias. También aprenderás a diferenciar el filtrado de filas con `WHERE` del filtrado de grupos con `HAVING`, y a usar funciones integradas para transformar texto, números y fechas sin alterar los datos almacenados. La evidencia de dominio será observable en tus predicciones, en las consultas ejecutadas, en la interpretación de resultados y en las tareas espejo que resolverás después de cada ejemplo. Una respuesta correcta que no puedes explicar todavía no representa dominio completo; una consulta que puedes justificar, modificar y validar sí.

### B. Desempeños observables

- construir consultas `SELECT` con columnas, alias y expresiones;
- aplicar filtros básicos y especializados;
- interpretar rangos y patrones de texto;
- resumir datos mediante agregaciones;
- agrupar y filtrar grupos;
- aplicar funciones de cadena, numéricas y fecha;
- diagnosticar errores frecuentes de sintaxis o lógica;
- validar una consulta comparando la salida con una predicción.

### C. Criterio de dominio

Demuestras dominio cuando puedes resolver T01-T08 sin copiar literalmente los ejemplos, explicar por qué cada consulta devuelve ese conjunto y corregir al menos un error controlado identificando su causa.

## 3. Antes de iniciar

### Debes saber

- qué representa una tabla, fila y columna;
- cómo conectarte a una instancia de SQL Server disponible para el laboratorio;
- que las consultas de esta sesión son de lectura: no necesitas modificar datos.

### Debes tener disponible

- SQL Server accesible;
- SQL Server Management Studio;
- carpeta `FASE_15` descargada;
- permisos para crear una base de laboratorio o una Northwind institucional ya preparada.

### No se asumirá todavía

No necesitas dominar `JOIN`, CTE, funciones ventana ni subconsultas avanzadas. No los uses para resolver las tareas de esta guía.

## 4. Herramientas y recursos para la práctica

### SQL Server

Es el motor que ejecutará T-SQL. Para un equipo nuevo, Microsoft mantiene SQL Server 2025 y permite seleccionar una edición gratuita de desarrollo durante la instalación. En un laboratorio institucional, utiliza la versión indicada por el docente si ya está instalada.

**Sitio oficial:** https://learn.microsoft.com/en-us/sql/database-engine/install-windows/install-sql-server-from-the-installation-wizard-setup?view=sql-server-ver17

**Validación mínima:** después de instalar/iniciar la instancia, debes poder conectarte desde SSMS y ejecutar `SELECT @@VERSION;`.

### SQL Server Management Studio (SSMS)

Es el cliente gráfico usado para conectarse, abrir scripts, ejecutarlos y observar resultados. La línea vigente es SSMS 22.

**Instalación oficial:** https://learn.microsoft.com/en-us/ssms/install/install

Al descargar SSMS 22, Microsoft distribuye el bootstrapper `vs_SSMS.exe`, que abre Visual Studio Installer. Instala SSMS y, al finalizar, inicia el programa.

**Validación mínima:** SSMS abre, muestra el cuadro de conexión y, después de conectar, puedes crear una ventana de consulta.

### Laboratorio FASE 15

Abre `FASE_15_Laboratorio.zip`, extrae la carpeta y localiza:

- `00_Crear_Northwind_S02_Lab.sql`;
- carpeta `estudiante/` con EJ01-EJ08;
- `README_Laboratorio.md`.

No necesitas Git para esta sesión.

## 5. Ruta de trabajo

```text
VERIFICAR SQL SERVER Y SSMS
  -> PREPARAR Northwind_S02_Lab
  -> EJ01 -> T01
  -> EJ02 -> T02
  -> EJ03 -> T03
  -> EJ04 -> T04
  -> EJ05 -> T05
  -> EJ06 -> T06
  -> EJ07 -> T07
  -> EJ08 -> T08
  -> CIERRE
```

## 6. Preparación y verificación del entorno desde cero

### Ruta A - Ya tienes SQL Server y SSMS

1. Abre SSMS y conéctate a la instancia del laboratorio.
2. Abre una nueva consulta y ejecuta:

```sql
SELECT @@VERSION AS VersionServidor;
```

3. Si recibes una fila con la versión, la conexión funciona.
4. Abre `00_Crear_Northwind_S02_Lab.sql` y ejecútalo completo. El script crea o reutiliza `Northwind_S02_Lab` y reconstruye solo sus tablas de práctica.
5. En Object Explorer, actualiza `Databases` y verifica que aparece `Northwind_S02_Lab`.
6. Ejecuta:

```sql
USE Northwind_S02_Lab;
GO
SELECT COUNT(*) AS Productos FROM dbo.Products;
```

Debes obtener una fila con un conteo mayor que cero.

**Error frecuente:** ejecutar los ejemplos en `master`.  
**Corrección:** verifica que cada archivo inicia con `USE Northwind_S02_Lab;` o selecciona explícitamente esa base en el desplegable de SSMS.

### Ruta B - Equipo sin preparar

1. Instala una edición de SQL Server adecuada para desarrollo/laboratorio desde la documentación oficial.
2. Durante el asistente selecciona una edición gratuita de desarrollo si el equipo es personal de práctica; no uses Developer para producción.
3. Completa la instalación del motor y conserva el nombre de la instancia para conectarte.
4. Instala SSMS 22 con `vs_SSMS.exe` desde la página oficial.
5. Abre SSMS y conecta al motor instalado.
6. Valida con `SELECT @@VERSION;`.
7. Ejecuta `00_Crear_Northwind_S02_Lab.sql`.
8. Verifica `Northwind_S02_Lab` y el conteo de Products.

Si tu institución ya entrega una Northwind completa, no crees la base reducida salvo indicación del docente; usa la base institucional y adapta solo la línea `USE` de los ejemplos.

## 7. Desarrollo del laboratorio mediante ejemplos resueltos


# EJ01 - SELECT, columnas, alias y expresión calculada

## Qué vamos a resolver

Construir una consulta legible sobre Products que muestre identificador, nombre, precio, stock y el valor estimado del stock.

## Qué aprenderás aquí

Seleccionar columnas específicas, asignar alias y crear una expresión calculada sin modificar los datos.

## Archivo del laboratorio

`FASE_15/estudiante/EJ01_Select_Alias_Expresion.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

`SELECT` define qué datos devuelve la consulta. Un alias cambia el encabezado del resultado, no el nombre real de la columna. Una expresión como `UnitPrice * UnitsInStock` calcula un valor para cada fila.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT ProductID,
       ProductName AS Producto,
       UnitPrice AS Precio,
       UnitsInStock AS Stock,
       UnitPrice * UnitsInStock AS StockValorizado
FROM dbo.Products;
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

Antes de ejecutar, predice si `StockValorizado` tendrá el mismo valor para todos los productos. Explica por qué.

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Agrega `QuantityPerUnit` y cambia el alias `StockValorizado` por `ValorInventario`.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Escribe temporalmente `UnitPrice * ProductName` en la expresión calculada.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

La multiplicación requiere tipos numéricos; `ProductName` es texto. SQL Server no puede aplicar esa operación de forma válida.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T01 - Tarea espejo

Muestra `EmployeeID`, `FirstName`, `LastName` y una columna calculada/concatenada `NombreCompleto` usando `CONCAT(FirstName,  , LastName)`.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** La salida debe tener una fila por empleado y el nombre completo debe combinar nombre y apellido.


# EJ02 - WHERE con comparadores

## Qué vamos a resolver

Filtrar clientes por país y productos por precio para observar cómo WHERE restringe filas.

## Qué aprenderás aquí

Aplicar condiciones básicas con `=`, `<>`, `>`, `>=`, `<` y `<=`.

## Archivo del laboratorio

`FASE_15/estudiante/EJ02_Where_Comparadores.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

`WHERE` se evalúa fila por fila. Solo pasan al resultado las filas cuya condición es verdadera. El filtro no cambia la tabla; cambia el conjunto devuelto.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT CustomerID, CompanyName, Country
FROM dbo.Customers
WHERE Country = N'Germany';

SELECT ProductID, ProductName, UnitPrice
FROM dbo.Products
WHERE UnitPrice > 20;
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

¿La segunda consulta incluirá un producto con precio exactamente 20.00? Justifica.

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Cambia `> 20` por `>= 20` y compara el número de filas.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Cambia `Country = NGermany` por `Country = Germany` sin comillas.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

Un literal de texto debe escribirse entre comillas. Sin ellas, SQL Server interpreta `Germany` como un identificador.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T02 - Tarea espejo

Muestra `CustomerID`, `CompanyName` y `Country` de clientes cuyo país sea distinto de `USA`.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** No debe aparecer ninguna fila con Country = USA.


# EJ03 - BETWEEN e IN

## Qué vamos a resolver

Seleccionar productos por rango de stock y por pertenencia a un conjunto de categorías.

## Qué aprenderás aquí

Usar `BETWEEN` para rangos inclusivos e `IN` para conjuntos discretos.

## Archivo del laboratorio

`FASE_15/estudiante/EJ03_Between_In.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

`BETWEEN a AND b` incluye los extremos. `IN (v1, v2, ...)` equivale conceptualmente a comprobar si el valor pertenece a uno de los elementos listados.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT ProductID, ProductName, UnitsInStock
FROM dbo.Products
WHERE UnitsInStock BETWEEN 10 AND 20
ORDER BY ProductName;

SELECT ProductID, ProductName, CategoryID
FROM dbo.Products
WHERE CategoryID IN (1, 4, 6);
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

¿Un producto con UnitsInStock = 10 debe aparecer? ¿Y uno con 20?

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Cambia el rango a `BETWEEN 12 AND 18` y explica qué filas límite cambian.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Invierte el rango: `BETWEEN 20 AND 10`.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

La forma directa espera límite inferior seguido del superior; con los valores invertidos no se satisfacen filas en este conjunto.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T03 - Tarea espejo

Lista productos cuyo stock esté entre 10 y 30 unidades y cuya categoría pertenezca al conjunto `(2, 7, 8)`.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** Cada fila debe cumplir simultáneamente el rango de stock y una de las categorías indicadas.


# EJ04 - LIKE y comodines

## Qué vamos a resolver

Buscar clientes por patrones de texto sin conocer el valor completo.

## Qué aprenderás aquí

Usar `%`, `_` y listas/rangos de caracteres soportados por LIKE en SQL Server.

## Archivo del laboratorio

`FASE_15/estudiante/EJ04_Like_Comodines.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

`LIKE` compara texto con un patrón. `%` representa cero o más caracteres; `_` representa un carácter; expresiones entre corchetes permiten listas o rangos de caracteres en SQL Server.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT CustomerID, CompanyName
FROM dbo.Customers
WHERE CompanyName LIKE N'C%';

SELECT CustomerID, CompanyName
FROM dbo.Customers
WHERE CompanyName LIKE N'%on%';

SELECT CustomerID, CompanyName
FROM dbo.Customers
WHERE CompanyName LIKE N'[A-C]%';
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

¿`Cactus Comidas para llevar` aparecerá en la primera y la tercera consulta?

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Prueba `CompanyName LIKE N_r%` y describe qué exige el guion bajo.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Usa `CompanyName = NC%` esperando el mismo comportamiento.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

El operador `=` compara el texto literal; los comodines solo adquieren significado con `LIKE`.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T04 - Tarea espejo

Lista clientes cuyo nombre de empresa contenga la cadena `Food` o empiece con una letra entre A y C. Resuelve cada patrón en una consulta separada.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** Debes demostrar al menos un patrón con `%` y otro con rango `[A-C]`.


# EJ05 - Funciones de agregación

## Qué vamos a resolver

Obtener resúmenes numéricos del catálogo de productos.

## Qué aprenderás aquí

Interpretar `AVG`, `MAX`, `MIN`, `SUM` y `COUNT` como valores resumen.

## Archivo del laboratorio

`FASE_15/estudiante/EJ05_Agregaciones.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

Una función de agregación procesa un conjunto de filas y devuelve un valor resumido. Sin `GROUP BY`, el conjunto es el resultado completo filtrado por la consulta.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT AVG(UnitPrice) AS PrecioPromedio,
       MAX(UnitPrice) AS PrecioMayor,
       MIN(UnitPrice) AS PrecioMenor,
       SUM(UnitsInStock) AS StockTotal,
       COUNT(*) AS NumeroProductos
FROM dbo.Products;
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

¿Cuántas filas esperas en la salida de esta consulta? ¿Por qué?

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Agrega `WHERE CategoryID = 2` antes de ejecutar y explica qué conjunto se resume ahora.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Agrega `ProductName` al SELECT sin agruparlo.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

Al combinar agregaciones con una columna no agregada, SQL Server requiere que esa columna forme parte del `GROUP BY`.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T05 - Tarea espejo

Obtén el precio promedio, el precio máximo, el precio mínimo y la cantidad total de productos de la categoría 4.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** La salida debe ser una sola fila y los cálculos deben considerar únicamente CategoryID = 4.


# EJ06 - GROUP BY

## Qué vamos a resolver

Contar pedidos por cliente y calcular precio promedio por categoría.

## Qué aprenderás aquí

Dividir filas en grupos y producir un resumen por cada valor de agrupación.

## Archivo del laboratorio

`FASE_15/estudiante/EJ06_GroupBy.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

`GROUP BY` crea grupos según una o más columnas. Cada grupo produce una fila de salida cuando el SELECT contiene la columna agrupada y funciones de agregación.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT CustomerID, COUNT(OrderID) AS NroPedidos
FROM dbo.Orders
GROUP BY CustomerID
ORDER BY NroPedidos DESC;

SELECT CategoryID, AVG(UnitPrice) AS PrecioPromedio
FROM dbo.Products
GROUP BY CategoryID
ORDER BY CategoryID;
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

¿Esperas una fila por pedido o una fila por cliente en la primera consulta?

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Cambia la primera agrupación a `EmployeeID` y compara qué significa cada fila.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Elimina `CustomerID` del `GROUP BY` pero déjalo en el SELECT.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

SQL Server no puede elegir un único CustomerID para un conjunto agregado si esa columna no define el grupo.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T06 - Tarea espejo

Muestra `EmployeeID` y la cantidad de pedidos atendidos por cada empleado, ordenando de mayor a menor.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** Debe existir una fila por EmployeeID presente en Orders y una columna de conteo.


# EJ07 - HAVING y ORDER BY sobre grupos

## Qué vamos a resolver

Mostrar solo clientes con un número de pedidos superior a un umbral.

## Qué aprenderás aquí

Diferenciar el filtrado de filas (`WHERE`) del filtrado de grupos (`HAVING`).

## Archivo del laboratorio

`FASE_15/estudiante/EJ07_Having_OrderBy.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

`HAVING` se aplica después de formar grupos y permite usar condiciones sobre agregaciones como `COUNT`. `ORDER BY` organiza el resultado final.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT CustomerID, COUNT(OrderID) AS NroPedidos
FROM dbo.Orders
GROUP BY CustomerID
HAVING COUNT(OrderID) > 2
ORDER BY NroPedidos DESC;
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

¿Qué diferencia habría entre filtrar una fecha con WHERE y filtrar `COUNT(OrderID)` con HAVING?

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Cambia el umbral de `> 2` a `>= 2` y analiza quiénes entran al resultado.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Escribe `WHERE COUNT(OrderID) > 2` antes del GROUP BY.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

`WHERE` filtra filas antes de agrupar y no puede usar directamente el resultado de esa agregación en ese punto.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T07 - Tarea espejo

Muestra `EmployeeID` y `NroPedidos` solo para empleados con más de 3 pedidos; ordena de mayor a menor.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** La condición del conteo debe estar en HAVING y el orden debe ser descendente por el alias o la agregación.


# EJ08 - Funciones de cadena, numéricas y fecha

## Qué vamos a resolver

Transformar valores sin modificar la tabla: texto, números y fechas.

## Qué aprenderás aquí

Aplicar funciones integradas y leer su resultado en una consulta.

## Archivo del laboratorio

`FASE_15/estudiante/EJ08_Funciones.sql`

## Punto de partida

`Northwind_S02_Lab` debe existir y la ventana de consulta debe estar conectada al servidor correcto.

## Concepto justo a tiempo

Las funciones integradas reciben valores y devuelven un resultado calculado. Pueden aplicarse a literales o columnas, y no cambian los datos almacenados salvo que se usen en una operación de modificación.

## Paso 1 - Lee la consulta antes de ejecutarla

```sql
USE Northwind_S02_Lab;
GO
SELECT ProductName, UPPER(ProductName) AS NombreMayusculas,
       LEFT(ProductName, 5) AS Prefijo, LEN(ProductName) AS Longitud
FROM dbo.Products;

SELECT ProductName, UnitPrice, ROUND(UnitPrice * 0.90, 2) AS PrecioConDescuento
FROM dbo.Products;

SELECT OrderID, OrderDate,
       DATEPART(year, OrderDate) AS Anio,
       DATEADD(day, 7, OrderDate) AS FechaMas7Dias,
       DATEDIFF(day, OrderDate, GETDATE()) AS DiasHastaHoy
FROM dbo.Orders;
```

Identifica en voz alta o por escrito: tabla origen, columnas devueltas, condición o cálculo principal y alias utilizados.

## Paso 2 - Antes de ejecutar: predicción

¿`ROUND(UnitPrice * 0.90, 2)` cambia el precio almacenado en Products?

## Paso 3 - Ejecuta y compara

Ejecuta el bloque completo. Comprueba que no hay error y compara el resultado con tu predicción. No te limites a contar filas: observa los encabezados y los valores que justifican la salida.

## Variante A - cambia una condición

Sustituye `LEFT(ProductName, 5)` por `RIGHT(ProductName, 4)` y compara el fragmento devuelto.

Vuelve a ejecutar y describe qué cambió y qué permaneció igual.

## Error controlado

Usa `SUBSTRING(ProductName, 0, 5)` esperando los cinco primeros caracteres.

Ejecuta solo si el docente indica que es seguro. Después restaura la versión correcta.

## Por qué ocurrió

En T-SQL la posición inicial habitual de `SUBSTRING` comienza en 1; usar 0 cambia el fragmento esperado.

## Corrección razonada

Vuelve al archivo original, restaura la expresión correcta y confirma que la consulta ejecuta sin error y produce la evidencia esperada.

## Qué debes poder explicar con tus palabras

1. ¿Qué parte de la consulta decide las columnas de salida?
2. ¿Qué parte decide qué filas o grupos permanecen?
3. ¿Qué resultado específico usaste para validar la consulta?

## T08 - Tarea espejo

Genera un reporte con ProductName, nombre en minúsculas, los 3 primeros caracteres y precio con 10% de descuento redondeado a 2 decimales. Luego, en una consulta separada, muestra OrderID, OrderDate y el mes con DATEPART.

**Restricción:** resuélvela con conceptos presentados hasta este ejemplo; no introduzcas joins ni subconsultas.

**Pista:** parte del ejemplo, pero cambia la tabla/columnas o la condición; no copies la solución final del ejemplo.

**Evidencia:** consulta ejecutada y captura o copia del resultado principal.

**Criterio para saber si está correcta:** Debe evidenciar al menos una función de cadena, ROUND y DATEPART con alias legibles.


## 8. Tareas espejo de consolidación

| TAREA_ID | EJEMPLO_ID | Evidencia mínima |
|---|---|---|
| T01 | EJ01 | Consulta + resultado + explicación breve de validación |
| T02 | EJ02 | Consulta + resultado + explicación breve de validación |
| T03 | EJ03 | Consulta + resultado + explicación breve de validación |
| T04 | EJ04 | Consulta + resultado + explicación breve de validación |
| T05 | EJ05 | Consulta + resultado + explicación breve de validación |
| T06 | EJ06 | Consulta + resultado + explicación breve de validación |
| T07 | EJ07 | Consulta + resultado + explicación breve de validación |
| T08 | EJ08 | Consulta + resultado + explicación breve de validación |


## 9. Cierre de la sesión

Hoy pasaste de escribir consultas aisladas a razonar sobre cómo SQL Server construye un resultado. Empezaste seleccionando columnas y expresiones, después restringiste filas con `WHERE`, rangos y conjuntos, y comprobaste que los patrones de `LIKE` permiten búsquedas que no requieren conocer el texto completo. Luego cambiaste de escala: en lugar de mirar una fila, resumiste conjuntos con funciones de agregación, formaste grupos con `GROUP BY` y aplicaste `HAVING` cuando el criterio dependía del resultado agregado. Finalmente utilizaste funciones integradas para transformar texto, números y fechas sin modificar los datos almacenados.

Antes de dar por terminada la sesión, revisa tus T01-T08 y asegúrate de que puedes explicar por qué cada consulta es correcta. Si una salida te sorprende, no la memorices: vuelve a la condición, predice de nuevo y compara. Esa disciplina será importante en las siguientes sesiones, donde las consultas podrán incorporar nuevas formas de relacionar o transformar información.

Para continuar repasando, puedes revisar recursos académicos de Lideratec Academy:  
Blog: https://lideratecacademy.com/blog/  
Canal: https://www.youtube.com/@LideratecAcademy
