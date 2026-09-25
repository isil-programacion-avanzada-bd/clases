---
course_id: PABD-ISIL
session_id: S04
module_id: UA1
course_name: Programación Avanzada de Base de Datos
session_topic: Subconsultas
source_origin: PPT
status: draft
estado_canon: PROVISIONAL
---


# FASE 13 - Guia docente

## Proposito docente

Conducir una clase-laboratorio de subconsultas en SQL Server donde cada ejemplo siga la secuencia: objetivo -> construccion -> explicacion -> prediccion -> ejecucion -> interpretacion -> variacion -> error -> correccion -> tarea espejo.

La clase usa Northwind. Las operaciones DML se ejecutan sobre tablas temporales para conservar el dataset. No se debe desactivar integridad referencial ni modificar tablas reales para hacer coincidir una captura.

## Apertura sugerida

Explica al grupo que el reto de las subconsultas no consiste en poner un `SELECT` dentro de parentesis. La decision importante es identificar que tipo de resultado produce la consulta interna y como lo va a consumir la consulta externa. En cada ejemplo, detente antes de ejecutar y pide una prediccion. No aceptes “debe funcionar” como prediccion: el estudiante debe decir que filas espera conservar, excluir o modificar y por que.

## Validacion de entorno

Ejecutar primero:

```sql
USE Northwind;
GO
SELECT DB_NAME() AS BaseActual;
SELECT SERVERPROPERTY('ProductVersion') AS ProductVersion,
       SERVERPROPERTY('ProductMajorVersion') AS ProductMajorVersion,
       SERVERPROPERTY('Edition') AS Edition;
```

Si Northwind no existe, no improvisar otra base en medio de la sesion: recuperar el entorno de las clases anteriores. Si el motor es anterior a SQL Server 2022, EJ06 debe usar la rama compatible incluida.


# EJ01 - Subconsulta escalar: productos por encima del promedio

## Objetivo docente

Construir una consulta que use un valor agregado calculado por una subconsulta y combinarlo con unidades vendidas por producto.

## Que debe estar visible antes de comenzar

Northwind está seleccionada y las tablas Products y [Order Details] responden.

## Que decir antes de escribir

Mostrar primero el promedio por separado. Preguntar por qué no conviene escribir ese valor como constante antes de revelar la consulta completa.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
USE Northwind;
GO

SELECT AVG(UnitPrice) AS PrecioPromedio
FROM Products;
```

### Mientras escribe, explicar

La subconsulta que luego irá dentro de WHERE produce un único valor: el promedio de UnitPrice. Ese resultado puede compararse con el precio de cada producto. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT ProductID,
       SUM(Quantity) AS UnitsSold
FROM [Order Details]
GROUP BY ProductID;
```

### Mientras escribe, explicar

Esta consulta resume el detalle de pedidos por ProductID. La usamos como tabla derivada para disponer de una cantidad total vendida por producto. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

SELECT p.ProductName,
       p.UnitPrice,
       od.UnitsSold
FROM Products AS p
JOIN (
    SELECT ProductID,
           SUM(Quantity) AS UnitsSold
    FROM [Order Details]
    GROUP BY ProductID
) AS od
    ON p.ProductID = od.ProductID
WHERE p.UnitPrice > (
    SELECT AVG(UnitPrice)
    FROM Products
)
ORDER BY p.UnitPrice DESC;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- La subconsulta de AVG devolverá un solo valor.
- Solo aparecerán productos cuyo UnitPrice sea mayor que ese promedio.
- UnitsSold proviene del resumen de [Order Details], no de Products.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Una cuadrícula con ProductName, UnitPrice y UnitsSold. Todas las filas deben tener UnitPrice mayor que el promedio calculado por la consulta interna.

## Interpretacion frente al grupo

El filtro depende de un dato que no estaba escrito como constante: se calcula primero como resultado de otra consulta. La tabla derivada agrega otra dimensión práctica: total vendido por producto.

## Variante A

Cambiar `>` por `<` y anticipar cómo cambia el conjunto.

## Error controlado

Sustituye temporalmente `AVG(UnitPrice)` por `SELECT UnitPrice FROM Products` sin agregación.

## Explicacion del error

El operador `>` espera un valor escalar, pero esa subconsulta devuelve varias filas.

## Correccion

Restaurar una subconsulta que garantice una sola fila, por ejemplo `SELECT AVG(UnitPrice) FROM Products`.

# T01 - Tarea espejo

## Enunciado

Obtén ProductName y UnitPrice de los productos activos (`Discontinued = 0`) cuyo precio sea menor que el promedio de todos los productos.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ01.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

SELECT p.ProductName,
       p.UnitPrice
FROM Products AS p
WHERE p.Discontinued = 0
  AND p.UnitPrice < (
      SELECT AVG(UnitPrice)
      FROM Products
  )
ORDER BY p.UnitPrice;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ01, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La consulta contiene una única subconsulta con AVG, filtra Discontinued = 0 y usa `UnitPrice < (...)`. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

Ahora que una subconsulta produce un único valor, pasamos a subconsultas que devuelven un conjunto y que se consumen con IN / NOT IN.


# EJ02 - Subconsulta de varias filas: IN y NOT IN

## Objetivo docente

Filtrar clientes a partir de un conjunto de ProductID y contrastar inclusión frente a exclusión.

## Que debe estar visible antes de comenzar

EJ01 puede estar cerrado; este ejemplo es independiente y usa Customers, Orders, [Order Details] y Products.

## Que decir antes de escribir

Hacer que el grupo identifique el tipo de salida de la subconsulta: no es un valor escalar, sino una lista de identificadores.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
SELECT ProductID
FROM Products
WHERE UnitPrice > 20;
```

### Mientras escribe, explicar

Esta subconsulta devuelve varios ProductID. Por eso no se compara con `=`; se usa como conjunto para `IN`. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT DISTINCT c.CompanyName,
       c.ContactName
FROM Customers AS c
JOIN Orders AS o
    ON c.CustomerID = o.CustomerID
JOIN [Order Details] AS od
    ON o.OrderID = od.OrderID
WHERE od.ProductID IN (
    SELECT ProductID
    FROM Products
    WHERE UnitPrice > 20
);
```

### Mientras escribe, explicar

La consulta externa recorre pedidos y clientes; el IN pregunta si cada ProductID pertenece al conjunto producido por la subconsulta. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

-- Parte A: clientes que SI tienen pedidos con productos > 20
SELECT DISTINCT c.CustomerID,
       c.CompanyName,
       c.ContactName
FROM Customers AS c
JOIN Orders AS o
    ON c.CustomerID = o.CustomerID
JOIN [Order Details] AS od
    ON o.OrderID = od.OrderID
WHERE od.ProductID IN (
    SELECT ProductID
    FROM Products
    WHERE UnitPrice > 20
)
ORDER BY c.CompanyName;

-- Parte B: clientes que NO tienen pedidos con productos > 20
SELECT c.CustomerID,
       c.CompanyName,
       c.ContactName
FROM Customers AS c
WHERE c.CustomerID NOT IN (
    SELECT DISTINCT o.CustomerID
    FROM Orders AS o
    JOIN [Order Details] AS od
        ON o.OrderID = od.OrderID
    WHERE od.ProductID IN (
        SELECT ProductID
        FROM Products
        WHERE UnitPrice > 20
    )
)
ORDER BY c.CompanyName;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- La primera consulta debe devolver clientes con al menos una coincidencia.
- La segunda debe excluir a todos esos clientes.
- El DISTINCT de la primera evita repetir clientes por múltiples pedidos/detalles.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Dos resultados: inclusión y exclusión. Ningún cliente de la segunda lista debería cumplir la condición usada por la primera.

## Interpretacion frente al grupo

IN evalúa pertenencia a un conjunto de valores. NOT IN invierte esa pertenencia, por lo que conviene vigilar si la subconsulta puede devolver NULL; en este caso se selecciona CustomerID proveniente de Orders.

## Variante A

Cambiar el umbral de 20 a 50 solo después de predecir si la primera lista crecerá o disminuirá.

## Error controlado

Quita `DISTINCT` de la primera consulta y compara la cantidad de filas.

## Explicacion del error

Un cliente puede aparecer en varios pedidos o detalles que cumplan la condición; sin DISTINCT, el resultado repite entidades.

## Correccion

Restaurar DISTINCT cuando el objetivo sea una lista única de clientes.

# T02 - Tarea espejo

## Enunciado

Obtén los clientes que NO han realizado pedidos que incluyan productos descontinuados.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ02.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

SELECT c.CustomerID,
       c.CompanyName,
       c.ContactName
FROM Customers AS c
WHERE c.CustomerID NOT IN (
    SELECT DISTINCT o.CustomerID
    FROM Orders AS o
    JOIN [Order Details] AS od
        ON o.OrderID = od.OrderID
    JOIN Products AS p
        ON od.ProductID = p.ProductID
    WHERE p.Discontinued = 1
)
ORDER BY c.CompanyName;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ02, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La subconsulta devuelve CustomerID y la consulta externa usa `CustomerID NOT IN (...)`. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

Después de trabajar conjuntos, volvemos a una subconsulta de una sola fila para ver por qué `=` sí es apropiado cuando el resultado es escalar.


# EJ03 - Operador de comparación con subconsulta de una fila

## Objetivo docente

Usar el resultado de una subconsulta de una fila como valor de comparación de la consulta principal.

## Que debe estar visible antes de comenzar

Customers contiene el cliente ALFKI utilizado en el material de la sesión.

## Que decir antes de escribir

Preguntar: ¿qué ocurriría si la consulta interior devolviera dos ciudades? La respuesta prepara el error controlado del ejemplo.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
SELECT City
FROM Customers
WHERE CustomerID = 'ALFKI';
```

### Mientras escribe, explicar

El filtro por clave de cliente pretende producir una única ciudad. Esa ciudad será el valor de comparación de la consulta exterior. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT CustomerID,
       CompanyName,
       City
FROM Customers
WHERE City = (
    SELECT City
    FROM Customers
    WHERE CustomerID = 'ALFKI'
);
```

### Mientras escribe, explicar

El operador `=` es válido porque la subconsulta se diseña para devolver una sola fila y una sola columna. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

SELECT CustomerID,
       CompanyName,
       ContactName,
       City
FROM Customers
WHERE City = (
    SELECT City
    FROM Customers
    WHERE CustomerID = 'ALFKI'
)
ORDER BY CompanyName;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- Primero se obtiene la ciudad de ALFKI.
- La consulta externa conservará clientes cuya City sea exactamente esa ciudad.
- En la versión de Northwind mostrada en la sesión, ALFKI aparece en Berlin.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Una lista de clientes de la misma ciudad que ALFKI. El valor comparado no se escribe manualmente: proviene de la subconsulta.

## Interpretacion frente al grupo

Este patrón es útil cuando la condición depende de un dato localizado en otra fila. El punto crítico es garantizar que la subconsulta usada con `=` no devuelva múltiples filas.

## Variante A

Cambiar `=` por `<>` y predecir qué conjunto se obtiene.

## Error controlado

Elimina el filtro `WHERE CustomerID = 'ALFKI'` dentro de la subconsulta.

## Explicacion del error

La subconsulta pasa a devolver muchas ciudades; el operador escalar `=` no acepta múltiples filas.

## Correccion

Restaurar un criterio que garantice una sola fila o cambiar el patrón a un operador de conjunto cuando el problema realmente requiera varias filas.

# T03 - Tarea espejo

## Enunciado

Selecciona los clientes que están en la misma ciudad que el cliente `ANATR`.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ03.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

SELECT CustomerID,
       CompanyName,
       ContactName,
       City
FROM Customers
WHERE City = (
    SELECT City
    FROM Customers
    WHERE CustomerID = 'ANATR'
)
ORDER BY CompanyName;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ03, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La ciudad se obtiene por subconsulta y no aparece escrita como constante. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

Ahora compararemos un valor escalar con un conjunto completo usando ANY y SOME.


# EJ04 - ANY y SOME: verdadero si alguna comparación se cumple

## Objetivo docente

Comparar un precio con el conjunto de precios de un proveedor usando ANY y verificar la equivalencia conceptual de SOME.

## Que debe estar visible antes de comenzar

Products contiene SupplierID y UnitPrice.

## Que decir antes de escribir

Mostrar dos o tres precios del proveedor de referencia y pedir un ejemplo numérico que cumpla “mayor que alguno” pero no “mayor que todos”.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
SELECT ProductName,
       UnitPrice
FROM Products
WHERE SupplierID = 2
ORDER BY UnitPrice;
```

### Mientras escribe, explicar

Observamos primero el conjunto de precios contra el que se hará la comparación. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT ProductID,
       ProductName,
       UnitPrice,
       SupplierID
FROM Products
WHERE UnitPrice > ANY (
    SELECT UnitPrice
    FROM Products
    WHERE SupplierID = 2
)
  AND SupplierID <> 2;
```

### Mientras escribe, explicar

`> ANY` resulta verdadero si el precio del producto es mayor que al menos uno de los precios devueltos por la subconsulta. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

-- ANY
SELECT ProductID,
       ProductName,
       UnitPrice,
       SupplierID
FROM Products
WHERE UnitPrice > ANY (
    SELECT UnitPrice
    FROM Products
    WHERE SupplierID = 2
)
  AND SupplierID <> 2
ORDER BY UnitPrice;

-- SOME es equivalente a ANY
SELECT ProductID,
       ProductName,
       UnitPrice,
       SupplierID
FROM Products
WHERE UnitPrice > SOME (
    SELECT UnitPrice
    FROM Products
    WHERE SupplierID = 2
)
  AND SupplierID <> 2
ORDER BY UnitPrice;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- ANY y SOME deben devolver el mismo conjunto para la misma comparación.
- No es necesario superar todos los precios del proveedor 2; basta con superar al menos uno.
- La segunda condición evita mezclar productos del proveedor usado como referencia.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Dos conjuntos equivalentes de productos, uno con ANY y otro con SOME.

## Interpretacion frente al grupo

ANY y SOME son dos formas de expresar la misma condición en SQL Server: la comparación debe ser verdadera para al menos un valor de la subconsulta.

## Variante A

Cambiar `>` por `<` y razonar qué significa “menor que al menos uno”.

## Error controlado

Sustituye la subconsulta por `SELECT ProductID ...` manteniendo la comparación con UnitPrice.

## Explicacion del error

La columna de la subconsulta debe ser comparable con la expresión escalar; ProductID no representa el mismo dominio que UnitPrice.

## Correccion

Devolver UnitPrice desde la subconsulta cuando se compara contra UnitPrice.

# T04 - Tarea espejo

## Enunciado

Usa SOME para encontrar productos de otros proveedores cuyo UnitPrice sea mayor que al menos uno de los precios del proveedor con SupplierID = 3.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ04.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

SELECT ProductID,
       ProductName,
       UnitPrice,
       SupplierID
FROM Products
WHERE UnitPrice > SOME (
    SELECT UnitPrice
    FROM Products
    WHERE SupplierID = 3
)
  AND SupplierID <> 3
ORDER BY UnitPrice;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ04, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La consulta usa `UnitPrice > SOME (SELECT UnitPrice ... SupplierID = 3)`. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

El siguiente paso cambia el cuantificador: ALL exige que la comparación se cumpla frente a cada valor del conjunto.


# EJ05 - ALL: verdadero si todas las comparaciones se cumplen

## Objetivo docente

Diferenciar ANY/SOME de ALL al comparar un valor con todos los elementos de un conjunto.

## Que debe estar visible antes de comenzar

Se reutiliza Products sin depender del estado de EJ04.

## Que decir antes de escribir

Hacer explícita la comparación lógica: ANY/SOME = al menos uno; ALL = cada valor. No reducirlo todavía a MIN/MAX, porque el objetivo es dominar el operador.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
SELECT MIN(UnitPrice) AS Minimo,
       MAX(UnitPrice) AS Maximo
FROM Products
WHERE SupplierID = 2;
```

### Mientras escribe, explicar

Los extremos ayudan a razonar la condición. Para cumplir `> ALL`, un precio debe superar incluso el máximo del conjunto. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT ProductID,
       ProductName,
       UnitPrice,
       SupplierID
FROM Products
WHERE UnitPrice > ALL (
    SELECT UnitPrice
    FROM Products
    WHERE SupplierID = 2
)
  AND SupplierID <> 2;
```

### Mientras escribe, explicar

ALL exige que cada comparación individual sea verdadera. En este caso, el precio debe ser mayor que todos los precios devueltos. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

SELECT ProductID,
       ProductName,
       UnitPrice,
       SupplierID
FROM Products
WHERE UnitPrice > ALL (
    SELECT UnitPrice
    FROM Products
    WHERE SupplierID = 2
)
  AND SupplierID <> 2
ORDER BY UnitPrice;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- El conjunto de `> ALL` será igual o más restrictivo que el de `> ANY` para el mismo proveedor.
- Cada fila devuelta debe superar el mayor precio del conjunto de referencia.
- Si la subconsulta no devolviera filas, la lógica requiere un análisis especial; aquí trabajamos con un proveedor existente.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Productos cuyo UnitPrice es mayor que cada precio del proveedor 2.

## Interpretacion frente al grupo

ALL expresa una condición universal sobre el conjunto. La diferencia con ANY/SOME no es de sintaxis superficial: cambia la regla lógica que debe cumplirse.

## Variante A

Ejecutar la versión ANY del ejemplo anterior y comparar visualmente la cantidad de filas.

## Error controlado

Cambiar ALL por ANY sin cambiar la explicación del resultado.

## Explicacion del error

La consulta seguirá siendo válida, pero la semántica cambia de “todos” a “al menos uno”; interpretar ambas como equivalentes sería un error conceptual.

## Correccion

Restaurar ALL y validar contra el MAX del conjunto de referencia.

# T05 - Tarea espejo

## Enunciado

Encuentra productos de otros proveedores cuyo UnitPrice sea mayor que TODOS los precios del proveedor con SupplierID = 3.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ05.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

SELECT ProductID,
       ProductName,
       UnitPrice,
       SupplierID
FROM Products
WHERE UnitPrice > ALL (
    SELECT UnitPrice
    FROM Products
    WHERE SupplierID = 3
)
  AND SupplierID <> 3
ORDER BY UnitPrice;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ05, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La condición usa ALL y excluye SupplierID = 3 de la consulta externa. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

Pasamos a una comparación distinta: igualdad o diferencia que produce TRUE/FALSE incluso cuando interviene NULL.


# EJ06 - Comparación segura frente a NULL con IS [NOT] DISTINCT FROM

## Objetivo docente

Comparar valores con semántica determinista frente a NULL y mantener compatibilidad con motores anteriores a SQL Server 2022.

## Que debe estar visible antes de comenzar

El ejemplo detecta la versión del motor. La sintaxis nativa requiere SQL Server 2022 (16.x) o posterior.

## Que decir antes de escribir

Separar dos ideas: la versión de SSMS no determina la versión del motor; y la sintaxis IS [NOT] DISTINCT FROM llega a SQL Server 2022.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
SELECT SERVERPROPERTY('ProductVersion') AS ProductVersion,
       SERVERPROPERTY('ProductMajorVersion') AS ProductMajorVersion,
       SERVERPROPERTY('Edition') AS Edition;
```

### Mientras escribe, explicar

Antes de usar la sintaxis nativa, comprobamos la versión del motor. El cliente SSMS y el motor SQL Server son componentes distintos. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
DECLARE @categoria nvarchar(15) = N'Beverages';

SELECT p.ProductName,
       c.CategoryName
FROM Products AS p
LEFT JOIN Categories AS c
    ON p.CategoryID = c.CategoryID
WHERE c.CategoryName = @categoria
   OR (c.CategoryName IS NULL AND @categoria IS NULL);
```

### Mientras escribe, explicar

Esta es la forma compatible con versiones anteriores para expresar igualdad considerando el caso NULL = NULL como coincidencia lógica. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

DECLARE @categoria nvarchar(15) = N'Beverages';
DECLARE @major int = TRY_CONVERT(int, SERVERPROPERTY('ProductMajorVersion'));

IF @major >= 16
BEGIN
    DECLARE @sql nvarchar(max) = N'
        SELECT p.ProductName,
               c.CategoryName
        FROM Products AS p
        LEFT JOIN Categories AS c
            ON p.CategoryID = c.CategoryID
        WHERE c.CategoryName IS NOT DISTINCT FROM @categoria
        ORDER BY p.ProductName;';

    EXEC sys.sp_executesql
        @sql,
        N'@categoria nvarchar(15)',
        @categoria = @categoria;
END
ELSE
BEGIN
    SELECT p.ProductName,
           c.CategoryName
    FROM Products AS p
    LEFT JOIN Categories AS c
        ON p.CategoryID = c.CategoryID
    WHERE c.CategoryName = @categoria
       OR (c.CategoryName IS NULL AND @categoria IS NULL)
    ORDER BY p.ProductName;
END;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- En SQL Server 2022+ se ejecutará la rama con `IS NOT DISTINCT FROM`.
- En versiones anteriores se ejecutará una expresión equivalente para igualdad con NULL.
- Con @categoria = Beverages se esperan productos cuya categoría sea Beverages.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Una lista de productos de la categoría indicada. La ruta de ejecución depende de ProductMajorVersion, pero el objetivo lógico es el mismo.

## Interpretacion frente al grupo

`IS NOT DISTINCT FROM` produce una comparación de igualdad que siempre resuelve a verdadero o falso, incluso con NULL. La rama alternativa permite mantener la práctica en motores anteriores.

## Variante A

Asignar `NULL` a @categoria y observar cómo la comparación trata filas sin categoría.

## Error controlado

Escribir directamente `IS NOT DISTINCT FROM` en un motor anterior a SQL Server 2022.

## Explicacion del error

La sintaxis nativa no está disponible antes de SQL Server 2022 (16.x).

## Correccion

Usar el control de versión y la expresión compatible incluida en el ejemplo.

# T06 - Tarea espejo

## Enunciado

Obtén los productos cuya CategoryName sea distinta de `Beverages`, tratando NULL como un valor diferente de `Beverages`.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ06.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

DECLARE @categoria nvarchar(15) = N'Beverages';
DECLARE @major int = TRY_CONVERT(int, SERVERPROPERTY('ProductMajorVersion'));

IF @major >= 16
BEGIN
    DECLARE @sql nvarchar(max) = N'
        SELECT p.ProductName,
               c.CategoryName
        FROM Products AS p
        LEFT JOIN Categories AS c
            ON p.CategoryID = c.CategoryID
        WHERE c.CategoryName IS DISTINCT FROM @categoria
        ORDER BY p.ProductName;';

    EXEC sys.sp_executesql
        @sql,
        N'@categoria nvarchar(15)',
        @categoria = @categoria;
END
ELSE
BEGIN
    SELECT p.ProductName,
           c.CategoryName
    FROM Products AS p
    LEFT JOIN Categories AS c
        ON p.CategoryID = c.CategoryID
    WHERE c.CategoryName <> @categoria
       OR (c.CategoryName IS NULL AND @categoria IS NOT NULL)
       OR (c.CategoryName IS NOT NULL AND @categoria IS NULL)
    ORDER BY p.ProductName;
END;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ06, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La solución contiene una ruta nativa para 16+ y una ruta compatible para versiones anteriores. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

Los siguientes tres ejemplos mantienen el tema de subconsultas pero introducen DML. Para proteger Northwind, todas las modificaciones se harán sobre tablas temporales.


# EJ07 - DELETE con subconsulta en una copia temporal

## Objetivo docente

Usar una subconsulta para decidir qué filas eliminar sin modificar la tabla Orders real.

## Que debe estar visible antes de comenzar

La sesión está conectada a Northwind. El ejemplo crea #OrdersLab y solo borra dentro de esa tabla temporal.

## Que decir antes de escribir

Recordar la nota del material sobre integridad referencial. La decisión didáctica es no desactivar controles: aislamos la modificación en una tabla temporal.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
IF OBJECT_ID('tempdb..#OrdersLab') IS NOT NULL
    DROP TABLE #OrdersLab;

SELECT OrderID,
       CustomerID,
       EmployeeID,
       OrderDate,
       ShippedDate,
       ShipVia
INTO #OrdersLab
FROM Orders;
```

### Mientras escribe, explicar

SELECT INTO crea una copia temporal de las columnas necesarias. La tabla desaparece al cerrar la sesión y no tiene las claves foráneas de Orders. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT DISTINCT od.OrderID
FROM [Order Details] AS od
JOIN Products AS p
    ON od.ProductID = p.ProductID
WHERE p.UnitPrice > 50;
```

### Mientras escribe, explicar

Antes de borrar, observamos exactamente qué OrderID seleccionará la subconsulta. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

IF OBJECT_ID('tempdb..#OrdersLab') IS NOT NULL
    DROP TABLE #OrdersLab;

SELECT OrderID,
       CustomerID,
       EmployeeID,
       OrderDate,
       ShippedDate,
       ShipVia
INTO #OrdersLab
FROM Orders;

SELECT COUNT(*) AS FilasAntes
FROM #OrdersLab;

DELETE FROM #OrdersLab
WHERE OrderID IN (
    SELECT DISTINCT od.OrderID
    FROM [Order Details] AS od
    JOIN Products AS p
        ON od.ProductID = p.ProductID
    WHERE p.UnitPrice > 50
);

SELECT @@ROWCOUNT AS FilasEliminadas;

SELECT COUNT(*) AS FilasDespues
FROM #OrdersLab;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- FilasDespues debe ser menor o igual que FilasAntes.
- Orders real no cambia.
- La subconsulta determina los OrderID afectados.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Tres evidencias: conteo antes, número de filas eliminadas y conteo después. La operación ocurre solo en #OrdersLab.

## Interpretacion frente al grupo

El patrón DELETE + subconsulta permite decidir filas según datos de otras tablas. La tabla temporal elimina el riesgo de romper integridad referencial de Northwind durante la práctica.

## Variante A

Ejecutar primero la subconsulta sola y contar cuántos OrderID distintos devuelve.

## Error controlado

Intentar cambiar `DELETE FROM #OrdersLab` por `DELETE FROM Orders` en la base real.

## Explicacion del error

Orders participa en relaciones; el material advierte que una FK puede impedir el borrado y, además, sería una modificación destructiva del dataset de clase.

## Correccion

Mantener la eliminación sobre #OrdersLab. No desactivar claves ni borrar datos reales para completar el ejercicio.

# T07 - Tarea espejo

## Enunciado

En una nueva #OrdersLab, elimina las órdenes que contengan al menos un producto descontinuado (`Discontinued = 1`).

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ07.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

IF OBJECT_ID('tempdb..#OrdersLab') IS NOT NULL
    DROP TABLE #OrdersLab;

SELECT OrderID,
       CustomerID,
       EmployeeID,
       OrderDate,
       ShippedDate,
       ShipVia
INTO #OrdersLab
FROM Orders;

SELECT COUNT(*) AS FilasAntes
FROM #OrdersLab;

DELETE FROM #OrdersLab
WHERE OrderID IN (
    SELECT DISTINCT od.OrderID
    FROM [Order Details] AS od
    JOIN Products AS p
        ON od.ProductID = p.ProductID
    WHERE p.Discontinued = 1
);

SELECT @@ROWCOUNT AS FilasEliminadas;
SELECT COUNT(*) AS FilasDespues FROM #OrdersLab;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ07, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

Solo se ejecuta DELETE sobre #OrdersLab y la condición de la subconsulta usa Discontinued = 1. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

El siguiente DML inserta filas producidas por una consulta, también dentro de una tabla temporal.


# EJ08 - INSERT con subconsulta para seleccionar una categoría

## Objetivo docente

Insertar varias filas obtenidas de Products usando una subconsulta para resolver CategoryID.

## Que debe estar visible antes de comenzar

Se crea #OrderDetailsLab con la misma forma básica de [Order Details], pero vacía y sin modificar datos reales.

## Que decir antes de escribir

Antes del INSERT, pedir al grupo que cuente mentalmente cuántas filas se insertarían si la consulta SELECT devuelve N productos.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
IF OBJECT_ID('tempdb..#OrderDetailsLab') IS NOT NULL
    DROP TABLE #OrderDetailsLab;

SELECT TOP (0)
       OrderID,
       ProductID,
       UnitPrice,
       Quantity,
       Discount
INTO #OrderDetailsLab
FROM [Order Details];
```

### Mientras escribe, explicar

TOP (0) copia la estructura de las columnas seleccionadas sin copiar filas. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT CategoryID
FROM Categories
WHERE CategoryName = 'Beverages';
```

### Mientras escribe, explicar

Esta subconsulta devuelve el identificador que se usará para filtrar Products. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

IF OBJECT_ID('tempdb..#OrderDetailsLab') IS NOT NULL
    DROP TABLE #OrderDetailsLab;

SELECT TOP (0)
       OrderID,
       ProductID,
       UnitPrice,
       Quantity,
       Discount
INTO #OrderDetailsLab
FROM [Order Details];

INSERT INTO #OrderDetailsLab
    (OrderID, ProductID, UnitPrice, Quantity, Discount)
SELECT 22077,
       ProductID,
       UnitPrice,
       10,
       0.05
FROM Products
WHERE CategoryID = (
    SELECT CategoryID
    FROM Categories
    WHERE CategoryName = 'Beverages'
);

SELECT @@ROWCOUNT AS FilasInsertadas;

SELECT *
FROM #OrderDetailsLab
ORDER BY ProductID;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- Se insertará una fila por cada producto de la categoría Beverages.
- Quantity será 10 en todas las filas insertadas.
- La tabla real [Order Details] no se modifica.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Una tabla temporal con filas de productos de Beverages y los valores fijos definidos para OrderID, Quantity y Discount.

## Interpretacion frente al grupo

INSERT ... SELECT permite insertar tantas filas como produzca la consulta. La subconsulta escalar resuelve el CategoryID a partir del nombre de categoría.

## Variante A

Cambiar Quantity de 10 a 5 y predecir qué columnas cambian.

## Error controlado

Cambiar el destino a `[Order Details]` real manteniendo OrderID 22077.

## Explicacion del error

El valor de OrderID podría no existir o podría chocar con restricciones. Además, modificaríamos el dataset de la clase.

## Correccion

Mantener #OrderDetailsLab como destino de la práctica.

# T08 - Tarea espejo

## Enunciado

Crea #OrderDetailsLab e inserta 5 unidades, descuento 0, de cada producto de la categoría `Condiments`, usando OrderID 22078.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ08.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

IF OBJECT_ID('tempdb..#OrderDetailsLab') IS NOT NULL
    DROP TABLE #OrderDetailsLab;

SELECT TOP (0)
       OrderID,
       ProductID,
       UnitPrice,
       Quantity,
       Discount
INTO #OrderDetailsLab
FROM [Order Details];

INSERT INTO #OrderDetailsLab
    (OrderID, ProductID, UnitPrice, Quantity, Discount)
SELECT 22078,
       ProductID,
       UnitPrice,
       5,
       0
FROM Products
WHERE CategoryID = (
    SELECT CategoryID
    FROM Categories
    WHERE CategoryName = 'Condiments'
);

SELECT @@ROWCOUNT AS FilasInsertadas;
SELECT * FROM #OrderDetailsLab ORDER BY ProductID;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ08, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La tabla real no cambia y la selección de categoría usa una subconsulta. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

El tercer DML actualiza una copia temporal y usa una subconsulta correlacionada con la fila que se está actualizando.


# EJ09 - UPDATE con subconsulta correlacionada sobre una copia temporal

## Objetivo docente

Actualizar una fila usando una subconsulta que referencia el ProductID de la fila exterior.

## Que debe estar visible antes de comenzar

Se crea #ProductsLab; Products real permanece intacta.

## Que decir antes de escribir

Advertir que la fórmula se conserva como práctica de subconsulta correlacionada. No presentarla como modelo general de cálculo de stock.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
IF OBJECT_ID('tempdb..#ProductsLab') IS NOT NULL
    DROP TABLE #ProductsLab;

SELECT ProductID,
       ProductName,
       UnitsInStock
INTO #ProductsLab
FROM Products;
```

### Mientras escribe, explicar

La copia temporal nos permite observar el cambio de estado antes y después sin alterar inventario real. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT od.ProductID,
       SUM(od.Quantity) AS TotalQuantity,
       SUM(od.Quantity * od.Discount) AS AjusteDescuento
FROM [Order Details] AS od
JOIN Orders AS o
    ON od.OrderID = o.OrderID
WHERE o.ShippedDate IS NULL
  AND od.ProductID = 7
GROUP BY od.ProductID;
```

### Mientras escribe, explicar

Esta consulta deja visible la fórmula que luego se incorporará como subconsulta correlacionada. Se conserva el planteamiento didáctico del caso de la sesión. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

IF OBJECT_ID('tempdb..#ProductsLab') IS NOT NULL
    DROP TABLE #ProductsLab;

SELECT ProductID,
       ProductName,
       UnitsInStock
INTO #ProductsLab
FROM Products;

SELECT ProductID, ProductName, UnitsInStock AS Antes
FROM #ProductsLab
WHERE ProductID = 7;

UPDATE pl
SET UnitsInStock = (
    SELECT CAST(
        SUM(CONVERT(decimal(18,2), od.Quantity))
        - SUM(CONVERT(decimal(18,2), od.Quantity)
              * CONVERT(decimal(18,4), od.Discount))
        AS smallint
    )
    FROM [Order Details] AS od
    JOIN Orders AS o
        ON od.OrderID = o.OrderID
    WHERE o.ShippedDate IS NULL
      AND od.ProductID = pl.ProductID
)
FROM #ProductsLab AS pl
WHERE pl.ProductID = 7;

SELECT ProductID, ProductName, UnitsInStock AS Despues
FROM #ProductsLab
WHERE ProductID = 7;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- Solo ProductID 7 de #ProductsLab puede cambiar.
- La subconsulta se evalúa en relación con `pl.ProductID`.
- Si no existen filas que cumplan ShippedDate IS NULL para ese producto, el agregado puede producir NULL.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Dos lecturas de ProductID 7, antes y después. El valor final refleja la fórmula del caso didáctico, no una política real de inventario.

## Interpretacion frente al grupo

La correlación aparece en `od.ProductID = pl.ProductID`: la subconsulta necesita el valor de la fila que UPDATE está procesando.

## Variante A

Ejecutar la subconsulta aislada con ProductID = 7 antes de hacer UPDATE.

## Error controlado

Eliminar `od.ProductID = pl.ProductID` dentro de la subconsulta.

## Explicacion del error

La subconsulta dejaría de depender de la fila exterior y calcularía un agregado global de todas las filas no enviadas.

## Correccion

Restaurar la condición de correlación por ProductID.

# T09 - Tarea espejo

## Enunciado

Repite el patrón sobre #ProductsLab para ProductID = 8, conservando la correlación por ProductID.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ09.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

IF OBJECT_ID('tempdb..#ProductsLab') IS NOT NULL
    DROP TABLE #ProductsLab;

SELECT ProductID,
       ProductName,
       UnitsInStock
INTO #ProductsLab
FROM Products;

SELECT ProductID, ProductName, UnitsInStock AS Antes
FROM #ProductsLab
WHERE ProductID = 8;

UPDATE pl
SET UnitsInStock = (
    SELECT CAST(
        SUM(CONVERT(decimal(18,2), od.Quantity))
        - SUM(CONVERT(decimal(18,2), od.Quantity)
              * CONVERT(decimal(18,4), od.Discount))
        AS smallint
    )
    FROM [Order Details] AS od
    JOIN Orders AS o
        ON od.OrderID = o.OrderID
    WHERE o.ShippedDate IS NULL
      AND od.ProductID = pl.ProductID
)
FROM #ProductsLab AS pl
WHERE pl.ProductID = 8;

SELECT ProductID, ProductName, UnitsInStock AS Despues
FROM #ProductsLab
WHERE ProductID = 8;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ09, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

El UPDATE apunta a #ProductsLab y la subconsulta contiene `od.ProductID = pl.ProductID`. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

Cerramos con EXISTS: en vez de traer el conjunto para compararlo, preguntamos si existen filas relacionadas.


# EJ10 - EXISTS y NOT EXISTS: comprobar existencia de filas relacionadas

## Objetivo docente

Filtrar filas de una consulta principal según exista o no exista al menos una fila relacionada en la subconsulta.

## Que debe estar visible antes de comenzar

Customers, Orders, Products, [Order Details] y Shippers están disponibles en Northwind.

## Que decir antes de escribir

Preguntar qué columna necesita devolver EXISTS. La respuesta esperada es: ninguna columna de negocio; interesa la presencia de filas.

## Paso 1 - Accion del docente

Abrir una consulta nueva y escribir la primera pieza sin mostrar todavia la solucion final.

### Codigo que escribe

```sql
SELECT c.CustomerID,
       c.CompanyName
FROM Customers AS c
WHERE EXISTS (
    SELECT 1
    FROM Orders AS o
    WHERE o.CustomerID = c.CustomerID
);
```

### Mientras escribe, explicar

EXISTS no necesita devolver columnas útiles para la consulta exterior; solo necesita saber si la subconsulta produce al menos una fila. `SELECT 1` hace visible esa intención. Subraya verbalmente la cardinalidad esperada: una fila, varias filas o un conjunto usado para verificar existencia.

## Paso 2 - Construir la relacion con la consulta exterior

### Codigo que escribe

```sql
SELECT c.CustomerID,
       c.CompanyName
FROM Customers AS c
WHERE NOT EXISTS (
    SELECT 1
    FROM Orders AS o
    WHERE o.CustomerID = c.CustomerID
);
```

### Mientras escribe, explicar

NOT EXISTS invierte la condición: conserva clientes para los que no existe una fila relacionada en Orders. Pide al grupo que identifique el operador que consume el resultado y que indique si existe correlacion.

## Paso 3 - Mostrar el estado completo

```sql
USE Northwind;
GO

-- Clientes con al menos un pedido
SELECT c.CustomerID,
       c.CompanyName,
       c.ContactName
FROM Customers AS c
WHERE EXISTS (
    SELECT 1
    FROM Orders AS o
    WHERE o.CustomerID = c.CustomerID
)
ORDER BY c.CompanyName;

-- Productos que aparecen en un detalle con Quantity >= 10
SELECT p.ProductID,
       p.ProductName,
       p.CategoryID
FROM Products AS p
WHERE EXISTS (
    SELECT 1
    FROM [Order Details] AS od
    WHERE od.ProductID = p.ProductID
      AND od.Quantity >= 10
)
ORDER BY p.ProductName;
```

## Pregunta antes de ejecutar

- Que parte se evalua como dato intermedio?
- Que filas esperan ver o modificar?
- Que condicion podria provocar un resultado diferente?

## Respuesta esperada

- La primera consulta devuelve clientes con al menos una fila en Orders.
- La segunda devuelve productos para los que existe al menos un detalle con Quantity >= 10.
- El valor literal `1` de SELECT 1 no se usa como dato de salida.

## Ejecutar

Ejecutar el bloque completo y no pasar de inmediato al siguiente ejemplo. Dejar visible el resultado para interpretarlo.

## Resultado esperado

Dos conjuntos filtrados por existencia. La correlación aparece en CustomerID y ProductID respectivamente.

## Interpretacion frente al grupo

EXISTS expresa una pregunta booleana: “¿hay al menos una fila que cumpla?”. Puede ser más claro que recuperar una lista cuando solo interesa la existencia; el rendimiento concreto depende del plan de ejecución y del diseño de datos.

## Variante A

Cambiar EXISTS por NOT EXISTS en la primera consulta y observar si existen clientes sin pedidos.

## Error controlado

Eliminar la condición `o.CustomerID = c.CustomerID` en la primera subconsulta.

## Explicacion del error

La subconsulta deja de estar correlacionada; si Orders tiene al menos una fila, EXISTS será verdadero para todos los clientes.

## Correccion

Restaurar la condición que conecta la fila exterior con la subconsulta.

# T10 - Tarea espejo

## Enunciado

Encuentra los pedidos que contienen productos del proveedor SupplierID = 2 y que fueron enviados por la compañía `Speedy Express`, usando EXISTS.

## Solucion docente paso a paso

1. Conservar la misma estructura conceptual de EJ10.
2. Aplicar la modificacion pedida sin cambiar a un patron distinto.
3. Ejecutar primero la subconsulta aislada cuando sea posible.
4. Ejecutar la solucion completa y revisar la evidencia.

### Codigo de solucion

```sql
USE Northwind;
GO

SELECT o.OrderID,
       o.CustomerID,
       o.OrderDate,
       o.ShipVia
FROM Orders AS o
WHERE EXISTS (
    SELECT 1
    FROM [Order Details] AS od
    JOIN Products AS p
        ON od.ProductID = p.ProductID
    WHERE od.OrderID = o.OrderID
      AND p.SupplierID = 2
)
AND EXISTS (
    SELECT 1
    FROM Shippers AS s
    WHERE s.ShipperID = o.ShipVia
      AND s.CompanyName = 'Speedy Express'
)
ORDER BY o.OrderID;
```

## Por que funciona

La solucion conserva el operador y la forma de consumo de la subconsulta que se practicaron en EJ10, pero cambia la condicion o el contexto suficiente para exigir transferencia.

## Como validarla

La consulta externa parte de Orders y usa EXISTS correlacionados por OrderID y ShipVia. La evidencia del estudiante debe coincidir con esa regla, no con un numero fijo de filas.

## Error probable

El estudiante puede copiar literalmente el ejemplo y cambiar solo un literal, perder la condicion de correlacion o ejecutar DML sobre una tabla real. Pedir que explique la consulta antes de aceptar el resultado.

## Transicion al siguiente ejemplo

Cerrar la sesión integrando los patrones: escalar, conjunto, cuantificadores, DML seguro y existencia.


# Cierre docente

Cierra pidiendo al grupo que clasifique los diez ejemplos por el tipo de resultado de la subconsulta: escalar, conjunto, cuantificador, correlacion o existencia. Recupera los tres DML y pregunta que mecanismo de seguridad se uso. La respuesta debe ser: tablas temporales y ausencia de cambios sobre las tablas reales.

Refuerza que `EXISTS` expresa una comprobacion de existencia; no presentes como regla que siempre sera mas rapido que `IN`. En rendimiento real intervienen el plan de ejecucion, la cardinalidad, los indices y la forma de la consulta.

Para EJ06, recuerda la compatibilidad: `IS [NOT] DISTINCT FROM` es nativo desde SQL Server 2022 (16.x). Si el aula usa un motor anterior, ejecuta la rama alternativa y conserva el objetivo conceptual.

El laboratorio de FASE 15 es un recurso opcional de validacion y arranque rapido; la guia del estudiante ya contiene todos los pasos esenciales.
