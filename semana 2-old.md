# PROGRAMACIÓN AVANZADA DE BASE DE DATOS
## Sesión 02 - Tipos de consultas (Parte 1)
### Guía del estudiante - SQL Server + Northwind

**Propósito de la práctica:** preparar la base de datos Northwind y utilizarla durante toda la sesión para ejecutar consultas simples, filtros, funciones de agregación, `GROUP BY`, `HAVING` y funciones de SQL Server.

**Punto de partida real:** tu instancia de SQL Server puede mostrar únicamente las bases del sistema: `master`, `model`, `msdb` y `tempdb`. Eso significa que el motor está disponible, pero **Northwind todavía no está instalada**.

**Resultado observable:** al finalizar podrás abrir SSMS, instalar Northwind de forma correcta, identificar sus tablas principales y ejecutar consultas alineadas con la Sesión 02.

---

# 1. Antes de comenzar

## 1.1 ¿Qué componentes vamos a utilizar?

| Componente | Para qué sirve | ¿Ya debe existir? |
|---|---|---|
| SQL Server | Motor que almacena y procesa la base de datos | Sí |
| SQL Server Management Studio (SSMS) | Interfaz para conectarnos, abrir scripts y ejecutar consultas | Sí |
| Northwind | Base de datos de ejemplo sobre clientes, pedidos, productos y empleados | La instalaremos |
| `instnwnd.sql` | Script oficial de Microsoft que crea los objetos y carga los datos de Northwind | Lo descargaremos |

> **Importante:** `master`, `model`, `msdb` y `tempdb` son bases del sistema. No las utilizaremos para desarrollar los ejercicios de la clase.

## 1.2 Recursos oficiales

- Microsoft Learn - bases de datos de ejemplo: https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/sql/linq/downloading-sample-databases
- Repositorio oficial Microsoft SQL Server Samples: https://github.com/microsoft/sql-server-samples/tree/master/samples/databases/northwind-pubs
- Script `instnwnd.sql`: https://github.com/microsoft/sql-server-samples/blob/master/samples/databases/northwind-pubs/instnwnd.sql

---

# 2. Preparación del entorno: instalar Northwind

## 2.1 ¿Por qué necesitamos Northwind?

La sesión utiliza tablas como:

- `Customers`
- `Employees`
- `Products`
- `Orders`
- `Order Details`
- `Categories`
- `Suppliers`

Si ejecutas una consulta como:

```sql
SELECT * FROM Customers;
```

sin haber instalado Northwind, SQL Server responderá que el objeto no existe.

La preparación del entorno forma parte de la práctica. Primero instalamos la base y luego usamos **la misma base durante toda la sesión**.

---

## 2.2 Paso 1 - Descargar el script oficial

1. Abre tu navegador.
2. Ingresa a:
   https://github.com/microsoft/sql-server-samples/tree/master/samples/databases/northwind-pubs
3. Ubica el archivo **`instnwnd.sql`**.
4. Ábrelo.
5. En GitHub utiliza la opción **Raw** para visualizar únicamente el contenido SQL.
6. Guarda el archivo en una carpeta fácil de ubicar, por ejemplo:

```text
Documentos\ISIL\Sesion02\instnwnd.sql
```

También puedes utilizar directamente la versión Raw:

https://raw.githubusercontent.com/microsoft/sql-server-samples/refs/heads/master/samples/databases/northwind-pubs/instnwnd.sql

### ¿Qué acabamos de descargar?

No descargamos un instalador `.exe`. Descargamos un **script Transact-SQL**. Ese script contiene la creación de tablas, relaciones, vistas y los datos de ejemplo de Northwind.

### Punto de control

Comprueba que el archivo termine en:

```text
instnwnd.sql
```

y no en `.txt`.

---

## 2.3 Paso 2 - Crear primero una base vacía llamada Northwind

La versión oficial actual de `instnwnd.sql` contiene una advertencia importante:

> El script no crea la base de datos; debe ejecutarse dentro de la base donde se crearán los objetos.

Por esa razón, primero crearemos la base vacía.

### Opción A - Desde la interfaz de SSMS

1. Vuelve a SSMS.
2. En **Explorador de objetos**, ubica la carpeta **Bases de datos**.
3. Clic derecho sobre **Bases de datos**.
4. Selecciona **Nueva base de datos...**.
5. En **Nombre de la base de datos** escribe:

```text
Northwind
```

6. No cambies rutas, tamaños ni opciones avanzadas para esta práctica.
7. Presiona **Aceptar**.
8. Si no la ves, clic derecho en **Bases de datos** -> **Actualizar**.

### Resultado esperado

Debes observar algo parecido a:

```text
Bases de datos
├── Bases de datos del sistema
└── Northwind
```

### Opción B - Con una consulta SQL

También puedes crearla desde una nueva consulta:

```sql
USE master;
GO

IF DB_ID(N'Northwind') IS NULL
    CREATE DATABASE Northwind;
GO
```

**¿Por qué usamos `master` aquí?**  
Porque estamos solicitando al servidor la creación de una nueva base. Después dejaremos de trabajar en `master` y cambiaremos a `Northwind`.

---

## 2.4 Paso 3 - Abrir `instnwnd.sql` en SSMS

1. En SSMS selecciona **Archivo -> Abrir -> Archivo...**
2. Busca `instnwnd.sql`.
3. Ábrelo.
4. SSMS mostrará el script en una ventana de consulta.

### Antes de ejecutar: revisión crítica

Observa el selector de base de datos de la barra superior de la consulta.

Debe indicar:

```text
Northwind
```

Si aparece `master`, **no ejecutes todavía**.

Cambia el selector a `Northwind`.

### ¿Por qué?

El script oficial crea los objetos en la base activa. Si lo ejecutaras accidentalmente sobre `master`, estarías colocando tablas de ejemplo dentro de una base del sistema.

---

## 2.5 Paso 4 - Ejecutar el script

1. Confirma que la base activa sea `Northwind`.
2. Ejecuta el script completo con el botón **Ejecutar** o con `F5`.
3. Espera a que termine.

El archivo es largo porque no solo crea tablas: también inserta los datos del ejemplo.

### Resultado esperado

La ejecución debe finalizar sin errores que impidan crear los objetos.

No cierres el archivo hasta completar la validación.

---

# 3. Validar que Northwind quedó instalada

Abre una **Nueva consulta** y ejecuta:

```sql
USE Northwind;
GO

SELECT DB_NAME() AS BaseActiva;
```

## ¿Qué debes observar?

La columna `BaseActiva` debe mostrar:

```text
Northwind
```

Ahora valida algunas tablas:

```sql
SELECT TOP (5) *
FROM dbo.Customers;

SELECT TOP (5) *
FROM dbo.Products;

SELECT TOP (5) *
FROM dbo.Orders;
```

Si aparecen registros, la instalación está lista.

También puedes verificar los objetos:

```sql
SELECT
    t.name AS Tabla
FROM sys.tables AS t
ORDER BY t.name;
```

Debes encontrar, entre otras:

```text
Categories
Customers
Employees
Order Details
Orders
Products
Shippers
Suppliers
```

---

# 4. Conocer la relación entre las tablas

Antes de consultar datos conviene comprender qué representa cada tabla.

| Tabla | Representa |
|---|---|
| `Customers` | Clientes |
| `Employees` | Empleados |
| `Products` | Productos |
| `Categories` | Categorías de productos |
| `Suppliers` | Proveedores |
| `Orders` | Cabecera de pedidos |
| `Order Details` | Detalle de productos vendidos en cada pedido |

## Crear el diagrama en SSMS

1. Expande `Northwind`.
2. Ubica **Diagramas de base de datos**.
3. Clic derecho -> **Nuevo diagrama de base de datos**.
4. Si SSMS solicita instalar objetos de soporte, confirma para este laboratorio.
5. Selecciona las tablas que quieras visualizar.
6. Agrega al menos:
   - `Customers`
   - `Orders`
   - `Order Details`
   - `Products`
   - `Categories`
   - `Suppliers`
   - `Employees`
7. Observa las relaciones.
8. Guarda el diagrama si deseas reutilizarlo.

### Lectura guiada

```text
Customers
   |
   v
Orders
   |
   v
Order Details
   |
   v
Products
   |
   +------> Categories
   |
   +------> Suppliers
```

Por ahora utilizaremos principalmente consultas sobre una tabla y agregaciones. Más adelante, estas relaciones permitirán combinar información de varias tablas.

---

# 5. Ejemplo 1 - Consultas simples con SELECT

## 1. ¿Qué aprenderemos?

A recuperar filas y columnas de una tabla.

## 2. Antes de comenzar

Abre una nueva consulta y ejecuta siempre:

```sql
USE Northwind;
GO
```

Eso evita ejecutar las sentencias sobre otra base.

## 3. Mostrar todos los clientes

```sql
SELECT *
FROM dbo.Customers;
```

### ¿Qué significa?

- `SELECT`: indica qué información queremos recuperar.
- `*`: solicita todas las columnas.
- `FROM`: indica de qué tabla provienen los datos.
- `dbo.Customers`: tabla `Customers` del esquema `dbo`.

## 4. Mostrar solo columnas necesarias

```sql
SELECT
    CustomerID,
    CompanyName,
    ContactName,
    Country
FROM dbo.Customers;
```

### ¿Por qué es mejor?

Porque una consulta real debería recuperar solo los datos que necesita.

## 5. Consultar empleados

```sql
SELECT
    FirstName,
    LastName,
    Title
FROM dbo.Employees;
```

## 6. Consultar productos

```sql
SELECT
    ProductName,
    UnitPrice,
    UnitsInStock
FROM dbo.Products;
```

### Modificación con el estudiante

Agrega la columna `UnitsOnOrder`.

Antes de ejecutar, predice: ¿aparecerá una columna adicional o nuevas filas?

---

# 6. Ejemplo 2 - Columnas calculadas y alias

Queremos estimar el valor almacenado de cada producto.

```sql
SELECT
    ProductName,
    UnitPrice,
    UnitsInStock,
    UnitPrice * UnitsInStock AS ValorInventario
FROM dbo.Products;
```

## ¿Qué ocurre?

SQL Server no agrega una columna física a `Products`. Calcula el valor únicamente para el resultado de la consulta.

`AS ValorInventario` crea un alias legible.

### Modificación

Cambia el alias:

```sql
UnitPrice * UnitsInStock AS TotalStock
```

Comprueba que cambió el nombre mostrado, no los datos almacenados.

---

# 7. Ejemplo 3 - Filtrar filas con WHERE

## Productos con precio mayor a 50

```sql
SELECT
    ProductName,
    UnitPrice
FROM dbo.Products
WHERE UnitPrice > 50;
```

## Clientes que no pertenecen a USA

```sql
SELECT
    CompanyName,
    Country
FROM dbo.Customers
WHERE Country <> N'USA';
```

## Pedidos posteriores al 1 de enero de 1997

```sql
SELECT
    OrderID,
    CustomerID,
    OrderDate
FROM dbo.Orders
WHERE OrderDate > '19970101';
```

### Idea clave

`WHERE` filtra **filas antes de mostrarlas**.

---

# 8. Ejemplo 4 - BETWEEN, IN y LIKE

## BETWEEN - rango de valores

```sql
SELECT
    ProductName,
    UnitsInStock
FROM dbo.Products
WHERE UnitsInStock BETWEEN 10 AND 20
ORDER BY ProductName;
```

`BETWEEN` incluye los dos límites.

## IN - conjunto de valores

```sql
SELECT
    CompanyName,
    Country
FROM dbo.Customers
WHERE Country IN (N'Germany', N'France', N'UK');
```

## LIKE - patrones de texto

Clientes cuyo código postal comienza con `1`:

```sql
SELECT
    CompanyName,
    PostalCode
FROM dbo.Customers
WHERE PostalCode LIKE N'1%';
```

Clientes cuya ciudad contiene `London`:

```sql
SELECT
    CompanyName,
    City
FROM dbo.Customers
WHERE City LIKE N'%London%'
ORDER BY CompanyName;
```

### ¿Qué significa `%`?

Representa cero o más caracteres.

```text
'1%'        empieza con 1
'%London%'  contiene London
'%a'        termina con a
```

---

# 9. Ejemplo 5 - Funciones de agregación

Las funciones de agregación resumen un conjunto de filas.

## Promedio de precios

```sql
SELECT AVG(UnitPrice) AS PrecioPromedio
FROM dbo.Products;
```

## Precio máximo y mínimo

```sql
SELECT
    MAX(UnitPrice) AS PrecioMaximo,
    MIN(UnitPrice) AS PrecioMinimo
FROM dbo.Products;
```

## Cantidad de productos

```sql
SELECT COUNT(*) AS CantidadProductos
FROM dbo.Products;
```

## Cantidad total vendida

```sql
SELECT SUM(Quantity) AS UnidadesVendidas
FROM dbo.[Order Details];
```

### Observación importante

`[Order Details]` utiliza corchetes porque su nombre contiene un espacio.

---

# 10. Ejemplo 6 - GROUP BY

Queremos saber cuántas unidades se han vendido de cada producto.

```sql
SELECT
    ProductID,
    SUM(Quantity) AS CantidadTotalVendida
FROM dbo.[Order Details]
GROUP BY ProductID
ORDER BY CantidadTotalVendida DESC;
```

## ¿Qué hace `GROUP BY`?

Reúne todas las filas que tienen el mismo `ProductID` y permite calcular un resumen por grupo.

### Flujo

```text
Order Details
      |
      v
agrupar por ProductID
      |
      v
SUM(Quantity)
      |
      v
una fila por producto
```

---

# 11. Ejemplo 7 - HAVING

Ahora queremos conservar únicamente los productos cuya cantidad total vendida sea superior a 100.

```sql
SELECT
    ProductID,
    SUM(Quantity) AS CantidadTotalVendida
FROM dbo.[Order Details]
GROUP BY ProductID
HAVING SUM(Quantity) > 100
ORDER BY CantidadTotalVendida DESC;
```

## Diferencia entre WHERE y HAVING

```text
WHERE  -> filtra filas individuales
GROUP BY -> forma grupos
HAVING -> filtra los grupos resultantes
```

### Error frecuente

Esto es incorrecto para filtrar una suma agrupada:

```sql
-- No usar para este objetivo:
-- WHERE SUM(Quantity) > 100
```

La agregación se evalúa después de formar grupos; por eso corresponde `HAVING`.

---

# 12. Ejemplo 8 - Funciones de cadena

```sql
SELECT CHARINDEX('an', 'banana') AS Posicion;

SELECT CONCAT('North', 'wind') AS Nombre;

SELECT LEN('Hello, world! ') AS Longitud;

SELECT LEFT('Northwind', 5) AS ParteIzquierda;

SELECT RIGHT('Northwind', 4) AS ParteDerecha;

SELECT REPLACE('The quick brown fox', 'brown', 'red') AS Reemplazo;

SELECT SUBSTRING('Northwind', 2, 5) AS Fragmento;

SELECT LOWER('NORTHWIND') AS Minusculas;

SELECT UPPER('northwind') AS Mayusculas;
```

## Aplicación con datos reales

```sql
SELECT
    CompanyName,
    UPPER(CompanyName) AS EmpresaMayusculas,
    LEN(CompanyName) AS LongitudNombre
FROM dbo.Customers;
```

---

# 13. Ejemplo 9 - Funciones numéricas

```sql
SELECT ABS(-10) AS Absoluto;
SELECT CEILING(3.14) AS RedondeoSuperior;
SELECT FLOOR(3.14) AS RedondeoInferior;
SELECT POWER(2, 3) AS Potencia;
SELECT ROUND(3.14159, 2) AS Redondeado;
SELECT SIGN(-10) AS Signo;
```

## Aplicación al precio con descuento

```sql
SELECT
    ProductName,
    UnitPrice,
    ROUND(UnitPrice * 0.90, 2) AS PrecioConDescuento
FROM dbo.Products;
```

---

# 14. Ejemplo 10 - Funciones de fecha

```sql
SELECT DATEADD(day, 7, '20230420') AS FechaMasSieteDias;

SELECT DATEDIFF(day, '20230420', '20230427') AS DiferenciaDias;

SELECT DATEPART(year, '20230427') AS Anio;

SELECT GETDATE() AS FechaHoraActual;
```

## Aplicación con pedidos

```sql
SELECT
    OrderID,
    OrderDate,
    DATEPART(year, OrderDate) AS AnioPedido
FROM dbo.Orders;
```

---

# 15. Trabajo práctico de la sesión

Resuelve primero sin revisar la solución docente.

## Nivel A - Consultas directas

1. Mostrar todos los datos de `Customers`.
2. Mostrar nombres y apellidos de `Employees`.
3. Mostrar nombre, precio y cantidad en stock de `Products`.
4. Mostrar clientes cuyo código postal comience con `1`.
5. Mostrar clientes que no estén en `USA`.
6. Mostrar pedidos posteriores al 1 de enero de 1997.
7. Mostrar productos con stock entre 10 y 20, ordenados por nombre.
8. Mostrar clientes cuya ciudad contenga `London`.

## Nivel B - Cálculos y agregaciones

9. Mostrar cada producto y su precio con 10% de descuento usando `ROUND`.
10. Mostrar cantidad total vendida por `ProductID`, de mayor a menor.
11. Mostrar solo productos cuya cantidad total vendida sea superior a 100.
12. Mostrar para cada empleado la cantidad de pedidos registrados, conservando solo quienes superen 30 pedidos.

## Nivel C - Ejercicios que usan relaciones de Northwind

Algunos enunciados finales de la presentación requieren recuperar información que está distribuida entre varias tablas, por ejemplo nombre de producto + proveedor o cliente + total comprado.

Estos ejercicios sirven como **puente de integración**. La sesión actual se concentra en `SELECT`, filtros, agregaciones y funciones. El docente decidirá si los resuelve al cierre o los reserva para la sesión de consultas multitabla.

---

# 16. Evidencias de la práctica

Al finalizar conserva:

1. Captura de `Northwind` visible en SSMS.
2. Captura o consulta que muestre tablas instaladas.
3. Archivo `.sql` con tus consultas.
4. Resultado de al menos:
   - una consulta simple;
   - una consulta con `WHERE`;
   - una con `BETWEEN`, `IN` o `LIKE`;
   - una agregación;
   - una consulta `GROUP BY`;
   - una consulta `HAVING`;
   - una función de cadena, numérica o fecha.

---

# 17. Tabla de errores frecuentes

| Situación | Causa probable | Qué revisar |
|---|---|---|
| `Invalid object name 'Customers'` | La base activa no es Northwind o Northwind no está instalada | Ejecuta `USE Northwind;` y valida las tablas |
| El script crea objetos donde no corresponde | Se ejecutó con `master` seleccionado | Crear/seleccionar `Northwind` antes de ejecutar |
| `Database 'Northwind' already exists` | Intentaste crearla de nuevo | No la recrees; úsala si está vacía o valida su contenido |
| Error con `Order Details` | El nombre tiene espacio | Utiliza `[Order Details]` |
| `HAVING` produce error | Se usó sin agrupación adecuada | Revisa `GROUP BY` y la función agregada |
| `LIKE` no devuelve lo esperado | El comodín está mal colocado | Revisa `%` al inicio/final según el patrón |

---

# 18. Checklist final

- [ ] Northwind aparece en SSMS.
- [ ] `SELECT DB_NAME()` devuelve `Northwind`.
- [ ] Existen `Customers`, `Employees`, `Products`, `Orders` y `Order Details`.
- [ ] Puedo ejecutar `SELECT`.
- [ ] Puedo filtrar con `WHERE`.
- [ ] Puedo usar `BETWEEN`, `IN` y `LIKE`.
- [ ] Comprendo `AVG`, `MAX`, `MIN`, `SUM` y `COUNT`.
- [ ] Puedo agrupar con `GROUP BY`.
- [ ] Puedo filtrar grupos con `HAVING`.
- [ ] Puedo aplicar funciones de cadena, numéricas y fecha.

---

# 19. Mapa final de la sesión

```text
Instalar Northwind
       |
       v
SELECT
       |
       v
WHERE
       |
       +--> BETWEEN
       +--> IN
       +--> LIKE
       |
       v
Agregaciones
       |
       v
GROUP BY
       |
       v
HAVING
       |
       v
Funciones SQL Server
       |
       v
Trabajo práctico
```

## Cierre

En esta sesión no solo ejecutaste consultas: preparaste el entorno, comprobaste la base activa, interpretaste resultados y utilizaste Northwind como escenario común para toda la práctica.

**Blog:** https://lideratecacademy.com/  
**Canal:** https://www.youtube.com/@LideratecAcademy
