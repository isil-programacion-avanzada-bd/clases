# Lideratec Academy · Programación Avanzada de Bases de Datos
## Sesión 12 — Funciones escalares, funciones de tabla y cursores en SQL Server

**Entorno confirmado del curso:** SQL Server Management Studio 22, motor SQL Server configurado en la sesión 1 y base `Northwind_S02_Lab`. **Duración de encuentro:** 120 minutos. **Fuente académica:** presentación institucional `30627-S12-PRESENTACION.pptx`, diapositivas 1–30. Los ejemplos de `pubs` (diapositivas 27–29) se adaptan de forma explícita al esquema Northwind; NO se presupone que exista `pubs`.

## 1. Propósito de la sesión — lectura inicial del docente

Imaginemos que una empresa necesita consultar productos, empleados y pedidos una y otra vez. Si en cada reporte repetimos la misma fórmula de descuento o la misma combinación de nombres, el código crece y resulta más difícil mantenerlo. Hoy aprenderemos a guardar esas reglas dentro de **funciones de SQL Server**. Primero construiremos funciones escalares, cuya respuesta es un solo valor, y después funciones de tabla, cuya respuesta incluye varias filas. El objetivo es aprender no solo la sintaxis, sino dónde se invoca cada función, qué recibe, qué devuelve y qué ocurre cuando no encuentra datos.

Luego estudiaremos una herramienta distinta: los **cursores**. Mientras una consulta SQL suele actuar sobre un conjunto completo, el cursor permite leer filas sucesivas y ejecutar una acción particular por cada una. Eso exige abrirlo, recuperar filas, controlar la terminación y liberar recursos. Conoceremos su utilidad y también por qué se recomienda preferir instrucciones basadas en conjuntos cuando el problema no exige un recorrido individual.

Trabajaremos con la base Northwind del curso y haremos todas las modificaciones experimentales sobre copias temporales. Al finalizar veremos funciones creadas y verificadas, recorridos de cursor que terminan correctamente y una comparación real entre `UPDATE` fila por fila y `UPDATE` basado en conjuntos. La meta es justificar cada decisión técnica, no solamente copiar comandos.

## 2. Resultado de aprendizaje y evidencias observables

Al finalizar la sesión podrás construir una función escalar parametrizada y justificar sus valores de entrada y retorno; crear funciones de tabla en línea y multisentencia; ejecutar de principio a fin un cursor local de lectura; analizar una actualización simulada y compararla con una sentencia de conjuntos; y verificar que los resultados son correctos sin alterar datos originales. Por ejemplo, podrás recibir un identificador de categoría, obtener sus productos, calcular un precio promocional y producir el reporte en forma tabular. El dominio se evidencia cuando puedes explicar por qué una función escalar se usa en una expresión, una función de tabla en FROM, por qué `FETCH NEXT` aparece dos veces y qué recurso libera `DEALLOCATE`.

**Desempeños comprobables:** crear; ejecutar; predecir; interpretar; contrastar; diagnosticar; validar; justificar.

## 3. Antes de iniciar

**Ya debes conocer:** SELECT, WHERE, JOIN, GROUP BY, variables, creación de procedimientos (sesiones previas), transacciones y el funcionamiento general de Northwind.

**Debes tener:** SQL Server Management Studio 22 abierto, conexión activa al motor, permisos de `CREATE FUNCTION`/`ALTER` en `Northwind_S02_Lab`, y tablas estándar `Employees`, `Products`, `Customers`, `Suppliers`, `Categories`, `Orders`, `[Order Details]`. No se requieren descargas adicionales ni la base `pubs`.

**No asumiremos:** cursores distribuidos, optimización avanzada del plan de ejecución ni funciones con efectos secundarios.

### Ruta A: entorno ya configurado

1. Abre **SSMS 22** y conecta la instancia utilizada durante el curso.
2. En **Object Explorer**, despliega **Databases** y comprueba `Northwind_S02_Lab`.
3. Selecciona **New Query**; ejecuta la siguiente consulta con **F5** y revisa **Results**:

```sql
SELECT @@SERVERNAME AS Instancia, DB_NAME() AS BaseActual;
SELECT name FROM sys.databases WHERE name = N'Northwind_S02_Lab';
USE Northwind_S02_Lab;
GO
SELECT OBJECT_ID('dbo.Products') AS TablaProductos,
       OBJECT_ID('dbo.Employees') AS TablaEmpleados,
       OBJECT_ID('dbo.[Order Details]') AS TablaDetallePedidos;
```

**Cómo interpretar:** la primera cuadrícula identifica la instancia y base previa; `sys.databases` confirma el catálogo; los tres valores `OBJECT_ID` deben ser diferentes de NULL. Si alguna tabla no está disponible, revisa el nombre real antes de seguir: no crees una tabla inventada para resolver la discrepancia.

### Ruta B: equipo sin conexión

Instala o configura primero el **motor SQL Server** que utilizaste en la sesión 1; SSMS por sí solo es un cliente y no aloja una base de datos. Conecta el servidor institucional, comprueba que la base restaurada o adjuntada realmente se denomina `Northwind_S02_Lab`, abre una nueva consulta y repite la verificación anterior. Si careces de permisos de función, solicita habilitación al administrador: este material no requiere alterar permisos ni modificar objetos del sistema.

### Regla de ejecución que evita errores

Cada `EJxx.sql` es un script **independiente**. Dentro de SSMS abre una pestaña nueva, pega el código completo de un solo ejemplo y ejecuta **F5**. `GO` no es una orden de T-SQL: SSMS la interpreta como separador de lotes. Al definir una función, `CREATE OR ALTER FUNCTION` debe empezar su lote; por eso antes se coloca `GO`. Los ejemplos **EJ08** requieren ejecutar antes EJ02 y EJ03. Los cursores y las tablas `#` pertenecen a la conexión/pestaña actual de ejecución; no ejecutes solo fragmentos aislados que dependen de variables anteriores.

## 4. Plan realista para 120 minutos

| Minutos | Momento en aula | Evidencia |
|---|---|---|
| 0–12 | Contexto, objetivos y diagnóstico del entorno | Conexión y tres OBJECT_ID válidos |
| 12–27 | EJ01 y EJ02 guiados | Dos funciones escalares y casos límite |
| 27–45 | EJ03 y EJ04 | Funciones de tabla en línea y multisentencia |
| 45–63 | EJ05: cursor de lectura; práctica T05 | Salida de Messages y ciclo finito |
| 63–75 | EJ06: agregación anual; T06 | Contraste reporte agrupado vs cursor |
| 75–97 | EJ07: modificación temporal y operación de conjuntos | Comparación de precios equivalentes |
| 97–110 | EJ08: integración y contraste | Resultado promocional sin UPDATE |
| 110–120 | Puesta en común, predicción de errores y cierre | Explicación de elecciones técnicas |

**Trabajo autónomo posterior:** terminar T01–T04, T07 y T08, repetir las pruebas de parámetros y revisar los casos límite. La guía incorpora tareas adicionales que no es razonable resolver y comentar enteras en 120 minutos.

---

# EJ01 — Crear una función escalar para recuperar el nombre completo de un empleado

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

En una pantalla de pedidos necesitamos identificar al empleado responsable sin repetir una concatenación de nombres en cada consulta. Primero verificaremos los campos reales de Employees; luego construiremos una función escalar que recibe EmployeeID y produce un único texto. Finalmente la invocaremos para un empleado existente y otro inexistente, y explicaremos por qué el segundo resultado es NULL. El objetivo no consiste solo en obtener una cadena, sino en comprender el contrato de entrada, el tipo de retorno y la forma correcta de usar una función escalar dentro de SELECT.

## ¿Qué problema concreto resolveremos?

Una aplicación que arma reportes repite FirstName + LastName en varias consultas; encapsularemos esa regla en una función.

**Archivo que representa el resultado:** `EJ01.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Seleccionar la base y comprobar el dato de entrada

**Antes de escribir (lectura docente).** USE selecciona la base activa y GO separa lotes de SSMS. Esta consulta es de reconocimiento: verificamos nombres de columnas y elegimos IDs válidos antes de crear objetos.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) EmployeeID, FirstName, LastName FROM dbo.Employees ORDER BY EmployeeID;
```

**Después y por qué importa.** USE selecciona la base activa y GO separa lotes de SSMS. Esta consulta es de reconocimiento: verificamos nombres de columnas y elegimos IDs válidos antes de crear objetos. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Definir contrato de la función y variable local

**Antes de escribir (lectura docente).** CREATE OR ALTER evita un error por objeto existente. @EmpleadoID es la entrada, RETURNS fija el tipo de resultado, y @NombreCompleto conserva la cadena que vamos a devolver.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
CREATE OR ALTER FUNCTION dbo.fn_S12_NombreEmpleado(@EmpleadoID INT)
RETURNS NVARCHAR(100)
AS
BEGIN
    DECLARE @NombreCompleto NVARCHAR(100);
```

**Después y por qué importa.** CREATE OR ALTER evita un error por objeto existente. @EmpleadoID es la entrada, RETURNS fija el tipo de resultado, y @NombreCompleto conserva la cadena que vamos a devolver. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Recuperar la fila y devolver el valor

**Antes de escribir (lectura docente).** La igualdad filtra por clave. CONCAT evita que un nombre nulo convierta toda la concatenación en NULL. Cuando no hay empleado, la variable sigue en NULL; RETURN entrega ese único valor.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
    SELECT @NombreCompleto = CONCAT(FirstName, N' ', LastName)
    FROM dbo.Employees WHERE EmployeeID = @EmpleadoID;
    RETURN @NombreCompleto;
END;
GO
```

**Después y por qué importa.** La igualdad filtra por clave. CONCAT evita que un nombre nulo convierta toda la concatenación en NULL. Cuando no hay empleado, la variable sigue en NULL; RETURN entrega ese único valor. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Invocar y contrastar un caso límite

**Antes de escribir (lectura docente).** Una función escalar participa como expresión en SELECT. El valor -1 permite comprobar el caso de búsqueda sin coincidencia.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
SELECT dbo.fn_S12_NombreEmpleado(5) AS NombreEmpleado;
SELECT dbo.fn_S12_NombreEmpleado(-1) AS EmpleadoInexistente;
```

**Después y por qué importa.** Una función escalar participa como expresión en SELECT. El valor -1 permite comprobar el caso de búsqueda sin coincidencia. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ01

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) EmployeeID, FirstName, LastName FROM dbo.Employees ORDER BY EmployeeID;

CREATE OR ALTER FUNCTION dbo.fn_S12_NombreEmpleado(@EmpleadoID INT)
RETURNS NVARCHAR(100)
AS
BEGIN
    DECLARE @NombreCompleto NVARCHAR(100);

    SELECT @NombreCompleto = CONCAT(FirstName, N' ', LastName)
    FROM dbo.Employees WHERE EmployeeID = @EmpleadoID;
    RETURN @NombreCompleto;
END;
GO

SELECT dbo.fn_S12_NombreEmpleado(5) AS NombreEmpleado;
SELECT dbo.fn_S12_NombreEmpleado(-1) AS EmpleadoInexistente;
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** La primera consulta devuelve el nombre del empleado 5; la segunda devuelve NULL.

**Error común y corrección:** No ejecutar CREATE OR ALTER junto con otras sentencias en el mismo lote; GO delimita el lote.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# EJ02 — Calcular un precio con descuento mediante una función escalar

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

Un área comercial necesita aplicar descuentos sin escribir la misma fórmula en diez reportes. Primero verificaremos un precio en Products; construiremos una función escalar de dos parámetros y devolveremos el precio descontado con dos decimales. Agregaremos una comprobación para porcentajes fuera de rango y probaremos descuento cero, descuento válido y descuento inválido. El resultado hará visible la diferencia entre un cálculo de negocio reutilizable y una operación que actualiza datos: la función calcula un valor pero no cambia el precio guardado.

## ¿Qué problema concreto resolveremos?

Calcular descuentos en SELECT de manera coherente, sin actualizar la tabla Products.

**Archivo que representa el resultado:** `EJ02.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Observar un precio real

**Antes de escribir (lectura docente).** Antes de construir una operación hay que conocer el tipo de entrada y confirmar la existencia de precios no nulos.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) ProductID, ProductName, UnitPrice FROM dbo.Products WHERE UnitPrice IS NOT NULL ORDER BY ProductID;
```

**Después y por qué importa.** Antes de construir una operación hay que conocer el tipo de entrada y confirmar la existencia de precios no nulos. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Declarar la firma del cálculo

**Antes de escribir (lectura docente).** El primer parámetro es el precio y el segundo es el porcentaje. Ambos son decimales para evitar redondeos accidentales propios de FLOAT.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
CREATE OR ALTER FUNCTION dbo.fn_S12_PrecioDescuento(
    @Precio DECIMAL(12,2), @Descuento DECIMAL(5,2))
RETURNS DECIMAL(12,2)
AS
BEGIN
```

**Después y por qué importa.** El primer parámetro es el precio y el segundo es el porcentaje. Ambos son decimales para evitar redondeos accidentales propios de FLOAT. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Definir validación y devolución

**Antes de escribir (lectura docente).** IF rechaza entradas no pertinentes. ROUND conserva dos decimales. Dividir entre 100.0 fuerza cálculo decimal; la función no ejecuta UPDATE.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
    IF @Precio IS NULL OR @Descuento IS NULL
       OR @Precio < 0 OR @Descuento < 0 OR @Descuento > 100
        RETURN NULL;
    RETURN ROUND(@Precio * (1 - @Descuento / 100.0), 2);
END;
GO
```

**Después y por qué importa.** IF rechaza entradas no pertinentes. ROUND conserva dos decimales. Dividir entre 100.0 fuerza cálculo decimal; la función no ejecuta UPDATE. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Ejecutar tres pruebas

**Antes de escribir (lectura docente).** Para validar hay que probar un caso común, un borde válido y un dato fuera de contrato.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
SELECT dbo.fn_S12_PrecioDescuento(100.00, 15.00) AS ConDescuento,
       dbo.fn_S12_PrecioDescuento(100.00, 0.00) AS SinDescuento,
       dbo.fn_S12_PrecioDescuento(100.00, 110.00) AS Invalido;
```

**Después y por qué importa.** Para validar hay que probar un caso común, un borde válido y un dato fuera de contrato. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ02

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) ProductID, ProductName, UnitPrice FROM dbo.Products WHERE UnitPrice IS NOT NULL ORDER BY ProductID;

CREATE OR ALTER FUNCTION dbo.fn_S12_PrecioDescuento(
    @Precio DECIMAL(12,2), @Descuento DECIMAL(5,2))
RETURNS DECIMAL(12,2)
AS
BEGIN

    IF @Precio IS NULL OR @Descuento IS NULL
       OR @Precio < 0 OR @Descuento < 0 OR @Descuento > 100
        RETURN NULL;
    RETURN ROUND(@Precio * (1 - @Descuento / 100.0), 2);
END;
GO

SELECT dbo.fn_S12_PrecioDescuento(100.00, 15.00) AS ConDescuento,
       dbo.fn_S12_PrecioDescuento(100.00, 0.00) AS SinDescuento,
       dbo.fn_S12_PrecioDescuento(100.00, 110.00) AS Invalido;
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** ConDescuento = 85.00; SinDescuento = 100.00; Invalido = NULL.

**Error común y corrección:** No confundir un resultado calculado en SELECT con una actualización persistida.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# EJ03 — Crear una función de tabla en línea para filtrar productos por categoría

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

Un catálogo necesita recuperar de forma parametrizable los productos de una categoría. Primero miraremos CategoryID en Products; luego declararemos una función que devuelve filas, no un valor. Usaremos RETURNS TABLE AS RETURN con una consulta filtrada, y comprobaremos que el resultado puede combinarse con ORDER BY como cualquier otra fuente tabular. Compararemos dos categorías y explicaremos por qué aquí invocamos la función dentro de FROM, a diferencia de la función escalar del primer ejemplo.

## ¿Qué problema concreto resolveremos?

Permitir que un informe solicite distintas categorías mediante un parámetro sin duplicar SELECT.

**Archivo que representa el resultado:** `EJ03.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Confirmar categorías utilizadas

**Antes de escribir (lectura docente).** Inspeccionamos la relación ProductID–CategoryID para decidir qué filtrar.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (8) ProductID, ProductName, CategoryID, UnitPrice
FROM dbo.Products ORDER BY ProductID;
```

**Después y por qué importa.** Inspeccionamos la relación ProductID–CategoryID para decidir qué filtrar. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Declarar una función que retorna filas

**Antes de escribir (lectura docente).** RETURNS TABLE anuncia un conjunto de registros; a diferencia de una función escalar, la definición no usa RETURNS INT ni una variable acumuladora.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
CREATE OR ALTER FUNCTION dbo.fn_S12_ProductosCategoria(@CategoriaID INT)
RETURNS TABLE
AS
RETURN
(
```

**Después y por qué importa.** RETURNS TABLE anuncia un conjunto de registros; a diferencia de una función escalar, la definición no usa RETURNS INT ni una variable acumuladora. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Definir la selección parametrizada

**Antes de escribir (lectura docente).** La condición compara cada fila con el parámetro de entrada. Las columnas seleccionadas son el contrato tabular del resultado.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
    SELECT ProductID, ProductName, UnitPrice, UnitsInStock, CategoryID
    FROM dbo.Products
    WHERE CategoryID = @CategoriaID
);
GO
```

**Después y por qué importa.** La condición compara cada fila con el parámetro de entrada. Las columnas seleccionadas son el contrato tabular del resultado. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Consultar la función como una tabla

**Antes de escribir (lectura docente).** La primera ejecución muestra productos y la segunda agrega los registros devueltos. FROM, no una expresión escalar, es el lugar de invocación.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
SELECT ProductID, ProductName, UnitPrice
FROM dbo.fn_S12_ProductosCategoria(1) ORDER BY ProductName;
SELECT COUNT(*) AS CantidadCategoria2
FROM dbo.fn_S12_ProductosCategoria(2);
```

**Después y por qué importa.** La primera ejecución muestra productos y la segunda agrega los registros devueltos. FROM, no una expresión escalar, es el lugar de invocación. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ03

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (8) ProductID, ProductName, CategoryID, UnitPrice
FROM dbo.Products ORDER BY ProductID;

CREATE OR ALTER FUNCTION dbo.fn_S12_ProductosCategoria(@CategoriaID INT)
RETURNS TABLE
AS
RETURN
(

    SELECT ProductID, ProductName, UnitPrice, UnitsInStock, CategoryID
    FROM dbo.Products
    WHERE CategoryID = @CategoriaID
);
GO

SELECT ProductID, ProductName, UnitPrice
FROM dbo.fn_S12_ProductosCategoria(1) ORDER BY ProductName;
SELECT COUNT(*) AS CantidadCategoria2
FROM dbo.fn_S12_ProductosCategoria(2);
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** Se muestran productos de CategoryID = 1 y un conteo para CategoryID = 2; no se modifican datos.

**Error común y corrección:** No escribir SELECT dbo.fn_S12_ProductosCategoria(1): las funciones de tabla se invocan en FROM.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# EJ04 — Construir una función de tabla multisentencia con datos de proveedor

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

El coordinador del catálogo necesita mostrar los productos de una categoría junto con su proveedor, pero desea un resultado preparado para futuras reglas internas. Primero comprobamos la relación Products–Suppliers; después declararemos una variable de tabla con las columnas que tendrá el resultado. Insertaremos filas combinando las dos tablas, devolveremos el conjunto y revisaremos que ProductID no aparece duplicado por un JOIN innecesario con pedidos. Así reconoceremos cuándo una función de tabla multisentencia tiene sentido y qué diferencia existe respecto de una función en línea.

## ¿Qué problema concreto resolveremos?

Obtener un resumen por categoría y proveedor con esquema de salida declarado expresamente.

**Archivo que representa el resultado:** `EJ04.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Comprobar las claves de relación

**Antes de escribir (lectura docente).** La consulta previa demuestra que la relación se establece por SupplierID. Evitamos unir Order Details porque un producto con varios pedidos produciría múltiples filas.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) p.ProductID, p.ProductName, p.SupplierID, s.CompanyName
FROM dbo.Products AS p INNER JOIN dbo.Suppliers AS s ON s.SupplierID=p.SupplierID
ORDER BY p.ProductID;
```

**Después y por qué importa.** La consulta previa demuestra que la relación se establece por SupplierID. Evitamos unir Order Details porque un producto con varios pedidos produciría múltiples filas. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Declarar retorno tabular

**Antes de escribir (lectura docente).** RETURNS @Ficha TABLE fija la forma del resultado. SMALLINT es compatible con UnitsInStock de Northwind clásico; NVARCHAR permite texto Unicode.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
CREATE OR ALTER FUNCTION dbo.fn_S12_FichaProductosCategoria(@CategoriaID INT)
RETURNS @Ficha TABLE
(
    ProductID INT, ProductName NVARCHAR(50),
    Supplier NVARCHAR(100), UnitsInStock SMALLINT
)
AS
BEGIN
```

**Después y por qué importa.** RETURNS @Ficha TABLE fija la forma del resultado. SMALLINT es compatible con UnitsInStock de Northwind clásico; NVARCHAR permite texto Unicode. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Insertar el conjunto resultado

**Antes de escribir (lectura docente).** INSERT INTO llena la variable de tabla; las columnas del SELECT se alinean por posición y tipo con las declaradas. El JOIN aporta el nombre de proveedor.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
    INSERT INTO @Ficha(ProductID,ProductName,Supplier,UnitsInStock)
    SELECT p.ProductID,p.ProductName,s.CompanyName,p.UnitsInStock
    FROM dbo.Products AS p
    INNER JOIN dbo.Suppliers AS s ON s.SupplierID=p.SupplierID
    WHERE p.CategoryID=@CategoriaID;
```

**Después y por qué importa.** INSERT INTO llena la variable de tabla; las columnas del SELECT se alinean por posición y tipo con las declaradas. El JOIN aporta el nombre de proveedor. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Devolver y comprobar unicidad

**Antes de escribir (lectura docente).** RETURN entrega la variable tabular. Si las dos cantidades coinciden no hubo duplicación de ProductID en este resultado.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
    RETURN;
END;
GO
SELECT * FROM dbo.fn_S12_FichaProductosCategoria(2) ORDER BY ProductID;
SELECT COUNT(*) AS Filas,COUNT(DISTINCT ProductID) AS ProductosUnicos
FROM dbo.fn_S12_FichaProductosCategoria(2);
```

**Después y por qué importa.** RETURN entrega la variable tabular. Si las dos cantidades coinciden no hubo duplicación de ProductID en este resultado. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ04

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) p.ProductID, p.ProductName, p.SupplierID, s.CompanyName
FROM dbo.Products AS p INNER JOIN dbo.Suppliers AS s ON s.SupplierID=p.SupplierID
ORDER BY p.ProductID;

CREATE OR ALTER FUNCTION dbo.fn_S12_FichaProductosCategoria(@CategoriaID INT)
RETURNS @Ficha TABLE
(
    ProductID INT, ProductName NVARCHAR(50),
    Supplier NVARCHAR(100), UnitsInStock SMALLINT
)
AS
BEGIN

    INSERT INTO @Ficha(ProductID,ProductName,Supplier,UnitsInStock)
    SELECT p.ProductID,p.ProductName,s.CompanyName,p.UnitsInStock
    FROM dbo.Products AS p
    INNER JOIN dbo.Suppliers AS s ON s.SupplierID=p.SupplierID
    WHERE p.CategoryID=@CategoriaID;

    RETURN;
END;
GO
SELECT * FROM dbo.fn_S12_FichaProductosCategoria(2) ORDER BY ProductID;
SELECT COUNT(*) AS Filas,COUNT(DISTINCT ProductID) AS ProductosUnicos
FROM dbo.fn_S12_FichaProductosCategoria(2);
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** Listado de productos con proveedor y conteos Filas = ProductosUnicos.

**Error común y corrección:** En la PPT la unión adicional con Orders/OrderDetails puede duplicar productos; aquí se corrige con una adaptación declarada.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# EJ05 — Recorrer clientes con un cursor local de solo lectura

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

Un proceso heredado exige emitir una línea por cliente y queremos comprender exactamente cómo avanza un cursor. Primero identificaremos ContactName en Customers y limitaremos la muestra a cinco filas. Después declararemos un cursor LOCAL FAST_FORWARD para lectura, lo abriremos, recuperaremos la primera fila y continuaremos con WHILE mientras @@FETCH_STATUS sea cero. Cerraremos y liberaremos el cursor, y contrastaremos la salida de Messages con un SELECT equivalente. El recorrido muestra por qué es necesario hacer FETCH también dentro del ciclo: sin ese avance, la repetición no termina.

## ¿Qué problema concreto resolveremos?

Comprender el procesamiento fila por fila y sus diferencias con un SELECT de conjunto.

**Archivo que representa el resultado:** `EJ05.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Revisar las filas que se recorrerán

**Antes de escribir (lectura docente).** El orden de las cinco filas permite comprobar después cada mensaje. LIMITAR evita llenar la salida durante la demostración.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) CustomerID, ContactName FROM dbo.Customers ORDER BY CustomerID;
```

**Después y por qué importa.** El orden de las cinco filas permite comprobar después cada mensaje. LIMITAR evita llenar la salida durante la demostración. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Preparar variable y cursor

**Antes de escribir (lectura docente).** La variable recibirá una fila cada vez. LOCAL reduce el ámbito al lote y FAST_FORWARD crea un cursor de lectura con avance solo hacia adelante.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
DECLARE @Nombre NVARCHAR(50);
DECLARE cur_S12_Clientes CURSOR LOCAL FAST_FORWARD FOR
    SELECT TOP (5) ContactName FROM dbo.Customers ORDER BY CustomerID;
```

**Después y por qué importa.** La variable recibirá una fila cada vez. LOCAL reduce el ámbito al lote y FAST_FORWARD crea un cursor de lectura con avance solo hacia adelante. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Abrir y recuperar la primera fila

**Antes de escribir (lectura docente).** OPEN prepara el resultado. FETCH NEXT transfiere un valor a @Nombre y actualiza @@FETCH_STATUS.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
OPEN cur_S12_Clientes;
FETCH NEXT FROM cur_S12_Clientes INTO @Nombre;
```

**Después y por qué importa.** OPEN prepara el resultado. FETCH NEXT transfiere un valor a @Nombre y actualiza @@FETCH_STATUS. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Recorrer y avanzar correctamente

**Antes de escribir (lectura docente).** La condición 0 significa que se recuperó una fila. PRINT envía mensajes, no una cuadrícula. El segundo FETCH cambia el estado y permite la terminación.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT @Nombre;
    FETCH NEXT FROM cur_S12_Clientes INTO @Nombre;
END;
```

**Después y por qué importa.** La condición 0 significa que se recuperó una fila. PRINT envía mensajes, no una cuadrícula. El segundo FETCH cambia el estado y permite la terminación. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 5 — Cerrar, liberar y contrastar

**Antes de escribir (lectura docente).** CLOSE termina la obtención de filas; DEALLOCATE libera el objeto. SELECT genera una cuadrícula y suele ser más apropiado para reportes tabulares.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
CLOSE cur_S12_Clientes;
DEALLOCATE cur_S12_Clientes;
SELECT TOP (5) ContactName FROM dbo.Customers ORDER BY CustomerID;
```

**Después y por qué importa.** CLOSE termina la obtención de filas; DEALLOCATE libera el objeto. SELECT genera una cuadrícula y suele ser más apropiado para reportes tabulares. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ05

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) CustomerID, ContactName FROM dbo.Customers ORDER BY CustomerID;

DECLARE @Nombre NVARCHAR(50);
DECLARE cur_S12_Clientes CURSOR LOCAL FAST_FORWARD FOR
    SELECT TOP (5) ContactName FROM dbo.Customers ORDER BY CustomerID;

OPEN cur_S12_Clientes;
FETCH NEXT FROM cur_S12_Clientes INTO @Nombre;

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT @Nombre;
    FETCH NEXT FROM cur_S12_Clientes INTO @Nombre;
END;

CLOSE cur_S12_Clientes;
DEALLOCATE cur_S12_Clientes;
SELECT TOP (5) ContactName FROM dbo.Customers ORDER BY CustomerID;
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** Cinco nombres en la pestaña Messages y cinco filas equivalentes en Results.

**Error común y corrección:** Olvidar FETCH dentro de WHILE provoca un bucle sin progreso; ejecutar el bloque completo.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# EJ06 — Recorrer totales anuales de ventas con un cursor sobre datos agregados

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

Una jefatura solicita anunciar el total vendido por año, y la PPT propone recorrer ventas con cursor. En Northwind no existe una tabla sales de pubs, por lo que primero reuniremos pedidos y sus detalles y calcularemos subtotales con UnitPrice, Quantity y Discount. Agruparemos por año de OrderDate y luego recorreremos solo los totales ya agregados, no cada línea comercial. Imprimiremos un mensaje por año y explicaremos cuándo conviene reemplazar toda la salida del cursor por un SELECT agrupado.

## ¿Qué problema concreto resolveremos?

Traducir el caso de ventas anuales de pubs al esquema real de Northwind sin inventar columnas.

**Archivo que representa el resultado:** `EJ06.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Verificar las tablas comerciales

**Antes de escribir (lectura docente).** La verificación confirma la clave OrderID y la fórmula de importe. [Order Details] lleva corchetes por el espacio.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) o.OrderID,o.OrderDate,od.UnitPrice,od.Quantity,od.Discount
FROM dbo.Orders AS o INNER JOIN dbo.[Order Details] AS od ON o.OrderID=od.OrderID
WHERE o.OrderDate IS NOT NULL ORDER BY o.OrderID;
```

**Después y por qué importa.** La verificación confirma la clave OrderID y la fórmula de importe. [Order Details] lleva corchetes por el espacio. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Preparar variables y consulta agregada del cursor

**Antes de escribir (lectura docente).** YEAR define la dimensión anual, SUM suma importes de detalle, GROUP BY obtiene una fila por año. La agregación ocurre antes del cursor.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
DECLARE @Anio INT, @Monto DECIMAL(18,2);
DECLARE cur_S12_VentasAnuales CURSOR LOCAL FAST_FORWARD FOR
    SELECT YEAR(o.OrderDate) AS Anio,
           CAST(SUM(od.UnitPrice*od.Quantity*(1-od.Discount)) AS DECIMAL(18,2)) AS Monto
    FROM dbo.Orders AS o INNER JOIN dbo.[Order Details] AS od ON o.OrderID=od.OrderID
    WHERE o.OrderDate IS NOT NULL
    GROUP BY YEAR(o.OrderDate) ORDER BY YEAR(o.OrderDate);
```

**Después y por qué importa.** YEAR define la dimensión anual, SUM suma importes de detalle, GROUP BY obtiene una fila por año. La agregación ocurre antes del cursor. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Abrir y recorrer cada año

**Antes de escribir (lectura docente).** El cursor procesa pocos registros agregados. PRINT es útil para observar el orden de lectura, no para exportar un reporte.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
OPEN cur_S12_VentasAnuales;
FETCH NEXT FROM cur_S12_VentasAnuales INTO @Anio,@Monto;
WHILE @@FETCH_STATUS=0
BEGIN
    PRINT CONCAT(N'Año ',@Anio,N': ',@Monto);
    FETCH NEXT FROM cur_S12_VentasAnuales INTO @Anio,@Monto;
END;
```

**Después y por qué importa.** El cursor procesa pocos registros agregados. PRINT es útil para observar el orden de lectura, no para exportar un reporte. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Liberar los recursos

**Antes de escribir (lectura docente).** El cursor no debe quedar abierto al terminar. Revisar la pestaña Messages para los totales por año.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
CLOSE cur_S12_VentasAnuales;
DEALLOCATE cur_S12_VentasAnuales;
```

**Después y por qué importa.** El cursor no debe quedar abierto al terminar. Revisar la pestaña Messages para los totales por año. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ06

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
SELECT TOP (5) o.OrderID,o.OrderDate,od.UnitPrice,od.Quantity,od.Discount
FROM dbo.Orders AS o INNER JOIN dbo.[Order Details] AS od ON o.OrderID=od.OrderID
WHERE o.OrderDate IS NOT NULL ORDER BY o.OrderID;

DECLARE @Anio INT, @Monto DECIMAL(18,2);
DECLARE cur_S12_VentasAnuales CURSOR LOCAL FAST_FORWARD FOR
    SELECT YEAR(o.OrderDate) AS Anio,
           CAST(SUM(od.UnitPrice*od.Quantity*(1-od.Discount)) AS DECIMAL(18,2)) AS Monto
    FROM dbo.Orders AS o INNER JOIN dbo.[Order Details] AS od ON o.OrderID=od.OrderID
    WHERE o.OrderDate IS NOT NULL
    GROUP BY YEAR(o.OrderDate) ORDER BY YEAR(o.OrderDate);

OPEN cur_S12_VentasAnuales;
FETCH NEXT FROM cur_S12_VentasAnuales INTO @Anio,@Monto;
WHILE @@FETCH_STATUS=0
BEGIN
    PRINT CONCAT(N'Año ',@Anio,N': ',@Monto);
    FETCH NEXT FROM cur_S12_VentasAnuales INTO @Anio,@Monto;
END;

CLOSE cur_S12_VentasAnuales;
DEALLOCATE cur_S12_VentasAnuales;
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** Un mensaje por año existente en Orders y montos calculados a partir de [Order Details].

**Error común y corrección:** No confundir la columna Discount de [Order Details], expresada como fracción, con porcentaje de 0 a 100.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# EJ07 — Actualizar precios en una copia temporal y comparar cursor con UPDATE de conjuntos

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

La PPT muestra cómo un cursor aumenta precios, pero ejecutarlo sobre Products cambiaría los datos académicos compartidos. Prepararemos dos tablas temporales idénticas con cinco productos, una para un cursor y otra para una operación UPDATE de conjuntos. Aplicaremos un aumento del diez por ciento, compararemos precios y verificaremos que dbo.Products permanezca intacta. Este ejercicio enseña la mecánica de un cursor de actualización sin exponer el laboratorio a cambios permanentes y permite discutir legibilidad, bloqueos, recursos y rendimiento con evidencia concreta.

## ¿Qué problema concreto resolveremos?

Practicar cambios fila a fila de manera controlada y medir equivalencia con una actualización set-based.

**Archivo que representa el resultado:** `EJ07.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Preparar copias aisladas

**Antes de escribir (lectura docente).** Las tablas # viven en la sesión actual de SSMS. INTO copia cinco filas sin alterar Products; la segunda copia establece el punto de partida equivalente.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
DROP TABLE IF EXISTS #PreciosCursor;
DROP TABLE IF EXISTS #PreciosSet;
SELECT TOP (5) ProductID, CAST(UnitPrice AS DECIMAL(12,2)) AS UnitPrice
INTO #PreciosCursor FROM dbo.Products WHERE UnitPrice IS NOT NULL ORDER BY ProductID;
SELECT * INTO #PreciosSet FROM #PreciosCursor;
```

**Después y por qué importa.** Las tablas # viven en la sesión actual de SSMS. INTO copia cinco filas sin alterar Products; la segunda copia establece el punto de partida equivalente. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Declarar cursor y precio de cada fila

**Antes de escribir (lectura docente).** Leemos el conjunto en el cursor y actualizamos mediante UPDATE por clave. READ_ONLY impide actualizar mediante WHERE CURRENT OF, pero no impide ejecutar un UPDATE independiente sobre la tabla temporal.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
DECLARE @ProductoID INT, @Precio DECIMAL(12,2);
DECLARE cur_S12_Precios CURSOR LOCAL STATIC READ_ONLY FOR
    SELECT ProductID,UnitPrice FROM #PreciosCursor ORDER BY ProductID;
OPEN cur_S12_Precios;
FETCH NEXT FROM cur_S12_Precios INTO @ProductoID,@Precio;
```

**Después y por qué importa.** Leemos el conjunto en el cursor y actualizamos mediante UPDATE por clave. READ_ONLY impide actualizar mediante WHERE CURRENT OF, pero no impide ejecutar un UPDATE independiente sobre la tabla temporal. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Aplicar una actualización por cada clave

**Antes de escribir (lectura docente).** La variable @ProductoID identifica la fila que se modifica. FETCH dentro del ciclo garantiza avanzar. Cerramos antes de comparar.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
WHILE @@FETCH_STATUS=0
BEGIN
    UPDATE #PreciosCursor SET UnitPrice=ROUND(@Precio*1.10,2)
    WHERE ProductID=@ProductoID;
    FETCH NEXT FROM cur_S12_Precios INTO @ProductoID,@Precio;
END;
CLOSE cur_S12_Precios;
DEALLOCATE cur_S12_Precios;
```

**Después y por qué importa.** La variable @ProductoID identifica la fila que se modifica. FETCH dentro del ciclo garantiza avanzar. Cerramos antes de comparar. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Resolver el mismo problema con un único UPDATE

**Antes de escribir (lectura docente).** Una sola sentencia SQL opera sobre todas las filas. La cuadrícula demuestra si ambas técnicas dan el mismo resultado.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
UPDATE #PreciosSet SET UnitPrice=ROUND(UnitPrice*1.10,2);
SELECT c.ProductID,c.UnitPrice AS PrecioCursor,s.UnitPrice AS PrecioSet
FROM #PreciosCursor AS c INNER JOIN #PreciosSet AS s ON s.ProductID=c.ProductID
ORDER BY c.ProductID;
```

**Después y por qué importa.** Una sola sentencia SQL opera sobre todas las filas. La cuadrícula demuestra si ambas técnicas dan el mismo resultado. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 5 — Comprobar el origen sin modificar

**Antes de escribir (lectura docente).** Esta lectura vuelve a la tabla persistente y verifica que no se escribió sobre ella.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
SELECT TOP (5) ProductID,UnitPrice FROM dbo.Products
WHERE ProductID IN (SELECT ProductID FROM #PreciosCursor) ORDER BY ProductID;
```

**Después y por qué importa.** Esta lectura vuelve a la tabla persistente y verifica que no se escribió sobre ella. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ07

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
DROP TABLE IF EXISTS #PreciosCursor;
DROP TABLE IF EXISTS #PreciosSet;
SELECT TOP (5) ProductID, CAST(UnitPrice AS DECIMAL(12,2)) AS UnitPrice
INTO #PreciosCursor FROM dbo.Products WHERE UnitPrice IS NOT NULL ORDER BY ProductID;
SELECT * INTO #PreciosSet FROM #PreciosCursor;

DECLARE @ProductoID INT, @Precio DECIMAL(12,2);
DECLARE cur_S12_Precios CURSOR LOCAL STATIC READ_ONLY FOR
    SELECT ProductID,UnitPrice FROM #PreciosCursor ORDER BY ProductID;
OPEN cur_S12_Precios;
FETCH NEXT FROM cur_S12_Precios INTO @ProductoID,@Precio;

WHILE @@FETCH_STATUS=0
BEGIN
    UPDATE #PreciosCursor SET UnitPrice=ROUND(@Precio*1.10,2)
    WHERE ProductID=@ProductoID;
    FETCH NEXT FROM cur_S12_Precios INTO @ProductoID,@Precio;
END;
CLOSE cur_S12_Precios;
DEALLOCATE cur_S12_Precios;

UPDATE #PreciosSet SET UnitPrice=ROUND(UnitPrice*1.10,2);
SELECT c.ProductID,c.UnitPrice AS PrecioCursor,s.UnitPrice AS PrecioSet
FROM #PreciosCursor AS c INNER JOIN #PreciosSet AS s ON s.ProductID=c.ProductID
ORDER BY c.ProductID;

SELECT TOP (5) ProductID,UnitPrice FROM dbo.Products
WHERE ProductID IN (SELECT ProductID FROM #PreciosCursor) ORDER BY ProductID;
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** PrecioCursor = PrecioSet por ProductID; los valores originales de dbo.Products no cambian.

**Error común y corrección:** No ejecutar UPDATE dbo.Products para esta demostración; el destino debe ser una tabla # temporal.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# EJ08 — Integrar funciones y cursor en un resumen de productos de una categoría

## Objetivo técnico — Qué vamos a hacer y qué aprenderemos

Una persona encargada del inventario quiere reutilizar el filtro de categoría, calcular un precio promocional y emitir un resumen ordenado. Empezaremos comprobando que existen las funciones construidas anteriormente; seleccionaremos productos mediante una función de tabla y calcularemos un precio con la función escalar. Luego usaremos un cursor limitado a cinco filas para generar mensajes y cerraremos contrastando con un SELECT set-based. Así integraremos reutilización, lectura secuencial, validación y un criterio de decisión profesional: preferir consultas de conjuntos cuando no se necesita trabajo fila a fila.

## ¿Qué problema concreto resolveremos?

Reunir componentes reutilizables en un reporte controlado y justificar el costo de un cursor.

**Archivo que representa el resultado:** `EJ08.sql`. **Punto de partida:** pestaña nueva de SSMS conectada; no depende de archivos externos, salvo EJ08 que utiliza dos funciones de ejercicios anteriores.

### Paso 1 — Comprobar que las funciones previas existen

**Antes de escribir (lectura docente).** Si ambos valores son no nulos, el ambiente contiene las funciones necesarias. En caso contrario hay que completar EJ02 y EJ03.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
USE Northwind_S02_Lab;
GO
SELECT OBJECT_ID('dbo.fn_S12_ProductosCategoria') AS FuncionTabla,
       OBJECT_ID('dbo.fn_S12_PrecioDescuento') AS FuncionPrecio;
```

**Después y por qué importa.** Si ambos valores son no nulos, el ambiente contiene las funciones necesarias. En caso contrario hay que completar EJ02 y EJ03. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 2 — Preparar cursor sobre cinco productos

**Antes de escribir (lectura docente).** El cursor consume filas devueltas por la función de tabla. Seleccionar cinco filas permite una traza legible.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
DECLARE @Producto NVARCHAR(50),@Precio DECIMAL(12,2);
DECLARE cur_S12_Resumen CURSOR LOCAL FAST_FORWARD FOR
    SELECT TOP (5) ProductName,CAST(UnitPrice AS DECIMAL(12,2))
    FROM dbo.fn_S12_ProductosCategoria(1)
    WHERE UnitPrice IS NOT NULL ORDER BY ProductName;
```

**Después y por qué importa.** El cursor consume filas devueltas por la función de tabla. Seleccionar cinco filas permite una traza legible. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 3 — Leer y aplicar función escalar por fila

**Antes de escribir (lectura docente).** Cada iteración recupera un precio y aplica la lógica de descuento de EJ02. La función calcula, no modifica la tabla.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
OPEN cur_S12_Resumen;
FETCH NEXT FROM cur_S12_Resumen INTO @Producto,@Precio;
WHILE @@FETCH_STATUS=0
BEGIN
    PRINT CONCAT(@Producto,N' → promoción: ',dbo.fn_S12_PrecioDescuento(@Precio,10));
    FETCH NEXT FROM cur_S12_Resumen INTO @Producto,@Precio;
END;
```

**Después y por qué importa.** Cada iteración recupera un precio y aplica la lógica de descuento de EJ02. La función calcula, no modifica la tabla. Todavía no pruebes la definición parcial: continúa con el paso siguiente.

### Paso 4 — Cerrar y obtener el reporte recomendado

**Antes de escribir (lectura docente).** SELECT ofrece datos estructurados sin depender de Messages. La comparación muestra el mismo razonamiento resuelto sin iterar manualmente.

**Acción exacta en SSMS.** Escribe este bloque a continuación del anterior, sin sustituir ni borrar las líneas ya construidas. Si este bloque contiene `GO`, úsalo para separar lotes; si la definición está incompleta, **no ejecutes todavía**.

```sql
CLOSE cur_S12_Resumen;
DEALLOCATE cur_S12_Resumen;
SELECT TOP (5) ProductName,UnitPrice,
       dbo.fn_S12_PrecioDescuento(CAST(UnitPrice AS DECIMAL(12,2)),10) AS PrecioPromocion
FROM dbo.fn_S12_ProductosCategoria(1)
WHERE UnitPrice IS NOT NULL ORDER BY ProductName;
```

**Después y por qué importa.** SELECT ofrece datos estructurados sin depender de Messages. La comparación muestra el mismo razonamiento resuelto sin iterar manualmente. Ahora la construcción está completa: ejecuta el conjunto íntegro con F5 y revisa Results o Messages según corresponda.

### Archivo / artefacto completo al terminar EJ08

Copia esta versión solo **después** de haber comprendido los bloques anteriores. Es el mismo texto que se materializa en el laboratorio.

```sql
USE Northwind_S02_Lab;
GO
SELECT OBJECT_ID('dbo.fn_S12_ProductosCategoria') AS FuncionTabla,
       OBJECT_ID('dbo.fn_S12_PrecioDescuento') AS FuncionPrecio;

DECLARE @Producto NVARCHAR(50),@Precio DECIMAL(12,2);
DECLARE cur_S12_Resumen CURSOR LOCAL FAST_FORWARD FOR
    SELECT TOP (5) ProductName,CAST(UnitPrice AS DECIMAL(12,2))
    FROM dbo.fn_S12_ProductosCategoria(1)
    WHERE UnitPrice IS NOT NULL ORDER BY ProductName;

OPEN cur_S12_Resumen;
FETCH NEXT FROM cur_S12_Resumen INTO @Producto,@Precio;
WHILE @@FETCH_STATUS=0
BEGIN
    PRINT CONCAT(@Producto,N' → promoción: ',dbo.fn_S12_PrecioDescuento(@Precio,10));
    FETCH NEXT FROM cur_S12_Resumen INTO @Producto,@Precio;
END;

CLOSE cur_S12_Resumen;
DEALLOCATE cur_S12_Resumen;
SELECT TOP (5) ProductName,UnitPrice,
       dbo.fn_S12_PrecioDescuento(CAST(UnitPrice AS DECIMAL(12,2)),10) AS PrecioPromocion
FROM dbo.fn_S12_ProductosCategoria(1)
WHERE UnitPrice IS NOT NULL ORDER BY ProductName;
```

### Auditoría antes de ejecutar

| Elemento | Qué aporta | Cómo comprobarlo |
|---|---|---|
| Contexto / lote | Ejecutar contra Northwind y delimitar definiciones | `USE` y `GO` visibles |
| Entrada y lógica principal | Resuelve el problema identificado | Revisar parámetros y filtros |
| Resultado / limpieza | Devuelve datos o libera cursor | SELECT, PRINT o DEALLOCATE |

**Resultado observable:** Hasta cinco mensajes con precios promocionales y una cuadrícula equivalente, sin actualizar datos.

**Error común y corrección:** EJ08 depende de que EJ02 y EJ03 se hayan ejecutado previamente.

**Predicción previa:** antes de ejecutar, explica qué devuelve una entrada válida y qué pasa si faltan filas o el resultado está vacío.

**Evidencia sugerida:** captura del código completo más resultado o pestaña Messages.


# 5. Retos espejo para practicar lo aprendido

Cada reto Txx corresponde al EJxx. Resuelve estas preguntas **sin consultar la solución docente**. Los ejercicios que consumen funciones requieren haber ejecutado las funciones de los EJ anteriores. Debes capturar código y salida e indicar por qué la solución no altera datos persistentes.

## T01 ↔ EJ01 — Adaptar la función de empleado para mostrar nombre en orden Apellido, Nombre

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ01, pero toma una decisión nueva: adaptar la función de empleado para mostrar nombre en orden Apellido, Nombre.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Un texto con coma entre apellido y nombre para EmployeeID 5.

**Pista concreta:** revisa el archivo completo de EJ01; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

## T02 ↔ EJ02 — Aplicar el cálculo de descuento a tres productos reales sin actualizar

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ02, pero toma una decisión nueva: aplicar el cálculo de descuento a tres productos reales sin actualizar.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Tres productos con precio original y 20% de descuento calculado.

**Pista concreta:** revisa el archivo completo de EJ02; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

## T03 ↔ EJ03 — Consultar dos categorías mediante la función de tabla y comparar cantidades

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ03, pero toma una decisión nueva: consultar dos categorías mediante la función de tabla y comparar cantidades.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Una fila por categoría consultada y su cantidad de productos.

**Pista concreta:** revisa el archivo completo de EJ03; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

## T04 ↔ EJ04 — Mostrar los proveedores de la categoría 1 sin repetir nombre de proveedor

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ04, pero toma una decisión nueva: mostrar los proveedores de la categoría 1 sin repetir nombre de proveedor.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Lista de proveedores únicos obtenidos desde la función multisentencia.

**Pista concreta:** revisa el archivo completo de EJ04; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

## T05 ↔ EJ05 — Recorrer los primeros tres productos por nombre con cursor de solo lectura

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ05, pero toma una decisión nueva: recorrer los primeros tres productos por nombre con cursor de solo lectura.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Tres mensajes de producto, sin que quede el cursor abierto.

**Pista concreta:** revisa el archivo completo de EJ05; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

## T06 ↔ EJ06 — Obtener un reporte set-based de totales anuales para contrastar con el cursor

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ06, pero toma una decisión nueva: obtener un reporte set-based de totales anuales para contrastar con el cursor.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Una cuadrícula anual con el mismo cálculo que EJ06.

**Pista concreta:** revisa el archivo completo de EJ06; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

## T07 ↔ EJ07 — Simular una reducción de 5% con UPDATE de conjuntos solo en una tabla temporal

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ07, pero toma una decisión nueva: simular una reducción de 5% con UPDATE de conjuntos solo en una tabla temporal.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Cinco valores temporales con reducción de 5%, sin cambios permanentes.

**Pista concreta:** revisa el archivo completo de EJ07; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

## T08 ↔ EJ08 — Generar el reporte promocional para la categoría 2 mediante una única consulta

**Objetivo de práctica.** Reutiliza la habilidad demostrada en EJ08, pero toma una decisión nueva: generar el reporte promocional para la categoría 2 mediante una única consulta.

**Qué debes entregar:** una pestaña SQL ejecutable, una captura de Results o Messages, y una explicación en dos frases de por qué la consulta o cursor finaliza correctamente.

**Criterio observable:** Resultados de categoría 2 con promoción del 15%, sin cursor y sin UPDATE.

**Pista concreta:** revisa el archivo completo de EJ08; conserva la estructura de lotes y cambia solamente la operación o la consulta indicada en este reto. Si usas cursor, incluye OPEN, FETCH, WHILE, CLOSE y DEALLOCATE.

# 6. Diagnóstico de errores y casos límite

| Síntoma en SSMS | Causa probable | Acción verificable |
|---|---|---|
| `CREATE FUNCTION must be the first statement in a query batch` | No se separó el lote | Añade `GO` antes de `CREATE OR ALTER` y ejecuta el script completo |
| `Invalid object name 'OrderDetails'` | Northwind suele usar `[Order Details]` | Usa el nombre real validado en el explorador de objetos |
| `Cannot find either column ... or the user-defined function ...` | Función no creada en esa base o falta prefijo `dbo` | Confirma DB_NAME(), crea función y usa `dbo.fn_...` |
| Cursor already exists | Se ejecutó una sección parcial y se dejó cursor abierto | Termina ejecución anterior, cierra/libera o reinicia la pestaña; luego ejecuta todo el bloque |
| Salida vacía | Filtro sin coincidencias | Prueba una categoría visible en Products y COUNT(*) |
| Cambios no deseados en productos | UPDATE dirigido a dbo.Products en lugar de # temporal | No ejecutes; revisa destino y ejecuta únicamente el ejemplo con `#PreciosCursor` |

## 7. Cierre de la sesión — lectura sugerida

Hoy convertimos reglas repetidas de negocio en funciones que pueden invocarse de forma coherente desde diferentes consultas. Aprendimos a distinguir una función escalar, que retorna un único valor, de una función de tabla, que suministra filas a la cláusula FROM. También verificamos que el tipo de datos, los parámetros, los filtros y los casos sin coincidencia son parte del contrato de una función, no detalles secundarios.

En la segunda parte vimos que los cursores requieren un ciclo de vida completo: declarar, abrir, recuperar, evaluar el estado, volver a recuperar, cerrar y liberar. Utilizamos nombres de clientes, totales anuales de pedidos y un reporte de productos para observar cómo se produce una salida secuencial. La práctica de precios fue especialmente importante porque contrastamos una modificación fila por fila con una operación basada en conjuntos sin alterar la tabla Products. Esa comparación nos ayuda a reconocer que un cursor puede resultar útil para lógica secuencial muy específica, pero no debe elegirse por costumbre cuando un SELECT o UPDATE es suficiente.

Antes de cerrar, revisa cómo se invocan las funciones en SELECT y FROM; explica por qué el segundo FETCH evita ciclos indefinidos; y comprueba por qué un cambio sobre una tabla temporal no persiste en el catálogo original. En Lideratec Academy buscamos que cada consulta tenga un propósito visible, una validación y una justificación técnica. Para reforzar lo aprendido, consulta el blog https://lideratecacademy.com/blog/ y las clases https://www.youtube.com/@LideratecAcademy. El siguiente paso consiste en trasladar estas decisiones a problemas donde eficiencia y mantenibilidad importan tanto como la corrección del resultado.
