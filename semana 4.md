---
course_id: PABD-ISIL
session_id: S04
module_id: UA1
course_name: Programación Avanzada de Base de Datos
session_topic: Subconsultas
source_origin: PPT
status: regenerated_detailed
estado_canon: PROVISIONAL
---


# FASE 13 - Clase-laboratorio universitaria guiada y razonada

# 1. Proposito de la sesion

En esta sesion vas a trabajar con **subconsultas en SQL Server** siguiendo una regla central: antes de escribir cada consulta, primero identificaras que se busca, que dato intermedio falta y que forma debe tener ese resultado sobre la base de datos **Northwind**. El problema central es aprender a leer una consulta que depende del resultado de otra consulta y decidir si ese resultado debe tratarse como un valor unico, un conjunto de valores o una prueba de existencia.

Construiras diez ejemplos progresivos. Empezaras con una subconsulta escalar para comparar precios, continuaras con `IN`, `NOT IN`, `ANY`, `SOME`, `ALL`, comparaciones frente a `NULL`, y terminaras con operaciones `DELETE`, `INSERT`, `UPDATE` y `EXISTS`. Las operaciones de modificacion se realizan sobre **tablas temporales**, de modo que la base Northwind original se conserva intacta.

Herramientas: SQL Server, SQL Server Management Studio (SSMS) y Northwind.

Producto observable: consultas T-SQL ejecutables, resultados interpretados y diez tareas espejo que demuestran que puedes transferir el patron sin copiar literalmente el ejemplo.

# 2. Resultado observable

## A. Lectura sugerida del docente

Al finalizar esta practica podras distinguir tres preguntas antes de escribir una subconsulta: que devuelve la consulta interna, como sera consumido ese resultado y que relacion mantiene con la consulta externa. Esa decision evita muchos errores frecuentes. Si la subconsulta produce un solo valor, podras usarla con un operador escalar como `=` o `>`. Si devuelve un conjunto, podras usar patrones como `IN`, `ANY`, `SOME` o `ALL`. Si solo importa saber si existen filas relacionadas, podras usar `EXISTS` o `NOT EXISTS`.

Tambien practicaras subconsultas dentro de operaciones de modificacion. Para mantener el laboratorio seguro, no borraremos ni alteraremos datos reales de Northwind: copiaremos las filas necesarias a tablas temporales y comprobaremos el estado antes y despues. Comprender no significa solo obtener una cuadricula sin errores. Debes poder predecir el resultado, explicar por que una consulta devuelve una fila o varias, identificar la condicion que correlaciona dos niveles, y corregir un error seguro sin desactivar restricciones.

## B. Desempenos observables

- Construir subconsultas escalares y de varias filas.
- Ejecutar y explicar `IN`, `NOT IN`, `ANY`, `SOME` y `ALL`.
- Interpretar la correlacion entre consulta externa e interna.
- Comparar valores con `NULL` de forma compatible con la version del motor.
- Aplicar subconsultas en `DELETE`, `INSERT` y `UPDATE` dentro de un entorno aislado.
- Utilizar `EXISTS` y `NOT EXISTS` para comprobar filas relacionadas.
- Diagnosticar errores por cardinalidad, correlacion o version.
- Validar resultados mediante una segunda evidencia.

## C. Criterio de dominio

Hay dominio cuando puedes escribir una consulta equivalente sin copiar el ejemplo, anticipar si la subconsulta devuelve una fila o varias, justificar el operador elegido, ejecutar sin modificar datos reales y demostrar con resultados que la condicion se cumplio.

# 3. Antes de iniciar

## Debe saber

- Ejecutar `SELECT`, `WHERE`, `ORDER BY` y funciones agregadas basicas.
- Interpretar claves y relaciones entre tablas.
- Usar `JOIN` entre tablas relacionadas.
- Abrir una nueva consulta en SSMS y seleccionar una base de datos.

## Debe tener disponible

- Una instancia de SQL Server accesible.
- SQL Server Management Studio.
- Base de datos Northwind instalada de sesiones anteriores.
- Permiso para ejecutar consultas en esa base de laboratorio.

## No se asumira todavia

- Ajuste avanzado del optimizador.
- Planes de ejecucion en profundidad.
- Indices avanzados.
- Transacciones de produccion.
- Diseno de un modelo real de inventario.

# 4. Herramientas y recursos

## SQL Server

Es el motor que ejecuta las sentencias T-SQL. La sesion no exige reinstalar el motor si ya esta disponible en el entorno institucional. Para el ejemplo nativo de `IS [NOT] DISTINCT FROM`, el motor debe ser SQL Server 2022 (16.x) o posterior. En motores anteriores se incluye una alternativa compatible.

## SQL Server Management Studio (SSMS)

Es el cliente grafico usado para conectarse al motor, abrir una ventana de consulta, ejecutar sentencias y revisar resultados. A septiembre de 2026, SSMS 22 es la rama GA vigente; si el laboratorio institucional utiliza otra version compatible, mantenla para no romper la configuracion del curso.

Sitio oficial: https://learn.microsoft.com/sql/ssms/release-notes-ssms

No necesitas instalar extensiones, paquetes de terceros ni herramientas adicionales para esta sesion.

# 5. Preparacion del entorno

## Ruta A - El entorno ya existe

1. Abre SSMS.
2. Conectate a la instancia de laboratorio.
3. En Object Explorer confirma que exista `Northwind`.
4. Abre **New Query**.
5. Ejecuta:

```sql
USE Northwind;
GO

SELECT DB_NAME() AS BaseActual;
SELECT COUNT(*) AS Productos FROM Products;
SELECT COUNT(*) AS Clientes FROM Customers;
SELECT COUNT(*) AS Pedidos FROM Orders;
```

Debes observar `Northwind` como base actual y conteos sin error. No importa que tus cantidades exactas difieran de otra instalacion si el dataset fue adaptado por la institucion.

6. Comprueba version del motor:

```sql
SELECT SERVERPROPERTY('ProductVersion') AS ProductVersion,
       SERVERPROPERTY('ProductMajorVersion') AS ProductMajorVersion,
       SERVERPROPERTY('Edition') AS Edition;
```

Si `ProductMajorVersion` es 16 o superior, podras ejecutar de forma nativa `IS [NOT] DISTINCT FROM`.

## Ruta B - El equipo no esta preparado

Esta sesion presupone que **Northwind ya fue instalada en clases anteriores**. Si no tienes motor, cliente o base de datos:

1. Instala o solicita acceso a la edicion de SQL Server autorizada por tu institucion.
2. Instala SSMS desde Microsoft Learn.
3. Conectate a la instancia.
4. Recupera la misma base Northwind usada en las sesiones anteriores del curso; no uses una variante distinta sin validarla con el docente.
5. Repite las comprobaciones de la Ruta A.

No continues con los ejemplos si `USE Northwind` o `SELECT COUNT(*) FROM Products` falla. Primero corrige el entorno.

## Regla de seguridad del laboratorio

Los ejemplos `DELETE`, `INSERT` y `UPDATE` trabajan sobre tablas temporales (`#OrdersLab`, `#OrderDetailsLab`, `#ProductsLab`). No reemplaces esos nombres por tablas reales. No desactives claves foraneas ni otras restricciones para forzar una ejecucion.



# 5.1 Metodo de trabajo de esta FASE 13: primero pensar, luego escribir

En esta version de la guia **ningun ejemplo comienza directamente con SQL**. Antes de escribir debes poder explicar el problema en lenguaje natural.

Para cada `EJEMPLO_ID` seguiremos siempre este ciclo:

```text
CASO
  ->
QUE SE QUIERE OBTENER
  ->
TABLAS Y COLUMNAS NECESARIAS
  ->
DATO QUE TODAVIA NO CONOCEMOS
  ->
QUE DEBE DEVOLVER LA SUBCONSULTA
  ->
UNA FILA / VARIAS FILAS / EXISTENCIA
  ->
PLAN SIN SQL
  ->
PREDICCION
  ->
AHORA SI: ESCRIBIR
  ->
EJECUTAR POR PARTES
  ->
INTERPRETAR
  ->
INTEGRAR
  ->
VALIDAR
  ->
ERROR CONTROLADO
  ->
TAREA ESPEJO
```

## Ficha mental que debes completar antes de cada ejemplo

Antes de escribir, debes poder responder:

1. ¿Qué pregunta de negocio o de datos estoy resolviendo?
2. ¿Qué columnas quiero ver al final?
3. ¿Qué tablas contienen esos datos?
4. ¿Qué dato intermedio todavía no conozco?
5. ¿La subconsulta debe devolver un valor, varias filas o solo comprobar existencia?
6. ¿Qué operador consumirá ese resultado?
7. ¿Existe correlacion con la fila exterior?
8. ¿Qué resultado espero antes de ejecutar?

Si no puedes responder estas preguntas, todavía no es momento de escribir la consulta.

---

# EJ01 - Subconsulta escalar: productos por encima del promedio

## Que vamos a construir

Construir una consulta que use un valor agregado calculado por una subconsulta y combinarlo con unidades vendidas por producto.

## Que aprenderas aqui

**Concepto:** Subconsulta escalar + agregación + tabla derivada.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

Queremos localizar productos cuyo `UnitPrice` sea **mayor que el precio promedio de todos los productos**. Además, queremos mostrar cuántas unidades se han vendido de cada producto.

No empieces escribiendo `SELECT`. Primero separa el problema en preguntas:

1. ¿Cuál es el precio promedio de todos los productos?
2. ¿Cuántas unidades se han vendido de cada producto?
3. ¿Qué productos tienen un precio mayor que ese promedio?
4. ¿Cómo unimos el producto con su cantidad vendida?

### 2. ¿Qué resultado final buscamos?

Una cuadrícula con:

- `ProductName`;
- `UnitPrice`;
- total de unidades vendidas (`UnitsSold`);
- únicamente productos cuyo precio esté por encima del promedio.

### 3. ¿Qué tablas y columnas intervienen?

**Products**
- `ProductID`: identifica el producto.
- `ProductName`: nombre que mostraremos.
- `UnitPrice`: precio que compararemos y del que calcularemos el promedio.

**Order Details**
- `ProductID`: permite relacionar el detalle con el producto.
- `Quantity`: permite calcular las unidades vendidas.

### 4. ¿Qué información todavía no conocemos?

No conocemos de antemano el **precio promedio**. Ese valor debe calcularse con los datos existentes.

La pregunta interna será:

> ¿Cuál es el promedio de `UnitPrice` de todos los productos?

### 5. ¿Qué debe devolver la subconsulta?

**Una sola fila y una sola columna.**

Ese resultado es un valor escalar. Por eso puede usarse a la derecha del operador `>`.

### 6. ¿Qué otra pieza necesitamos?

También necesitamos resumir `Order Details` por `ProductID` para obtener `SUM(Quantity)`. Esa pieza produce varias filas, una por producto, y se utilizará como tabla derivada.

### 7. Plan de solución sin SQL

1. Calcular el promedio de precios.
2. Comprobar que el promedio sea un solo valor.
3. Calcular las unidades vendidas por producto.
4. Relacionar ese resumen con `Products`.
5. Comparar cada `UnitPrice` con el promedio.
6. Mostrar únicamente las filas que cumplan la condición.

### 8. Mapa mental

```text
Products.UnitPrice
      |
      +--> AVG(UnitPrice) --> un valor promedio
      |
Products + resumen de Order Details
      |
      +--> comparar UnitPrice > promedio
      |
      +--> resultado final
```

### 9. Predicción antes de escribir

Responde:

- ¿La subconsulta de promedio devolverá una fila o varias?
- ¿Podrías usar `IN` en lugar de `>`? ¿Por qué?
- ¿Qué ocurriría si escribieras un promedio fijo en lugar de calcularlo?

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ01_T01.sql

## Punto de partida

Northwind está seleccionada y las tablas Products y [Order Details] responden.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

```sql
USE Northwind;
GO

SELECT AVG(UnitPrice) AS PrecioPromedio
FROM Products;
```

### Que significa

La subconsulta que luego irá dentro de WHERE produce un único valor: el promedio de UnitPrice. Ese resultado puede compararse con el precio de cada producto.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

```sql
SELECT ProductID,
       SUM(Quantity) AS UnitsSold
FROM [Order Details]
GROUP BY ProductID;
```

### Que cambia respecto al paso anterior

Esta consulta resume el detalle de pedidos por ProductID. La usamos como tabla derivada para disponer de una cantidad total vendida por producto.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- La subconsulta de AVG devolverá un solo valor.
- Solo aparecerán productos cuyo UnitPrice sea mayor que ese promedio.
- UnitsSold proviene del resumen de [Order Details], no de Products.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Una cuadrícula con ProductName, UnitPrice y UnitsSold. Todas las filas deben tener UnitPrice mayor que el promedio calculado por la consulta interna.

## Como interpretarlo

El filtro depende de un dato que no estaba escrito como constante: se calcula primero como resultado de otra consulta. La tabla derivada agrega otra dimensión práctica: total vendido por producto.

## Variacion A

Cambiar `>` por `<` y anticipar cómo cambia el conjunto.

## Variacion B

Ejecutar solo la subconsulta de AVG y anotar el valor antes de ejecutar la consulta completa.

## Error controlado

Sustituye temporalmente `AVG(UnitPrice)` por `SELECT UnitPrice FROM Products` sin agregación.

## Por que ocurre

El operador `>` espera un valor escalar, pero esa subconsulta devuelve varias filas.

## Correccion razonada

Restaurar una subconsulta que garantice una sola fila, por ejemplo `SELECT AVG(UnitPrice) FROM Products`.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T01 - Tarea espejo de EJ01

## Enunciado

Obtén ProductName y UnitPrice de los productos activos (`Discontinued = 0`) cuyo precio sea menor que el promedio de todos los productos.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ01_T01.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

Debes mantener una subconsulta escalar en WHERE; no reemplazar el promedio por un número escrito manualmente.

## Pista

Empieza ejecutando el AVG por separado y luego úsalo dentro de la comparación `<`.

## Evidencia que debes mostrar

Captura o copia de la consulta final y una muestra de filas donde se observe que el filtro usa el promedio.

## Como saber si esta correcta

La consulta contiene una única subconsulta con AVG, filtra Discontinued = 0 y usa `UnitPrice < (...)`.


# EJ02 - Subconsulta de varias filas: IN y NOT IN

## Que vamos a construir

Filtrar clientes a partir de un conjunto de ProductID y contrastar inclusión frente a exclusión.

## Que aprenderas aqui

**Concepto:** IN / NOT IN con subconsulta de varias filas.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

Queremos identificar clientes que realizaron pedidos que contienen productos cuyo precio supera un valor determinado. Después contrastaremos el mismo razonamiento con `NOT IN`.

La clave es entender que primero necesitamos obtener **un conjunto de productos**, no un único producto.

### 2. ¿Qué resultado final buscamos?

Para la variante con `IN`:

- clientes que sí tienen pedidos con productos cuyo `UnitPrice > 20`.

Para la variante con `NOT IN`:

- clientes que no pertenecen al conjunto definido por la condición del ejercicio.

### 3. ¿Qué tablas intervienen?

- `Products`: decide qué `ProductID` cumplen el criterio de precio.
- `Order Details`: indica qué productos aparecen en cada pedido.
- `Orders`: relaciona el pedido con el cliente.
- `Customers`: aporta los datos del cliente.

### 4. ¿Qué información debemos obtener primero?

La pregunta interna es:

> ¿Qué identificadores de producto tienen precio superior a 20?

Esa pregunta puede devolver **muchos `ProductID`**.

### 5. ¿Qué debe devolver la subconsulta?

**Varias filas de una sola columna (`ProductID`).**

Por eso el operador natural es `IN`: la consulta principal preguntará si cada `od.ProductID` pertenece al conjunto devuelto.

### 6. Plan de solución sin SQL

1. Consultar `Products`.
2. Obtener los `ProductID` cuyo precio sea mayor a 20.
3. Verificar que la salida contiene varios IDs.
4. Recorrer pedidos y detalles.
5. Preguntar si el producto del detalle pertenece a ese conjunto.
6. Obtener el cliente asociado.
7. Usar `DISTINCT` para evitar repetir clientes cuando tengan varios pedidos coincidentes.

### 7. Mapa mental

```text
Products
  |
  +--> ProductID con UnitPrice > 20
                |
                v
      conjunto de IDs
                |
Order Details --IN--> Orders --> Customers
                |
                v
        clientes resultantes
```

### 8. Antes de escribir, decide

- ¿La subconsulta devuelve un valor o una lista?
- ¿Por qué `=` no representa bien este problema?
- ¿Por qué podría aparecer el mismo cliente varias veces sin `DISTINCT`?

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ02_T02.sql

## Punto de partida

EJ01 puede estar cerrado; este ejemplo es independiente y usa Customers, Orders, [Order Details] y Products.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

```sql
SELECT ProductID
FROM Products
WHERE UnitPrice > 20;
```

### Que significa

Esta subconsulta devuelve varios ProductID. Por eso no se compara con `=`; se usa como conjunto para `IN`.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

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

### Que cambia respecto al paso anterior

La consulta externa recorre pedidos y clientes; el IN pregunta si cada ProductID pertenece al conjunto producido por la subconsulta.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- La primera consulta debe devolver clientes con al menos una coincidencia.
- La segunda debe excluir a todos esos clientes.
- El DISTINCT de la primera evita repetir clientes por múltiples pedidos/detalles.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Dos resultados: inclusión y exclusión. Ningún cliente de la segunda lista debería cumplir la condición usada por la primera.

## Como interpretarlo

IN evalúa pertenencia a un conjunto de valores. NOT IN invierte esa pertenencia, por lo que conviene vigilar si la subconsulta puede devolver NULL; en este caso se selecciona CustomerID proveniente de Orders.

## Variacion A

Cambiar el umbral de 20 a 50 solo después de predecir si la primera lista crecerá o disminuirá.

## Variacion B

Ejecutar primero la subconsulta de ProductID para observar el conjunto que alimenta IN.

## Error controlado

Quita `DISTINCT` de la primera consulta y compara la cantidad de filas.

## Por que ocurre

Un cliente puede aparecer en varios pedidos o detalles que cumplan la condición; sin DISTINCT, el resultado repite entidades.

## Correccion razonada

Restaurar DISTINCT cuando el objetivo sea una lista única de clientes.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T02 - Tarea espejo de EJ02

## Enunciado

Obtén los clientes que NO han realizado pedidos que incluyan productos descontinuados.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ02_T02.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

Debes resolver la exclusión con `NOT IN` y una subconsulta que llegue a los CustomerID de Orders.

## Pista

La condición de productos es `Discontinued = 1`; conecta Order Details con Products dentro de la subconsulta.

## Evidencia que debes mostrar

Consulta final y resultado de clientes excluidos de pedidos con productos descontinuados.

## Como saber si esta correcta

La subconsulta devuelve CustomerID y la consulta externa usa `CustomerID NOT IN (...)`.


# EJ03 - Operador de comparación con subconsulta de una fila

## Que vamos a construir

Usar el resultado de una subconsulta de una fila como valor de comparación de la consulta principal.

## Que aprenderas aqui

**Concepto:** Subconsulta escalar con operador =.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

Primero debemos averiguar la ciudad del cliente `ALFKI`. Después utilizaremos esa ciudad para localizar a todos los clientes que viven en la misma ciudad.

### 2. ¿Qué resultado final buscamos?

Clientes cuya columna `City` sea igual a la ciudad obtenida para `ALFKI`.

### 3. ¿Qué tabla y columnas intervienen?

Solo `Customers`:

- `CustomerID`: permite localizar a `ALFKI`.
- `City`: es el valor que obtendremos en la subconsulta y compararemos.
- `CompanyName`: ayuda a reconocer las filas del resultado.

### 4. ¿Qué información no conocemos todavía?

La ciudad de `ALFKI`.

Pregunta interna:

> ¿Qué valor de `City` tiene el cliente con `CustomerID = 'ALFKI'`?

### 5. ¿Qué debe devolver la subconsulta?

Esperamos **una sola fila y una sola columna**.

Ese diseño permite utilizar el operador `=`.

### 6. Plan de solución sin SQL

1. Buscar a `ALFKI`.
2. Recuperar únicamente su ciudad.
3. Verificar que la consulta interna devuelve un solo valor.
4. Recorrer `Customers`.
5. Conservar filas cuya ciudad sea igual al valor obtenido.

### 7. Mapa mental

```text
CustomerID = ALFKI
       |
       v
     City
       |
       v
Customers.City = valor obtenido
       |
       v
clientes de la misma ciudad
```

### 8. Pregunta de control

¿Qué error conceptual existiría si la subconsulta pudiera devolver dos ciudades diferentes y todavía utilizáramos `=`?

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ03_T03.sql

## Punto de partida

Customers contiene el cliente ALFKI utilizado en el material de la sesión.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

```sql
SELECT City
FROM Customers
WHERE CustomerID = 'ALFKI';
```

### Que significa

El filtro por clave de cliente pretende producir una única ciudad. Esa ciudad será el valor de comparación de la consulta exterior.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

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

### Que cambia respecto al paso anterior

El operador `=` es válido porque la subconsulta se diseña para devolver una sola fila y una sola columna.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- Primero se obtiene la ciudad de ALFKI.
- La consulta externa conservará clientes cuya City sea exactamente esa ciudad.
- En la versión de Northwind mostrada en la sesión, ALFKI aparece en Berlin.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Una lista de clientes de la misma ciudad que ALFKI. El valor comparado no se escribe manualmente: proviene de la subconsulta.

## Como interpretarlo

Este patrón es útil cuando la condición depende de un dato localizado en otra fila. El punto crítico es garantizar que la subconsulta usada con `=` no devuelva múltiples filas.

## Variacion A

Cambiar `=` por `<>` y predecir qué conjunto se obtiene.

## Variacion B

Mostrar solo CustomerID, CompanyName y City para comprobar visualmente la igualdad.

## Error controlado

Elimina el filtro `WHERE CustomerID = 'ALFKI'` dentro de la subconsulta.

## Por que ocurre

La subconsulta pasa a devolver muchas ciudades; el operador escalar `=` no acepta múltiples filas.

## Correccion razonada

Restaurar un criterio que garantice una sola fila o cambiar el patrón a un operador de conjunto cuando el problema realmente requiera varias filas.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T03 - Tarea espejo de EJ03

## Enunciado

Selecciona los clientes que están en la misma ciudad que el cliente `ANATR`.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ03_T03.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

Usa una subconsulta de una fila; no consultes manualmente la ciudad para escribirla como literal.

## Pista

Cambia únicamente el CustomerID buscado dentro de la subconsulta y conserva el patrón `City = (...)`.

## Evidencia que debes mostrar

Consulta y resultado donde todas las filas comparten la ciudad obtenida para ANATR.

## Como saber si esta correcta

La ciudad se obtiene por subconsulta y no aparece escrita como constante.


# EJ04 - ANY y SOME: verdadero si alguna comparación se cumple

## Que vamos a construir

Comparar un precio con el conjunto de precios de un proveedor usando ANY y verificar la equivalencia conceptual de SOME.

## Que aprenderas aqui

**Concepto:** ANY / SOME con subconsulta de una columna.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

Compararemos el precio de cada producto con **varios precios** pertenecientes al proveedor `SupplierID = 2`.

El objetivo es comprender `ANY` y comprobar que `SOME` expresa el mismo tipo de condición.

### 2. ¿Qué resultado final buscamos?

Productos cuyo `UnitPrice` sea mayor que **al menos uno** de los precios devueltos por la subconsulta.

### 3. ¿Qué datos necesitamos?

De `Products`:

- `ProductID`;
- `ProductName`;
- `UnitPrice`;
- `SupplierID`.

### 4. ¿Qué información debe producir la consulta interna?

La lista de `UnitPrice` de los productos del proveedor 2.

La pregunta interna es:

> ¿Cuáles son todos los precios de los productos vendidos por el proveedor 2?

### 5. ¿Qué debe devolver la subconsulta?

**Varias filas de una sola columna (`UnitPrice`).**

### 6. ¿Cómo razonar `> ANY`?

`precio > ANY (lista)` será verdadero cuando el precio sea mayor que **por lo menos uno** de los valores de la lista.

Antes de escribir, observa el menor y el mayor precio del conjunto. Esto te ayudará a predecir qué filas pueden pasar el filtro.

### 7. Plan de solución sin SQL

1. Obtener la lista de precios del proveedor 2.
2. Ordenarla para entender sus extremos.
3. Elegir un producto de prueba y compararlo mentalmente contra la lista.
4. Aplicar `> ANY`.
5. Repetir la consulta cambiando `ANY` por `SOME`.
6. Comparar ambos resultados.

### 8. Mapa mental

```text
precios del proveedor 2
  10, 20, 30, ...
        |
        v
precio de otro producto
        |
      > ANY
        |
si supera al menos uno -> conservar
```

### 9. Predicción

Si la lista fuera `10, 20, 30`, ¿un precio de `15` cumpliría `> ANY`? Explica contra qué valor se cumple.

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ04_T04.sql

## Punto de partida

Products contiene SupplierID y UnitPrice.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

```sql
SELECT ProductName,
       UnitPrice
FROM Products
WHERE SupplierID = 2
ORDER BY UnitPrice;
```

### Que significa

Observamos primero el conjunto de precios contra el que se hará la comparación.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

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

### Que cambia respecto al paso anterior

`> ANY` resulta verdadero si el precio del producto es mayor que al menos uno de los precios devueltos por la subconsulta.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- ANY y SOME deben devolver el mismo conjunto para la misma comparación.
- No es necesario superar todos los precios del proveedor 2; basta con superar al menos uno.
- La segunda condición evita mezclar productos del proveedor usado como referencia.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Dos conjuntos equivalentes de productos, uno con ANY y otro con SOME.

## Como interpretarlo

ANY y SOME son dos formas de expresar la misma condición en SQL Server: la comparación debe ser verdadera para al menos un valor de la subconsulta.

## Variacion A

Cambiar `>` por `<` y razonar qué significa “menor que al menos uno”.

## Variacion B

Quitar temporalmente `SupplierID <> 2` para observar si aparecen productos del propio conjunto de referencia.

## Error controlado

Sustituye la subconsulta por `SELECT ProductID ...` manteniendo la comparación con UnitPrice.

## Por que ocurre

La columna de la subconsulta debe ser comparable con la expresión escalar; ProductID no representa el mismo dominio que UnitPrice.

## Correccion razonada

Devolver UnitPrice desde la subconsulta cuando se compara contra UnitPrice.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T04 - Tarea espejo de EJ04

## Enunciado

Usa SOME para encontrar productos de otros proveedores cuyo UnitPrice sea mayor que al menos uno de los precios del proveedor con SupplierID = 3.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ04_T04.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

Debes usar SOME, no MAX/MIN.

## Pista

Conserva el patrón del ejemplo y cambia el proveedor de referencia a 3.

## Evidencia que debes mostrar

Consulta y resultado; explica con una frase qué significa “mayor que SOME”.

## Como saber si esta correcta

La consulta usa `UnitPrice > SOME (SELECT UnitPrice ... SupplierID = 3)`.


# EJ05 - ALL: verdadero si todas las comparaciones se cumplen

## Que vamos a construir

Diferenciar ANY/SOME de ALL al comparar un valor con todos los elementos de un conjunto.

## Que aprenderas aqui

**Concepto:** ALL con subconsulta de una columna.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

Ahora utilizaremos el mismo tipo de conjunto que en el ejemplo anterior, pero cambiaremos la condición: el precio deberá ser mayor que **todos** los valores devueltos.

### 2. ¿Qué resultado final buscamos?

Productos cuyo `UnitPrice` sea mayor que cada uno de los precios del proveedor 2.

### 3. ¿Qué información necesitamos primero?

Los precios del proveedor 2 y, para razonar con claridad, especialmente su **máximo**.

### 4. ¿Qué debe devolver la subconsulta?

Varias filas con `UnitPrice`.

### 5. ¿Cómo razonar `> ALL`?

Para que `precio > ALL (lista)` sea verdadero, el precio debe superar incluso al valor más alto del conjunto.

Por eso, aunque SQL compara con el conjunto, mentalmente puedes usar el máximo como referencia para predecir.

### 6. Plan de solución sin SQL

1. Consultar mínimo y máximo de precios del proveedor 2.
2. Obtener la lista real de precios.
3. Predecir qué productos tienen un precio superior al máximo.
4. Aplicar `> ALL`.
5. Comparar el resultado con el obtenido usando `> ANY`.

### 7. Mapa mental

```text
precios proveedor 2
       |
       +--> máximo
       |
precio candidato
       |
    > ALL
       |
debe superar todos
```

### 8. Contraste obligatorio

Antes de escribir, completa:

- `ANY` significa: ______________________
- `ALL` significa: ______________________
- Si el precio supera solo un valor pero no supera el máximo, ¿aparece con `ALL`?

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ05_T05.sql

## Punto de partida

Se reutiliza Products sin depender del estado de EJ04.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

```sql
SELECT MIN(UnitPrice) AS Minimo,
       MAX(UnitPrice) AS Maximo
FROM Products
WHERE SupplierID = 2;
```

### Que significa

Los extremos ayudan a razonar la condición. Para cumplir `> ALL`, un precio debe superar incluso el máximo del conjunto.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

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

### Que cambia respecto al paso anterior

ALL exige que cada comparación individual sea verdadera. En este caso, el precio debe ser mayor que todos los precios devueltos.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- El conjunto de `> ALL` será igual o más restrictivo que el de `> ANY` para el mismo proveedor.
- Cada fila devuelta debe superar el mayor precio del conjunto de referencia.
- Si la subconsulta no devolviera filas, la lógica requiere un análisis especial; aquí trabajamos con un proveedor existente.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Productos cuyo UnitPrice es mayor que cada precio del proveedor 2.

## Como interpretarlo

ALL expresa una condición universal sobre el conjunto. La diferencia con ANY/SOME no es de sintaxis superficial: cambia la regla lógica que debe cumplirse.

## Variacion A

Ejecutar la versión ANY del ejemplo anterior y comparar visualmente la cantidad de filas.

## Variacion B

Calcular el MAX del proveedor 2 y comprobar que todas las filas de `> ALL` lo superan.

## Error controlado

Cambiar ALL por ANY sin cambiar la explicación del resultado.

## Por que ocurre

La consulta seguirá siendo válida, pero la semántica cambia de “todos” a “al menos uno”; interpretar ambas como equivalentes sería un error conceptual.

## Correccion razonada

Restaurar ALL y validar contra el MAX del conjunto de referencia.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T05 - Tarea espejo de EJ05

## Enunciado

Encuentra productos de otros proveedores cuyo UnitPrice sea mayor que TODOS los precios del proveedor con SupplierID = 3.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ05_T05.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

Debes usar ALL; no reemplazarlo por MAX en la solución principal.

## Pista

El patrón es `UnitPrice > ALL (SELECT UnitPrice ... WHERE SupplierID = 3)`.

## Evidencia que debes mostrar

Consulta final y una comprobación adicional con el MAX del proveedor 3.

## Como saber si esta correcta

La condición usa ALL y excluye SupplierID = 3 de la consulta externa.


# EJ06 - Comparación segura frente a NULL con IS [NOT] DISTINCT FROM

## Que vamos a construir

Comparar valores con semántica determinista frente a NULL y mantener compatibilidad con motores anteriores a SQL Server 2022.

## Que aprenderas aqui

**Concepto:** IS DISTINCT FROM / IS NOT DISTINCT FROM.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

La sesión introduce `IS [NOT] DISTINCT FROM` para comparar valores cuando puede aparecer `NULL`.

En este ejemplo no necesitamos forzar una subconsulta: el objetivo específico es comprender **cómo cambia una comparación cuando intervienen valores nulos**, manteniendo el contexto de productos y categorías de Northwind.

### 2. ¿Qué resultado final buscamos?

Poder distinguir dos preguntas:

- ¿Los valores deben considerarse iguales, incluso en el caso `NULL` con `NULL`?
- ¿Los valores deben considerarse diferentes, incluyendo comparaciones donde intervenga `NULL`?

### 3. ¿Qué tablas intervienen?

- `Products`;
- `Categories`.

Se usa `LEFT JOIN` porque queremos conservar también productos sin una categoría relacionada cuando el caso lo requiera.

### 4. ¿Qué debemos comprobar antes de escribir?

La versión del motor SQL Server, porque la sintaxis nativa `IS [NOT] DISTINCT FROM` está disponible en SQL Server 2022 (16.x) y posteriores.

### 5. ¿Qué problema tiene una comparación ordinaria con NULL?

`NULL` representa un valor desconocido. Expresiones como `NULL = NULL` no producen el mismo tipo de respuesta booleana que una comparación entre valores conocidos.

Por eso este predicado permite expresar una comparación con resultado verdadero o falso incluso cuando interviene `NULL`.

### 6. Plan de solución sin SQL

1. Comprobar la versión del motor.
2. Relacionar `Products` con `Categories` usando `LEFT JOIN`.
3. Identificar qué filas pueden tener categoría y cuáles pueden quedar sin coincidencia.
4. Aplicar la comparación nativa si el motor la soporta.
5. Si no la soporta, usar la expresión equivalente incluida en la guía.
6. Comparar ambos razonamientos.

### 7. Mapa mental

```text
Products --LEFT JOIN--> Categories
      |
      +--> valor conocido
      |
      +--> posible NULL
              |
              v
   comparación consciente de NULL
```

### 8. Pregunta de control

¿Por qué no debemos cambiar `LEFT JOIN` por `INNER JOIN` si queremos conservar la posibilidad de observar filas sin categoría relacionada?

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ06_T06.sql

## Punto de partida

El ejemplo detecta la versión del motor. La sintaxis nativa requiere SQL Server 2022 (16.x) o posterior.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

```sql
SELECT SERVERPROPERTY('ProductVersion') AS ProductVersion,
       SERVERPROPERTY('ProductMajorVersion') AS ProductMajorVersion,
       SERVERPROPERTY('Edition') AS Edition;
```

### Que significa

Antes de usar la sintaxis nativa, comprobamos la versión del motor. El cliente SSMS y el motor SQL Server son componentes distintos.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

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

### Que cambia respecto al paso anterior

Esta es la forma compatible con versiones anteriores para expresar igualdad considerando el caso NULL = NULL como coincidencia lógica.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- En SQL Server 2022+ se ejecutará la rama con `IS NOT DISTINCT FROM`.
- En versiones anteriores se ejecutará una expresión equivalente para igualdad con NULL.
- Con @categoria = Beverages se esperan productos cuya categoría sea Beverages.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Una lista de productos de la categoría indicada. La ruta de ejecución depende de ProductMajorVersion, pero el objetivo lógico es el mismo.

## Como interpretarlo

`IS NOT DISTINCT FROM` produce una comparación de igualdad que siempre resuelve a verdadero o falso, incluso con NULL. La rama alternativa permite mantener la práctica en motores anteriores.

## Variacion A

Asignar `NULL` a @categoria y observar cómo la comparación trata filas sin categoría.

## Variacion B

En SQL Server 2022+, cambiar a `IS DISTINCT FROM` para obtener los valores diferentes, incluyendo diferencias por NULL.

## Error controlado

Escribir directamente `IS NOT DISTINCT FROM` en un motor anterior a SQL Server 2022.

## Por que ocurre

La sintaxis nativa no está disponible antes de SQL Server 2022 (16.x).

## Correccion razonada

Usar el control de versión y la expresión compatible incluida en el ejemplo.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T06 - Tarea espejo de EJ06

## Enunciado

Obtén los productos cuya CategoryName sea distinta de `Beverages`, tratando NULL como un valor diferente de `Beverages`.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ06_T06.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

En SQL Server 2022+ usa `IS DISTINCT FROM`; en versiones anteriores usa una condición equivalente. Mantén el control de versión.

## Pista

La equivalencia para “distinto” debe considerar tres casos: `<>`, izquierda NULL/derecha no NULL, izquierda no NULL/derecha NULL.

## Evidencia que debes mostrar

Consulta ejecutada y salida; registra además el ProductMajorVersion del motor.

## Como saber si esta correcta

La solución contiene una ruta nativa para 16+ y una ruta compatible para versiones anteriores.


# EJ07 - DELETE con subconsulta en una copia temporal

## Que vamos a construir

Usar una subconsulta para decidir qué filas eliminar sin modificar la tabla Orders real.

## Que aprenderas aqui

**Concepto:** DELETE + IN + subconsulta.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

El caso de la sesión busca eliminar pedidos que contienen productos con `UnitPrice > 50`.

Sin embargo, no vamos a borrar datos reales de Northwind. Reproduciremos la lógica sobre una copia temporal de `Orders`.

### 2. ¿Qué resultado final buscamos?

Una tabla temporal `#OrdersLab` en la que hayan desaparecido los pedidos cuyos identificadores fueron seleccionados por la subconsulta.

### 3. ¿Qué información necesitamos antes de ejecutar DELETE?

Primero debemos responder:

> ¿Qué `OrderID` corresponden a pedidos que contienen productos con precio mayor a 50?

Esa consulta debe revisarse **antes** de borrar.

### 4. ¿Qué debe devolver la subconsulta?

Varias filas con `OrderID`.

Por eso el `DELETE` puede usar:

```text
WHERE OrderID IN (conjunto de OrderID)
```

### 5. ¿Por qué usamos una tabla temporal?

La fuente advierte que un `DELETE` real puede fallar por integridad referencial. Además, una clase no debe destruir el dataset compartido.

La tabla temporal permite practicar:

- selección de filas;
- borrado;
- conteo antes/después;
- validación;

sin tocar `Orders`.

### 6. Plan de solución sin SQL

1. Crear una copia temporal de `Orders`.
2. Contar cuántas filas tiene.
3. Ejecutar solo la consulta que obtiene los `OrderID` candidatos.
4. Revisar algunos IDs.
5. Ejecutar `DELETE` únicamente sobre `#OrdersLab`.
6. Volver a contar filas.
7. Comprobar que los IDs seleccionados ya no están en la copia.

### 7. Mapa mental

```text
Products > 50
     |
Order Details
     |
     +--> OrderID candidatos
               |
               v
      DELETE #OrdersLab
               |
               v
       validar antes/después
```

### 8. Regla de seguridad

Si tu sentencia dice `DELETE FROM Orders`, detente. En este laboratorio la sentencia correcta debe apuntar a `#OrdersLab`.

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ07_T07.sql

## Punto de partida

La sesión está conectada a Northwind. El ejemplo crea #OrdersLab y solo borra dentro de esa tabla temporal.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

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

### Que significa

SELECT INTO crea una copia temporal de las columnas necesarias. La tabla desaparece al cerrar la sesión y no tiene las claves foráneas de Orders.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

```sql
SELECT DISTINCT od.OrderID
FROM [Order Details] AS od
JOIN Products AS p
    ON od.ProductID = p.ProductID
WHERE p.UnitPrice > 50;
```

### Que cambia respecto al paso anterior

Antes de borrar, observamos exactamente qué OrderID seleccionará la subconsulta.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- FilasDespues debe ser menor o igual que FilasAntes.
- Orders real no cambia.
- La subconsulta determina los OrderID afectados.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Tres evidencias: conteo antes, número de filas eliminadas y conteo después. La operación ocurre solo en #OrdersLab.

## Como interpretarlo

El patrón DELETE + subconsulta permite decidir filas según datos de otras tablas. La tabla temporal elimina el riesgo de romper integridad referencial de Northwind durante la práctica.

## Variacion A

Ejecutar primero la subconsulta sola y contar cuántos OrderID distintos devuelve.

## Variacion B

Cerrar la ventana de consulta y comprobar que #OrdersLab ya no existe en otra sesión.

## Error controlado

Intentar cambiar `DELETE FROM #OrdersLab` por `DELETE FROM Orders` en la base real.

## Por que ocurre

Orders participa en relaciones; el material advierte que una FK puede impedir el borrado y, además, sería una modificación destructiva del dataset de clase.

## Correccion razonada

Mantener la eliminación sobre #OrdersLab. No desactivar claves ni borrar datos reales para completar el ejercicio.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T07 - Tarea espejo de EJ07

## Enunciado

En una nueva #OrdersLab, elimina las órdenes que contengan al menos un producto descontinuado (`Discontinued = 1`).

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ07_T07.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

La tabla real Orders no debe modificarse. Debes mostrar conteo antes y después.

## Pista

La subconsulta necesita [Order Details] + Products y debe devolver OrderID distintos.

## Evidencia que debes mostrar

FilasAntes, FilasEliminadas y FilasDespues, más la consulta usada.

## Como saber si esta correcta

Solo se ejecuta DELETE sobre #OrdersLab y la condición de la subconsulta usa Discontinued = 1.


# EJ08 - INSERT con subconsulta para seleccionar una categoría

## Que vamos a construir

Insertar varias filas obtenidas de Products usando una subconsulta para resolver CategoryID.

## Que aprenderas aqui

**Concepto:** INSERT ... SELECT + subconsulta escalar.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

Queremos insertar filas de productos de la categoría `Beverages` en una tabla de detalles de laboratorio.

La lógica combina dos niveles:

1. averiguar qué `CategoryID` corresponde a `Beverages`;
2. seleccionar todos los productos de esa categoría para formar las filas a insertar.

### 2. ¿Qué resultado final buscamos?

Varias filas nuevas dentro de `#OrderDetailsLab`, con:

- un `OrderID` de laboratorio;
- cada `ProductID` de `Beverages`;
- su `UnitPrice`;
- `Quantity = 10`;
- un descuento controlado.

### 3. ¿Qué información debemos obtener primero?

Pregunta interna:

> ¿Cuál es el `CategoryID` de la categoría `Beverages`?

### 4. ¿Qué debe devolver esa subconsulta?

Una sola fila y una sola columna: `CategoryID`.

Ese valor se utiliza para filtrar varias filas de `Products`.

### 5. ¿Qué produce el SELECT del INSERT?

Después de resolver el `CategoryID`, el `SELECT` exterior produce **varias filas**, una por producto de la categoría.

### 6. Plan de solución sin SQL

1. Crear `#OrderDetailsLab` vacía.
2. Buscar el identificador de `Beverages`.
3. Ejecutar solo el `SELECT` que devolverá los productos.
4. Contar cuántas filas se insertarían.
5. Ejecutar `INSERT ... SELECT`.
6. Consultar la tabla temporal y comprobar el número de filas.

### 7. Mapa mental

```text
Categories
   |
   +--> CategoryID de Beverages
             |
             v
Products filtrados
             |
             v
filas que alimentan INSERT
             |
             v
#OrderDetailsLab
```

### 8. Predicción

Si la categoría contiene 12 productos, ¿cuántas filas esperas que inserte `INSERT ... SELECT`? Justifica sin ejecutar.

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ08_T08.sql

## Punto de partida

Se crea #OrderDetailsLab con la misma forma básica de [Order Details], pero vacía y sin modificar datos reales.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

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

### Que significa

TOP (0) copia la estructura de las columnas seleccionadas sin copiar filas.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

```sql
SELECT CategoryID
FROM Categories
WHERE CategoryName = 'Beverages';
```

### Que cambia respecto al paso anterior

Esta subconsulta devuelve el identificador que se usará para filtrar Products.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- Se insertará una fila por cada producto de la categoría Beverages.
- Quantity será 10 en todas las filas insertadas.
- La tabla real [Order Details] no se modifica.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Una tabla temporal con filas de productos de Beverages y los valores fijos definidos para OrderID, Quantity y Discount.

## Como interpretarlo

INSERT ... SELECT permite insertar tantas filas como produzca la consulta. La subconsulta escalar resuelve el CategoryID a partir del nombre de categoría.

## Variacion A

Cambiar Quantity de 10 a 5 y predecir qué columnas cambian.

## Variacion B

Ejecutar primero la subconsulta de CategoryID y luego la consulta de Products que la consume.

## Error controlado

Cambiar el destino a `[Order Details]` real manteniendo OrderID 22077.

## Por que ocurre

El valor de OrderID podría no existir o podría chocar con restricciones. Además, modificaríamos el dataset de la clase.

## Correccion razonada

Mantener #OrderDetailsLab como destino de la práctica.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T08 - Tarea espejo de EJ08

## Enunciado

Crea #OrderDetailsLab e inserta 5 unidades, descuento 0, de cada producto de la categoría `Condiments`, usando OrderID 22078.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ08_T08.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

El CategoryID debe resolverse con una subconsulta por CategoryName; no escribir el ID manualmente.

## Pista

Cambia la categoría, el OrderID, Quantity y Discount, pero conserva INSERT ... SELECT.

## Evidencia que debes mostrar

Filas insertadas y contenido final de #OrderDetailsLab.

## Como saber si esta correcta

La tabla real no cambia y la selección de categoría usa una subconsulta.


# EJ09 - UPDATE con subconsulta correlacionada sobre una copia temporal

## Que vamos a construir

Actualizar una fila usando una subconsulta que referencia el ProductID de la fila exterior.

## Que aprenderas aqui

**Concepto:** UPDATE + subconsulta correlacionada.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

El caso trabaja un `UPDATE` cuyo nuevo valor se calcula mediante una subconsulta relacionada con el producto que se está actualizando.

Para practicar de forma segura, actualizaremos `#ProductsLab`, no `Products`.

### 2. ¿Qué resultado final buscamos?

Modificar `UnitsInStock` de un producto de laboratorio utilizando un cálculo obtenido desde `Order Details` y `Orders`.

### 3. ¿Qué información necesitamos calcular?

Para cada producto objetivo necesitamos resumir cantidades de detalles asociados a pedidos que cumplen la condición del caso.

La idea importante no es memorizar la fórmula, sino observar que la consulta interna usa el `ProductID` de la fila que está actualizando la consulta exterior.

### 4. ¿Qué hace que la subconsulta sea correlacionada?

Dentro de la consulta interna aparece una referencia al producto de la consulta exterior.

Conceptualmente:

```text
fila exterior: ProductID = X
          |
          v
subconsulta calcula datos para ProductID = X
```

### 5. ¿Qué debe devolver la subconsulta para el SET?

El lado derecho de:

```text
SET UnitsInStock = (...)
```

necesita un **valor escalar** por cada fila actualizada.

Por ello la consulta interna debe estar diseñada para producir un único valor agregado.

### 6. Plan de solución sin SQL

1. Copiar `Products` a `#ProductsLab`.
2. Elegir el producto objetivo.
3. Consultar su valor inicial.
4. Ejecutar por separado el cálculo agregado para ese `ProductID`.
5. Predecir el nuevo valor.
6. Integrar el cálculo en el `UPDATE`.
7. Ejecutar el cambio sobre la tabla temporal.
8. Consultar nuevamente la fila y comparar antes/después.

### 7. Mapa mental

```text
#ProductsLab fila ProductID = X
            |
            +--> subconsulta usa X
                    |
                    v
              valor agregado
                    |
                    v
             SET UnitsInStock
```

### 8. Pregunta de control

¿Qué ocurriría si quitaras la condición que conecta `od.ProductID` con el `ProductID` de la fila exterior?

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ09_T09.sql

## Punto de partida

Se crea #ProductsLab; Products real permanece intacta.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

```sql
IF OBJECT_ID('tempdb..#ProductsLab') IS NOT NULL
    DROP TABLE #ProductsLab;

SELECT ProductID,
       ProductName,
       UnitsInStock
INTO #ProductsLab
FROM Products;
```

### Que significa

La copia temporal nos permite observar el cambio de estado antes y después sin alterar inventario real.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

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

### Que cambia respecto al paso anterior

Esta consulta deja visible la fórmula que luego se incorporará como subconsulta correlacionada. Se conserva el planteamiento didáctico del caso de la sesión.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- Solo ProductID 7 de #ProductsLab puede cambiar.
- La subconsulta se evalúa en relación con `pl.ProductID`.
- Si no existen filas que cumplan ShippedDate IS NULL para ese producto, el agregado puede producir NULL.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Dos lecturas de ProductID 7, antes y después. El valor final refleja la fórmula del caso didáctico, no una política real de inventario.

## Como interpretarlo

La correlación aparece en `od.ProductID = pl.ProductID`: la subconsulta necesita el valor de la fila que UPDATE está procesando.

## Variacion A

Ejecutar la subconsulta aislada con ProductID = 7 antes de hacer UPDATE.

## Variacion B

Cambiar la fila exterior a ProductID 8 y anticipar si habrá datos correlacionados.

## Error controlado

Eliminar `od.ProductID = pl.ProductID` dentro de la subconsulta.

## Por que ocurre

La subconsulta dejaría de depender de la fila exterior y calcularía un agregado global de todas las filas no enviadas.

## Correccion razonada

Restaurar la condición de correlación por ProductID.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T09 - Tarea espejo de EJ09

## Enunciado

Repite el patrón sobre #ProductsLab para ProductID = 8, conservando la correlación por ProductID.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ09_T09.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

No modificar Products real y no eliminar la condición correlacionada.

## Pista

Cambia únicamente el ProductID de la fila exterior; la subconsulta debe seguir enlazada a `pl.ProductID`.

## Evidencia que debes mostrar

Valores Antes y Despues de ProductID 8 y la consulta completa.

## Como saber si esta correcta

El UPDATE apunta a #ProductsLab y la subconsulta contiene `od.ProductID = pl.ProductID`.


# EJ10 - EXISTS y NOT EXISTS: comprobar existencia de filas relacionadas

## Que vamos a construir

Filtrar filas de una consulta principal según exista o no exista al menos una fila relacionada en la subconsulta.

## Que aprenderas aqui

**Concepto:** EXISTS / NOT EXISTS con correlación.

## Antes de escribir - analiza el problema

### 1. ¿Qué nos está pidiendo el caso?

Ahora no necesitamos recuperar una lista para compararla ni un único valor para colocarlo a la derecha de `=`. Queremos responder una pregunta booleana:

> ¿Existe al menos una fila relacionada que cumpla la condición?

Ese es el papel de `EXISTS`.

### 2. ¿Qué resultado final buscamos?

Ejemplos de la sesión:

- clientes para los que existe al menos un pedido;
- productos para los que existe al menos un detalle con determinada cantidad;
- posteriormente, tareas que combinan existencia con otros criterios.

### 3. ¿Qué tablas intervienen?

Según el caso:

- `Customers` y `Orders`;
- `Products` y `Order Details`;
- otras tablas relacionadas en las tareas.

### 4. ¿Qué debe devolver la subconsulta?

Para `EXISTS`, **no importa el valor de las columnas devueltas**. Lo importante es si existe al menos una fila.

Por eso se suele escribir `SELECT 1`: comunica que la intención no es transportar un dato al exterior.

### 5. ¿Dónde está la correlación?

Ejemplo:

```text
Customers.CustomerID
          |
          v
Orders.CustomerID
```

La subconsulta se evalúa respecto de la fila exterior porque utiliza `c.CustomerID`.

### 6. Plan de solución sin SQL

1. Elegir una fila de la consulta exterior.
2. Formular la pregunta: “¿existe una fila relacionada?”.
3. Escribir la relación entre claves.
4. Agregar la condición adicional si existe.
5. Usar `EXISTS` para conservar la fila cuando la respuesta sea sí.
6. Cambiar a `NOT EXISTS` para conservarla cuando la respuesta sea no.

### 7. Mapa mental

```text
fila exterior
    |
    v
buscar fila relacionada
    |
    +--> existe     -> conservar con EXISTS
    |
    +--> no existe  -> conservar con NOT EXISTS
```

### 8. Predicción antes de escribir

Si eliminas accidentalmente la condición que relaciona la subconsulta con la fila exterior y `Orders` contiene al menos una fila, ¿qué podría ocurrir con el resultado de `EXISTS`?

## Archivos que vamos a crear o modificar

- Nueva ventana de consulta en SSMS / estudiante/EJ10_T10.sql

## Punto de partida

Customers, Orders, Products, [Order Details] y Shippers están disponibles en Northwind.

## Paso 1 - Preparar la primera pieza de la consulta

### Escribe / ejecuta

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

### Que significa

EXISTS no necesita devolver columnas útiles para la consulta exterior; solo necesita saber si la subconsulta produce al menos una fila. `SELECT 1` hace visible esa intención.

### Por que lo hacemos ahora

Aislamos primero la parte que produce el dato intermedio. Si entiendes esa salida antes de anidarla, puedes predecir con mas precision la consulta final.

### Que deberia ocurrir

La sentencia debe ejecutarse sin modificar datos reales y mostrar el conjunto o valor que luego sera consumido por la consulta principal.

## Paso 2 - Agregar la segunda pieza

### Escribe / ejecuta

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

### Que cambia respecto al paso anterior

NOT EXISTS invierte la condición: conserva clientes para los que no existe una fila relacionada en Orders.

### Flujo mental

```text
entrada de Northwind
      -> subconsulta / conjunto intermedio
      -> operador o correlacion
      -> filtro / cambio controlado
      -> resultado observable
```

## Paso 3 - Completar la consulta

Escribe la version completa. Ejecutala solo despues de leerla de arriba abajo y localizar la subconsulta.

## Archivo completo al terminar este ejemplo

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

## Antes de ejecutar: prediccion

- La primera consulta devuelve clientes con al menos una fila en Orders.
- La segunda devuelve productos para los que existe al menos un detalle con Quantity >= 10.
- El valor literal `1` de SELECT 1 no se usa como dato de salida.

## Ejecuta

En SSMS, selecciona el bloque completo del ejemplo y presiona **Execute**. Si tienes varios resultados, revisalos en el orden en que aparecen.

## Resultado esperado

Dos conjuntos filtrados por existencia. La correlación aparece en CustomerID y ProductID respectivamente.

## Como interpretarlo

EXISTS expresa una pregunta booleana: “¿hay al menos una fila que cumpla?”. Puede ser más claro que recuperar una lista cuando solo interesa la existencia; el rendimiento concreto depende del plan de ejecución y del diseño de datos.

## Variacion A

Cambiar EXISTS por NOT EXISTS en la primera consulta y observar si existen clientes sin pedidos.

## Variacion B

Cambiar Quantity >= 10 por Quantity > 20 y anticipar si el conjunto disminuye.

## Error controlado

Eliminar la condición `o.CustomerID = c.CustomerID` en la primera subconsulta.

## Por que ocurre

La subconsulta deja de estar correlacionada; si Orders tiene al menos una fila, EXISTS será verdadero para todos los clientes.

## Correccion razonada

Restaurar la condición que conecta la fila exterior con la subconsulta.

## Que debes poder explicar con tus palabras

1. Que devuelve la subconsulta de este ejemplo: una fila, varias filas o una prueba de existencia?
2. Que operador consume ese resultado?
3. Que condicion conecta la subconsulta con la consulta principal, si existe correlacion?
4. Que evidencia confirma que el resultado es correcto?

# T10 - Tarea espejo de EJ10

## Enunciado

Encuentra los pedidos que contienen productos del proveedor SupplierID = 2 y que fueron enviados por la compañía `Speedy Express`, usando EXISTS.

## Archivos a modificar

- La misma ventana de consulta o `estudiante/EJ10_T10.sql`.

## Antes de escribir tu solucion

No copies el patron de memoria. Completa primero esta ficha en tus apuntes:

- **Resultado final que debes obtener:** ...
- **Tablas que necesitas:** ...
- **Dato o conjunto que debe producir la subconsulta:** ...
- **Cardinalidad esperada:** un valor / varias filas / existencia.
- **Operador o relacion que consumira el resultado:** ...
- **Prediccion:** ¿que filas esperas conservar, excluir, insertar, borrar o actualizar?

Solo despues de responder, escribe la consulta por partes y ejecuta primero la pieza interna cuando sea posible.

## Restriccion

El pedido debe filtrarse con EXISTS. No conviertas la consulta principal en una combinación que repita pedidos.

## Pista

Usa un EXISTS para [Order Details] + Products y otro EXISTS para relacionar ShipVia con Shippers.

## Evidencia que debes mostrar

Consulta y lista de OrderID resultantes, sin duplicados causados por los detalles.

## Como saber si esta correcta

La consulta externa parte de Orders y usa EXISTS correlacionados por OrderID y ShipVia.


# 6. Cierre de la sesion

Hoy construiste una secuencia completa de subconsultas sobre Northwind. Empezaste calculando un valor agregado que alimenta un filtro y luego cambiaste de modelo mental: con `IN` y `NOT IN` la subconsulta produce un conjunto; con `ANY`, `SOME` y `ALL` ese conjunto participa en comparaciones cuantificadas; con `EXISTS` la pregunta ya no es que valores devuelve la subconsulta, sino si existe al menos una fila relacionada.

Tambien practicaste tres operaciones de modificacion. La diferencia importante es que las ejecutaste sobre tablas temporales. Eso te permitio observar `DELETE`, `INSERT` y `UPDATE` sin dañar la base usada por el resto de la clase. Durante la practica provocaste errores seguros: subconsultas que devuelven demasiadas filas, perdida de correlacion y uso de sintaxis no disponible en todas las versiones. Corregir esos errores forma parte del aprendizaje.

Antes de cerrar, repasa esta pregunta para cada consulta: **que devuelve la subconsulta y como usa ese resultado la consulta exterior?** Si puedes responderla antes de ejecutar, ya no estas copiando sintaxis: estas razonando la consulta.

Como recurso opcional posterior, puedes utilizar el laboratorio ejecutable de FASE 15 para validar tus archivos o comenzar desde una version preparada.

Lideratec Academy: https://lideratecacademy.com/  
Blog: https://lideratecacademy.com/blog/  
YouTube: https://www.youtube.com/@LideratecAcademy
