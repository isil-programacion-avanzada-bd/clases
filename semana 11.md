# GUÍA AMPLIADA DE CLASE PRÁCTICA — FASE_13

## Semana 11 | Desencadenantes (Triggers) en SQL Server

**Curso:** Programación Avanzada de Base de Datos  
**Institución:** ISIL · **Material didáctico:** Lideratec Academy  
**Software:** SQL Server Management Studio 22 (SSMS 22)  
**Base de datos de trabajo:** `Northwind_S02_Lab`  
**Duración de la sesión:** 120 minutos  
**Formato:** Guía completa para estudiantes y apoyo a la explicación del docente.

> **¿Qué vamos a enseñar hoy?** Aprenderemos a automatizar respuestas dentro de SQL Server cuando una tabla recibe INSERT, UPDATE o DELETE. Diferenciaremos triggers AFTER e INSTEAD OF, veremos qué datos aparecen en las tablas lógicas inserted y deleted, construiremos auditorías y validaciones y comprobaremos cómo interactúan con una transacción.

## 1. Objetivo de la sesión

**Objetivo general.** Al finalizar la clase, el estudiante **implementará, ejecutará y comprobará desencadenantes DML en SQL Server**, diferenciando `AFTER` e `INSTEAD OF` y utilizando correctamente `inserted` y `deleted` para automatizar auditorías, validar operaciones y analizar los efectos de las transacciones en una base de datos de laboratorio.

**Objetivos específicos.** Al terminar la clase podrás:

1. Explicar qué es un trigger y qué evento provoca su ejecución.
2. Crear un trigger `AFTER INSERT`, otro `AFTER UPDATE` y otro `AFTER DELETE` para conservar evidencia de cambios.
3. Reconocer cuándo consultar `inserted`, cuándo consultar `deleted` y cuándo unir ambas tablas.
4. Aplicar `INSTEAD OF INSERT`, `INSTEAD OF UPDATE` e `INSTEAD OF DELETE` para reemplazar la acción solicitada por una lógica de negocio.
5. Probar cada solución con `SELECT`, anticipar resultados y justificar las diferencias entre el estado anterior y posterior.
6. Relacionar el tema con `BEGIN TRAN`, `ROLLBACK` y `COMMIT` de la Semana 10.

**Producto esperado:** siete demostraciones reproducibles (EJ01–EJ07), consultas de verificación y siete tareas espejo (T01–T07); en cada ejercicio el estudiante explicará qué instrucción activó el trigger, qué filas leyó y cuál fue el cambio real.

## 2. ¿Qué se aprenderá y en qué orden?

| Momento | Tema central | Qué evidencia obtendremos |
|---|---|---|
| 0–12 min | Preparación, triggers y tablas lógicas | Base correcta, reglas claras, predicción inicial |
| 12–31 min | EJ01: AFTER INSERT | Pedido y registro automático de auditoría |
| 31–51 min | EJ02: AFTER UPDATE | Precio anterior y precio nuevo |
| 51–66 min | EJ03: AFTER DELETE | Historial de productos eliminados |
| 66–86 min | EJ04: INSTEAD OF INSERT | Aceptación o rechazo de proveedores |
| 86–104 min | EJ05: INSTEAD OF UPDATE | Rebaja permitida y aumento bloqueado |
| 104–115 min | EJ06: INSTEAD OF DELETE | Proveedor conservado e intento registrado |
| 115–120 min | EJ07: trigger + transacción | Visión inicial de ROLLBACK; completar su prueba como repaso |

**Distribución sugerida:** las siete demostraciones quedan explicadas íntegramente aquí; para respetar 120 minutos, el docente realiza el núcleo de EJ07 al cierre y puede encargar las tareas espejo más largas como práctica posterior.

## 3. Antes de comenzar: escenario que resolveremos

Imagina una aplicación que registra pedidos, administra precios y maneja proveedores. Las instrucciones de la aplicación hacen su trabajo principal, pero surgen nuevas necesidades: dejar rastros de altas y bajas, verificar cambios de precios, rechazar un proveedor sin correo y evitar eliminaciones indebidas. Implementaremos esas respuestas **dentro del servidor**, sin tener que pedirle al operador que ejecute una segunda consulta manual después de cada cambio.

Un trigger no se ejecuta haciendo `EXEC nombre_trigger`. SQL Server lo dispara automáticamente cuando ocurre el evento configurado sobre la tabla correspondiente.

### 3.1 AFTER e INSTEAD OF no significan lo mismo

| Tipo | Qué sucede con la operación solicitada | Ejemplo de hoy |
|---|---|---|
| AFTER | La sentencia de modificación se ejecuta y el trigger actúa en el mismo contexto transaccional; todavía puede revertirse. | Crear historial después de insertar, actualizar o eliminar. |
| INSTEAD OF | El trigger **reemplaza** la operación solicitada; para persistir un cambio válido debe realizarlo explícitamente. | Validar correo, restringir aumentos o bloquear bajas. |

### 3.2 ¿Qué guardan las tablas lógicas inserted y deleted?

| Evento | `inserted` | `deleted` |
|---|---|---|
| INSERT | Filas nuevas | Sin filas del INSERT |
| UPDATE | Versiones nuevas / propuestas | Versiones anteriores |
| DELETE | Sin filas del DELETE | Filas eliminadas / propuestas para eliminación |

Estas tablas son **lógicas y temporales del trigger**: no las crearemos con `CREATE TABLE`. Se utilizan dentro de su definición, no como tablas permanentes de la base. Para operaciones de varias filas, contienen el conjunto afectado; por eso usaremos `INSERT ... SELECT` y `JOIN`, evitando asumir que solo existe una fila.

### 3.3 ¿Por qué utilizamos tablas de laboratorio?

La presentación de la Semana 11 plantea ejemplos sobre pedidos, productos y proveedores de Northwind. Para que cada estudiante repita las pruebas sin alterar información original, crearemos objetos con nombres `dbo.S11_EJ01_*` hasta `dbo.S11_EJ07_*` **dentro de `Northwind_S02_Lab`**. Seguimos los mismos conceptos y escenarios, pero aislamos las modificaciones.

## 4. Preparación de SSMS 22 paso a paso

**Paso A — Abrir el software.** Inicia SQL Server Management Studio 22. En el cuadro de conexión usa el servidor y método de autenticación que ya configuraste en las primeras sesiones. Espera a que aparezca el Explorador de objetos.

**Paso B — Comprobar la base.** En el Explorador de objetos expande **Databases** y localiza `Northwind_S02_Lab`. Si no existe, no cambies a otra base por intuición: confirma el nombre con el docente.

**Paso C — Abrir la consulta.** Haz clic en **New Query / Nueva consulta**. Ejecuta el siguiente bloque usando el botón **Execute / Ejecutar** o la tecla **F5**:

```sql
USE Northwind_S02_Lab;
GO
SELECT DB_NAME() AS BaseActiva, @@SERVERNAME AS ServidorActual;
```

**Antes de ejecutar:** la pestaña podría estar usando otra base. **Después de ejecutar:** `BaseActiva` debe ser `Northwind_S02_Lab`. Si se muestra otra, corrige el contexto antes de continuar.

**Paso D — Forma de trabajar cada ejercicio.** Cada EJxx tiene tres bloques principales: (1) crear estructuras y datos iniciales, (2) definir el trigger y (3) provocar el evento y verificar. Ejecuta **cada bloque completo** y en el orden indicado. No ejecutes `CREATE OR ALTER TRIGGER` incompleto. Los `GO` son separadores de lote que interpreta SSMS; no son instrucciones T-SQL para guardar en la tabla.

**Paso E — Entender el reinicio.** Los ejemplos incluyen `DROP TABLE` solo para sus tablas `S11_EJxx_*`. Si ya realizaste una tarea Txx y vuelves a ejecutar el primer bloque del mismo ejemplo, se perderán esos datos de práctica. Guarda capturas antes de reiniciar. **Nunca ejecutes los DROP sobre las tablas originales de Northwind.**

---

# 5.1. EJ01 — Registrar automáticamente pedidos insertados

### Tema y objetivo del ejemplo

**Tema:** Auditoría AFTER.

**Situación real que resolveremos.** En una plataforma de ventas se registra un pedido, pero el equipo de operaciones necesita una evidencia automática de ese alta.

**Objetivo específico.** Crear dos tablas, instalar un AFTER INSERT y comprobar que una sola sentencia que agrega dos pedidos produce dos registros de auditoría.

**Lo que construiremos.** Una tabla operativa de práctica, una tabla de evidencia y un trigger que responda automáticamente al evento. Al terminar responderemos: ¿qué cambió, por qué cambió y qué consulta demuestra el resultado?

### Paso 1 — Comprender el mecanismo antes de escribir SQL

INSERT agrega filas a la tabla operativa. El trigger AFTER INSERT se activa por la sentencia; la tabla lógica inserted contiene todas las filas nuevas. El INSERT ... SELECT del trigger traslada ese conjunto a auditoría.

**Estado de partida esperado:** 0 pedidos y 0 auditorías.

**Predicción a registrar antes de la prueba:** Después de insertar dos pedidos: 2 filas en Pedidos y 2 en Auditoria.

### Paso 2 — Preparar las tablas y los datos del caso

**Antes:** este ejercicio todavía no tiene sus estructuras de trabajo, o podría haber quedado con datos de una ejecución anterior. Abre una pestaña de consulta nueva y revisa `USE`.

**Acción exacta:** selecciona el siguiente bloque completo y presiona **F5**. No lo ejecutes línea por línea, porque algunas instrucciones dependen de las tablas recién creadas.

```sql
USE Northwind_S02_Lab;
GO
IF OBJECT_ID('dbo.S11_EJ01_Auditoria','U') IS NOT NULL DROP TABLE dbo.S11_EJ01_Auditoria;
IF OBJECT_ID('dbo.S11_EJ01_Pedidos','U') IS NOT NULL DROP TABLE dbo.S11_EJ01_Pedidos;
CREATE TABLE dbo.S11_EJ01_Pedidos (PedidoID int NOT NULL PRIMARY KEY, Cliente nvarchar(60) NOT NULL);
CREATE TABLE dbo.S11_EJ01_Auditoria (AuditoriaID int IDENTITY PRIMARY KEY, PedidoID int NOT NULL, Accion varchar(10) NOT NULL, Fecha datetime2 NOT NULL DEFAULT SYSDATETIME());
SELECT COUNT(*) AS PedidosIniciales FROM dbo.S11_EJ01_Pedidos;
SELECT COUNT(*) AS AuditoriasIniciales FROM dbo.S11_EJ01_Auditoria;
```

**Por qué lo hacemos:** `USE` selecciona la base correcta; `GO` termina el lote de selección del contexto. Los `DROP TABLE` solo limpian objetos de este ejercicio para poder empezar desde cero. `CREATE TABLE` define dónde estarán los datos principales y, cuando corresponde, dónde quedará la auditoría. Las claves `PRIMARY KEY` impiden duplicar identificadores y la columna `IDENTITY` (si aparece) numera automáticamente evidencias.

**Después de ejecutar:** comprueba que no aparece un error de objeto inexistente o permisos. 0 pedidos y 0 auditorías. Si ya habías resuelto la tarea T01, recuerda que este reinicio elimina esa evidencia de laboratorio.

### Paso 3 — Construir y entender el trigger

**Antes:** las tablas ya existen, pero la operación principal aún no genera automáticamente la conducta deseada. Ahora añadiremos esa regla dentro de SQL Server.

**Acción exacta:** ejecuta el siguiente código en la misma ventana, después del bloque anterior, o en una nueva pestaña. Conserva ambos `GO`: `CREATE OR ALTER TRIGGER` debe comenzar su lote.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER TRIGGER dbo.S11_EJ01_trg_AfterInsert
ON dbo.S11_EJ01_Pedidos AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.S11_EJ01_Auditoria (PedidoID, Accion)
    SELECT i.PedidoID, 'INSERT' FROM inserted AS i;
END;
GO
```

**Lectura guiada de la estructura:**

1. `CREATE OR ALTER TRIGGER` crea el objeto por primera vez o actualiza su definición si ya existía; evita tener que borrarlo manualmente.
2. `ON dbo.S11_EJ01_...` identifica la **tabla vigilada**. Revisa su nombre en el código: el trigger no se activará por cambiar otra tabla.
3. La cláusula `AFTER` o `INSTEAD OF` junto a `INSERT`, `UPDATE` o `DELETE` determina **cuándo y por qué evento** se dispara.
4. `AS BEGIN ... END` delimita el cuerpo que SQL Server ejecuta automáticamente.
5. `SET NOCOUNT ON` evita mensajes adicionales de recuento de filas durante la ejecución del trigger; no desactiva ni cancela cambios.
6. Las referencias a `inserted` y/o `deleted` representan las **filas del evento** y las sentencias dentro del cuerpo realizan la auditoría, validación o sustitución de la operación.

**Decisión técnica más importante aquí:** La instrucción INSERT ... SELECT lee el conjunto inserted: no declares una variable única para PedidoID, porque el evento puede contener múltiples filas.

**Después de ejecutar:** el mensaje de SSMS debe indicar que la instrucción se completó correctamente. No verás nuevas filas por el solo hecho de crear el trigger: necesitamos provocar el evento con la próxima instrucción DML.

### Paso 4 — Predecir el resultado y activar el trigger

**Antes:** sin ejecutar todavía el bloque siguiente, escribe en una nota cuántas filas crees que habrá en la tabla principal y cuántas evidencias aparecerán. Explica qué parte de SQL activará el trigger.

**Acción exacta:** ejecuta el bloque completo para provocar el evento y consultar el resultado.

```sql
USE Northwind_S02_Lab;
GO
INSERT INTO dbo.S11_EJ01_Pedidos (PedidoID, Cliente)
VALUES (1101,N'Ana'),(1102,N'Luis');
SELECT PedidoID,Cliente FROM dbo.S11_EJ01_Pedidos ORDER BY PedidoID;
SELECT PedidoID,Accion FROM dbo.S11_EJ01_Auditoria ORDER BY AuditoriaID;
```

**¿Por qué ejecutamos estas consultas?** Las instrucciones `INSERT`, `UPDATE` o `DELETE` permiten observar el comportamiento real, no solo que la definición fue aceptada por SQL Server. Los `SELECT` posteriores no activan estos triggers DML: **sirven para medir y demostrar** qué ocurrió. Si aparece `TRY ... CATCH`, es para mostrar de forma controlada un rechazo previsto por la regla de negocio.

**Después / evidencia que debes observar:** Pedidos: 1101/Ana y 1102/Luis; Auditoría: PedidoID 1101 y 1102, ambos con acción INSERT.

### Paso 5 — Comprobar el resultado mediante una segunda consulta

**Objetivo de esta consulta adicional:** no repetir por repetir; contrastaremos la operación principal con su efecto o historial, usando un criterio verificable.

```sql
-- Comprobación adicional sin modificar datos
SELECT p.PedidoID, p.Cliente, COUNT(a.AuditoriaID) AS Auditorias
FROM dbo.S11_EJ01_Pedidos AS p
LEFT JOIN dbo.S11_EJ01_Auditoria AS a ON a.PedidoID=p.PedidoID
GROUP BY p.PedidoID,p.Cliente
ORDER BY p.PedidoID;
```

**Cómo interpretar el resultado:** Cada pedido debe mostrar Auditorias = 1. Un LEFT JOIN permite detectar si falta su evidencia.

**Pregunta de comprensión:** Explica por qué dos VALUES desencadenan un solo evento de sentencia, pero generan dos filas de auditoría.

### Paso 6 — Analizar un caso límite y prevenir errores

**Caso límite:** Si intentas repetir PedidoID = 1101, la clave primaria impide el INSERT; no debe generarse una tercera auditoría.

**Si no coincide tu resultado:** (a) ejecuta `SELECT DB_NAME();` y comprueba la base; (b) confirma que ejecutaste Paso 2 antes del Paso 3; (c) verifica que el trigger existe en **Tables → [tabla] → Triggers** o consultando `sys.triggers`; los triggers DML de estos ejercicios pertenecen a tablas; (d) examina los datos iniciales; (e) reconstruye solo este EJxx desde el Paso 2 si necesitas reiniciar.

### T01 — Tarea espejo para practicar sin copiar la solución

**Consigna:** T01 — Añade una tercera fila en una sola instrucción INSERT y demuestra con una consulta agrupada que cada PedidoID tiene una auditoría; predice antes los conteos (3 pedidos, 3 auditorías).

**Procedimiento recomendado:** predice por escrito el resultado; ejecuta tu nueva instrucción DML **después de completar EJ01**, sin reiniciar las tablas; prepara un `SELECT` que compruebe el efecto; compara tu predicción con la salida; explica en tres frases qué hizo el trigger y qué filas utilizó. Tu evidencia debe contener el código y la cuadrícula de resultados, no solo un mensaje de «comando completado».

**Criterio de logro:** la respuesta se considera correcta si produce el estado esperado, demuestra que el trigger actuó (o bloqueó la operación) y explica la causa técnica.

---

# 5.2. EJ02 — Comparar precio anterior y precio nuevo

### Tema y objetivo del ejemplo

**Tema:** Auditoría AFTER.

**Situación real que resolveremos.** El área comercial modifica precios y necesita recuperar el valor previo de cada producto, no solamente su valor actual.

**Objetivo específico.** Construir un AFTER UPDATE que compare el estado anterior y el nuevo, y registre únicamente cambios efectivos de UnitPrice.

**Lo que construiremos.** Una tabla operativa de práctica, una tabla de evidencia y un trigger que responda automáticamente al evento. Al terminar responderemos: ¿qué cambió, por qué cambió y qué consulta demuestra el resultado?

### Paso 1 — Comprender el mecanismo antes de escribir SQL

En un UPDATE, deleted contiene la versión anterior de las filas afectadas e inserted la versión propuesta/actualizada. El JOIN por ProductoID permite comparar pares de la misma fila.

**Estado de partida esperado:** Té cuesta 10 y Café 20; tabla Cambios vacía.

**Predicción a registrar antes de la prueba:** Tras incrementar ambos precios en 2, aparecerán dos cambios: 10→12 y 20→22. Cambiar solo Nombre no agrega auditoría de precio.

### Paso 2 — Preparar las tablas y los datos del caso

**Antes:** este ejercicio todavía no tiene sus estructuras de trabajo, o podría haber quedado con datos de una ejecución anterior. Abre una pestaña de consulta nueva y revisa `USE`.

**Acción exacta:** selecciona el siguiente bloque completo y presiona **F5**. No lo ejecutes línea por línea, porque algunas instrucciones dependen de las tablas recién creadas.

```sql
USE Northwind_S02_Lab;
GO
IF OBJECT_ID('dbo.S11_EJ02_Cambios','U') IS NOT NULL DROP TABLE dbo.S11_EJ02_Cambios;
IF OBJECT_ID('dbo.S11_EJ02_Productos','U') IS NOT NULL DROP TABLE dbo.S11_EJ02_Productos;
CREATE TABLE dbo.S11_EJ02_Productos (ProductoID int PRIMARY KEY, Nombre nvarchar(60), UnitPrice decimal(10,2));
CREATE TABLE dbo.S11_EJ02_Cambios (CambioID int IDENTITY PRIMARY KEY, ProductoID int, PrecioAnterior decimal(10,2), PrecioNuevo decimal(10,2));
INSERT INTO dbo.S11_EJ02_Productos VALUES (1,N'Té',10),(2,N'Café',20);
SELECT * FROM dbo.S11_EJ02_Productos;
```

**Por qué lo hacemos:** `USE` selecciona la base correcta; `GO` termina el lote de selección del contexto. Los `DROP TABLE` solo limpian objetos de este ejercicio para poder empezar desde cero. `CREATE TABLE` define dónde estarán los datos principales y, cuando corresponde, dónde quedará la auditoría. Las claves `PRIMARY KEY` impiden duplicar identificadores y la columna `IDENTITY` (si aparece) numera automáticamente evidencias.

**Después de ejecutar:** comprueba que no aparece un error de objeto inexistente o permisos. Té cuesta 10 y Café 20; tabla Cambios vacía. Si ya habías resuelto la tarea T02, recuerda que este reinicio elimina esa evidencia de laboratorio.

### Paso 3 — Construir y entender el trigger

**Antes:** las tablas ya existen, pero la operación principal aún no genera automáticamente la conducta deseada. Ahora añadiremos esa regla dentro de SQL Server.

**Acción exacta:** ejecuta el siguiente código en la misma ventana, después del bloque anterior, o en una nueva pestaña. Conserva ambos `GO`: `CREATE OR ALTER TRIGGER` debe comenzar su lote.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER TRIGGER dbo.S11_EJ02_trg_AfterUpdate
ON dbo.S11_EJ02_Productos AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.S11_EJ02_Cambios (ProductoID,PrecioAnterior,PrecioNuevo)
    SELECT i.ProductoID,d.UnitPrice,i.UnitPrice
    FROM inserted AS i JOIN deleted AS d ON d.ProductoID=i.ProductoID
    WHERE i.UnitPrice <> d.UnitPrice;
END;
GO
```

**Lectura guiada de la estructura:**

1. `CREATE OR ALTER TRIGGER` crea el objeto por primera vez o actualiza su definición si ya existía; evita tener que borrarlo manualmente.
2. `ON dbo.S11_EJ02_...` identifica la **tabla vigilada**. Revisa su nombre en el código: el trigger no se activará por cambiar otra tabla.
3. La cláusula `AFTER` o `INSTEAD OF` junto a `INSERT`, `UPDATE` o `DELETE` determina **cuándo y por qué evento** se dispara.
4. `AS BEGIN ... END` delimita el cuerpo que SQL Server ejecuta automáticamente.
5. `SET NOCOUNT ON` evita mensajes adicionales de recuento de filas durante la ejecución del trigger; no desactiva ni cancela cambios.
6. Las referencias a `inserted` y/o `deleted` representan las **filas del evento** y las sentencias dentro del cuerpo realizan la auditoría, validación o sustitución de la operación.

**Decisión técnica más importante aquí:** El filtro WHERE i.UnitPrice <> d.UnitPrice evita confundir «se ejecutó UPDATE» con «cambió el precio». Para NULL se requeriría una comparación adicional, aunque esta tabla usa valores no nulos en los datos de prueba.

**Después de ejecutar:** el mensaje de SSMS debe indicar que la instrucción se completó correctamente. No verás nuevas filas por el solo hecho de crear el trigger: necesitamos provocar el evento con la próxima instrucción DML.

### Paso 4 — Predecir el resultado y activar el trigger

**Antes:** sin ejecutar todavía el bloque siguiente, escribe en una nota cuántas filas crees que habrá en la tabla principal y cuántas evidencias aparecerán. Explica qué parte de SQL activará el trigger.

**Acción exacta:** ejecuta el bloque completo para provocar el evento y consultar el resultado.

```sql
USE Northwind_S02_Lab;
GO
UPDATE dbo.S11_EJ02_Productos SET UnitPrice=UnitPrice+2 WHERE ProductoID IN (1,2);
SELECT * FROM dbo.S11_EJ02_Cambios ORDER BY CambioID;
UPDATE dbo.S11_EJ02_Productos SET Nombre=N'Té especial' WHERE ProductoID=1;
SELECT COUNT(*) AS CambiosDePrecio FROM dbo.S11_EJ02_Cambios;
```

**¿Por qué ejecutamos estas consultas?** Las instrucciones `INSERT`, `UPDATE` o `DELETE` permiten observar el comportamiento real, no solo que la definición fue aceptada por SQL Server. Los `SELECT` posteriores no activan estos triggers DML: **sirven para medir y demostrar** qué ocurrió. Si aparece `TRY ... CATCH`, es para mostrar de forma controlada un rechazo previsto por la regla de negocio.

**Después / evidencia que debes observar:** Dos registros en Cambios; al modificar el nombre de Té, permanece el conteo 2.

### Paso 5 — Comprobar el resultado mediante una segunda consulta

**Objetivo de esta consulta adicional:** no repetir por repetir; contrastaremos la operación principal con su efecto o historial, usando un criterio verificable.

```sql
-- Comprobación del historial frente al estado actual
SELECT p.ProductoID,p.Nombre,p.UnitPrice AS PrecioActual,
       c.PrecioAnterior,c.PrecioNuevo
FROM dbo.S11_EJ02_Productos AS p
LEFT JOIN dbo.S11_EJ02_Cambios AS c ON c.ProductoID=p.ProductoID
ORDER BY p.ProductoID,c.CambioID;
```

**Cómo interpretar el resultado:** La columna PrecioActual refleja el estado final; PrecioAnterior y PrecioNuevo describen una transición histórica.

**Pregunta de comprensión:** ¿Qué ocurriría si UPDATE afecta cinco productos? ¿Cuántos pares de filas existirían en inserted/deleted?

### Paso 6 — Analizar un caso límite y prevenir errores

**Caso límite:** El trigger sí se activa cuando cambia solo el nombre; simplemente no inserta filas de historial gracias al WHERE.

**Si no coincide tu resultado:** (a) ejecuta `SELECT DB_NAME();` y comprueba la base; (b) confirma que ejecutaste Paso 2 antes del Paso 3; (c) verifica que el trigger existe en **Tables → [tabla] → Triggers** o consultando `sys.triggers`; los triggers DML de estos ejercicios pertenecen a tablas; (d) examina los datos iniciales; (e) reconstruye solo este EJxx desde el Paso 2 si necesitas reiniciar.

### T02 — Tarea espejo para practicar sin copiar la solución

**Consigna:** T02 — Eleva solo el precio del producto 2 en 3 unidades y verifica que aparece una tercera auditoría con precio anterior 22 y nuevo 25.

**Procedimiento recomendado:** predice por escrito el resultado; ejecuta tu nueva instrucción DML **después de completar EJ02**, sin reiniciar las tablas; prepara un `SELECT` que compruebe el efecto; compara tu predicción con la salida; explica en tres frases qué hizo el trigger y qué filas utilizó. Tu evidencia debe contener el código y la cuadrícula de resultados, no solo un mensaje de «comando completado».

**Criterio de logro:** la respuesta se considera correcta si produce el estado esperado, demuestra que el trigger actuó (o bloqueó la operación) y explica la causa técnica.

---

# 5.3. EJ03 — Auditar eliminaciones sin perder la información

### Tema y objetivo del ejemplo

**Tema:** Auditoría AFTER.

**Situación real que resolveremos.** Se elimina un producto del catálogo; luego el equipo necesita saber qué producto desapareció.

**Objetivo específico.** Construir AFTER DELETE y preservar una copia de la información eliminada en una tabla de bajas.

**Lo que construiremos.** Una tabla operativa de práctica, una tabla de evidencia y un trigger que responda automáticamente al evento. Al terminar responderemos: ¿qué cambió, por qué cambió y qué consulta demuestra el resultado?

### Paso 1 — Comprender el mecanismo antes de escribir SQL

En DELETE, las filas removidas se pueden leer desde deleted dentro del trigger. inserted no contiene las filas eliminadas.

**Estado de partida esperado:** Tres productos de ejemplo y cero bajas.

**Predicción a registrar antes de la prueba:** El DELETE de productos 1 y 2 deja un producto en catálogo y dos filas históricas.

### Paso 2 — Preparar las tablas y los datos del caso

**Antes:** este ejercicio todavía no tiene sus estructuras de trabajo, o podría haber quedado con datos de una ejecución anterior. Abre una pestaña de consulta nueva y revisa `USE`.

**Acción exacta:** selecciona el siguiente bloque completo y presiona **F5**. No lo ejecutes línea por línea, porque algunas instrucciones dependen de las tablas recién creadas.

```sql
USE Northwind_S02_Lab;
GO
IF OBJECT_ID('dbo.S11_EJ03_Bajas','U') IS NOT NULL DROP TABLE dbo.S11_EJ03_Bajas;
IF OBJECT_ID('dbo.S11_EJ03_Productos','U') IS NOT NULL DROP TABLE dbo.S11_EJ03_Productos;
CREATE TABLE dbo.S11_EJ03_Productos(ProductoID int PRIMARY KEY, Nombre nvarchar(60));
CREATE TABLE dbo.S11_EJ03_Bajas(BajaID int IDENTITY PRIMARY KEY, ProductoID int, Nombre nvarchar(60), Fecha datetime2 DEFAULT SYSDATETIME());
INSERT INTO dbo.S11_EJ03_Productos VALUES(1,N'Cuaderno'),(2,N'Lapicero'),(3,N'Borrador');
```

**Por qué lo hacemos:** `USE` selecciona la base correcta; `GO` termina el lote de selección del contexto. Los `DROP TABLE` solo limpian objetos de este ejercicio para poder empezar desde cero. `CREATE TABLE` define dónde estarán los datos principales y, cuando corresponde, dónde quedará la auditoría. Las claves `PRIMARY KEY` impiden duplicar identificadores y la columna `IDENTITY` (si aparece) numera automáticamente evidencias.

**Después de ejecutar:** comprueba que no aparece un error de objeto inexistente o permisos. Tres productos de ejemplo y cero bajas. Si ya habías resuelto la tarea T03, recuerda que este reinicio elimina esa evidencia de laboratorio.

### Paso 3 — Construir y entender el trigger

**Antes:** las tablas ya existen, pero la operación principal aún no genera automáticamente la conducta deseada. Ahora añadiremos esa regla dentro de SQL Server.

**Acción exacta:** ejecuta el siguiente código en la misma ventana, después del bloque anterior, o en una nueva pestaña. Conserva ambos `GO`: `CREATE OR ALTER TRIGGER` debe comenzar su lote.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER TRIGGER dbo.S11_EJ03_trg_AfterDelete
ON dbo.S11_EJ03_Productos AFTER DELETE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.S11_EJ03_Bajas(ProductoID,Nombre)
    SELECT d.ProductoID,d.Nombre FROM deleted AS d;
END;
GO
```

**Lectura guiada de la estructura:**

1. `CREATE OR ALTER TRIGGER` crea el objeto por primera vez o actualiza su definición si ya existía; evita tener que borrarlo manualmente.
2. `ON dbo.S11_EJ03_...` identifica la **tabla vigilada**. Revisa su nombre en el código: el trigger no se activará por cambiar otra tabla.
3. La cláusula `AFTER` o `INSTEAD OF` junto a `INSERT`, `UPDATE` o `DELETE` determina **cuándo y por qué evento** se dispara.
4. `AS BEGIN ... END` delimita el cuerpo que SQL Server ejecuta automáticamente.
5. `SET NOCOUNT ON` evita mensajes adicionales de recuento de filas durante la ejecución del trigger; no desactiva ni cancela cambios.
6. Las referencias a `inserted` y/o `deleted` representan las **filas del evento** y las sentencias dentro del cuerpo realizan la auditoría, validación o sustitución de la operación.

**Decisión técnica más importante aquí:** La copia se realiza en la misma transacción del DELETE; «AFTER» no significa «ya confirmado con COMMIT».

**Después de ejecutar:** el mensaje de SSMS debe indicar que la instrucción se completó correctamente. No verás nuevas filas por el solo hecho de crear el trigger: necesitamos provocar el evento con la próxima instrucción DML.

### Paso 4 — Predecir el resultado y activar el trigger

**Antes:** sin ejecutar todavía el bloque siguiente, escribe en una nota cuántas filas crees que habrá en la tabla principal y cuántas evidencias aparecerán. Explica qué parte de SQL activará el trigger.

**Acción exacta:** ejecuta el bloque completo para provocar el evento y consultar el resultado.

```sql
USE Northwind_S02_Lab;
GO
DELETE FROM dbo.S11_EJ03_Productos WHERE ProductoID IN (1,2);
SELECT * FROM dbo.S11_EJ03_Productos ORDER BY ProductoID;
SELECT ProductoID,Nombre FROM dbo.S11_EJ03_Bajas ORDER BY BajaID;
```

**¿Por qué ejecutamos estas consultas?** Las instrucciones `INSERT`, `UPDATE` o `DELETE` permiten observar el comportamiento real, no solo que la definición fue aceptada por SQL Server. Los `SELECT` posteriores no activan estos triggers DML: **sirven para medir y demostrar** qué ocurrió. Si aparece `TRY ... CATCH`, es para mostrar de forma controlada un rechazo previsto por la regla de negocio.

**Después / evidencia que debes observar:** Catálogo: permanece el producto 3. Bajas: quedan anotados los productos 1 y 2.

### Paso 5 — Comprobar el resultado mediante una segunda consulta

**Objetivo de esta consulta adicional:** no repetir por repetir; contrastaremos la operación principal con su efecto o historial, usando un criterio verificable.

```sql
-- Buscar evidencias de las filas que ya no están en catálogo
SELECT b.* FROM dbo.S11_EJ03_Bajas AS b ORDER BY b.BajaID;
SELECT * FROM dbo.S11_EJ03_Productos ORDER BY ProductoID;
```

**Cómo interpretar el resultado:** Contrasta ambos resultados: las filas ausentes en la tabla operativa permanecen en la tabla histórica.

**Pregunta de comprensión:** ¿Por qué no sería posible reconstruir el nombre del producto borrado mirando solo la tabla física?

### Paso 6 — Analizar un caso límite y prevenir errores

**Caso límite:** Un DELETE con condición que no coincide con ninguna fila no produce registros de baja.

**Si no coincide tu resultado:** (a) ejecuta `SELECT DB_NAME();` y comprueba la base; (b) confirma que ejecutaste Paso 2 antes del Paso 3; (c) verifica que el trigger existe en **Tables → [tabla] → Triggers** o consultando `sys.triggers`; los triggers DML de estos ejercicios pertenecen a tablas; (d) examina los datos iniciales; (e) reconstruye solo este EJxx desde el Paso 2 si necesitas reiniciar.

### T03 — Tarea espejo para practicar sin copiar la solución

**Consigna:** T03 — Elimina el producto 3, verifica que el catálogo quede vacío y que haya tres registros en la tabla de bajas.

**Procedimiento recomendado:** predice por escrito el resultado; ejecuta tu nueva instrucción DML **después de completar EJ03**, sin reiniciar las tablas; prepara un `SELECT` que compruebe el efecto; compara tu predicción con la salida; explica en tres frases qué hizo el trigger y qué filas utilizó. Tu evidencia debe contener el código y la cuadrícula de resultados, no solo un mensaje de «comando completado».

**Criterio de logro:** la respuesta se considera correcta si produce el estado esperado, demuestra que el trigger actuó (o bloqueó la operación) y explica la causa técnica.

---

# 5.4. EJ04 — Validar registros antes de permitir una inserción

### Tema y objetivo del ejemplo

**Tema:** Reemplazo con INSTEAD OF.

**Situación real que resolveremos.** Un proveedor sin correo no debería aceptarse en el catálogo, incluso si la aplicación envía múltiples filas.

**Objetivo específico.** Implementar INSTEAD OF INSERT con validación obligatoria de correo y persistencia explícita de inserciones válidas.

**Lo que construiremos.** Una tabla operativa de práctica, un trigger que responda automáticamente al evento. Al terminar responderemos: ¿qué cambió, por qué cambió y qué consulta demuestra el resultado?

### Paso 1 — Comprender el mecanismo antes de escribir SQL

INSTEAD OF INSERT sustituye la acción original. inserted representa los registros que se pretendía insertar. Si el trigger no ejecuta su propio INSERT, esas filas no se guardan.

**Estado de partida esperado:** Tabla Proveedores vacía.

**Predicción a registrar antes de la prueba:** Proveedor A, con correo, se guarda. Proveedor B, con correo vacío, es rechazado y no queda en la tabla.

### Paso 2 — Preparar las tablas y los datos del caso

**Antes:** este ejercicio todavía no tiene sus estructuras de trabajo, o podría haber quedado con datos de una ejecución anterior. Abre una pestaña de consulta nueva y revisa `USE`.

**Acción exacta:** selecciona el siguiente bloque completo y presiona **F5**. No lo ejecutes línea por línea, porque algunas instrucciones dependen de las tablas recién creadas.

```sql
USE Northwind_S02_Lab;
GO
IF OBJECT_ID('dbo.S11_EJ04_Proveedores','U') IS NOT NULL DROP TABLE dbo.S11_EJ04_Proveedores;
CREATE TABLE dbo.S11_EJ04_Proveedores(ProveedorID int PRIMARY KEY, Nombre nvarchar(60), Correo nvarchar(100) NULL);
```

**Por qué lo hacemos:** `USE` selecciona la base correcta; `GO` termina el lote de selección del contexto. Los `DROP TABLE` solo limpian objetos de este ejercicio para poder empezar desde cero. `CREATE TABLE` define dónde estarán los datos principales y, cuando corresponde, dónde quedará la auditoría. Las claves `PRIMARY KEY` impiden duplicar identificadores y la columna `IDENTITY` (si aparece) numera automáticamente evidencias.

**Después de ejecutar:** comprueba que no aparece un error de objeto inexistente o permisos. Tabla Proveedores vacía. Si ya habías resuelto la tarea T04, recuerda que este reinicio elimina esa evidencia de laboratorio.

### Paso 3 — Construir y entender el trigger

**Antes:** las tablas ya existen, pero la operación principal aún no genera automáticamente la conducta deseada. Ahora añadiremos esa regla dentro de SQL Server.

**Acción exacta:** ejecuta el siguiente código en la misma ventana, después del bloque anterior, o en una nueva pestaña. Conserva ambos `GO`: `CREATE OR ALTER TRIGGER` debe comenzar su lote.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER TRIGGER dbo.S11_EJ04_trg_InsteadInsert
ON dbo.S11_EJ04_Proveedores INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    IF EXISTS (SELECT 1 FROM inserted WHERE NULLIF(LTRIM(RTRIM(Correo)),N'') IS NULL)
    BEGIN
        THROW 51004, 'Correo obligatorio: inserción rechazada.', 1;
    END;
    INSERT INTO dbo.S11_EJ04_Proveedores(ProveedorID,Nombre,Correo)
    SELECT ProveedorID,Nombre,Correo FROM inserted;
END;
GO
```

**Lectura guiada de la estructura:**

1. `CREATE OR ALTER TRIGGER` crea el objeto por primera vez o actualiza su definición si ya existía; evita tener que borrarlo manualmente.
2. `ON dbo.S11_EJ04_...` identifica la **tabla vigilada**. Revisa su nombre en el código: el trigger no se activará por cambiar otra tabla.
3. La cláusula `AFTER` o `INSTEAD OF` junto a `INSERT`, `UPDATE` o `DELETE` determina **cuándo y por qué evento** se dispara.
4. `AS BEGIN ... END` delimita el cuerpo que SQL Server ejecuta automáticamente.
5. `SET NOCOUNT ON` evita mensajes adicionales de recuento de filas durante la ejecución del trigger; no desactiva ni cancela cambios.
6. Las referencias a `inserted` y/o `deleted` representan las **filas del evento** y las sentencias dentro del cuerpo realizan la auditoría, validación o sustitución de la operación.

**Decisión técnica más importante aquí:** EXISTS recorre el conjunto completo; NULLIF(LTRIM(RTRIM(Correo)),N'') trata NULL, cadena vacía y espacios como correo ausente. THROW detiene la sentencia. El INSERT posterior materializa los datos válidos.

**Después de ejecutar:** el mensaje de SSMS debe indicar que la instrucción se completó correctamente. No verás nuevas filas por el solo hecho de crear el trigger: necesitamos provocar el evento con la próxima instrucción DML.

### Paso 4 — Predecir el resultado y activar el trigger

**Antes:** sin ejecutar todavía el bloque siguiente, escribe en una nota cuántas filas crees que habrá en la tabla principal y cuántas evidencias aparecerán. Explica qué parte de SQL activará el trigger.

**Acción exacta:** ejecuta el bloque completo para provocar el evento y consultar el resultado.

```sql
USE Northwind_S02_Lab;
GO
INSERT INTO dbo.S11_EJ04_Proveedores VALUES(1,N'Proveedor A',N'a@ejemplo.test');
BEGIN TRY
 INSERT INTO dbo.S11_EJ04_Proveedores VALUES(2,N'Proveedor B',N'');
END TRY
BEGIN CATCH
 SELECT ERROR_MESSAGE() AS ErrorEsperado;
END CATCH;
SELECT * FROM dbo.S11_EJ04_Proveedores;
```

**¿Por qué ejecutamos estas consultas?** Las instrucciones `INSERT`, `UPDATE` o `DELETE` permiten observar el comportamiento real, no solo que la definición fue aceptada por SQL Server. Los `SELECT` posteriores no activan estos triggers DML: **sirven para medir y demostrar** qué ocurrió. Si aparece `TRY ... CATCH`, es para mostrar de forma controlada un rechazo previsto por la regla de negocio.

**Después / evidencia que debes observar:** Solo existe ProveedorID 1. La prueba controlada devuelve «Correo obligatorio: inserción rechazada».

### Paso 5 — Comprobar el resultado mediante una segunda consulta

**Objetivo de esta consulta adicional:** no repetir por repetir; contrastaremos la operación principal con su efecto o historial, usando un criterio verificable.

```sql
-- Verificar cuántos proveedores existen y cuáles son
SELECT COUNT(*) AS ProveedoresAceptados
FROM dbo.S11_EJ04_Proveedores;
SELECT ProveedorID, Nombre, Correo
FROM dbo.S11_EJ04_Proveedores ORDER BY ProveedorID;
```

**Cómo interpretar el resultado:** Solo la fila válida debe estar persistida. El mensaje de error controlado explica el rechazo.

**Pregunta de comprensión:** Predice qué ocurre si en una misma sentencia llegan tres proveedores y el segundo no tiene correo.

### Paso 6 — Analizar un caso límite y prevenir errores

**Caso límite:** La regla establecida es todo o nada para una misma sentencia: si alguna fila propuesta es inválida, no se guarda ninguna de esa sentencia.

**Si no coincide tu resultado:** (a) ejecuta `SELECT DB_NAME();` y comprueba la base; (b) confirma que ejecutaste Paso 2 antes del Paso 3; (c) verifica que el trigger existe en **Tables → [tabla] → Triggers** o consultando `sys.triggers`; los triggers DML de estos ejercicios pertenecen a tablas; (d) examina los datos iniciales; (e) reconstruye solo este EJxx desde el Paso 2 si necesitas reiniciar.

### T04 — Tarea espejo para practicar sin copiar la solución

**Consigna:** T04 — Intenta insertar dos proveedores juntos: uno válido y otro con correo NULL. Comprueba que no se inserta ninguno de esos dos y explica la atomicidad por instrucción.

**Procedimiento recomendado:** predice por escrito el resultado; ejecuta tu nueva instrucción DML **después de completar EJ04**, sin reiniciar las tablas; prepara un `SELECT` que compruebe el efecto; compara tu predicción con la salida; explica en tres frases qué hizo el trigger y qué filas utilizó. Tu evidencia debe contener el código y la cuadrícula de resultados, no solo un mensaje de «comando completado».

**Criterio de logro:** la respuesta se considera correcta si produce el estado esperado, demuestra que el trigger actuó (o bloqueó la operación) y explica la causa técnica.

---

# 5.5. EJ05 — Autorizar solo reducciones de precio

### Tema y objetivo del ejemplo

**Tema:** Reemplazo con INSTEAD OF.

**Situación real que resolveremos.** El área comercial autoriza rebajas de precios, pero no aumentos mediante el módulo de mantenimiento.

**Objetivo específico.** Aplicar INSTEAD OF UPDATE para controlar la política y persistir únicamente actualizaciones válidas.

**Lo que construiremos.** Una tabla operativa de práctica, un trigger que responda automáticamente al evento. Al terminar responderemos: ¿qué cambió, por qué cambió y qué consulta demuestra el resultado?

### Paso 1 — Comprender el mecanismo antes de escribir SQL

En INSTEAD OF UPDATE, deleted contiene valores anteriores e inserted los valores propuestos. El trigger decide si ejecutar un UPDATE físico.

**Estado de partida esperado:** Café 30; Té 20.

**Predicción a registrar antes de la prueba:** Bajar Café a 25 se acepta; intentar subirlo a 28 produce un error y Café permanece en 25.

### Paso 2 — Preparar las tablas y los datos del caso

**Antes:** este ejercicio todavía no tiene sus estructuras de trabajo, o podría haber quedado con datos de una ejecución anterior. Abre una pestaña de consulta nueva y revisa `USE`.

**Acción exacta:** selecciona el siguiente bloque completo y presiona **F5**. No lo ejecutes línea por línea, porque algunas instrucciones dependen de las tablas recién creadas.

```sql
USE Northwind_S02_Lab;
GO
IF OBJECT_ID('dbo.S11_EJ05_Productos','U') IS NOT NULL DROP TABLE dbo.S11_EJ05_Productos;
CREATE TABLE dbo.S11_EJ05_Productos(ProductoID int PRIMARY KEY, Nombre nvarchar(60), UnitPrice decimal(10,2) NOT NULL);
INSERT INTO dbo.S11_EJ05_Productos VALUES(1,N'Café',30),(2,N'Té',20);
```

**Por qué lo hacemos:** `USE` selecciona la base correcta; `GO` termina el lote de selección del contexto. Los `DROP TABLE` solo limpian objetos de este ejercicio para poder empezar desde cero. `CREATE TABLE` define dónde estarán los datos principales y, cuando corresponde, dónde quedará la auditoría. Las claves `PRIMARY KEY` impiden duplicar identificadores y la columna `IDENTITY` (si aparece) numera automáticamente evidencias.

**Después de ejecutar:** comprueba que no aparece un error de objeto inexistente o permisos. Café 30; Té 20. Si ya habías resuelto la tarea T05, recuerda que este reinicio elimina esa evidencia de laboratorio.

### Paso 3 — Construir y entender el trigger

**Antes:** las tablas ya existen, pero la operación principal aún no genera automáticamente la conducta deseada. Ahora añadiremos esa regla dentro de SQL Server.

**Acción exacta:** ejecuta el siguiente código en la misma ventana, después del bloque anterior, o en una nueva pestaña. Conserva ambos `GO`: `CREATE OR ALTER TRIGGER` debe comenzar su lote.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER TRIGGER dbo.S11_EJ05_trg_InsteadUpdate
ON dbo.S11_EJ05_Productos INSTEAD OF UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF EXISTS(SELECT 1 FROM inserted AS i JOIN deleted AS d ON d.ProductoID=i.ProductoID WHERE i.UnitPrice>d.UnitPrice)
        THROW 51005, 'No se permiten aumentos de precio.', 1;
    UPDATE p SET p.Nombre=i.Nombre,p.UnitPrice=i.UnitPrice
    FROM dbo.S11_EJ05_Productos AS p JOIN inserted AS i ON p.ProductoID=i.ProductoID;
END;
GO
```

**Lectura guiada de la estructura:**

1. `CREATE OR ALTER TRIGGER` crea el objeto por primera vez o actualiza su definición si ya existía; evita tener que borrarlo manualmente.
2. `ON dbo.S11_EJ05_...` identifica la **tabla vigilada**. Revisa su nombre en el código: el trigger no se activará por cambiar otra tabla.
3. La cláusula `AFTER` o `INSTEAD OF` junto a `INSERT`, `UPDATE` o `DELETE` determina **cuándo y por qué evento** se dispara.
4. `AS BEGIN ... END` delimita el cuerpo que SQL Server ejecuta automáticamente.
5. `SET NOCOUNT ON` evita mensajes adicionales de recuento de filas durante la ejecución del trigger; no desactiva ni cancela cambios.
6. Las referencias a `inserted` y/o `deleted` representan las **filas del evento** y las sentencias dentro del cuerpo realizan la auditoría, validación o sustitución de la operación.

**Decisión técnica más importante aquí:** IF EXISTS compara propuestas contra valores anteriores por ProductoID. THROW impide el UPDATE. El UPDATE ... JOIN implementa el cambio autorizado sin asumir una sola fila.

**Después de ejecutar:** el mensaje de SSMS debe indicar que la instrucción se completó correctamente. No verás nuevas filas por el solo hecho de crear el trigger: necesitamos provocar el evento con la próxima instrucción DML.

### Paso 4 — Predecir el resultado y activar el trigger

**Antes:** sin ejecutar todavía el bloque siguiente, escribe en una nota cuántas filas crees que habrá en la tabla principal y cuántas evidencias aparecerán. Explica qué parte de SQL activará el trigger.

**Acción exacta:** ejecuta el bloque completo para provocar el evento y consultar el resultado.

```sql
USE Northwind_S02_Lab;
GO
UPDATE dbo.S11_EJ05_Productos SET UnitPrice=25 WHERE ProductoID=1;
BEGIN TRY
 UPDATE dbo.S11_EJ05_Productos SET UnitPrice=28 WHERE ProductoID=1;
END TRY BEGIN CATCH SELECT ERROR_MESSAGE() AS ErrorEsperado; END CATCH;
SELECT * FROM dbo.S11_EJ05_Productos ORDER BY ProductoID;
```

**¿Por qué ejecutamos estas consultas?** Las instrucciones `INSERT`, `UPDATE` o `DELETE` permiten observar el comportamiento real, no solo que la definición fue aceptada por SQL Server. Los `SELECT` posteriores no activan estos triggers DML: **sirven para medir y demostrar** qué ocurrió. Si aparece `TRY ... CATCH`, es para mostrar de forma controlada un rechazo previsto por la regla de negocio.

**Después / evidencia que debes observar:** Café sigue en 25 y Té en 20; el mensaje «No se permiten aumentos de precio» se captura.

### Paso 5 — Comprobar el resultado mediante una segunda consulta

**Objetivo de esta consulta adicional:** no repetir por repetir; contrastaremos la operación principal con su efecto o historial, usando un criterio verificable.

```sql
-- Comprobación final de la política
SELECT ProductoID, Nombre, UnitPrice
FROM dbo.S11_EJ05_Productos
ORDER BY ProductoID;
```

**Cómo interpretar el resultado:** No debe aparecer 28. El rechazo de la segunda instrucción conserva los valores previamente aceptados.

**Pregunta de comprensión:** ¿Qué pasa si una sola sentencia intenta rebajar Té y aumentar Café? Explica por qué se rechaza el conjunto.

### Paso 6 — Analizar un caso límite y prevenir errores

**Caso límite:** INSTEAD OF reemplaza la modificación; si omites el UPDATE interno, ni siquiera las rebajas válidas modificarán la tabla.

**Si no coincide tu resultado:** (a) ejecuta `SELECT DB_NAME();` y comprueba la base; (b) confirma que ejecutaste Paso 2 antes del Paso 3; (c) verifica que el trigger existe en **Tables → [tabla] → Triggers** o consultando `sys.triggers`; los triggers DML de estos ejercicios pertenecen a tablas; (d) examina los datos iniciales; (e) reconstruye solo este EJxx desde el Paso 2 si necesitas reiniciar.

### T05 — Tarea espejo para practicar sin copiar la solución

**Consigna:** T05 — Ejecuta un UPDATE de dos productos que rebaje uno pero eleve el otro; verifica que ambos valores se conservan por el rechazo conjunto.

**Procedimiento recomendado:** predice por escrito el resultado; ejecuta tu nueva instrucción DML **después de completar EJ05**, sin reiniciar las tablas; prepara un `SELECT` que compruebe el efecto; compara tu predicción con la salida; explica en tres frases qué hizo el trigger y qué filas utilizó. Tu evidencia debe contener el código y la cuadrícula de resultados, no solo un mensaje de «comando completado».

**Criterio de logro:** la respuesta se considera correcta si produce el estado esperado, demuestra que el trigger actuó (o bloqueó la operación) y explica la causa técnica.

---

# 5.6. EJ06 — Bloquear un borrado y registrar el intento

### Tema y objetivo del ejemplo

**Tema:** Reemplazo con INSTEAD OF.

**Situación real que resolveremos.** Se intenta eliminar proveedores que deben mantenerse activos, pero el equipo de control necesita registrar esos intentos.

**Objetivo específico.** Crear INSTEAD OF DELETE que no borre proveedores y sí registre cada solicitud de baja.

**Lo que construiremos.** Una tabla operativa de práctica, una tabla de evidencia y un trigger que responda automáticamente al evento. Al terminar responderemos: ¿qué cambió, por qué cambió y qué consulta demuestra el resultado?

### Paso 1 — Comprender el mecanismo antes de escribir SQL

En INSTEAD OF DELETE, deleted contiene las filas que el DELETE intentaría eliminar. Si el trigger no ejecuta el DELETE sobre la tabla, los datos operativos permanecen.

**Estado de partida esperado:** Dos proveedores y tabla de intentos vacía.

**Predicción a registrar antes de la prueba:** Al solicitar DELETE de ambos, se conservarán 2 proveedores y se registrarán 2 intentos.

### Paso 2 — Preparar las tablas y los datos del caso

**Antes:** este ejercicio todavía no tiene sus estructuras de trabajo, o podría haber quedado con datos de una ejecución anterior. Abre una pestaña de consulta nueva y revisa `USE`.

**Acción exacta:** selecciona el siguiente bloque completo y presiona **F5**. No lo ejecutes línea por línea, porque algunas instrucciones dependen de las tablas recién creadas.

```sql
USE Northwind_S02_Lab;
GO
IF OBJECT_ID('dbo.S11_EJ06_Intentos','U') IS NOT NULL DROP TABLE dbo.S11_EJ06_Intentos;
IF OBJECT_ID('dbo.S11_EJ06_Proveedores','U') IS NOT NULL DROP TABLE dbo.S11_EJ06_Proveedores;
CREATE TABLE dbo.S11_EJ06_Proveedores(ProveedorID int PRIMARY KEY, Nombre nvarchar(60));
CREATE TABLE dbo.S11_EJ06_Intentos(IntentoID int IDENTITY PRIMARY KEY, ProveedorID int, Motivo nvarchar(100));
INSERT INTO dbo.S11_EJ06_Proveedores VALUES(1,N'Editorial Sur'),(2,N'Comercial Norte');
```

**Por qué lo hacemos:** `USE` selecciona la base correcta; `GO` termina el lote de selección del contexto. Los `DROP TABLE` solo limpian objetos de este ejercicio para poder empezar desde cero. `CREATE TABLE` define dónde estarán los datos principales y, cuando corresponde, dónde quedará la auditoría. Las claves `PRIMARY KEY` impiden duplicar identificadores y la columna `IDENTITY` (si aparece) numera automáticamente evidencias.

**Después de ejecutar:** comprueba que no aparece un error de objeto inexistente o permisos. Dos proveedores y tabla de intentos vacía. Si ya habías resuelto la tarea T06, recuerda que este reinicio elimina esa evidencia de laboratorio.

### Paso 3 — Construir y entender el trigger

**Antes:** las tablas ya existen, pero la operación principal aún no genera automáticamente la conducta deseada. Ahora añadiremos esa regla dentro de SQL Server.

**Acción exacta:** ejecuta el siguiente código en la misma ventana, después del bloque anterior, o en una nueva pestaña. Conserva ambos `GO`: `CREATE OR ALTER TRIGGER` debe comenzar su lote.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER TRIGGER dbo.S11_EJ06_trg_InsteadDelete
ON dbo.S11_EJ06_Proveedores INSTEAD OF DELETE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.S11_EJ06_Intentos(ProveedorID,Motivo)
    SELECT ProveedorID,N'Borrado bloqueado' FROM deleted;
END;
GO
```

**Lectura guiada de la estructura:**

1. `CREATE OR ALTER TRIGGER` crea el objeto por primera vez o actualiza su definición si ya existía; evita tener que borrarlo manualmente.
2. `ON dbo.S11_EJ06_...` identifica la **tabla vigilada**. Revisa su nombre en el código: el trigger no se activará por cambiar otra tabla.
3. La cláusula `AFTER` o `INSTEAD OF` junto a `INSERT`, `UPDATE` o `DELETE` determina **cuándo y por qué evento** se dispara.
4. `AS BEGIN ... END` delimita el cuerpo que SQL Server ejecuta automáticamente.
5. `SET NOCOUNT ON` evita mensajes adicionales de recuento de filas durante la ejecución del trigger; no desactiva ni cancela cambios.
6. Las referencias a `inserted` y/o `deleted` representan las **filas del evento** y las sentencias dentro del cuerpo realizan la auditoría, validación o sustitución de la operación.

**Decisión técnica más importante aquí:** No confundas registrar un intento con confirmar un borrado. El trigger está reemplazando la acción, no completándola.

**Después de ejecutar:** el mensaje de SSMS debe indicar que la instrucción se completó correctamente. No verás nuevas filas por el solo hecho de crear el trigger: necesitamos provocar el evento con la próxima instrucción DML.

### Paso 4 — Predecir el resultado y activar el trigger

**Antes:** sin ejecutar todavía el bloque siguiente, escribe en una nota cuántas filas crees que habrá en la tabla principal y cuántas evidencias aparecerán. Explica qué parte de SQL activará el trigger.

**Acción exacta:** ejecuta el bloque completo para provocar el evento y consultar el resultado.

```sql
USE Northwind_S02_Lab;
GO
DELETE FROM dbo.S11_EJ06_Proveedores WHERE ProveedorID IN(1,2);
SELECT COUNT(*) AS ProveedoresConservados FROM dbo.S11_EJ06_Proveedores;
SELECT ProveedorID,Motivo FROM dbo.S11_EJ06_Intentos ORDER BY IntentoID;
```

**¿Por qué ejecutamos estas consultas?** Las instrucciones `INSERT`, `UPDATE` o `DELETE` permiten observar el comportamiento real, no solo que la definición fue aceptada por SQL Server. Los `SELECT` posteriores no activan estos triggers DML: **sirven para medir y demostrar** qué ocurrió. Si aparece `TRY ... CATCH`, es para mostrar de forma controlada un rechazo previsto por la regla de negocio.

**Después / evidencia que debes observar:** Ambos proveedores siguen en Proveedores; Intentos incluye cada clave con motivo «Borrado bloqueado».

### Paso 5 — Comprobar el resultado mediante una segunda consulta

**Objetivo de esta consulta adicional:** no repetir por repetir; contrastaremos la operación principal con su efecto o historial, usando un criterio verificable.

```sql
-- Vincular cada proveedor con sus intentos
SELECT p.ProveedorID,p.Nombre,COUNT(i.IntentoID) AS IntentosDeBaja
FROM dbo.S11_EJ06_Proveedores AS p
LEFT JOIN dbo.S11_EJ06_Intentos AS i ON i.ProveedorID=p.ProveedorID
GROUP BY p.ProveedorID,p.Nombre
ORDER BY p.ProveedorID;
```

**Cómo interpretar el resultado:** Cada proveedor debe continuar presente y mostrar un intento después de la prueba base.

**Pregunta de comprensión:** Si repites el DELETE para el mismo proveedor, ¿su fila desaparece? ¿Se incrementa el historial?

### Paso 6 — Analizar un caso límite y prevenir errores

**Caso límite:** El DELETE devuelve el control tras la ejecución del trigger, pero ello no implica que se haya eliminado ninguna fila.

**Si no coincide tu resultado:** (a) ejecuta `SELECT DB_NAME();` y comprueba la base; (b) confirma que ejecutaste Paso 2 antes del Paso 3; (c) verifica que el trigger existe en **Tables → [tabla] → Triggers** o consultando `sys.triggers`; los triggers DML de estos ejercicios pertenecen a tablas; (d) examina los datos iniciales; (e) reconstruye solo este EJxx desde el Paso 2 si necesitas reiniciar.

### T06 — Tarea espejo para practicar sin copiar la solución

**Consigna:** T06 — Repite el intento sobre el proveedor 2 y comprueba que el historial registra un segundo intento para esa clave, sin borrar el proveedor.

**Procedimiento recomendado:** predice por escrito el resultado; ejecuta tu nueva instrucción DML **después de completar EJ06**, sin reiniciar las tablas; prepara un `SELECT` que compruebe el efecto; compara tu predicción con la salida; explica en tres frases qué hizo el trigger y qué filas utilizó. Tu evidencia debe contener el código y la cuadrícula de resultados, no solo un mensaje de «comando completado».

**Criterio de logro:** la respuesta se considera correcta si produce el estado esperado, demuestra que el trigger actuó (o bloqueó la operación) y explica la causa técnica.

---

# 5.7. EJ07 — Comprobar que auditoría y dato comparten transacción

### Tema y objetivo del ejemplo

**Tema:** Integración de triggers con transacciones.

**Situación real que resolveremos.** Un pedido activó una auditoría automática, pero la operación comercial completa se canceló.

**Objetivo específico.** Demostrar que el pedido y la auditoría pertenecen a una misma transacción y que ROLLBACK revierte ambos.

**Lo que construiremos.** Una tabla operativa de práctica, una tabla de evidencia y un trigger que responda automáticamente al evento. Al terminar responderemos: ¿qué cambió, por qué cambió y qué consulta demuestra el resultado?

### Paso 1 — Comprender el mecanismo antes de escribir SQL

El trigger DML ejecuta sus cambios en el contexto transaccional de la sentencia que lo activó. Una fila visible dentro de una transacción abierta no significa que esté confirmada.

**Estado de partida esperado:** Cero pedidos y cero auditorías.

**Predicción a registrar antes de la prueba:** Dentro de la transacción habrá 1 pedido y 1 auditoría; después de ROLLBACK volverán a 0 y 0.

### Paso 2 — Preparar las tablas y los datos del caso

**Antes:** este ejercicio todavía no tiene sus estructuras de trabajo, o podría haber quedado con datos de una ejecución anterior. Abre una pestaña de consulta nueva y revisa `USE`.

**Acción exacta:** selecciona el siguiente bloque completo y presiona **F5**. No lo ejecutes línea por línea, porque algunas instrucciones dependen de las tablas recién creadas.

```sql
USE Northwind_S02_Lab;
GO
IF OBJECT_ID('dbo.S11_EJ07_Auditoria','U') IS NOT NULL DROP TABLE dbo.S11_EJ07_Auditoria;
IF OBJECT_ID('dbo.S11_EJ07_Pedidos','U') IS NOT NULL DROP TABLE dbo.S11_EJ07_Pedidos;
CREATE TABLE dbo.S11_EJ07_Pedidos(PedidoID int PRIMARY KEY, Cliente nvarchar(60));
CREATE TABLE dbo.S11_EJ07_Auditoria(AuditoriaID int IDENTITY PRIMARY KEY,PedidoID int,Accion varchar(10));
```

**Por qué lo hacemos:** `USE` selecciona la base correcta; `GO` termina el lote de selección del contexto. Los `DROP TABLE` solo limpian objetos de este ejercicio para poder empezar desde cero. `CREATE TABLE` define dónde estarán los datos principales y, cuando corresponde, dónde quedará la auditoría. Las claves `PRIMARY KEY` impiden duplicar identificadores y la columna `IDENTITY` (si aparece) numera automáticamente evidencias.

**Después de ejecutar:** comprueba que no aparece un error de objeto inexistente o permisos. Cero pedidos y cero auditorías. Si ya habías resuelto la tarea T07, recuerda que este reinicio elimina esa evidencia de laboratorio.

### Paso 3 — Construir y entender el trigger

**Antes:** las tablas ya existen, pero la operación principal aún no genera automáticamente la conducta deseada. Ahora añadiremos esa regla dentro de SQL Server.

**Acción exacta:** ejecuta el siguiente código en la misma ventana, después del bloque anterior, o en una nueva pestaña. Conserva ambos `GO`: `CREATE OR ALTER TRIGGER` debe comenzar su lote.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER TRIGGER dbo.S11_EJ07_trg_AfterInsert
ON dbo.S11_EJ07_Pedidos AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO dbo.S11_EJ07_Auditoria(PedidoID,Accion)
    SELECT PedidoID,'INSERT' FROM inserted;
END;
GO
```

**Lectura guiada de la estructura:**

1. `CREATE OR ALTER TRIGGER` crea el objeto por primera vez o actualiza su definición si ya existía; evita tener que borrarlo manualmente.
2. `ON dbo.S11_EJ07_...` identifica la **tabla vigilada**. Revisa su nombre en el código: el trigger no se activará por cambiar otra tabla.
3. La cláusula `AFTER` o `INSTEAD OF` junto a `INSERT`, `UPDATE` o `DELETE` determina **cuándo y por qué evento** se dispara.
4. `AS BEGIN ... END` delimita el cuerpo que SQL Server ejecuta automáticamente.
5. `SET NOCOUNT ON` evita mensajes adicionales de recuento de filas durante la ejecución del trigger; no desactiva ni cancela cambios.
6. Las referencias a `inserted` y/o `deleted` representan las **filas del evento** y las sentencias dentro del cuerpo realizan la auditoría, validación o sustitución de la operación.

**Decisión técnica más importante aquí:** BEGIN TRAN inicia la transacción; el INSERT dispara la auditoría; los SELECT intermedios ven los cambios de la misma sesión; ROLLBACK cancela ambos. No ejecutes otro ejercicio con una transacción abierta.

**Después de ejecutar:** el mensaje de SSMS debe indicar que la instrucción se completó correctamente. No verás nuevas filas por el solo hecho de crear el trigger: necesitamos provocar el evento con la próxima instrucción DML.

### Paso 4 — Predecir el resultado y activar el trigger

**Antes:** sin ejecutar todavía el bloque siguiente, escribe en una nota cuántas filas crees que habrá en la tabla principal y cuántas evidencias aparecerán. Explica qué parte de SQL activará el trigger.

**Acción exacta:** ejecuta el bloque completo para provocar el evento y consultar el resultado.

```sql
USE Northwind_S02_Lab;
GO
BEGIN TRAN;
INSERT INTO dbo.S11_EJ07_Pedidos VALUES(701,N'Cliente temporal');
SELECT COUNT(*) AS PedidosDentro FROM dbo.S11_EJ07_Pedidos;
SELECT COUNT(*) AS AuditoriasDentro FROM dbo.S11_EJ07_Auditoria;
ROLLBACK;
SELECT COUNT(*) AS PedidosDespues FROM dbo.S11_EJ07_Pedidos;
SELECT COUNT(*) AS AuditoriasDespues FROM dbo.S11_EJ07_Auditoria;
```

**¿Por qué ejecutamos estas consultas?** Las instrucciones `INSERT`, `UPDATE` o `DELETE` permiten observar el comportamiento real, no solo que la definición fue aceptada por SQL Server. Los `SELECT` posteriores no activan estos triggers DML: **sirven para medir y demostrar** qué ocurrió. Si aparece `TRY ... CATCH`, es para mostrar de forma controlada un rechazo previsto por la regla de negocio.

**Después / evidencia que debes observar:** Los conteos antes de ROLLBACK son 1/1; los posteriores son 0/0.

### Paso 5 — Comprobar el resultado mediante una segunda consulta

**Objetivo de esta consulta adicional:** no repetir por repetir; contrastaremos la operación principal con su efecto o historial, usando un criterio verificable.

```sql
-- Estado final tras ROLLBACK
SELECT COUNT(*) AS PedidosConfirmados FROM dbo.S11_EJ07_Pedidos;
SELECT COUNT(*) AS AuditoriasConfirmadas FROM dbo.S11_EJ07_Auditoria;
SELECT @@TRANCOUNT AS TransaccionesAbiertas;
```

**Cómo interpretar el resultado:** Los tres resultados deben ser 0. Si @@TRANCOUNT no es 0, revisa si dejaste una transacción abierta en esta conexión.

**Pregunta de comprensión:** Repite el flujo con otro PedidoID y COMMIT: ¿qué cambia frente a ROLLBACK?

### Paso 6 — Analizar un caso límite y prevenir errores

**Caso límite:** AFTER INSERT no implica confirmación durable. La auditoría también se revierte si la transacción falla o se deshace.

**Si no coincide tu resultado:** (a) ejecuta `SELECT DB_NAME();` y comprueba la base; (b) confirma que ejecutaste Paso 2 antes del Paso 3; (c) verifica que el trigger existe en **Tables → [tabla] → Triggers** o consultando `sys.triggers`; los triggers DML de estos ejercicios pertenecen a tablas; (d) examina los datos iniciales; (e) reconstruye solo este EJxx desde el Paso 2 si necesitas reiniciar.

### T07 — Tarea espejo para practicar sin copiar la solución

**Consigna:** T07 — Realiza otra transacción con el pedido 702 y confirma con COMMIT. Verifica que los conteos finales sean 1 y 1 y justifica la diferencia frente a ROLLBACK.

**Procedimiento recomendado:** predice por escrito el resultado; ejecuta tu nueva instrucción DML **después de completar EJ07**, sin reiniciar las tablas; prepara un `SELECT` que compruebe el efecto; compara tu predicción con la salida; explica en tres frases qué hizo el trigger y qué filas utilizó. Tu evidencia debe contener el código y la cuadrícula de resultados, no solo un mensaje de «comando completado».

**Criterio de logro:** la respuesta se considera correcta si produce el estado esperado, demuestra que el trigger actuó (o bloqueó la operación) y explica la causa técnica.

---

# 6. Cierre de la sesión y evaluación formativa

## 6.1 ¿Qué hemos aprendido hoy?

Hemos probado que un trigger ejecuta una respuesta automatizada a eventos de modificación y que su comportamiento depende de la decisión `AFTER` frente a `INSTEAD OF`. Vimos cuándo leer `inserted`, `deleted` o ambos, cómo capturar auditorías de múltiples filas, cómo detener una modificación por regla de negocio y por qué la auditoría se revierte cuando se hace `ROLLBACK` de su transacción.

## 6.2 Preguntas de salida

1. ¿Qué sentencia concreta dispara un `AFTER INSERT` y qué contiene `inserted`?
2. En un `AFTER UPDATE`, ¿cómo obtienes el precio anterior y el precio nuevo de cada producto?
3. ¿Por qué `deleted` sirve para conservar información después de una eliminación?
4. ¿Qué diferencia práctica hay entre `AFTER INSERT` e `INSTEAD OF INSERT`?
5. ¿Cómo impedirías que una sola instrucción actualizara cinco productos si uno viola una política?
6. Si haces `BEGIN TRAN`, provocas un trigger y luego `ROLLBACK`, ¿qué sucede con la auditoría?

## 6.3 Evidencias mínimas de clase

Entrega o conserva un `.sql` por ejemplo, los resultados de sus `SELECT`, una breve explicación de lo ocurrido y la solución de sus tareas espejo cuando se asignen. No es suficiente mostrar que «el script corrió»: debes relacionar la operación, los registros lógicos involucrados y el estado final.

## 6.4 Conexión con la clase anterior

En la Semana 10 vimos cómo `COMMIT` confirma una transacción y `ROLLBACK` revierte sus cambios. En esta Semana 11 vimos que una acción adicional provocada por un trigger **forma parte de ese mismo contexto transaccional**. La combinación de ambos temas permite diseñar operaciones de bases de datos con evidencia auditable y coherencia entre datos principales e históricos.

---

**Alcance y fuente:** elaborado con base en `30627-S11-PRESENTACION.pptx` y la secuencia EJ01–EJ07 / T01–T07 de la FASE_13 previa. La PPT incluye descripciones que pueden confundir los contenidos de `inserted` y `deleted`; en esta guía se emplea la semántica correcta de SQL Server. La ejecución final de los scripts debe comprobarse en la instalación local de SSMS 22 y SQL Server.
