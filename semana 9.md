---
course_id: ISIL-PABD
session_id: S09
module_id: UA3
source_origin: PPT
status: revisado_para_Northwind_S02_Lab
---

# FASE 13 — Guía de clase y práctica (sesión 09)

**Curso:** Programación Avanzada de Bases de Datos · **Duración presencial:** 120 minutos · **Entorno ya instalado:** Microsoft SQL Server + SSMS 22 · **ÚNICA base de esta sesión:** `Northwind_S02_Lab`.

> **Objetivo central:** diseñar, crear, ejecutar y validar procedimientos almacenados en la base `Northwind_S02_Lab` usando datos reales de lectura de `dbo.Products` y `dbo.Categories`, y realizar operaciones INSERT, UPDATE y DELETE solamente sobre las tablas auxiliares `dbo.S09_ProductosTrabajo` y `dbo.S09_CategoriasTrabajo` de esa MISMA base. No se crea una segunda base de datos.

## 1. Apertura para explicar en clase

Durante sesiones anteriores practicamos consultas y mantenimiento de registros utilizando SQL Server Management Studio. Pensemos ahora en una aplicación que necesita consultar una categoría de productos todos los días, registrar nuevas entradas al inventario o informar por qué una operación no pudo completarse. Si reescribimos instrucciones SQL en cada ocasión, repetimos lógica y aumentamos la posibilidad de errores. Los **procedimientos almacenados** permiten conservar instrucciones dentro de SQL Server y ejecutarlas mediante un nombre y, cuando haga falta, parámetros.

En esta clase trabajaremos sobre la misma base `Northwind_S02_Lab` que ya existe desde nuestras primeras prácticas. Empezaremos observando productos y categorías REALES que contiene esa base, transformaremos una consulta con JOIN en un procedimiento, agregaremos parámetros para filtrar resultados y aprenderemos tres formas diferentes de obtener información: filas de SELECT, valores de salida OUTPUT y códigos de estado RETURN. Finalmente utilizaremos procedimientos para registrar, modificar y eliminar registros de manera controlada. Para proteger los datos anteriores, estas operaciones se ejecutarán únicamente en dos tablas auxiliares de la propia base.

Al terminar, no bastará con que un código aparezca como «ejecutado correctamente». Cada estudiante deberá explicar qué fila se buscaba, por qué se aceptó o rechazó la operación, qué valores entraron y salieron y qué SELECT demuestra el estado final. Esta disciplina es fundamental para desarrollar mantenimiento de datos sin depender de copiar scripts.

## 2. Resultado de aprendizaje y criterios

Al concluir, podrás abrir SSMS 22, seleccionar explícitamente `Northwind_S02_Lab`, crear ocho procedimientos que resuelven situaciones distintas, probar parámetros de entrada y salida, y distinguir un procedimiento que devuelve filas de otro que retorna una cifra o un código. También podrás construir una operación de mantenimiento restringida por `WHERE`, explicar su validación y presentar evidencias comparando antes y después en la tabla correcta. Demostrarás dominio cuando puedas justificar una decisión de validación, capturar `OUTPUT` en una variable y probar un caso exitoso y uno fallido sin modificar las tablas originales.

**Desempeños evaluables:** identificar el contexto de la base, construir procedimientos, pasar parámetros, diferenciar SELECT/OUTPUT/RETURN, insertar, actualizar, borrar con condiciones de seguridad y contrastar los resultados.

**Ya debes conocer:** SELECT, JOIN, WHERE, GROUP BY, COUNT, INSERT, UPDATE, DELETE y variables; se repasarán brevemente cuando se utilicen. **No se adelantarán:** transacciones, triggers y optimización de planes (otras sesiones).

## 3. Guía de operación exacta en SSMS 22

1. Abre **SQL Server Management Studio 22** y conéctate a la misma instancia de las clases anteriores.
2. En **Explorador de objetos → Bases de datos**, busca `Northwind_S02_Lab`. No la cambies de nombre ni adjuntes otra base para esta sesión.
3. Pulsa **Nueva consulta**. Copia o abre `FASE_15/00_Explorar_Estructura.sql` y ejecútalo completo con **F5**. Debes obtener `BaseEsperada` distinto de NULL, `BaseEnUso = Northwind_S02_Lab`, y tablas `dbo.Products` y `dbo.Categories` con sus columnas. Si alguna falta, **detente**: nuestra guía asume la estructura Northwind estándar y no puede afirmar sin verificación cómo es tu copia.
4. Elige una pestaña nueva, abre `01_Preparar_Tabla_S09.sql` y ejecuta todo. Este archivo **NO crea otra base**; crea solo `S09_CategoriasTrabajo` y `S09_ProductosTrabajo` dentro de la misma Northwind y copia hasta veinte productos existentes para practicar modificaciones sin alterar la fuente. La primera ejecución debe mostrar categorías y hasta 20 filas auxiliares; la cantidad exacta depende de los datos reales. Si el servidor no permite crear tablas, consulta al docente por permisos en esta base de laboratorio.
5. Antes de EJ05–EJ08, cuando necesites comenzar con los datos de origen, abre `02_Restablecer_Solo_S09.sql`. Este script BORRA Y RECARGA **únicamente** `dbo.S09_ProductosTrabajo`. No lo ejecutes si quieres conservar altas o actualizaciones propias; no funciona como “deshacer” general de la base.
6. Para cada ejemplo, abre una pestaña **Nueva consulta** o un `.sql` distinto, comprueba que el script empieza con `USE Northwind_S02_Lab; GO`, escribe/observa la definición completa y ejecuta F5 **solo cuando el procedimiento esté cerrado**. Usa `GO` antes del siguiente lote; `CREATE OR ALTER PROCEDURE` debe iniciar su lote.
7. Con F5 sobre el bloque `EXEC`, mira la pestaña **Resultados**, no solo **Mensajes**. Las instrucciones SELECT devuelven cuadrículas; `OUTPUT` y `RETURN` requieren variables y SELECT externo para ser visibles.

**Seguridad:** `dbo.Products`, `dbo.Categories`, `dbo.Orders`, `dbo.[Order Details]` son de SOLO LECTURA en esta sesión. Todas las modificaciones se limitan a objetos S09_. Nunca uses DROP, TRUNCATE ni DELETE sin saber qué objeto exacto está seleccionado.

### 3.1 Diagnóstico visible sin cambiar registros

```sql
/* S09 - EXPLORACION SIN CAMBIOS DE DATOS. Ejecutar PRIMERO en SSMS 22. */
SELECT @@SERVERNAME AS Servidor, DB_NAME() AS ContextoActual, @@VERSION AS VersionSQL;
SELECT DB_ID(N'Northwind_S02_Lab') AS BaseEsperada;
IF DB_ID(N'Northwind_S02_Lab') IS NULL
BEGIN
    THROW 51000, 'No existe Northwind_S02_Lab en esta conexion. Detener y consultar al docente.', 1;
END;
GO
USE Northwind_S02_Lab;
GO
SELECT DB_NAME() AS BaseEnUso;
SELECT t.name AS Tabla, s.name AS Esquema
FROM sys.tables AS t JOIN sys.schemas AS s ON t.schema_id=s.schema_id
WHERE t.name IN (N'Products',N'Categories') ORDER BY t.name;
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA='dbo' AND TABLE_NAME IN ('Products','Categories')
ORDER BY TABLE_NAME, ORDINAL_POSITION;
IF OBJECT_ID(N'dbo.Products',N'U') IS NULL OR OBJECT_ID(N'dbo.Categories',N'U') IS NULL
    THROW 51001, 'Faltan las tablas dbo.Products y/o dbo.Categories. No continuar.', 1;
IF EXISTS (
 SELECT 1 FROM (VALUES (N'Products',N'ProductID'),(N'Products',N'ProductName'),
 (N'Products',N'CategoryID'),(N'Products',N'UnitPrice'),(N'Products',N'UnitsInStock'),
 (N'Categories',N'CategoryID'),(N'Categories',N'CategoryName')) AS r(Tabla,Columna)
 WHERE COL_LENGTH(N'dbo.'+r.Tabla,r.Columna) IS NULL)
    THROW 51002,'La estructura no coincide con Northwind estandar. Revisar columnas antes de preparar.',1;
EXEC sys.sp_executesql N'SELECT TOP (5) ProductID,ProductName,CategoryID,UnitPrice,UnitsInStock FROM dbo.Products ORDER BY ProductID;';
EXEC sys.sp_executesql N'SELECT TOP (5) CategoryID,CategoryName FROM dbo.Categories ORDER BY CategoryID;';
```

**Qué deberías observar:** primero el nombre del servidor y el contexto, después un identificador numérico de base, las columnas efectivamente encontradas y ejemplos de productos y categorías. Si `BaseEsperada` es NULL, la conexión apunta a otra instancia o la base no está disponible: NO sustituyas la base por `master`, `Northwind` o `PABD_S09_Lab` para intentar continuar.

### 3.2 Preparación única dentro de Northwind_S02_Lab

```sql
/* S09 - UNA SOLA BASE: Northwind_S02_Lab. Objetos nuevos solo con prefijo S09_.
   Ejecutar despues de 00_Explorar_Estructura.sql y solo en un entorno docente de prueba.
   No modifica dbo.Products / dbo.Categories originales. */
IF DB_ID(N'Northwind_S02_Lab') IS NULL
    THROW 51010,'Base de sesion 02 no disponible',1;
GO
USE Northwind_S02_Lab;
GO
IF DB_NAME() <> N'Northwind_S02_Lab'
    THROW 51019, 'Contexto incorrecto: cancelar cambios; vuelve a seleccionar Northwind_S02_Lab.', 1;
IF OBJECT_ID(N'dbo.Products',N'U') IS NULL OR OBJECT_ID(N'dbo.Categories',N'U') IS NULL
    THROW 51011,'No hay esquema Northwind esperado; cancelar creacion',1;
GO
IF DB_NAME() <> N'Northwind_S02_Lab'
    THROW 51019, 'Contexto incorrecto: cancelar cambios; vuelve a seleccionar Northwind_S02_Lab.', 1;
IF OBJECT_ID(N'dbo.S09_CategoriasTrabajo',N'U') IS NULL
BEGIN
    CREATE TABLE dbo.S09_CategoriasTrabajo
    (
        CategoriaID INT NOT NULL PRIMARY KEY,
        Nombre NVARCHAR(50) NOT NULL
    );
END;
GO
IF DB_NAME() <> N'Northwind_S02_Lab'
    THROW 51019, 'Contexto incorrecto: cancelar cambios; vuelve a seleccionar Northwind_S02_Lab.', 1;
IF OBJECT_ID(N'dbo.S09_ProductosTrabajo',N'U') IS NULL
BEGIN
    CREATE TABLE dbo.S09_ProductosTrabajo
    (
        ProductoID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        OrigenProductID INT NULL,  -- Identifica copia de Northwind; NULL = alta de esta practica.
        Nombre NVARCHAR(80) NOT NULL,
        CategoriaID INT NOT NULL,
        Precio DECIMAL(10,2) NOT NULL CHECK (Precio >= 0),
        Stock INT NOT NULL CHECK (Stock >= 0),
        CONSTRAINT FK_S09_Productos_Categorias FOREIGN KEY (CategoriaID)
            REFERENCES dbo.S09_CategoriasTrabajo(CategoriaID)
    );
    CREATE UNIQUE INDEX UX_S09_OrigenProductID
    ON dbo.S09_ProductosTrabajo(OrigenProductID)
    WHERE OrigenProductID IS NOT NULL;
END;
GO
IF DB_NAME() <> N'Northwind_S02_Lab'
    THROW 51019, 'Contexto incorrecto: cancelar cambios; vuelve a seleccionar Northwind_S02_Lab.', 1;
/* S09: volver a poblar categorias si no estaban cargadas */
INSERT INTO dbo.S09_CategoriasTrabajo (CategoriaID,Nombre)
SELECT c.CategoryID,c.CategoryName
FROM dbo.Categories AS c
WHERE NOT EXISTS (SELECT 1 FROM dbo.S09_CategoriasTrabajo AS l
                  WHERE l.CategoriaID=c.CategoryID);
GO
IF DB_NAME() <> N'Northwind_S02_Lab'
    THROW 51019, 'Contexto incorrecto: cancelar cambios; vuelve a seleccionar Northwind_S02_Lab.', 1;
/* Tomar productos REALES de esta Northwind, sin modificar la fuente. */
IF NOT EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo)
BEGIN
    INSERT INTO dbo.S09_ProductosTrabajo (OrigenProductID,Nombre,CategoriaID,Precio,Stock)
    SELECT TOP (20) p.ProductID,p.ProductName,p.CategoryID,
           COALESCE(CONVERT(DECIMAL(10,2),p.UnitPrice),0),
           COALESCE(CONVERT(INT,p.UnitsInStock),0)
    FROM dbo.Products AS p
    WHERE p.CategoryID IS NOT NULL
      AND EXISTS (SELECT 1 FROM dbo.S09_CategoriasTrabajo AS c WHERE c.CategoriaID=p.CategoryID)
    ORDER BY p.ProductID;
END;
GO
IF DB_NAME() <> N'Northwind_S02_Lab'
    THROW 51019, 'Contexto incorrecto: cancelar cambios; vuelve a seleccionar Northwind_S02_Lab.', 1;
SELECT COUNT(*) AS CategoriasTrabajo FROM dbo.S09_CategoriasTrabajo;
SELECT COUNT(*) AS ProductosTrabajo FROM dbo.S09_ProductosTrabajo;
SELECT TOP(20) ProductoID,OrigenProductID,Nombre,CategoriaID,Precio,Stock
FROM dbo.S09_ProductosTrabajo ORDER BY OrigenProductID;
```

**Interpretación importante:** `OrigenProductID` conserva el identificador del producto de origen en la copia de práctica. Las filas INSERTADAS por nuestros procedimientos dejan esta columna NULL; el motor genera `ProductoID` interno mediante IDENTITY. `Precio` y `Stock` se convierten a tipos adecuados para practicar. La clave foránea conecta el producto de práctica con una categoría de práctica, mientras `CHECK` impide precios y stocks negativos. Estas restricciones son una protección adicional: los procedimientos deberían rechazar las entradas inválidas antes de alcanzar la restricción.

### 3.3 Plan presencial de 120 minutos

| Minutos | Secuencia docente y evidencia |
|---|---|
| 00–10 | Comprobar Northwind_S02_Lab, estructura real y crear tablas auxiliares. |
| 10–21 | EJ01 y T01: pasar de JOIN a procedimiento, comparación con consulta directa. |
| 21–34 | EJ02 y T02: filtros parametrizados, dos casos. |
| 34–47 | EJ03 y T03: OUTPUT + agregación; cero coincidencias. |
| 47–60 | EJ04 y T04: RETURN frente a OUTPUT con caso inexistente. |
| 60–79 | EJ05 y T05: INSERT de práctica, validación y SCOPE_IDENTITY. |
| 79–95 | EJ06 y T06: UPDATE condicionado y precio anterior. |
| 95–111 | EJ07 y T07: stock disponible y rechazo. |
| 111–120 | EJ08 guiado: DELETE protegido; T08 como cierre independiente o práctica posterior. |

**Regla de conducción:** las tareas T01–T07 pueden iniciarse en los tramos indicados y completarse autónomamente si un ejemplo necesita más discusión; T08 queda como transferencia cuando solo resten los 9 minutos finales. Nunca acortes la demostración de errores para “hacer que entren” todas las tareas.

---


# EJ01 — Convertir una consulta con JOIN en procedimiento reutilizable

**Objetivo técnico — qué construiremos:** Obtener hasta diez productos reales, mostrando nombre de producto, categoría, precio y unidades; verificar que un procedimiento sin parámetros puede devolver filas con columnas relacionadas. **Tiempo presencial orientativo:** 11 minutos.

**¿Qué problema concreto resolveremos?** El docente consulta el catálogo de productos y sus categorías varias veces. Copiar un SELECT con JOIN en cada clase facilita perder una columna o alterar el orden. Hoy guardaremos esa consulta en un procedimiento de la MISMA Northwind_S02_Lab, sin duplicar bases ni modificar registros.

**Predicción antes de ejecutar:** ¿Veremos una tabla de cuatro o cinco columnas? ¿La creación inserta productos nuevos?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (10) p.ProductID, p.ProductName,
       c.CategoryName, p.UnitPrice, p.UnitsInStock
FROM dbo.Products AS p
INNER JOIN dbo.Categories AS c ON c.CategoryID = p.CategoryID
ORDER BY p.ProductID;
```

**Por qué:** Primero ejecutamos el SELECT aislado. p identifica Products y c identifica Categories; INNER JOIN une los códigos de categoría compatibles, TOP limita filas, ORDER BY estabiliza su orden. Esta consulta NO crea procedimiento: establece la respuesta de referencia con la cual compararemos la invocación.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_Catalogo
```

**Por qué tiene esta firma:** El nombre dbo.usp_S09_Catalogo identifica un procedimiento del esquema dbo y no recibe argumentos. La consulta relacionará productos con sus categorías, pero el encabezado solo crea el contrato con SQL Server: todavía no describe qué filas obtener. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
```

**Razonamiento de negocio:** En un procedimiento sin validaciones de entrada, AS abre el cuerpo y SET NOCOUNT ON evita mensajes innecesarios; no limita las filas ni oculta resultados. Esta separación permite ver la estructura sin confundirla con la consulta que realmente aporta datos. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
SELECT TOP (10) p.ProductID, p.ProductName,
       c.CategoryName, p.UnitPrice, p.UnitsInStock
FROM dbo.Products AS p
INNER JOIN dbo.Categories AS c ON c.CategoryID = p.CategoryID
ORDER BY p.ProductID;
```

**Explicación de las decisiones y consecuencias:** INNER JOIN se ejecuta dentro del servidor cada vez que invocamos el procedimiento; el número de filas depende de TOP(10), pero CategoryName proviene de Categories y no de Products.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_Catalogo
AS
BEGIN
    SET NOCOUNT ON;
    SELECT TOP (10) p.ProductID, p.ProductName,
           c.CategoryName, p.UnitPrice, p.UnitsInStock
    FROM dbo.Products AS p
    INNER JOIN dbo.Categories AS c ON c.CategoryID = p.CategoryID
    ORDER BY p.ProductID;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
USE Northwind_S02_Lab;
GO
EXEC dbo.usp_S09_Catalogo;
```

**Cómo leer la evidencia y su significado:** EXEC busca el procedimiento guardado; no se repite el SELECT. Deben verse hasta 10 filas con CategoryName. Compara la primera fila con el SELECT directo: deben coincidir en ProductID y datos. Tras crear el objeto, el mismo conjunto de filas que arrojó el SELECT directo debe aparecer en EXEC. La primera diferencia observable es que ahora invocamos un NOMBRE en lugar de pegar una consulta.

**Resultado que debes contrastar:** Dos consultas devuelven el mismo conjunto ordenado; no cambia el total de dbo.Products. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Sustituye TOP (10) por TOP (5), vuelve a ejecutar CREATE OR ALTER y luego EXEC. Comprueba cinco filas, no diez.

**Error/caso límite para analizar:** Error Invalid object name si dbo.Categories no existe, o si se seleccionó otra base.

**Corrección segura:** Ejecuta 00_Explorar_Estructura.sql, comprueba DB_NAME() y esquema dbo; no sustituyas sin verificar la estructura.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T01 — Tarea espejo: transferir lo aprendido

**Caso:** T01: crea dbo.usp_S09_CategoriasConProductos que liste CategoriaID, CategoryName y cantidad de productos por categoría, usando LEFT JOIN + GROUP BY y orden por cantidad descendente. Deben aparecer las categorías aunque tengan 0 productos.

**Pista:** Agrupa por c.CategoryID, c.CategoryName y usa COUNT(p.ProductID), no COUNT(*).

**Entrega:** abre `estudiante/tareas/T01_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** La suma de ConteoProductos de todas las categorías debe coincidir con COUNT(*) de dbo.Products si cada producto tiene una categoría válida.

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# EJ02 — Filtrar el catálogo por categoría y precio mínimo

**Objetivo técnico — qué construiremos:** Recibir @CategoriaID y @PrecioMinimo, devolver solo productos de la categoría seleccionada cuyo precio sea mayor o igual al umbral, y comparar dos ejecuciones sin modificar el código del procedimiento. **Tiempo presencial orientativo:** 13 minutos.

**¿Qué problema concreto resolveremos?** El área académica solicita el mismo reporte de productos para distintas categorías y umbrales de precio. Una consulta que obliga a reescribir números en WHERE no es reutilizable. Pasaremos esos valores por parámetros y validaremos que sean razonables.

**Predicción antes de ejecutar:** Con el mismo ID de categoría, ¿qué sucede al subir el precio mínimo? ¿Qué devuelve una categoría sin coincidencias?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT p.ProductID,p.ProductName,p.UnitPrice
FROM dbo.Products AS p
WHERE p.CategoryID = 1 AND p.UnitPrice >= 15
ORDER BY p.UnitPrice DESC, p.ProductID;
```

**Por qué:** Estas condiciones fijas ayudan a describir la necesidad: la categoría delimita el grupo y el precio mínimo restringe dentro de él. AND exige ambas condiciones. Ahora necesitamos reemplazar 1 y 15 por entradas variables.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ProductosPorCategoriaPrecio
    @CategoriaID INT,
    @PrecioMinimo DECIMAL(10,2) = 0
```

**Por qué tiene esta firma:** La firma incluye @CategoriaID obligatorio y @PrecioMinimo opcional gracias al =0. DECIMAL(10,2) permite decimales monetarios; el procedimiento no debe recibir precios como texto. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
IF @CategoriaID <= 0 OR @PrecioMinimo < 0
    BEGIN
        PRINT N'Parámetros inválidos: revisar categoría y precio mínimo.';
        RETURN 20;
    END;
```

**Razonamiento de negocio:** La comprobación IF se ubica ANTES del SELECT. Si el precio es negativo o la categoría no puede ser válida, RETURN 20 evita ejecutar el resto; no se confunde una búsqueda vacía con una petición rechazada. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
SELECT p.ProductID,p.ProductName,c.CategoryName,p.UnitPrice
FROM dbo.Products AS p
INNER JOIN dbo.Categories AS c ON c.CategoryID=p.CategoryID
WHERE p.CategoryID=@CategoriaID AND p.UnitPrice>=@PrecioMinimo
ORDER BY p.UnitPrice DESC,p.ProductID;
RETURN 0;
```

**Explicación de las decisiones y consecuencias:** WHERE combina categoría y precio con AND. Al cambiar solo el segundo parámetro la selección se vuelve más restrictiva; ORDER BY precio descendente hace que un estudiante pueda verificar visualmente el umbral.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ProductosPorCategoriaPrecio
    @CategoriaID INT,
    @PrecioMinimo DECIMAL(10,2) = 0
AS
BEGIN
    SET NOCOUNT ON;
    IF @CategoriaID <= 0 OR @PrecioMinimo < 0
    BEGIN
        PRINT N'Parámetros inválidos: revisar categoría y precio mínimo.';
        RETURN 20;
    END;
    SELECT p.ProductID,p.ProductName,c.CategoryName,p.UnitPrice
    FROM dbo.Products AS p
    INNER JOIN dbo.Categories AS c ON c.CategoryID=p.CategoryID
    WHERE p.CategoryID=@CategoriaID AND p.UnitPrice>=@PrecioMinimo
    ORDER BY p.UnitPrice DESC,p.ProductID;
    RETURN 0;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
DECLARE @Estado INT;
EXEC @Estado=dbo.usp_S09_ProductosPorCategoriaPrecio
     @CategoriaID=1,@PrecioMinimo=15;
SELECT @Estado AS CodigoEjecucion;
EXEC @Estado=dbo.usp_S09_ProductosPorCategoriaPrecio
     @CategoriaID=1,@PrecioMinimo=25;
SELECT @Estado AS CodigoEjecucion;
```

**Cómo leer la evidencia y su significado:** Se reutiliza la misma definición. Cambia solo @PrecioMinimo y la segunda tabla será igual o menor en filas que la primera; ambas llamadas devuelven CodigoEjecucion=0, salvo problemas ajenos a la validación. Dos ejecuciones muestran el mismo algoritmo con distintos valores: en la segunda, ninguna fila puede tener precio inferior a 25; los códigos indican validez de entrada.

**Resultado que debes contrastar:** Dos cuadrículas con productos de categoría 1; la segunda no puede contener precios menores de 25. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Invoca sin @PrecioMinimo y después con -1. Compara comportamiento del valor predeterminado con el rechazo 20.

**Error/caso límite para analizar:** Usar WHERE CategoryID=CategoryID por omitir @ produce comparaciones fila consigo misma y desactiva el filtro.

**Corrección segura:** Escribe p.CategoryID=@CategoriaID; confirma categoría de todas las filas retornadas.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T02 — Tarea espejo: transferir lo aprendido

**Caso:** T02: desarrolla dbo.usp_S09_BuscarNombrePrecio con @Texto NVARCHAR(40) y @PrecioMaximo DECIMAL(10,2); usa LIKE y <= para localizar productos por fragmento de nombre, rechazando texto vacío y precio negativo.

**Pista:** p.ProductName LIKE N'%' + @Texto + N'%'; comprueba TRIM/LTRIM(RTRIM) o LEN tras recortar espacios.

**Entrega:** abre `estudiante/tareas/T02_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** Prueba con texto N'Ch' y un precio alto; con texto vacío debe devolver un código de rechazo, no todas las filas.

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# EJ03 — Devolver una cantidad mediante OUTPUT sin perder la consulta

**Objetivo técnico — qué construiremos:** Recibir el ID de categoría, calcular COUNT(*) desde Products y capturar @Total mediante OUTPUT incluso cuando no hay coincidencias, distinguiendo ese valor de una fila visual en Results. **Tiempo presencial orientativo:** 13 minutos.

**¿Qué problema concreto resolveremos?** El coordinador no necesita una lista completa, sino cuántos productos hay en determinada categoría para un indicador. Si devolvemos únicamente una cuadrícula, otro bloque SQL tendría que leerla manualmente. Con OUTPUT capturaremos el conteo en una variable.

**Predicción antes de ejecutar:** ¿El conteo de categoría inexistente será NULL o 0? ¿Qué ocurre si omitimos OUTPUT en EXEC?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT CategoryID,COUNT(*) AS Total
FROM dbo.Products
WHERE CategoryID=1
GROUP BY CategoryID;
```

**Por qué:** Esta salida sirve para confirmar la cifra real de la instancia. GROUP BY puede no devolver ninguna fila para una categoría ausente; en el procedimiento utilizaremos COUNT(*) sin GROUP BY para asignar el cero a la salida.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ContarProductos
    @CategoriaID INT,
    @Total INT OUTPUT
```

**Por qué tiene esta firma:** El primer parámetro identifica qué categoría contaremos y el segundo, @Total INT OUTPUT, es un canal de salida hacia la variable de quien llama. No es una tabla nueva ni un SELECT de visualización. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
```

**Razonamiento de negocio:** El cuerpo de este procedimiento no necesita rechazar categorías ausentes porque su contrato es responder 0 cuando no haya productos. NO declarar RETURN es aceptable: SQL Server devuelve normalmente estado 0 si termina sin RETURN explícito. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
SELECT @Total=COUNT(*)
FROM dbo.Products
WHERE CategoryID=@CategoriaID;
```

**Explicación de las decisiones y consecuencias:** SELECT @Total=COUNT(*) es una ASIGNACIÓN. A diferencia de SELECT COUNT(*) AS Total, este código no genera automáticamente una cuadrícula; para verlo el lote llamador hará SELECT @Cantidad.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ContarProductos
    @CategoriaID INT,
    @Total INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT @Total=COUNT(*)
    FROM dbo.Products
    WHERE CategoryID=@CategoriaID;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
DECLARE @Cantidad INT;
EXEC dbo.usp_S09_ContarProductos
     @CategoriaID=1,@Total=@Cantidad OUTPUT;
SELECT @Cantidad AS TotalCategoria1;
EXEC dbo.usp_S09_ContarProductos
     @CategoriaID=999,@Total=@Cantidad OUTPUT;
SELECT @Cantidad AS TotalCategoriaInexistente;
```

**Cómo leer la evidencia y su significado:** DECLARE reserva la variable receptora; en EXEC se utiliza OUTPUT nuevamente para copiar el valor de salida. Los dos SELECT externos son los que realmente muestran  un valor en Results; el segundo debe ser 0. Al usar categoría inexistente, COUNT(*) entrega cero. El estudiante debe poder explicar por qué el resultado visible aparece solamente en el SELECT posterior a EXEC.

**Resultado que debes contrastar:** TotalCategoria1 coincide con COUNT(*) en Products; TotalCategoriaInexistente=0. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Quita SOLO la palabra OUTPUT de una invocación y observa el valor previo de @Cantidad; vuelve a colocarla para recuperar el resultado. Reinicia la variable en NULL antes de cada intento para que la prueba sea concluyente.

**Error/caso límite para analizar:** Creer que asignar @Total genera automáticamente una tabla visible.

**Corrección segura:** Ejecutar SELECT @Cantidad después de EXEC; confirmar OUTPUT en declaración y llamada.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T03 — Tarea espejo: transferir lo aprendido

**Caso:** T03: crea dbo.usp_S09_SumarUnidadesPorCategoria que devuelva @StockTotal INT OUTPUT usando SUM(UnitsInStock) y COALESCE para convertir NULL en cero.

**Pista:** COALESCE(SUM(CONVERT(INT,UnitsInStock)),0); usa @CategoriaID como filtro.

**Entrega:** abre `estudiante/tareas/T03_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** Compara con SUM en un SELECT directo para categoría 1; para 999 espera 0.

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# EJ04 — Separar el dato OUTPUT del código RETURN

**Objetivo técnico — qué construiremos:** Obtener el stock de dbo.Products mediante @Stock OUTPUT y distinguir RETURN 0 (encontrado), 10 (no existe) y 20 (identificador inválido), sin escribir en ninguna tabla. **Tiempo presencial orientativo:** 13 minutos.

**¿Qué problema concreto resolveremos?** Si un producto tiene 0 unidades, no debemos confundirlo con un producto inexistente. Se necesitan dos informaciones distintas: el stock real y si la búsqueda tuvo éxito. Probaremos un ID existente, uno que no existe y uno inválido.

**Predicción antes de ejecutar:** ¿Puede un producto existente tener stock cero y RETURN=0? ¿Qué respuesta daremos ante ID -5?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP(5) ProductID,ProductName,UnitsInStock
FROM dbo.Products
ORDER BY ProductID;
SELECT COUNT(*) AS Encontrados FROM dbo.Products WHERE ProductID=999999;
```

**Por qué:** El primer SELECT descubre IDs reales sin presuponer un producto específico. COUNT(*) permite diferenciar entre fila inexistente y fila cuyo stock es 0; la identidad de un producto no depende de su cantidad.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ConsultarStockEstado
    @ProductoID INT,
    @Stock INT OUTPUT
```

**Por qué tiene esta firma:** @ProductoID determina la búsqueda y @Stock OUTPUT devuelve el dato. Reservamos RETURN 0/10/20 para clasificar el resultado de ejecución, sin utilizar la cifra del inventario como indicador de éxito. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
SET @Stock=NULL;
    IF @ProductoID <= 0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.Products WHERE ProductID=@ProductoID)
        RETURN 10;
```

**Razonamiento de negocio:** SET @Stock=NULL limpia un posible resultado anterior. Primero rechazamos un ID <=0, luego buscamos existencia por ProductID; este orden permite un diagnóstico distinto para entradas inválidas y ausencias legítimas. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
SELECT @Stock=CONVERT(INT,COALESCE(UnitsInStock,0))
FROM dbo.Products WHERE ProductID=@ProductoID;
RETURN 0;
```

**Explicación de las decisiones y consecuencias:** COALESCE(UnitsInStock,0) solo se consulta si la fila existe. Por eso Stock=0 con Codigo=0 es una respuesta válida y muy distinta de Stock=NULL con Codigo=10.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ConsultarStockEstado
    @ProductoID INT,
    @Stock INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @Stock=NULL;
    IF @ProductoID <= 0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.Products WHERE ProductID=@ProductoID)
        RETURN 10;
    SELECT @Stock=CONVERT(INT,COALESCE(UnitsInStock,0))
    FROM dbo.Products WHERE ProductID=@ProductoID;
    RETURN 0;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
DECLARE @Id INT=(SELECT MIN(ProductID) FROM dbo.Products);
DECLARE @Estado INT,@Stock INT;
EXEC @Estado=dbo.usp_S09_ConsultarStockEstado @ProductoID=@Id,@Stock=@Stock OUTPUT;
SELECT @Id AS IdBuscado,@Estado AS Codigo,@Stock AS StockEncontrado;
EXEC @Estado=dbo.usp_S09_ConsultarStockEstado @ProductoID=999999,@Stock=@Stock OUTPUT;
SELECT @Estado AS Codigo,@Stock AS StockEncontrado;
EXEC @Estado=dbo.usp_S09_ConsultarStockEstado @ProductoID=-5,@Stock=@Stock OUTPUT;
SELECT @Estado AS Codigo,@Stock AS StockEncontrado;
```

**Cómo leer la evidencia y su significado:** El ID válido se obtiene desde esta base y no se inventa. Esperamos 0 y una cifra (posiblemente 0), después 10/NULL y por último 20/NULL. Es un ejemplo donde RETURN y OUTPUT cambian independientemente. Se necesitan tres ejecuciones porque dos casos no demuestran los tres estados. Deben aparecer Codigo=0,10,20 en ese orden, con una cifra solo para el primero.

**Resultado que debes contrastar:** Códigos 0,10,20; solo el caso exitoso proporciona un dato en @Stock. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Busca deliberadamente en dbo.Products un producto con UnitsInStock=0, si existe; si no existe, explica por qué no puede demostrarse el contraste numéricamente en esta copia.

**Error/caso límite para analizar:** Escribir RETURN @Stock para devolver stock como dato o usar RETURN NULL.

**Corrección segura:** Deja RETURN exclusivamente para códigos INT; mantén la información de negocio en OUTPUT.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T04 — Tarea espejo: transferir lo aprendido

**Caso:** T04: implementa dbo.usp_S09_ConsultarPrecioEstado con @Precio DECIMAL(10,2) OUTPUT y los estados 0,10,20 para consultar UnitPrice; exige salida NULL en códigos distintos de cero.

**Pista:** Una vez encontrada la fila, asigna CONVERT(DECIMAL(10,2),COALESCE(UnitPrice,0)).

**Entrega:** abre `estudiante/tareas/T04_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** Usa MIN(ProductID) de Products, 999999 y -3; compara @Precio con el SELECT directo.

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# EJ05 — Registrar un producto de práctica y recuperar su ID

**Objetivo técnico — qué construiremos:** Insertar una fila solamente en dbo.S09_ProductosTrabajo, validar precio, stock, nombre y categoría, capturar el identificador nuevo y demostrar que dbo.Products original no ha cambiado. **Tiempo presencial orientativo:** 19 minutos.

**¿Qué problema concreto resolveremos?** La base original tiene productos reales, pero no debemos insertar allí nombres de prueba. En una tabla auxiliar de Northwind_S02_Lab registraremos un nuevo artículo, rechazaremos categorías inexistentes y devolveremos el identificador que SQL Server genera automáticamente.

**Predicción antes de ejecutar:** ¿Qué valor asignará IDENTITY? ¿Es obligatorio conocerlo antes de insertar? ¿Una categoría 9999 debería registrarse?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (8) CategoriaID,Nombre FROM dbo.S09_CategoriasTrabajo ORDER BY CategoriaID;
SELECT COUNT(*) AS Copia FROM dbo.S09_ProductosTrabajo;
SELECT COUNT(*) AS FuenteOriginal FROM dbo.Products;
```

**Por qué:** La categoría se escoge de las copias locales y ProductosTrabajo es el ÚNICO destino de la escritura. Nunca escribiremos INSERT INTO dbo.Products. Los conteos anteriores nos servirán para demostrar qué tabla aumentó.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_RegistrarProducto
    @Nombre NVARCHAR(80),
    @CategoriaID INT,
    @Precio DECIMAL(10,2),
    @Stock INT,
    @NuevoID INT OUTPUT
```

**Por qué tiene esta firma:** En este contrato, @Nombre, @CategoriaID, @Precio y @Stock son entradas; @NuevoID OUTPUT recibirá la identidad generada. El nombre y los tipos se alinean con las columnas AUXILIARES, no con las de dbo.Products original. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
SET @NuevoID=NULL;
    IF NULLIF(LTRIM(RTRIM(@Nombre)),N'') IS NULL OR @Precio IS NULL OR @Precio < 0
        OR @Stock IS NULL OR @Stock < 0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_CategoriasTrabajo WHERE CategoriaID=@CategoriaID)
        RETURN 10;
```

**Razonamiento de negocio:** Antes de tocar datos, hacemos dos validaciones distintas: calidad de nombre/precio/stock (estado 20) y existencia de la categoría en S09_CategoriasTrabajo (estado 10). Iniciamos @NuevoID a NULL para que una operación rechazada no parezca exitosa. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
INSERT INTO dbo.S09_ProductosTrabajo (OrigenProductID,Nombre,CategoriaID,Precio,Stock)
VALUES (NULL,@Nombre,@CategoriaID,@Precio,@Stock);
SET @NuevoID=CONVERT(INT,SCOPE_IDENTITY());
RETURN 0;
```

**Explicación de las decisiones y consecuencias:** INSERT solo apunta a S09_ProductosTrabajo. OrigenProductID=NULL marca un alta creada aquí; SCOPE_IDENTITY se ejecuta inmediatamente después de INSERT y captura la nueva clave en el ámbito actual. RETURN 0 confirma una operación permitida.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_RegistrarProducto
    @Nombre NVARCHAR(80),
    @CategoriaID INT,
    @Precio DECIMAL(10,2),
    @Stock INT,
    @NuevoID INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @NuevoID=NULL;
    IF NULLIF(LTRIM(RTRIM(@Nombre)),N'') IS NULL OR @Precio IS NULL OR @Precio < 0
        OR @Stock IS NULL OR @Stock < 0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_CategoriasTrabajo WHERE CategoriaID=@CategoriaID)
        RETURN 10;
    INSERT INTO dbo.S09_ProductosTrabajo (OrigenProductID,Nombre,CategoriaID,Precio,Stock)
    VALUES (NULL,@Nombre,@CategoriaID,@Precio,@Stock);
    SET @NuevoID=CONVERT(INT,SCOPE_IDENTITY());
    RETURN 0;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
DECLARE @Categoria INT=(SELECT MIN(CategoriaID) FROM dbo.S09_CategoriasTrabajo);
DECLARE @NuevoID INT,@Codigo INT;
EXEC @Codigo=dbo.usp_S09_RegistrarProducto @Nombre=N'Galletas laboratorio S09',
     @CategoriaID=@Categoria,@Precio=12.50,@Stock=18,@NuevoID=@NuevoID OUTPUT;
SELECT @Codigo AS Codigo,@NuevoID AS NuevoID;
SELECT ProductoID,OrigenProductID,Nombre,Precio,Stock
FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@NuevoID;
EXEC @Codigo=dbo.usp_S09_RegistrarProducto @Nombre=N'Categoría inválida S09',
     @CategoriaID=9999,@Precio=12,@Stock=1,@NuevoID=@NuevoID OUTPUT;
SELECT @Codigo AS CodigoRechazo,@NuevoID AS IDNoCreado;
```

**Cómo leer la evidencia y su significado:** La primera llamada debe producir Codigo=0, nuevo ID numérico y una fila con OrigenProductID=NULL. La segunda debe producir CodigoRechazo=10, IDNoCreado=NULL y ninguna fila para ese nombre. El número exacto de IDENTITY puede variar entre alumnos y reinicios. La primera llamada debe incrementar solo el conteo auxiliar y entregar un NuevoID real. La segunda, con categoría 9999, debe mantener ese conteo y devolver 10/NULL.

**Resultado que debes contrastar:** Una fila nueva exclusivamente en S09_ProductosTrabajo; dbo.Products mantiene el conteo previo. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Prueba Stock=-1 y verifica 20; compara conteos antes/después para confirmar que no se insertó.

**Error/caso límite para analizar:** Usar @@IDENTITY o asumir que NuevoID siempre será 21; también confundir dbo.Products con la tabla auxiliar.

**Corrección segura:** Usa SCOPE_IDENTITY; revisa SELECT DB_NAME() y prefijo S09_ antes de F5; nunca supongas secuencias IDENTITY consecutivas tras pruebas.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T05 — Tarea espejo: transferir lo aprendido

**Caso:** T05: implementa dbo.usp_S09_RegistrarProductoConStockMinimo para aceptar solamente nuevos productos con Stock >= 5; usa códigos 0,10,20,30 (30 cuando Stock<5), y @NuevoID OUTPUT.

**Pista:** Valida que @Stock no sea NULL o negativo; después comprueba IF @Stock<5 RETURN 30, antes del INSERT.

**Entrega:** abre `estudiante/tareas/T05_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** Prueba un producto con stock 6 (éxito y fila), otro con stock 2 (código 30 y sin fila) y categoría 9999 (10).

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# EJ06 — Actualizar precio, conservar el valor anterior y comprobar una sola fila

**Objetivo técnico — qué construiremos:** Cambiar Precio de un ProductoID de la tabla auxiliar, devolver @PrecioAnterior, informar @Filas INT OUTPUT, y contrastar una fila antes/después contra la base fuente. **Tiempo presencial orientativo:** 16 minutos.

**¿Qué problema concreto resolveremos?** El encargado corrige el precio de un artículo. No basta con emitir UPDATE; también necesita saber cuál era el precio anterior y asegurar que solo se modificó el producto solicitado. El procedimiento debe protegernos de un WHERE ausente y de cifras negativas.

**Predicción antes de ejecutar:** Si el ID no existe, ¿cuántas filas se actualizan? ¿Un precio negativo debería reemplazar al anterior?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP(5) ProductoID,OrigenProductID,Nombre,Precio FROM dbo.S09_ProductosTrabajo
ORDER BY ProductoID;
```

**Por qué:** Se utiliza un ID consultado, no uno supuesto. Anota PrecioAntes y Nombre de la fila. Es importante escribir desde el principio que la tabla objetivo será S09_ProductosTrabajo y que WHERE tendrá una clave concreta.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ActualizarPrecio
    @ProductoID INT,
    @NuevoPrecio DECIMAL(10,2),
    @PrecioAnterior DECIMAL(10,2) OUTPUT,
    @Filas INT OUTPUT
```

**Por qué tiene esta firma:** La firma combina el ID buscado, el precio nuevo, el precio anterior de salida y la cantidad de filas afectadas. Separar @PrecioAnterior de @Filas nos permite verificar dos hechos distintos: historia de valor y cardinalidad. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
SET @PrecioAnterior=NULL;
    SET @Filas=0;
    IF @NuevoPrecio IS NULL OR @NuevoPrecio<0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID)
        RETURN 10;
```

**Razonamiento de negocio:** Se ponen @PrecioAnterior=NULL y @Filas=0 antes de cualquier RETURN. Luego se rechaza un precio negativo y se revisa existencia, evitando que una llamada fallida herede datos anteriores. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
SELECT @PrecioAnterior=Precio FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID;
UPDATE dbo.S09_ProductosTrabajo
   SET Precio=@NuevoPrecio
 WHERE ProductoID=@ProductoID;
SET @Filas=@@ROWCOUNT;
RETURN 0;
```

**Explicación de las decisiones y consecuencias:** Capturamos PrecioAnterior ANTES del UPDATE. El UPDATE limita el objetivo por clave y @@ROWCOUNT debe guardarse en la instrucción siguiente, porque otra sentencia puede cambiarlo. Verificar 1 evita dar por exitosa una alteración masiva.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_ActualizarPrecio
    @ProductoID INT,
    @NuevoPrecio DECIMAL(10,2),
    @PrecioAnterior DECIMAL(10,2) OUTPUT,
    @Filas INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @PrecioAnterior=NULL;
    SET @Filas=0;
    IF @NuevoPrecio IS NULL OR @NuevoPrecio<0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID)
        RETURN 10;
    SELECT @PrecioAnterior=Precio FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID;
    UPDATE dbo.S09_ProductosTrabajo
       SET Precio=@NuevoPrecio
     WHERE ProductoID=@ProductoID;
    SET @Filas=@@ROWCOUNT;
    RETURN 0;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
DECLARE @Id INT=(SELECT MIN(ProductoID) FROM dbo.S09_ProductosTrabajo);
DECLARE @Previo DECIMAL(10,2),@Filas INT,@Estado INT;
SELECT ProductoID,Precio AS PrecioAntes FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@Id;
EXEC @Estado=dbo.usp_S09_ActualizarPrecio @ProductoID=@Id,@NuevoPrecio=29.90,
     @PrecioAnterior=@Previo OUTPUT,@Filas=@Filas OUTPUT;
SELECT @Estado AS Codigo,@Previo AS PrecioAnterior,@Filas AS FilasAfectadas;
SELECT ProductoID,Precio AS PrecioDespues FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@Id;
```

**Cómo leer la evidencia y su significado:** Espera Codigo=0 y FilasAfectadas=1. PrecioAnterior debe coincidir con PrecioAntes; PrecioDespues debe ser 29.90. Si el precio ya era 29.90, la actualización sigue afectando esa fila; no significa que el valor cambió desde un importe distinto. Comprueba una fila antes y después; que @PrecioAnterior coincida con el SELECT inicial y que @Filas sea 1 es más informativo que el simple mensaje «completado». 

**Resultado que debes contrastar:** Una sola fila afectada; se conserva el precio previo en OUTPUT; fuente dbo.Products intacta. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Prueba @NuevoPrecio=-0.50 y ProductoID=999999 en llamadas separadas; espera respectivamente 20 y 10 sin nuevas modificaciones.

**Error/caso límite para analizar:** Capturar @@ROWCOUNT después de SELECT @PrecioAnterior... o ejecutar UPDATE sin WHERE.

**Corrección segura:** Coloca SET @Filas=@@ROWCOUNT inmediatamente después del UPDATE, y exige verificar WHERE antes de F5.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T06 — Tarea espejo: transferir lo aprendido

**Caso:** T06: crea dbo.usp_S09_CambiarCategoria que actualice CategoriaID en la copia y devuelva la categoría anterior y el número de filas; rechaza IDs de categorías que no existen.

**Pista:** Compara tabla S09_CategoriasTrabajo y S09_ProductosTrabajo. Estados sugeridos: 0 OK, 10 producto ausente, 20 categoría ausente.

**Entrega:** abre `estudiante/tareas/T06_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** Elige un producto y dos categorías reales diferentes; antes/después debe mostrar exactamente un cambio de clave foránea.

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# EJ07 — Descontar stock sin permitir una cantidad imposible

**Objetivo técnico — qué construiremos:** Recibir ID y Cantidad, efectuar un descuento únicamente si Stock>=Cantidad, devolver stock final y distinguir cantidad inválida, producto ausente y stock insuficiente. **Tiempo presencial orientativo:** 16 minutos.

**¿Qué problema concreto resolveremos?** Una venta solicita unidades de un producto. Si el stock es insuficiente, la actualización no debe realizarse y el procedimiento debe comunicar por qué. La condición debe evaluarse en el propio UPDATE para que nunca descuente más de lo disponible en esta demostración.

**Predicción antes de ejecutar:** ¿Qué debe pasar si se intenta vender una unidad más que el stock actual? ¿El stock será NULL, cero o permanecerá igual?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP(10) ProductoID,Nombre,Stock
FROM dbo.S09_ProductosTrabajo ORDER BY ProductoID;
```

**Por qué:** Elegimos una fila con Stock>0 (si no existe, restablecemos datos y comprobamos). La regla esperada es StockFinal = StockAntes - Cantidad y StockFinal nunca negativo. La fuente para la operación es solo S09_ProductosTrabajo.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_DescontarStock
    @ProductoID INT,
    @Cantidad INT,
    @StockFinal INT OUTPUT
```

**Por qué tiene esta firma:** @Cantidad representa una salida de almacén, no el stock absoluto. @StockFinal OUTPUT mostrará el saldo que quedó después del UPDATE; el código RETURN distinguirá tres rechazos. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
SET @StockFinal=NULL;
    IF @Cantidad IS NULL OR @Cantidad<=0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID)
        RETURN 10;
```

**Razonamiento de negocio:** SET @StockFinal=NULL evita residuos de ejecuciones anteriores. Un descuento nulo o no positivo da 20; un producto que no existe da 10. Solo entonces debe intentarse el descuento. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
UPDATE dbo.S09_ProductosTrabajo
   SET Stock=Stock-@Cantidad
 WHERE ProductoID=@ProductoID AND Stock>=@Cantidad;
IF @@ROWCOUNT=0 RETURN 30;
SELECT @StockFinal=Stock FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID;
RETURN 0;
```

**Explicación de las decisiones y consecuencias:** La condición Stock>=@Cantidad se escribe dentro de WHERE del UPDATE para proteger el valor persistido. @@ROWCOUNT=0 da 30 (no había suficiente). Solo después se consulta el stock final y se retorna 0.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_DescontarStock
    @ProductoID INT,
    @Cantidad INT,
    @StockFinal INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @StockFinal=NULL;
    IF @Cantidad IS NULL OR @Cantidad<=0 RETURN 20;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID)
        RETURN 10;
    UPDATE dbo.S09_ProductosTrabajo
       SET Stock=Stock-@Cantidad
     WHERE ProductoID=@ProductoID AND Stock>=@Cantidad;
    IF @@ROWCOUNT=0 RETURN 30;
    SELECT @StockFinal=Stock FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID;
    RETURN 0;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
DECLARE @Id INT=(SELECT TOP(1) ProductoID FROM dbo.S09_ProductosTrabajo
                     WHERE Stock>=2 ORDER BY ProductoID);
DECLARE @Antes INT=(SELECT Stock FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@Id);
DECLARE @Nuevo INT,@Estado INT;
EXEC @Estado=dbo.usp_S09_DescontarStock @ProductoID=@Id,
     @Cantidad=2,@StockFinal=@Nuevo OUTPUT;
SELECT @Antes AS StockAntes,@Estado AS Codigo,@Nuevo AS StockDespues;
EXEC @Estado=dbo.usp_S09_DescontarStock @ProductoID=@Id,
     @Cantidad=100000,@StockFinal=@Nuevo OUTPUT;
SELECT @Estado AS CodigoRechazo,@Nuevo AS StockNoActualizado;
SELECT Stock AS StockPersistido FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@Id;
```

**Cómo leer la evidencia y su significado:** Si existe al menos una fila con stock>=2, la primera ejecución retorna 0 y reduce exactamente 2. La segunda retorna 30 y deja el stock persistido sin modificaciones adicionales. Si @Id queda NULL, restablece el entorno o elige otra fila antes de ejecutar. Primero veremos un descuento exactamente de dos unidades. Luego forzaremos una cantidad demasiado alta y comprobaremos que el stock persistido quedó igual.

**Resultado que debes contrastar:** StockDespues=StockAntes-2, luego CodigoRechazo=30 y StockPersistido sin una segunda reducción. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Prueba Cantidad=0 y espera 20. Restablece solo tabla S09_ antes de repetir el escenario si quieres resultados iniciales.

**Error/caso límite para analizar:** Escribir SET Stock=@Cantidad o quitar Stock>=@Cantidad de WHERE.

**Corrección segura:** Comprueba ambas condiciones y utiliza una SELECT antes/después; no resuelvas stock negativo con un UPDATE correctivo posterior.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T07 — Tarea espejo: transferir lo aprendido

**Caso:** T07: crea dbo.usp_S09_ReponerStock que SUME unidades al inventario de la copia, rechace Cantidad<=0 y devuelva @StockFinal y estado; valida un ID inexistente.

**Pista:** El cambio es Stock=Stock+@Cantidad con filtro por ProductoID; el caso inexistente debe dar 10.

**Entrega:** abre `estudiante/tareas/T07_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** Para un mismo producto, stock después de reponer 5 debe ser exactamente stock antes + 5.

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# EJ08 — Eliminar solamente un registro nuevo y verificar el efecto

**Objetivo técnico — qué construiremos:** Implementar DELETE en dbo.S09_ProductosTrabajo únicamente sobre altas creadas por nosotros; devolver número de filas afectadas y códigos para inexistente o fila de origen protegido. **Tiempo presencial orientativo:** 9 minutos.

**¿Qué problema concreto resolveremos?** Una prueba de registro quedó duplicada. Tenemos que eliminarla sin afectar datos originales ni las filas copiadas desde Northwind. La marca OrigenProductID=NULL será una regla que delimita qué se puede borrar durante esta sesión.

**Predicción antes de ejecutar:** ¿Qué sucede al solicitar la eliminación de un ID que proviene de dbo.Products? ¿Y si el ID no existe?

## Paso 1 — Inspeccionar el dato real antes de programar

**Antes:** no asumimos que la copia Northwind tenga exactamente los mismos precios y stocks en todos los equipos. Vamos a descubrir el resultado real con un SELECT visible antes de encapsularlo.

**Acción en SSMS:** en una pestaña de **Nueva consulta** escribe `USE Northwind_S02_Lab;` y `GO`; debajo pega esta consulta de referencia. **Ejecuta este paso por separado** con F5, porque NO es una definición incompleta.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (10) ProductoID,OrigenProductID,Nombre
FROM dbo.S09_ProductosTrabajo ORDER BY ProductoID;
-- Las altas con OrigenProductID NULL son candidatas a eliminación en S09.
```

**Por qué:** Las filas replicadas tienen OrigenProductID con un valor fuente. Las altas de EJ05 tienen NULL. Esta distinción protege incluso nuestra copia didáctica; jamás se propone borrar dbo.Products.

**Después:** anota o captura una fila, una cantidad o un estado inicial. Esto servirá para detectar una definición incorrecta incluso si CREATE PROCEDURE termina sin mostrar errores. **Falta:** sustituir la instrucción pegada por un procedimiento que podamos invocar repetidamente.

## Paso 2 — Establecer el contrato del procedimiento y su ubicación

**Antes:** la consulta directa ya mostró el resultado buscado, pero no existe todavía un procedimiento que lo reproduzca.

**Acción exacta:** abre **OTRA pestaña Nueva consulta**, distinta de la del diagnóstico. Copia el siguiente encabezado en una ventana vacía. **No ejecutes F5** porque todavía faltan `AS`, el cuerpo y `END`.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_EliminarAlta
    @ProductoID INT,
    @Filas INT OUTPUT
```

**Por qué tiene esta firma:** @ProductoID identifica la fila candidata y @Filas OUTPUT permitirá comprobar cuántos registros se borraron. El procedimiento solo tiene permiso lógico para borrar altas propias de la sesión. `USE Northwind_S02_Lab` garantiza que la definición se guarde en la MISMA base de la sesión 02. `GO` hace que `CREATE OR ALTER` sea la primera instrucción del lote siguiente; `dbo.usp_S09_...` identifica el objeto en nuestro esquema de trabajo.

**Después:** ya hay un nombre permanente y una lista precisa de datos que la llamada suministrará o recibirá. Aún NO se ha creado el procedimiento. **Falta:** el bloque de operaciones que SQL Server ejecutará al recibir el `EXEC`.

## Paso 3 — Abrir el cuerpo y controlar entradas antes de consultar o modificar

**Antes:** el encabezado terminó en el nombre y los parámetros; ahora necesitamos introducir la lógica en la misma pestaña.

**Acción exacta:** coloca el cursor justo al FINAL del encabezado del paso 2. Pulsa Enter y añade SOLO estas líneas, sin borrar lo escrito ni abrir una segunda definición. **Todavía no se ejecuta**: `BEGIN` queda abierto hasta el paso 5.

```sql
AS
BEGIN
    SET NOCOUNT ON;
SET @Filas=0;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID)
        RETURN 10;
    IF EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo
               WHERE ProductoID=@ProductoID AND OrigenProductID IS NOT NULL)
        RETURN 20;
```

**Razonamiento de negocio:** El flujo de comprobaciones diferencia ID inexistente (10) y copia original protegida (20). La marca OrigenProductID NULL es intencional: identifica altas de la sesión, no productos que provenían de Northwind. `AS` separa la firma del cuerpo; `BEGIN` abre un grupo de instrucciones y `SET NOCOUNT ON` evita mensajes de recuento intermedios, sin ocultar las filas de un SELECT.

**Después:** ahora el cuerpo contiene la preparación y las validaciones que corresponden a la operación. **Falta:** la consulta o el mantenimiento que realmente produce el resultado.

## Paso 4 — Implementar la operación que resuelve el problema

**Antes:** ya tenemos el encabezado, `BEGIN` y las validaciones; lo que todavía no existe es la instrucción que lee o modifica el dato central.

**Acción exacta:** en la MISMA pestaña, al FINAL del paso 3, añade este bloque. Mantén las condiciones WHERE y el orden de sus instrucciones. No pulses F5 hasta cerrar el procedimiento.

```sql
DELETE FROM dbo.S09_ProductosTrabajo
WHERE ProductoID=@ProductoID AND OrigenProductID IS NULL;
SET @Filas=@@ROWCOUNT;
RETURN 0;
```

**Explicación de las decisiones y consecuencias:** El DELETE vuelve a incluir OrigenProductID IS NULL para que la protección no dependa únicamente de las comprobaciones previas. @@ROWCOUNT=1 evidencia una baja válida, sin tocar el catálogo original.

**Después:** ya está programado el comportamiento, pero el lote sigue abierto. Si lo ejecutas incompleto obtendrás un error de sintaxis o guardarás menos lógica de la necesaria. **Falta:** cerrar `BEGIN` y enviar la definición completa.

## Paso 5 — Cerrar, compilar y localizar la definición guardada

**Antes:** en el editor sigue abierto `BEGIN`; aún no se ha ejecutado la definición completa.

**Acción exacta:** al FINAL del paso 4 agrega el cierre siguiente y después selecciona con `Ctrl+A` toda esta pestaña de definición (pasos 2 a 5) y presiona **F5** una sola vez.

```sql
END;
GO
```

**Por qué:** `END` termina el cuerpo y `GO` indica a SSMS que envíe el lote al motor. Una ejecución sin error confirma que SQL Server acepta la definición, **no** que sus casos de negocio se comporten correctamente.

**Después:** el procedimiento debe figurar en `Northwind_S02_Lab → Programación → Procedimientos almacenados` (puede requerir **Actualizar** en el Explorador de objetos). **Falta:** llamarlo y observar datos, variables o estado.

**Vista completa para contrastar el montaje de los fragmentos (no es un nuevo procedimiento):**

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S09_EliminarAlta
    @ProductoID INT,
    @Filas INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET @Filas=0;
    IF NOT EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@ProductoID)
        RETURN 10;
    IF EXISTS (SELECT 1 FROM dbo.S09_ProductosTrabajo
               WHERE ProductoID=@ProductoID AND OrigenProductID IS NOT NULL)
        RETURN 20;
    DELETE FROM dbo.S09_ProductosTrabajo
    WHERE ProductoID=@ProductoID AND OrigenProductID IS NULL;
    SET @Filas=@@ROWCOUNT;
    RETURN 0;
END;
GO
```

## Paso 6 — Ejecutar pruebas con evidencias, no solo mensajes

**Antes:** SQL Server ya conserva el objeto, pero no sabemos si responde correctamente a entradas distintas o si cambia exactamente la fila esperada.

**Acción exacta:** abre la pestaña de pruebas (NO la de definición) y ejecuta este lote COMPLETO con F5. Si encuentras `DECLARE`, inclúyelo dentro de la selección para que sus variables estén disponibles durante EXEC y SELECT.

```sql
USE Northwind_S02_Lab;
GO
DECLARE @Categoria INT=(SELECT MIN(CategoriaID) FROM dbo.S09_CategoriasTrabajo);
DECLARE @Id INT,@AltasEstado INT,@Borradas INT,@BajaEstado INT;
EXEC @AltasEstado=dbo.usp_S09_RegistrarProducto @Nombre=N'Producto temporal a borrar S09',
 @CategoriaID=@Categoria,@Precio=5,@Stock=0,@NuevoID=@Id OUTPUT;
EXEC @BajaEstado=dbo.usp_S09_EliminarAlta @ProductoID=@Id,@Filas=@Borradas OUTPUT;
SELECT @AltasEstado AS Alta,@BajaEstado AS Baja,@Borradas AS FilasEliminadas;
SELECT COUNT(*) AS DebeSerCero FROM dbo.S09_ProductosTrabajo WHERE ProductoID=@Id;
DECLARE @IdProtegido INT=(SELECT MIN(ProductoID) FROM dbo.S09_ProductosTrabajo
                          WHERE OrigenProductID IS NOT NULL);
EXEC @BajaEstado=dbo.usp_S09_EliminarAlta @ProductoID=@IdProtegido,@Filas=@Borradas OUTPUT;
SELECT @BajaEstado AS CodigoProtegido,@Borradas AS SinBorrado;
```

**Cómo leer la evidencia y su significado:** Tras ejecutar EJ05 y definir este procedimiento, la alta temporal devuelve 0, la baja 0 y FilasEliminadas 1. El contador del registro queda 0. Intentar borrar una copia de origen debe devolver 20 y 0 filas. Ejecuta el último rechazo solo si existen filas copiadas (preparación correcta). Después de crear una alta temporal podremos borrarla y obtener Filas=1; al probar la copia protegida veremos Codigo=20/Filas=0. En ambos casos dbo.Products original conserva sus filas.

**Resultado que debes contrastar:** Alta y baja exitosa solo en tabla auxiliar; intento de borrar copia de origen: código 20 y cero filas. Cuando trabajes con datos reales, identifica la fila o cantidad consultada en el paso 1 antes de afirmar que todos los equipos tendrán exactamente el mismo valor.

## Paso 7 — Modificar, diagnosticar y demostrar dominio

**Variación que altera el resultado o una regla:** Invoca sobre 999999 para confirmar 10 y Filas=0. Verifica que dbo.Products original conserva la misma cantidad de filas.

**Error/caso límite para analizar:** Ejecutar DELETE FROM dbo.Products o borrar una fila sin condición de origen.

**Corrección segura:** Verifica el prefijo de tabla S09_ y WHERE OrigenProductID IS NULL; jamás ejecutes un borrado de prueba sobre Orders, OrderDetails o Products.

**Cómo saber si dominaste el ejemplo:** puedes explicar la entrada, el resultado y la tabla implicada sin leer el código, y presentar una prueba en Results o SELECT antes/después. No alcanza con mostrar que el comando se ejecutó sin errores.

## T08 — Tarea espejo: transferir lo aprendido

**Caso:** T08: crea dbo.usp_S09_EliminarAltaSinStock; además de OrigenProductID NULL exige Stock=0; devuelve 30 si el registro nuevo aún tiene unidades.

**Pista:** Comprueba la regla de stock antes de DELETE y repite Stock=0 en WHERE.

**Entrega:** abre `estudiante/tareas/T08_plantilla.sql`. Reemplaza TODOS los TODO por un procedimiento real, crea dos llamadas con entradas distintas, ejecuta un caso incorrecto y presenta SELECT antes/después si hay cambios. **Criterio de éxito:** Inserta nueva alta con stock 4 (rechazada con código 30); actualiza con stock 0 en laboratorio y entonces elimina exactamente 1.

**Pregunta de reflexión:** ¿qué condición restringe el resultado, por qué se necesita OUTPUT o RETURN si aparece, y cuál consulta demuestra que la operación fue correcta?

---


# 5. Cierre docente y reflexión técnica

Compara cómo se usan las tres salidas fundamentales. **SELECT** produce filas y columnas; **OUTPUT** permite recuperar un dato escalar en una variable del llamador; **RETURN** comunica un estado entero. Un procedimiento también puede combinar mecanismos sin que sean equivalentes. En la segunda mitad construimos mantenimiento de registros sobre una COPIA de trabajo dentro de la misma base; la prueba esencial siempre fue verificar primero la fila objetivo, después ejecutar y finalmente consultar qué cambió.

**Pregunta de síntesis 1:** ¿Por qué `WHERE ProductoID=@ProductoID` es obligatorio al actualizar un precio? **Respuesta explicada:** sin ese filtro, UPDATE afectaría potencialmente muchas filas, aunque el procedimiento se ejecute sin error. 

**Pregunta de síntesis 2:** ¿Por qué un producto con stock cero no es lo mismo que uno inexistente? **Respuesta:** la existencia se valida por la clave; OUTPUT aporta el dato y RETURN informa si la búsqueda tuvo éxito.

**Pregunta de síntesis 3:** ¿Por qué recuperamos el ID con `SCOPE_IDENTITY()`? **Respuesta:** toma la identidad generada en el ámbito actual, evitando la ambigüedad de `@@IDENTITY` en escenarios con desencadenantes.

**Evidencias finales:** un SELECT de consulta sobre Northwind real, un EXEC parametrizado, un ejemplo de OUTPUT, un caso RETURN de error, y un antes/después de S09_ProductosTrabajo con 1 fila afectada. Las tareas T01–T08 tienen archivos separados y soluciones docentes en FASE 15.

**Advertencia de ejecución:** la guía se ha revisado por coherencia y sintaxis, pero NO se ha conectado a la instancia SQL Server del estudiante; los valores observados reales deben confirmarse en SSMS.
