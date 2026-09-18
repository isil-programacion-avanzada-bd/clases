---
course_id: ISIL-PABD
session_id: S03
module_id: UA1
course_version: 2026.09
source_origin: PPT
status: validated
---
# Guía del Estudiante - S03

# Tipos de consultas (Parte 2): JOIN, CASE y UNION

**Curso:** Programación Avanzada de Base de Datos  
**Base de práctica:** Northwind  
**Tecnología:** Transact-SQL en Microsoft SQL Server  

## 1. Propósito de la sesión

En esta sesión construirás y ejecutarás consultas sobre varias tablas de Northwind. Aprenderás a decidir cuándo necesitas conservar solo coincidencias, cuándo debes mantener filas sin pareja, cuándo quieres generar combinaciones, cómo representar una relación de una tabla consigo misma, cómo clasificar resultados con CASE y cómo reunir varios SELECT con UNION.

El producto observable será un conjunto de **8 consultas resueltas y 8 tareas espejo**. Cada ejemplo se construye de forma incremental: primero se identifica la pregunta, luego se escriben las partes de la consulta, se predice el resultado, se ejecuta, se interpreta y finalmente se modifica.

## 2. Resultado observable

### A. Lectura sugerida del docente

Al terminar esta práctica no bastará con decir que una consulta “funciona”. Deberás explicar qué conjunto se preserva, qué clave relaciona las tablas, qué condición evalúa CASE o por qué las ramas de un UNION son compatibles. El objetivo es que puedas anticipar el comportamiento antes de ejecutar y que puedas detectar por qué una consulta devuelve demasiadas filas, pierde filas que querías conservar o falla por incompatibilidad de estructura. Construirás los ejemplos sobre Northwind, una base con relaciones entre clientes, pedidos, detalles, productos, categorías, proveedores y empleados. El resultado correcto no se valida solo porque SQL Server muestre una cuadrícula: debes leer qué representa cada fila y comprobar que la operación elegida responde a la pregunta original.

### B. Desempeños observables

1. Construir JOINs con condiciones ON correctas.
2. Predecir qué filas conservará INNER, LEFT, RIGHT y FULL OUTER JOIN.
3. Estimar el tamaño de un CROSS JOIN.
4. Representar una jerarquía con SELF JOIN.
5. Crear columnas calculadas con CASE.
6. Integrar CASE con JOIN y SUM.
7. Reunir SELECT compatibles con UNION y UNION ALL.
8. Diagnosticar y corregir errores seguros de consulta.

### C. Criterio de dominio

Demuestras dominio cuando puedes escribir una consulta equivalente sin copiar, justificar cada relación o condición y explicar el resultado esperado antes de ejecutarla.

## 3. Antes de iniciar

### Debe saber

- Conectarse a una instancia de SQL Server.
- Abrir una ventana de consulta.
- Ejecutar SELECT simples y filtros básicos.
- Reconocer claves principales y foráneas en el modelo Northwind.

### Debe tener disponible

- SQL Server institucional o local.
- SQL Server Management Studio (SSMS). La versión actual de la herramienta es SSMS 22; si el laboratorio institucional usa una versión anterior compatible, las construcciones T-SQL de esta sesión siguen siendo las mismas.
- Base `Northwind`.
- Permiso de lectura sobre las tablas de práctica.

### No se asumirá todavía

- Procedimientos almacenados.
- Funciones definidas por el usuario.
- CTE, ventanas analíticas o PIVOT.
- Optimización avanzada de planes de ejecución.

## 4. Herramientas y recursos

### SQL Server Management Studio (SSMS)

SSMS es el entorno usado para conectarse al motor, abrir consultas y revisar resultados. Microsoft distribuye SSMS 22 mediante el instalador de Visual Studio. Si ya tienes una instalación institucional funcional, no necesitas reemplazarla solo para esta sesión.

**Sitio oficial:** https://learn.microsoft.com/es-es/ssms/install/install

**Validación mínima:** abre SSMS, conéctate y ejecuta:

```sql
SELECT @@VERSION AS VersionServidor;
```

Debe aparecer una fila con información de la instancia.

### Northwind

La sesión continúa sobre Northwind, que la PPT considera instalada en la clase anterior. Si no la tienes, Microsoft mantiene el script histórico `instnwnd.sql` dentro del repositorio oficial de ejemplos. La instalación debe realizarse en un entorno de laboratorio, no sobre producción.

**Referencia oficial:** https://github.com/microsoft/sql-server-samples/tree/master/samples/databases/northwind-pubs

## 5. Preparación del entorno desde cero

### Ruta A - El entorno ya existe

1. Abre SSMS.
2. Conéctate a la instancia usada en clase.
3. En Object Explorer, verifica que aparezca `Northwind`.
4. Abre **New Query**.
5. Ejecuta:

```sql
USE Northwind;
GO
SELECT DB_NAME() AS BaseActual;
```

6. Comprueba que `BaseActual` sea `Northwind`.
7. Valida las tablas esenciales:

```sql
SELECT
    OBJECT_ID('dbo.Customers') AS CustomersId,
    OBJECT_ID('dbo.Orders') AS OrdersId,
    OBJECT_ID('dbo.Products') AS ProductsId,
    OBJECT_ID('dbo.Employees') AS EmployeesId;
```

Los identificadores no deben ser NULL.

### Ruta B - El equipo no está preparado

1. Instala o utiliza la instancia SQL Server definida por tu institución.
2. Instala SSMS desde la documentación oficial indicada arriba si no cuentas con cliente gráfico.
3. Descarga el script oficial de Northwind únicamente desde el repositorio de Microsoft.
4. Abre el script en SSMS y ejecútalo en una instancia de laboratorio siguiendo su README.
5. Verifica que la base aparezca en Object Explorer.
6. Ejecuta el bloque `USE Northwind` y las validaciones de tablas.

> Si la institución ya suministra Northwind, conserva esa copia para mantener coherencia con los datos usados por el curso.

## 6. Proyecto base de la práctica

Esta sesión no necesita un proyecto de software. El “proyecto base” es una ventana de consulta conectada a Northwind. Crea un archivo llamado `S03_Consultas_Parte2.sql` y escribe al inicio:

```sql
USE Northwind;
GO
```

Durante la clase puedes mantener cada ejemplo separado por comentarios o utilizar los archivos individuales de FASE 15 como recurso opcional de validación posterior.

---
# EJ01 - INNER JOIN: productos y categorías
## Qué vamos a construir

Relacionar Products con Categories y devolver únicamente productos que encuentran una categoría coincidente.

## Qué aprenderás aquí

Identificar la clave de relación, ubicar la condición en ON y leer el resultado combinado.

## Archivos que vamos a crear o modificar

- `EJ01_inner_join_productos_categorias.sql`

## Punto de partida

Northwind disponible y una ventana de consulta nueva en SSMS.
## Paso 1 - Seleccionar columnas del producto

### Escribe / ejecuta

```sql
SELECT
    p.ProductID,
    p.ProductName
```

### Qué significa

Empezamos por definir qué datos del producto deben verse. El alias p evita repetir el nombre completo de la tabla.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Agregar la tabla principal

### Escribe / ejecuta

```sql
FROM dbo.Products AS p
```

### Qué significa

Products es el conjunto de partida. Cada fila representa un producto.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 3 - Relacionar Categories

### Escribe / ejecuta

```sql
INNER JOIN dbo.Categories AS c
    ON p.CategoryID = c.CategoryID
```

### Qué significa

INNER JOIN conserva solo las parejas que cumplen la igualdad de CategoryID. La condición ON expresa la relación entre ambas tablas.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 4 - Incluir el dato de la categoría

### Escribe / ejecuta

```sql
c.CategoryName
```

### Qué significa

El resultado ahora mezcla columnas procedentes de las dos tablas en una misma fila.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

SELECT
    p.ProductID,
    p.ProductName,
    c.CategoryName
FROM dbo.Products AS p
INNER JOIN dbo.Categories AS c
    ON p.CategoryID = c.CategoryID
ORDER BY c.CategoryName, p.ProductName;
```
## Antes de ejecutar: predicción

1. ¿Aparecerá una fila si el producto no tiene CategoryID coincidente?
2. ¿Qué columnas provienen de Products y cuál de Categories?

## Ejecuta

Selecciona el bloque completo de EJ01 y ejecuta **Execute** en SSMS.

## Resultado esperado

Una lista de productos acompañados por CategoryName. Solo aparecen filas que cumplen p.CategoryID = c.CategoryID.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Sustituye INNER JOIN por la sintaxis implícita `FROM Products p, Categories c WHERE p.CategoryID = c.CategoryID` y compara el resultado. La sesión la reconoce como JOIN implícito; para el laboratorio se prioriza la forma explícita porque mantiene la relación junto al JOIN.

## Error controlado

Elimina temporalmente la cláusula ON. SQL Server no tendrá una condición válida para ese INNER JOIN y la consulta no podrá compilar correctamente.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Restaurar `ON p.CategoryID = c.CategoryID` y comprobar que las columnas comparadas pertenecen a la relación correcta.

## Qué debes poder explicar con tus palabras
- Qué papel cumple `ON`.
- Por qué `p` y `c` son alias distintos.
- Qué significa que INNER JOIN descarte filas sin coincidencia.

## T01 - Tarea espejo

Obtén el nombre del proveedor y el nombre de cada producto que suministra, usando Suppliers y Products con INNER JOIN.

### Archivos a modificar

- `tareas/T01_EJ01.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

Usa alias de tabla y coloca la relación SupplierID en ON. No uses subconsultas.

### Pista

Products contiene SupplierID y Suppliers contiene SupplierID.

### Evidencia que debes mostrar

Captura o copia del resultado y consulta SQL final, con al menos las columnas CompanyName y ProductName.

### Cómo saber si está correcta

La consulta relaciona Suppliers con Products por SupplierID y no produce producto cartesiano.

---
# EJ02 - LEFT OUTER JOIN: clientes con o sin pedidos
## Qué vamos a construir

Preservar todos los clientes y mostrar sus pedidos cuando existan.

## Qué aprenderás aquí

Distinguir el lado preservado de un LEFT OUTER JOIN e interpretar NULL en columnas de la tabla derecha.

## Archivos que vamos a crear o modificar

- `EJ02_left_join_clientes_pedidos.sql`

## Punto de partida

Northwind activa. Ya se comprende INNER JOIN.
## Paso 1 - Partir de Customers

### Escribe / ejecuta

```sql
FROM dbo.Customers AS c
```

### Qué significa

La tabla ubicada a la izquierda es la que deseamos preservar completa.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Agregar Orders con LEFT OUTER JOIN

### Escribe / ejecuta

```sql
LEFT OUTER JOIN dbo.Orders AS o
    ON c.CustomerID = o.CustomerID
```

### Qué significa

Se buscan pedidos coincidentes. Si no existen, el cliente permanece y las columnas de Orders quedan en NULL.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 3 - Seleccionar columnas de ambos lados

### Escribe / ejecuta

```sql
c.CustomerID, c.ContactName, o.OrderID, o.OrderDate
```

### Qué significa

Las columnas de Orders permiten observar explícitamente cuándo hubo coincidencia.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

SELECT
    c.CustomerID,
    c.ContactName,
    o.OrderID,
    o.OrderDate
FROM dbo.Customers AS c
LEFT OUTER JOIN dbo.Orders AS o
    ON c.CustomerID = o.CustomerID
ORDER BY c.CustomerID, o.OrderDate;
```
## Antes de ejecutar: predicción

1. Si un cliente no tiene pedidos, ¿desaparece o permanece?
2. ¿Qué valor esperas en OrderID cuando no haya coincidencia?

## Ejecuta

Selecciona el bloque completo de EJ02 y ejecuta **Execute** en SSMS.

## Resultado esperado

Todos los clientes del lado izquierdo; para los clientes sin pedido, OrderID y OrderDate aparecen como NULL.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Agrega temporalmente `WHERE o.OrderID IS NOT NULL`. Observa que las filas sin pedido dejan de aparecer; el filtro elimina el efecto de preservación que queríamos observar.

## Error controlado

Colocar un filtro obligatorio sobre una columna de Orders en WHERE puede descartar las filas NULL que LEFT JOIN pretendía conservar.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Si el objetivo es preservar clientes sin pedidos, no filtres después por una condición que exija una fila de Orders, o mueve una condición pertinente a la relación ON cuando conceptualmente corresponda.

## Qué debes poder explicar con tus palabras
- Qué significa “lado izquierdo”.
- Qué representa NULL en las columnas de Orders.
- Por qué LEFT JOIN responde una pregunta distinta de INNER JOIN.

## T02 - Tarea espejo

Muestra todos los productos y su categoría cuando exista, usando Products como lado izquierdo y Categories como lado derecho.

### Archivos a modificar

- `tareas/T02_EJ02.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

Usa LEFT OUTER JOIN y conserva ProductID, ProductName y CategoryName.

### Pista

La relación usa CategoryID.

### Evidencia que debes mostrar

Consulta y resultado; explica en una frase qué significaría un CategoryName NULL.

### Cómo saber si está correcta

Products queda a la izquierda y el JOIN se realiza por CategoryID.

---
# EJ03 - RIGHT y FULL OUTER JOIN: qué lado se preserva
## Qué vamos a construir

Comparar RIGHT OUTER JOIN y FULL OUTER JOIN y reconocer qué filas se conservan.

## Qué aprenderás aquí

Leer la orientación del JOIN y distinguir conservación del lado derecho frente a conservación de ambos lados.

## Archivos que vamos a crear o modificar

- `EJ03_right_full_outer_join.sql`

## Punto de partida

Se domina la idea de preservación de LEFT OUTER JOIN.
## Paso 1 - Leer primero el RIGHT JOIN

### Escribe / ejecuta

```sql
FROM dbo.Customers AS c
RIGHT OUTER JOIN dbo.Orders AS o
    ON c.CustomerID = o.CustomerID
```

### Qué significa

El conjunto obligatorio es Orders porque está a la derecha. Customers aporta datos cuando hay coincidencia.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Comparar con FULL OUTER JOIN

### Escribe / ejecuta

```sql
FROM dbo.Products AS p
FULL OUTER JOIN dbo.[Order Details] AS od
    ON p.ProductID = od.ProductID
```

### Qué significa

FULL preserva ambos lados. Si un lado no encuentra pareja, las columnas del otro lado aparecen como NULL.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

-- A. Todos los pedidos, con el cliente correspondiente cuando exista
SELECT
    c.CustomerID,
    c.ContactName,
    o.OrderID,
    o.OrderDate
FROM dbo.Customers AS c
RIGHT OUTER JOIN dbo.Orders AS o
    ON c.CustomerID = o.CustomerID
ORDER BY o.OrderID;

-- B. Todos los productos y todos los detalles de pedido, coincidan o no
SELECT
    p.ProductID,
    p.ProductName,
    od.OrderID,
    od.Quantity
FROM dbo.Products AS p
FULL OUTER JOIN dbo.[Order Details] AS od
    ON p.ProductID = od.ProductID
ORDER BY p.ProductID, od.OrderID;
```
## Antes de ejecutar: predicción

1. En el primer SELECT, ¿qué tabla se preserva completa?
2. En el segundo, ¿qué dos tipos de filas sin coincidencia podrían existir en teoría?

## Ejecuta

Selecciona el bloque completo de EJ03 y ejecuta **Execute** en SSMS.

## Resultado esperado

RIGHT JOIN mantiene Orders; FULL OUTER JOIN combina coincidencias y además preserva filas no emparejadas de ambos lados.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Reescribe el primer RIGHT JOIN como `Orders o LEFT JOIN Customers c` manteniendo la misma condición y compara el significado. Esto muestra que RIGHT JOIN puede expresarse cambiando el orden de tablas.

## Error controlado

Cambiar el orden de las tablas sin cambiar el tipo de OUTER JOIN puede cambiar qué conjunto se preserva.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Antes de ejecutar, verbaliza: “quiero preservar ___”. Luego coloca esa tabla en el lado que el JOIN preserva o usa LEFT JOIN reordenando las tablas.

## Qué debes poder explicar con tus palabras
- Qué preserva RIGHT.
- Qué preserva FULL.
- Por qué la orientación importa aunque la condición ON use igualdad.

## T03 - Tarea espejo

Construye dos consultas con Suppliers y Products: una con RIGHT OUTER JOIN que preserve Products y otra con FULL OUTER JOIN que preserve ambos conjuntos.

### Archivos a modificar

- `tareas/T03_EJ03.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

Ambas consultas deben usar SupplierID como condición de relación y mostrar CompanyName, ProductID y ProductName.

### Pista

En la primera consulta, coloca Products a la derecha.

### Evidencia que debes mostrar

Dos consultas y una explicación de una frase sobre la diferencia entre RIGHT y FULL.

### Cómo saber si está correcta

RIGHT preserva Products; FULL preserva Suppliers y Products, con NULL donde no hay coincidencia.

---
# EJ04 - CROSS JOIN: combinaciones posibles
## Qué vamos a construir

Generar un producto cartesiano y estimar su tamaño antes de visualizarlo.

## Qué aprenderás aquí

Relacionar el número de filas de cada tabla con la cantidad de combinaciones producidas.

## Archivos que vamos a crear o modificar

- `EJ04_cross_join.sql`

## Punto de partida

Northwind activa. Se reconoce que CROSS JOIN no usa condición ON.
## Paso 1 - Contar antes de listar

### Escribe / ejecuta

```sql
SELECT COUNT(*) AS TotalCombinaciones
FROM dbo.Products AS p
CROSS JOIN dbo.Suppliers AS s;
```

### Qué significa

Cada fila de Products se combina con cada fila de Suppliers. Contar primero evita interpretar un resultado grande sin contexto.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Visualizar una muestra

### Escribe / ejecuta

```sql
SELECT TOP (20) p.ProductName, s.CompanyName
FROM dbo.Products AS p
CROSS JOIN dbo.Suppliers AS s;
```

### Qué significa

TOP limita la visualización; el concepto sigue siendo el producto cartesiano completo.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

SELECT COUNT(*) AS TotalCombinaciones
FROM dbo.Products AS p
CROSS JOIN dbo.Suppliers AS s;

SELECT TOP (20)
    p.ProductName,
    s.CompanyName
FROM dbo.Products AS p
CROSS JOIN dbo.Suppliers AS s
ORDER BY p.ProductName, s.CompanyName;
```
## Antes de ejecutar: predicción

1. Si Products tiene P filas y Suppliers S filas, ¿cuántas combinaciones produce el CROSS JOIN?
2. ¿Por qué no hay cláusula ON?

## Ejecuta

Selecciona el bloque completo de EJ04 y ejecuta **Execute** en SSMS.

## Resultado esperado

El conteo es P x S y cada producto aparece combinado con múltiples proveedores.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Sustituye las tablas por Categories y Shippers y calcula primero el número esperado de combinaciones a mano.

## Error controlado

Usar CROSS JOIN cuando en realidad se necesita relacionar claves produce muchas filas que no representan relaciones reales.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Si existe una relación lógica entre tablas, utiliza el JOIN correspondiente con ON; reserva CROSS JOIN para combinaciones deliberadas.

## Qué debes poder explicar con tus palabras
- Qué es un producto cartesiano.
- Por qué el tamaño crece multiplicativamente.
- Por qué CROSS JOIN no necesita ON.

## T04 - Tarea espejo

Combina todas las categorías con todos los transportistas (Shippers) usando CROSS JOIN. Primero devuelve el total de combinaciones y luego una lista con CategoryName y CompanyName.

### Archivos a modificar

- `tareas/T04_EJ04.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

No uses ON. Ejecuta el COUNT antes de listar las filas.

### Pista

Número esperado = filas de Categories x filas de Shippers.

### Evidencia que debes mostrar

Resultado del COUNT y consulta de listado.

### Cómo saber si está correcta

La consulta usa CROSS JOIN y el estudiante justifica el total como producto de los tamaños de los conjuntos.

---
# EJ05 - SELF JOIN: empleado y supervisor
## Qué vamos a construir

Usar dos alias de Employees para representar dos roles distintos de la misma tabla.

## Qué aprenderás aquí

Interpretar una relación recursiva mediante ReportsTo y EmployeeID.

## Archivos que vamos a crear o modificar

- `EJ05_self_join_empleados.sql`

## Punto de partida

Se domina INNER JOIN y el uso de alias.
## Paso 1 - Asignar dos roles a Employees

### Escribe / ejecuta

```sql
FROM dbo.Employees AS e
INNER JOIN dbo.Employees AS s
```

### Qué significa

No son dos tablas físicas distintas. Los alias e y s permiten leer la misma tabla como “empleado” y “supervisor”.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Relacionar ReportsTo con EmployeeID

### Escribe / ejecuta

```sql
ON e.ReportsTo = s.EmployeeID
```

### Qué significa

ReportsTo del empleado apunta al EmployeeID de su supervisor.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 3 - Construir nombres legibles

### Escribe / ejecuta

```sql
e.FirstName + ' ' + e.LastName AS EmployeeName
```

### Qué significa

Se concatenan nombre y apellido para interpretar mejor cada rol en el resultado.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

SELECT
    e.EmployeeID,
    e.FirstName + ' ' + e.LastName AS EmployeeName,
    s.EmployeeID AS SupervisorID,
    s.FirstName + ' ' + s.LastName AS SupervisorName
FROM dbo.Employees AS e
INNER JOIN dbo.Employees AS s
    ON e.ReportsTo = s.EmployeeID
ORDER BY e.EmployeeID;
```
## Antes de ejecutar: predicción

1. ¿Por qué necesitamos dos alias si solo existe una tabla Employees?
2. ¿Qué ocurre con un empleado cuyo ReportsTo sea NULL cuando usamos INNER JOIN?

## Ejecuta

Selecciona el bloque completo de EJ05 y ejecuta **Execute** en SSMS.

## Resultado esperado

Cada fila muestra un empleado y el supervisor al que apunta ReportsTo. Un empleado sin supervisor no aparece con INNER JOIN.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Cambia INNER JOIN por LEFT OUTER JOIN para observar también al empleado que no tiene supervisor directo.

## Error controlado

Usar `ON e.EmployeeID = s.EmployeeID` relaciona cada empleado consigo mismo y no modela la jerarquía.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Volver a la relación recursiva `e.ReportsTo = s.EmployeeID`.

## Qué debes poder explicar con tus palabras
- Qué representan e y s.
- Qué columna actúa como referencia al supervisor.
- Por qué SELF JOIN sigue siendo un JOIN normal entre dos fuentes lógicas.

## T05 - Tarea espejo

Muestra EmployeeName, EmployeeTitle, SupervisorName y SupervisorTitle para los empleados que tienen supervisor, usando SELF JOIN.

### Archivos a modificar

- `tareas/T05_EJ05.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

Usa INNER JOIN y conserva la relación ReportsTo = EmployeeID.

### Pista

Title existe en Employees y debe leerse con ambos alias.

### Evidencia que debes mostrar

Consulta y resultado con cuatro columnas descriptivas.

### Cómo saber si está correcta

Los alias representan roles diferentes y la relación usa ReportsTo.

---
# EJ06 - CASE: clasificar el estado de un pedido
## Qué vamos a construir

Crear una columna calculada que clasifique un pedido según ShippedDate.

## Qué aprenderás aquí

Evaluar una condición con CASE WHEN y devolver textos alternativos.

## Archivos que vamos a crear o modificar

- `EJ06_case_estado_pedido.sql`

## Punto de partida

Se sabe seleccionar columnas de Orders.
## Paso 1 - Seleccionar la evidencia original

### Escribe / ejecuta

```sql
OrderID, ShippedDate
```

### Qué significa

Conservar ShippedDate permite comprobar de dónde sale la clasificación.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Abrir la expresión CASE

### Escribe / ejecuta

```sql
CASE
    WHEN ShippedDate IS NOT NULL THEN 'Enviado'
    ELSE 'No enviado'
END AS Estado
```

### Qué significa

CASE evalúa la condición y devuelve una expresión de resultado. Microsoft la documenta técnicamente como una expresión CASE.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 3 - Asignar alias a la columna calculada

### Escribe / ejecuta

```sql
END AS Estado
```

### Qué significa

El alias permite tratar el resultado de CASE como una columna legible.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

SELECT
    OrderID,
    ShippedDate,
    CASE
        WHEN ShippedDate IS NOT NULL THEN 'Enviado'
        ELSE 'No enviado'
    END AS Estado
FROM dbo.Orders
ORDER BY OrderID;
```
## Antes de ejecutar: predicción

1. ¿Qué Estado tendrá una fila con ShippedDate NULL?
2. ¿Qué Estado tendrá una fila con una fecha de envío?

## Ejecuta

Selecciona el bloque completo de EJ06 y ejecuta **Execute** en SSMS.

## Resultado esperado

Cada pedido aparece con “Enviado” cuando ShippedDate tiene valor y “No enviado” cuando es NULL.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Filtra visualmente algunas filas con ShippedDate NULL y comprueba que coincidan con Estado = No enviado.

## Error controlado

Usar `ShippedDate = NULL` no evalúa NULL como una igualdad normal y no expresa la condición correctamente.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Usar `IS NULL` o `IS NOT NULL` según la condición requerida.

## Qué debes poder explicar con tus palabras
- Qué evalúa WHEN.
- Qué devuelve THEN/ELSE.
- Por qué el alias Estado no existe físicamente en Orders.

## T06 - Tarea espejo

Clasifica los productos por UnitsInStock: “Sin stock” cuando sea 0, “Stock bajo” cuando sea menor que 20 y “Stock disponible” en los demás casos.

### Archivos a modificar

- `tareas/T06_EJ06.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

Usa CASE buscado con al menos dos WHEN y un ELSE.

### Pista

Evalúa primero la condición más específica: UnitsInStock = 0.

### Evidencia que debes mostrar

Consulta con ProductName, UnitsInStock y una columna EstadoStock.

### Cómo saber si está correcta

Las condiciones están ordenadas para que 0 no quede absorbido por la condición < 20.

---
# EJ07 - CASE + JOIN + SUM: clasificar clientes por ventas
## Qué vamos a construir

Combinar JOIN, agregación y CASE para clasificar clientes según sus ventas acumuladas.

## Qué aprenderás aquí

Integrar datos de Customers, Orders y Order Details y reutilizar la misma expresión agregada dentro de CASE.

## Archivos que vamos a crear o modificar

- `EJ07_case_join_ventas.sql`

## Punto de partida

Se dominan JOIN y CASE básicos. La función SUM forma parte del ejemplo mostrado en la sesión.
## Paso 1 - Construir la cadena de relaciones

### Escribe / ejecuta

```sql
FROM dbo.Customers AS c
INNER JOIN dbo.Orders AS o ON c.CustomerID = o.CustomerID
INNER JOIN dbo.[Order Details] AS od ON o.OrderID = od.OrderID
```

### Qué significa

Customers se conecta con Orders y Orders con Order Details; así llegamos al precio y cantidad de cada detalle vendido.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Calcular ventas acumuladas

### Escribe / ejecuta

```sql
SUM(od.UnitPrice * od.Quantity) AS VentasTotales
```

### Qué significa

Cada detalle aporta precio por cantidad; SUM acumula esos importes por cliente.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 3 - Clasificar el agregado

### Escribe / ejecuta

```sql
CASE
    WHEN SUM(od.UnitPrice * od.Quantity) > 10000 THEN 'Grande'
    WHEN SUM(od.UnitPrice * od.Quantity) > 5000 THEN 'Mediano'
    ELSE 'Pequeño'
END AS Tamano
```

### Qué significa

Las condiciones se evalúan de arriba hacia abajo. El umbral mayor debe ir primero.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 4 - Agrupar por cliente

### Escribe / ejecuta

```sql
GROUP BY c.CustomerID, c.ContactName
```

### Qué significa

La agrupación define una fila de resultado por cliente.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

SELECT
    c.CustomerID,
    c.ContactName,
    SUM(od.UnitPrice * od.Quantity) AS VentasTotales,
    CASE
        WHEN SUM(od.UnitPrice * od.Quantity) > 10000 THEN 'Grande'
        WHEN SUM(od.UnitPrice * od.Quantity) > 5000 THEN 'Mediano'
        ELSE 'Pequeño'
    END AS Tamano
FROM dbo.Customers AS c
INNER JOIN dbo.Orders AS o
    ON c.CustomerID = o.CustomerID
INNER JOIN dbo.[Order Details] AS od
    ON o.OrderID = od.OrderID
GROUP BY c.CustomerID, c.ContactName
ORDER BY VentasTotales DESC;
```
## Antes de ejecutar: predicción

1. ¿Por qué no basta con unir tablas si queremos una fila por cliente?
2. ¿Qué ocurriría si evaluamos > 5000 antes que > 10000?

## Ejecuta

Selecciona el bloque completo de EJ07 y ejecuta **Execute** en SSMS.

## Resultado esperado

Una fila por cliente con ventas acumuladas y clasificación Grande, Mediano o Pequeño.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Cambia temporalmente los umbrales y observa qué clientes cambian de categoría sin modificar las relaciones JOIN.

## Error controlado

Quitar GROUP BY mientras se seleccionan columnas no agregadas produce una consulta inválida en SQL Server.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Agrupar por las columnas no agregadas que identifican al cliente.

## Qué debes poder explicar con tus palabras
- La ruta Customers -> Orders -> Order Details.
- La diferencia entre detalle y agregado.
- Por qué el orden de WHEN cambia el resultado.

## T07 - Tarea espejo

Calcula ventas acumuladas por empleado y clasifícalas como “Alta” (> 50000), “Media” (> 20000) o “Baja” (resto), usando Employees, Orders y Order Details.

### Archivos a modificar

- `tareas/T07_EJ07.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

Una fila por empleado; usa INNER JOIN, SUM, GROUP BY y CASE.

### Pista

Orders contiene EmployeeID y Order Details contiene OrderID.

### Evidencia que debes mostrar

Consulta con EmployeeName, VentasTotales y NivelVentas.

### Cómo saber si está correcta

La cadena de JOIN es correcta, hay agrupación por empleado y CASE evalúa primero el umbral mayor.

---
# EJ08 - UNION y UNION ALL: reunir resultados compatibles
## Qué vamos a construir

Combinar resultados de SELECT con la misma cantidad y orden de columnas y observar la diferencia entre UNION y UNION ALL.

## Qué aprenderás aquí

Verificar compatibilidad posicional de columnas y reconocer que UNION elimina duplicados mientras UNION ALL los conserva.

## Archivos que vamos a crear o modificar

- `EJ08_union_union_all.sql`

## Punto de partida

Northwind activa. Se comprenden SELECT y columnas calculadas literales.
## Paso 1 - Construir el primer SELECT

### Escribe / ejecuta

```sql
SELECT p.ProductName AS Nombre, 'Producto' AS Tipo
FROM dbo.Products AS p
```

### Qué significa

Define dos columnas: un nombre y una etiqueta de tipo.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 2 - Agregar el segundo SELECT con UNION

### Escribe / ejecuta

```sql
UNION
SELECT c.CategoryName, 'Categoria'
FROM dbo.Categories AS c
```

### Qué significa

El segundo SELECT devuelve también dos columnas en posiciones compatibles.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Paso 3 - Comparar UNION y UNION ALL

### Escribe / ejecuta

```sql
SELECT Country FROM dbo.Customers
UNION
SELECT Country FROM dbo.Suppliers;

SELECT Country FROM dbo.Customers
UNION ALL
SELECT Country FROM dbo.Suppliers;
```

### Qué significa

La primera variante elimina filas duplicadas; la segunda conserva todas las filas.

### Por qué lo hacemos ahora

Este bloque introduce solo la decisión necesaria para avanzar al siguiente estado de la consulta. Así puedes verificar cada relación antes de sumar otra parte.

### Qué debería ocurrir

La consulta queda estructuralmente más cerca del objetivo, pero todavía no se considera terminada hasta revisar el archivo completo y ejecutar la versión final.
## Archivo completo al terminar este ejemplo

```sql
USE Northwind;
GO

-- A. Una sola lista con productos y categorías
SELECT
    p.ProductName AS Nombre,
    'Producto' AS Tipo
FROM dbo.Products AS p
UNION
SELECT
    c.CategoryName,
    'Categoria'
FROM dbo.Categories AS c
ORDER BY Tipo, Nombre;

-- B. Comparación de duplicados por país
SELECT Country
FROM dbo.Customers
UNION
SELECT Country
FROM dbo.Suppliers;

SELECT Country
FROM dbo.Customers
UNION ALL
SELECT Country
FROM dbo.Suppliers;
```
## Antes de ejecutar: predicción

1. ¿Qué debe coincidir entre los SELECT unidos?
2. ¿Cuál de las dos variantes puede devolver más filas: UNION o UNION ALL?

## Ejecuta

Selecciona el bloque completo de EJ08 y ejecuta **Execute** en SSMS.

## Resultado esperado

La primera consulta genera una lista común de nombres con su tipo. En la comparación de países, UNION ALL puede contener repeticiones que UNION elimina.

## Cómo interpretarlo

No leas la cuadrícula como filas aisladas. Identifica qué tabla aporta cada columna y qué regla decidió que una fila apareciera, se conservara, se clasificara o se reuniera.

## Variación A

Intercambia el orden de los SELECT y comprueba que los nombres de columnas finales se toman del primer SELECT.

## Error controlado

Intentar unir un SELECT de una columna con otro de dos columnas genera incompatibilidad de estructura.

## Por qué ocurre

El error cambia una condición estructural de la operación SQL, por lo que el motor ya no puede representar correctamente la intención original o devuelve un conjunto distinto del esperado.

## Corrección razonada

Asegurar la misma cantidad de columnas y tipos de datos compatibles en posiciones equivalentes.

## Qué debes poder explicar con tus palabras
- Qué combina UNION: filas de resultados, no columnas de tablas relacionadas.
- Qué diferencia hay entre UNION y JOIN.
- Por qué importa el orden de las columnas.

## T08 - Tarea espejo

Combina los países de Customers y Employees. Ejecuta una versión con UNION y otra con UNION ALL, y compara la cantidad de filas.

### Archivos a modificar

- `tareas/T08_EJ08.sql` en el laboratorio de FASE 15, si decides utilizarlo como punto de partida opcional.

### Restricción

Ambos SELECT deben devolver una sola columna Country. No uses DISTINCT adicional.

### Pista

UNION ya elimina duplicados; UNION ALL no.

### Evidencia que debes mostrar

Dos consultas, sus cantidades de filas y una explicación de la diferencia.

### Cómo saber si está correcta

Las dos ramas tienen estructura compatible y se identifica correctamente el efecto sobre duplicados.

---
# 7. Cierre de sesión

Hoy construiste consultas que responden preguntas diferentes aunque todas parten de SELECT. Con INNER JOIN aprendiste a conservar únicamente coincidencias; con OUTER JOIN analizaste qué ocurre cuando una fila no encuentra pareja; con CROSS JOIN comprobaste que combinar todo con todo produce un crecimiento multiplicativo; y con SELF JOIN representaste la relación empleado-supervisor usando dos alias de la misma tabla. Después utilizaste CASE para convertir condiciones en una columna calculada y lo integraste con JOIN y SUM. Finalmente, con UNION y UNION ALL reuniste resultados compatibles y distinguías entre eliminar y conservar duplicados.

El punto clave para repasar es que **JOIN, CASE y UNION no son intercambiables**: cada uno resuelve una dimensión distinta del problema. Antes de ejecutar una consulta, formula qué filas deberían aparecer y por qué. Si el resultado sorprende, revisa primero la relación ON, el lado preservado, el orden de las condiciones CASE o la compatibilidad de las ramas UNION.

Para reforzar lo trabajado puedes revisar los materiales del curso en **Lideratec Academy**, el blog https://lideratecacademy.com/blog/ y el canal https://www.youtube.com/@LideratecAcademy.

Si deseas validar tu implementación o trabajar con una versión completa de los scripts, puedes utilizar el laboratorio ejecutable de la FASE 15 como recurso opcional.
