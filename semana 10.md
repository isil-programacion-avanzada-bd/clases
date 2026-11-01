---
course_id: ISIL-PABD
session_id: S10
source_origin: PPT
editor: SQL Server Management Studio 22
database: Northwind_S02_Lab
status: reconstruido_validacion_estructural
---

# TEMA DE CLASE — SESIÓN 10: TRANSACCIONES EN SQL SERVER

**Curso:** Programación Avanzada de Base de Datos — ISIL.  
**Duración:** 120 minutos (7 ejemplos en aula y 1 ejemplo de transferencia).  
**Herramienta:** SQL Server Management Studio 22 (SSMS 22).  
**Base de datos:** `Northwind_S02_Lab`.  
**Caso práctico:** una tienda gestiona clientes, stock y ventas; debe evitar cambios incompletos.  
**Conceptos centrales:** autoconfirmación (autocommit), transacciones implícitas y explícitas, `BEGIN TRANSACTION`, `COMMIT`, `ROLLBACK`, `TRY/CATCH`, `XACT_STATE()` y `SAVE TRANSACTION`.

## ¿Qué aprenderemos hoy?

Al finalizar la clase, podrás **explicar con pruebas SQL** cuándo un cambio queda guardado y cuándo se cancela. No se trata de memorizar comandos: aprenderás a identificar *qué dato se quiere modificar, cuál es el riesgo de modificarlo a medias y qué consulta verifica que el resultado es correcto*. La meta observable es construir una venta que reste inventario y genere su registro como una sola operación; si falla cualquiera de los pasos, se restaura el estado anterior.

### La pregunta que resolveremos durante la sesión

> Si un cliente compra dos unidades, el sistema reduce el stock de 20 a 18, pero falla al crear el registro de venta: ¿la base de datos quedó bien? **No**. Ahora el inventario afirma que se vendieron dos unidades, mientras que las ventas no lo reflejan. Ese es el problema de consistencia que solucionan las transacciones.

**Camino de aprendizaje:** veremos primero una sola instrucción guardada automáticamente (EJ01), luego una modificación que queda pendiente y se cancela (EJ02). Después abriremos una transacción para confirmar (EJ03) y otra para revertir (EJ04). Con esos fundamentos, simularemos una venta con dos cambios inseparables (EJ05), provocaremos un error y verificaremos la reversión (EJ06), practicaremos retroceso parcial (EJ07) y finalmente encapsularemos todo en un procedimiento almacenado (EJ08).

## Cómo interpretar las consultas: no ejecutamos SQL por ejecutarlo

En una clase práctica aparecen numerosas sentencias SQL, pero **no tienen todas la misma función**. Antes de pulsar F5, pregúntate siempre: «¿Esta sentencia está observando un dato, modificándolo, decidiendo si continúa la operación o demostrando el resultado?».

| Instrucción | Pregunta concreta que responde | ¿Modifica información? | Para qué la usaremos |
|---|---|---|---|
| `SELECT ... FROM ... WHERE ...` | ¿Qué valor tiene exactamente el cliente o producto que me interesa? | No | Conocer el **antes** y comprobar el **después**. |
| `UPDATE ... SET ... WHERE ...` | ¿Qué fila debemos cambiar y cuál debe ser su nuevo valor? | Sí | Cambiar contacto, país o stock. |
| `INSERT INTO ... VALUES/SELECT ...` | ¿Qué nuevo registro debe existir? | Sí | Guardar un evento, venta o auditoría. |
| `@@ROWCOUNT` | ¿Cuántas filas afectó la instrucción anterior? | No | Evitar confirmar un UPDATE que no encontró el registro. |
| `BEGIN TRANSACTION` | ¿Desde dónde deben tratarse las operaciones como una sola unidad? | Abre una transacción | Asegurar que varios cambios tengan una decisión común. |
| `COMMIT TRANSACTION` | ¿Las operaciones fueron válidas y deben permanecer? | Confirma | Hacer permanentes los cambios de la transacción. |
| `ROLLBACK TRANSACTION` | ¿Debemos descartar los cambios realizados en la transacción? | Revierte | Impedir datos inconsistentes. |
| `@@TRANCOUNT` | ¿Hay una transacción todavía abierta en esta conexión? | No | Comprobar que el ejercicio no deja una operación pendiente. |
| `TRY/CATCH` / `XACT_STATE()` | ¿Ocurrió un error y la transacción puede confirmarse o debe revertirse? | Controla ejecución | Cerrar de manera segura un error de prueba. |
| `SAVE TRANSACTION` | ¿Desde qué punto podemos cancelar solo la parte posterior? | Crea un punto de guardado | Conservar el evento obligatorio y descartar el opcional. |

**Regla esencial:** un `SELECT` no guarda ni revierte nada; hace visible un estado. Usamos `SELECT` antes y después para **probar una afirmación**, no para llenar la pantalla de resultados. `@@TRANCOUNT = 0` solo demuestra que no hay transacción activa en esa conexión; para demostrar que un cambio se confirmó o revirtió, también debemos consultar el valor de los datos.

### Cómo leer un SELECT de esta clase

```sql
SELECT CustomerID, ContactName, Country
FROM dbo.S10_ClientesLab
WHERE CustomerID = 'ALFKI';
```

En palabras sencillas: «Muéstrame el identificador, el contacto y el país **del cliente ALFKI** de la tabla de pruebas». `SELECT` elige las columnas que queremos observar; `FROM` indica de qué tabla se leen; `WHERE` evita confundir el resultado con otros clientes. **La consulta no cambia al cliente.** Si la ejecutamos antes y después de `UPDATE`, podemos demostrar si la modificación quedó guardada.

### Diferencia entre guardar y comprobar

Supongamos que el contacto del cliente ALFKI se modifica temporalmente dentro de una transacción. Un `SELECT` ejecutado **en la misma conexión antes del COMMIT** puede mostrar el valor nuevo, pero eso **no significa que esté confirmado**. Por eso, las prácticas incluyen `COMMIT` o `ROLLBACK` y una consulta final **después** de cerrar la transacción. Esta comparación es la evidencia que queremos enseñar, no solo una captura de SQL ejecutado.

### ¿Por qué no cambiamos las tablas originales de Northwind?

`dbo.Customers` y `dbo.Products` nos sirven para obtener nombres y precios reales de la base con la que venimos trabajando. Para practicar cancelaciones, ventas y errores, copiamos un conjunto pequeño de datos a tablas `dbo.S10_*Lab`. En estas tablas auxiliares podemos iniciar, validar y revertir cambios sin alterar el historial original. **El stock 20 es un valor inicial didáctico**, no una afirmación sobre las existencias reales de Northwind. Si repites las ventas, el stock disminuirá nuevamente; la preparación lo reinicia.

### Acuerdo de ejecución en SSMS 22

En cada ejemplo escribiremos el código **por microbloques**, explicando por qué cada sentencia va allí. Sin embargo, **no debes ejecutar una definición incompleta**, un `TRY` sin su `CATCH`, ni una transacción sin su cierre. Al terminar los pasos, revisarás el **Código completo**, lo ejecutarás con F5 y buscarás sus resultados en el mismo orden en que aparecen los `SELECT`. La primera lectura será «cómo estaba», la segunda «qué ocurrió» y la última «cómo quedó». Si aparece un error inesperado, no continúes ejecutando otros ejemplos en esa conexión hasta entender el estado de la transacción.

---


**Curso:** Programación Avanzada de Base de Datos · ISIL.  **Duración de aula:** 120 minutos. **Caso transversal:** venta de productos a clientes Northwind sobre COPIAS auxiliares S10.

## 1. ¿Qué vamos a construir y por qué?

Imagina que una tienda recibe una solicitud de compra. En la base de datos existe el cliente, un producto y un número de unidades disponibles. Si el sistema descuenta unidades del almacén pero no registra la venta, el inventario y el historial comercial no coinciden. Si, por el contrario, registra una venta que no pudo descontar stock, también aparece información incorrecta. Una transacción permite tratar esas acciones relacionadas como una sola unidad lógica: se confirman juntas o se revierten juntas cuando una condición no se cumple.

Durante esta clase no iremos directamente a un procedimiento complejo. Primero observaremos qué pasa cuando SQL Server guarda automáticamente una instrucción; después probaremos una transacción implícita que necesita cierre; luego abriremos transacciones explícitas para practicar COMMIT y ROLLBACK. Más adelante, aplicaremos esas decisiones a un registro de venta con descuento de stock; provocaremos un error para ver cómo TRY/CATCH protege datos; practicaremos un savepoint y terminaremos convirtiendo la venta en un procedimiento reutilizable. En cada caso leeremos el estado inicial, ejecutaremos la operación, compararemos qué cambió y justificaremos por qué ese resultado es correcto.

Trabajaremos con **SSMS 22** y **Northwind_S02_Lab**, la base usada en las primeras clases. `dbo.Customers` y `dbo.Products` se leen para crear el conjunto de pruebas, pero **nunca se alteran**: las modificaciones se realizan únicamente en `dbo.S10_ClientesLab`, `dbo.S10_ProductosLab`, `dbo.S10_VentasLab`, `dbo.S10_EventosLab` y `dbo.S10_AuditoriaLab`.

## 2. Resultado de aprendizaje y cómo lo demostraremos

Al terminar, podrás reconocer qué transacciones se confirman automáticamente, cuáles comienzan de forma implícita y cuándo necesitas abrir una transacción explícita para evitar cambios parciales. Podrás justificar por qué una venta válida modifica simultáneamente inventario y registro comercial, y por qué una venta rechazada conserva los datos anteriores. También podrás observar un error dentro de TRY/CATCH, utilizar XACT_STATE para limpiar una transacción y retroceder solo hasta un punto de guardado sin perder las acciones anteriores. Se considerará logrado el objetivo cuando muestres una consulta SQL **antes y después**, expliques el motivo de usar COMMIT o ROLLBACK, y compruebes que `@@TRANCOUNT` no queda abierto.

**Conocimientos previos:** SELECT, UPDATE, INSERT, WHERE, variables `DECLARE`, `IF`, `GO`, y procedimientos de la sesión 9. **No se introducen:** niveles de aislamiento, concurrencia avanzada, índices ni administración del transaction log.

## 3. Mapa de la clase: conoce los ocho problemas antes de programarlos

| Orden | Tiempo sugerido | Caso concreto | Qué se cambia | Decisión que aprenderás | Prueba observable |
|---|---:|---|---|---|---|
| EJ01 | 10 min | Registrar una revisión de cliente | Un evento de bitácora | Autocommit | Marca guardada; contador 0 |
| EJ02 | 12 min | Proponer cambio de país sin aprobación | País de ALFKI | Transacción implícita + ROLLBACK | País recuperado; contador 0 |
| EJ03 | 13 min | Aprobar nuevo contacto | ContactName de ALFKI | Transacción explícita + COMMIT | Nombre cambiado de forma persistente |
| EJ04 | 12 min | Rechazar modificación de país | Country de ANATR | ROLLBACK explícito | País anterior restaurado |
| EJ05 | 20 min | Vender 2 unidades de Chai | Stock + fila de venta | Atomicidad entre dos tablas | Stock baja y existe venta |
| EJ06 | 18 min | Detectar un registro de auditoría duplicado | Stock provisional + evento inválido | TRY/CATCH + XACT_STATE | Stock no se modifica al fallar |
| EJ07 | 13 min | Conservar evento base y quitar nota | Dos marcas de bitácora | SAVE TRANSACTION | Solo marca base persiste |
| EJ08 | Autónomo | Ejecutar ventas repetibles con auditoría | Stock + venta + auditoría | Procedimiento transaccional | Caso válido sí; rechazo no |

**En los 120 minutos:** 12 min de apertura/preparación + 98 min de EJ01–EJ07 + 10 min de verificación final. **EJ08 y T08** se desarrollan como trabajo autónomo; las tareas espejo se inician durante clase y se completan si falta tiempo. No se obliga a ejecutar 16 scripts completos en dos horas.

## 4. Cómo comenzar en SSMS 22 (sin adivinar pasos)

1. Abre SSMS 22 y conéctate a la misma instancia que utilizaste en sesiones anteriores.
2. En Explorador de objetos localiza `Bases de datos > Northwind_S02_Lab`. Pulsa **Nueva consulta** y comprueba el contexto escribiendo `SELECT DB_NAME() AS BaseActual,@@TRANCOUNT AS Pendientes;` y presionando **F5**. Debe verse `Northwind_S02_Lab` y, en una conexión limpia, `0`.
3. Copia el script de preparación que figura **íntegro a continuación** a una segunda consulta nueva. El archivo equivalente de FASE 15 se llama `00_preparacion/00_Preparar_Laboratorio.sql`.
4. **Antes de pulsar F5**, comprueba que aparece `USE Northwind_S02_Lab;` y que se crean solamente tablas `S10_*Lab`. El script **reinicia filas de esas tablas**: guarda capturas previas si necesitas conservar evidencias. No lo ejecutes en producción.
5. Ejecuta todo el script con **F5**. En Resultados deben verse `ALFKI`, `ANATR`, los productos `1` y `2` con **Stock=20**, y la base actual con `TransaccionesPendientes=0`. Si falta una tabla o esos IDs, detente: tu variante de Northwind no coincide con esta preparación y necesita adaptación, no cambios arbitrarios.
6. Para cada ejemplo abre una pestaña nueva, usa el orden de sus pasos, **no pulses F5 sobre código incompleto**, y al terminar verifica `@@TRANCOUNT=0`. Si un error deja transacción pendiente, revisa la operación antes de decidir una reversión; nunca confirmes datos a ciegas.


### ¿Por qué ejecutamos las consultas de preparación?

En el script de preparación hay **cuatro decisiones distintas** que necesitas poder explicar antes de comenzar:

1. `SELECT ... FROM dbo.Customers` y `SELECT ... FROM dbo.Products` se usan para **copiar registros conocidos** de Northwind a nuestro escenario de prueba. Esto responde: *¿con qué clientes y productos vamos a practicar?* No son consultas de venta; son lecturas de origen.
2. `CREATE TABLE dbo.S10_*Lab` crea espacios de pruebas independientes, con claves y restricciones. Lo hacemos para que las próximas demostraciones con `UPDATE` y `ROLLBACK` no alteren datos originales.
3. Los `DELETE` de la preparación eliminan **solo filas de las cinco tablas de laboratorio**. Los ejecutamos antes de volver a copiar clientes y productos porque necesitamos un punto de partida reproducible. Al repetir la preparación, se perderán las ventas y evidencias anteriores de S10.
4. Los `SELECT` del final son una **lista de control**: debes reconocer dos clientes (`ALFKI`, `ANATR`), dos productos (IDs 1 y 2), stock de prueba 20 y `TransaccionesPendientes=0`. Si no aparecen esos datos, no continúes: los ejemplos posteriores dependen de ese escenario.

**Regla docente:** mostrar primero la consulta de “antes”, verbalizar el resultado esperado y solo entonces explicar qué `UPDATE` o `INSERT` lo modifica. La mayor dificultad de esta sesión no es escribir `COMMIT`, sino **entender qué datos confirma**.

### Preparación completa (reproducible sin abrir otro documento)

```sql
-- S10 | Preparación: solo afecta tablas auxiliares dbo.S10_*Lab.
-- REQUIERE dbo.Customers (ALFKI, ANATR) y dbo.Products (1,2).
-- Ejecute completo en SSMS 22 sobre la base académica Northwind_S02_Lab.
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SET XACT_ABORT ON;
IF @@TRANCOUNT <> 0 THROW 51000, N'Cierre de forma consciente la transacción pendiente antes de preparar.', 1;
IF OBJECT_ID(N'dbo.Customers',N'U') IS NULL OR OBJECT_ID(N'dbo.Products',N'U') IS NULL
    THROW 51001, N'Esta variante de Northwind necesita dbo.Customers y dbo.Products.', 1;
IF (SELECT COUNT(*) FROM dbo.Customers WHERE CustomerID IN ('ALFKI','ANATR')) <> 2
    THROW 51002, N'No se encuentran los clientes ALFKI y ANATR en Northwind.', 1;
IF (SELECT COUNT(*) FROM dbo.Products WHERE ProductID IN (1,2)) <> 2
    THROW 51003, N'No se encuentran los productos 1 y 2 en Northwind.', 1;
GO
IF OBJECT_ID(N'dbo.S10_ClientesLab',N'U') IS NULL
BEGIN
    CREATE TABLE dbo.S10_ClientesLab (
        CustomerID NCHAR(5) NOT NULL PRIMARY KEY,
        CompanyName NVARCHAR(40) NOT NULL,
        ContactName NVARCHAR(30) NULL,
        Country NVARCHAR(15) NULL
    );
END;
GO
IF OBJECT_ID(N'dbo.S10_ProductosLab',N'U') IS NULL
BEGIN
    CREATE TABLE dbo.S10_ProductosLab (
        ProductID INT NOT NULL PRIMARY KEY,
        ProductName NVARCHAR(40) NOT NULL,
        Stock INT NOT NULL CONSTRAINT CK_S10_Stock CHECK (Stock >= 0),
        Precio DECIMAL(10,2) NOT NULL
    );
END;
GO
IF OBJECT_ID(N'dbo.S10_VentasLab',N'U') IS NULL
BEGIN
    CREATE TABLE dbo.S10_VentasLab (
        VentaID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        CustomerID NCHAR(5) NOT NULL REFERENCES dbo.S10_ClientesLab(CustomerID),
        ProductID INT NOT NULL REFERENCES dbo.S10_ProductosLab(ProductID),
        Cantidad INT NOT NULL CONSTRAINT CK_S10_Cantidad CHECK (Cantidad > 0),
        Precio DECIMAL(10,2) NOT NULL,
        Fecha DATETIME2(0) NOT NULL CONSTRAINT DF_S10_Ventas_Fecha DEFAULT SYSUTCDATETIME()
    );
END;
GO
IF OBJECT_ID(N'dbo.S10_EventosLab',N'U') IS NULL
BEGIN
    CREATE TABLE dbo.S10_EventosLab (
        Codigo VARCHAR(30) NOT NULL PRIMARY KEY,
        Descripcion NVARCHAR(180) NOT NULL,
        Fecha DATETIME2(0) NOT NULL CONSTRAINT DF_S10_Eventos_Fecha DEFAULT SYSUTCDATETIME()
    );
END;
GO
IF OBJECT_ID(N'dbo.S10_AuditoriaLab',N'U') IS NULL
BEGIN
    CREATE TABLE dbo.S10_AuditoriaLab (
        AuditoriaID INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
        VentaID INT NULL,
        Descripcion NVARCHAR(180) NOT NULL,
        Fecha DATETIME2(0) NOT NULL CONSTRAINT DF_S10_Audit_Fecha DEFAULT SYSUTCDATETIME()
    );
END;
GO
-- REINICIO: elimina filas de laboratorio; guardar evidencias antes.
-- No se hace DELETE/UPDATE de dbo.Customers ni dbo.Products.
BEGIN TRY
    BEGIN TRANSACTION;
    DELETE FROM dbo.S10_AuditoriaLab;
    DELETE FROM dbo.S10_VentasLab;
    DELETE FROM dbo.S10_EventosLab;
    DELETE FROM dbo.S10_ProductosLab;
    DELETE FROM dbo.S10_ClientesLab;

    INSERT INTO dbo.S10_ClientesLab(CustomerID,CompanyName,ContactName,Country)
    SELECT CustomerID,CompanyName,ContactName,Country
    FROM dbo.Customers WHERE CustomerID IN ('ALFKI','ANATR');

    -- Tomamos los nombres y precios de Northwind, pero usamos stock de prueba = 20.
    INSERT INTO dbo.S10_ProductosLab(ProductID,ProductName,Stock,Precio)
    SELECT ProductID,ProductName,20,CONVERT(DECIMAL(10,2),COALESCE(UnitPrice,10))
    FROM dbo.Products WHERE ProductID IN (1,2);

    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES ('DUPLICADO',N'Marca preexistente para provocar error controlado en EJ06');
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
SELECT CustomerID,CompanyName,ContactName,Country FROM dbo.S10_ClientesLab ORDER BY CustomerID;
SELECT ProductID,ProductName,Stock,Precio FROM dbo.S10_ProductosLab ORDER BY ProductID;
SELECT @@TRANCOUNT AS TransaccionesPendientes,DB_NAME() AS BaseActual;
GO
```

**Qué se preparó y para qué:** `S10_ClientesLab` guarda el pequeño conjunto de clientes de Northwind; `S10_ProductosLab` copia dos nombres/precios y establece 20 unidades de prueba; `S10_VentasLab` registra ventas con IDENTITY; `S10_EventosLab` permite mostrar autocommit, duplicados y savepoints; `S10_AuditoriaLab` registra eventos comerciales de la rutina final. Todas viven en la misma base ya conocida, pero son independientes de sus tablas históricas.

**Nota importante:** en SQL Server **autoconfirmación e implícita NO son sinónimos**. En autocommit, la sentencia aislada confirma al terminar; con `SET IMPLICIT_TRANSACTIONS ON`, una instrucción elegible inicia una transacción que deberá cerrarse. `BEGIN...END` solo agrupa instrucciones de control y no inicia por sí mismo una transacción.

## Cómo justificar cada ejecución durante la clase

Cada vez que se ejecute un ejemplo, se debe completar oralmente esta secuencia: **1) ¿Qué problema del caso queremos resolver? 2) ¿Qué fila y columna representan el problema? 3) ¿Por qué esta consulta es necesaria? 4) ¿Qué resultado esperamos antes de ejecutarla? 5) ¿Qué dato demuestra que COMMIT o ROLLBACK hicieron lo correcto?**. Si una persona no puede responder estas cinco preguntas, todavía no ha terminado el ejercicio aunque SSMS muestre «Comandos completados correctamente».

Para facilitar el aprendizaje, distingue en tus notas estas tres clases de consultas: **consulta de diagnóstico** (lee estado anterior), **operación de negocio** (modifica datos bajo una regla) y **consulta de evidencia** (comprueba que el estado final coincide con el objetivo). En EJ05, por ejemplo, un `UPDATE` que disminuye stock no basta para declarar que la venta se completó: también hay que encontrar la nueva fila de venta. Esta relación es el corazón de las transacciones.

## 5. Recorrido guiado de los ejemplos

---

## EJ01 — Registrar un evento sin COMMIT: autoconfirmación

### Tema específico y objetivo de aprendizaje del EJ01

**Tema que trataremos:** Modo de autoconfirmación para una única operación.

**Objetivo del ejemplo (qué lograremos al terminar):** Registrar **una sola marca de atención** sin abrir una transacción explícita y demostrar, mediante una lectura posterior, que SQL Server la conservó y que no hay una transacción pendiente.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** El negocio solo necesita dejar una constancia breve: “revisamos el cliente”. A diferencia de una venta, aquí hay un único INSERT. Antes de aprender transacciones de varios pasos, comprobamos el comportamiento normal de una instrucción aislada para tener un punto de comparación.

**¿Qué datos vamos a observar y modificar?** `dbo.S10_EventosLab`: se agrega el código único `EJ01` y su descripción; **ningún cliente ni producto se modifica**.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. Primer `SELECT` por `Codigo=EJ01` | ¿Ya estaba registrada esa revisión? Un resultado vacío significa que todavía no existe la marca. |
| 2. `INSERT` condicionado por `IF NOT EXISTS` | ¿Hace falta registrar la marca? Evita repetir el mismo evento si ejecutamos dos veces la práctica. |
| 3. Segundo `SELECT` y `@@TRANCOUNT` | ¿La fila ahora existe y quedó la conexión limpia? Debe aparecer EJ01 y el contador debe ser 0. |

**Prueba de que alcanzamos el objetivo:** Una fila con `Codigo=EJ01`, incluso al consultar de nuevo, y `TransaccionesPendientes=0`.

**Idea que no debemos confundir:** Autocommit significa que cada instrucción elegible se confirma por sí sola en ese modo; **no** significa que cualquier conjunto de varias instrucciones forme automáticamente una transacción.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** El área de atención necesita dejar constancia de que se revisó un cliente. Es una sola operación de INSERT, no una venta con varias tablas.

**Qué haremos, en orden:** Insertaremos una marca en dbo.S10_EventosLab sin abrir transacción explícita; verificaremos que la fila persiste y que @@TRANCOUNT termina en cero.

**Qué tenemos ANTES:** La marca EJ01 no existe tras preparar el laboratorio.

**Qué debe existir DESPUÉS:** Aparece exactamente la marca EJ01 y TransaccionesPendientes=0.

**Qué significa el resultado:** La instrucción INSERT se confirmó por sí misma porque el modo implícito está apagado y no se abrió una transacción explícita.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Fijar la base y el modo de autoconfirmación

**Antes (explicación para entender lo que viene):** Abrimos una pestaña nueva en SSMS 22; aún no hay código.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
```

**Objetivo de esta consulta o bloque:** Preparar la sesión.

**Por qué lo necesitamos precisamente en este momento:** No esperamos filas de resultados: esta parte configura **dónde** se ejecutará el ejercicio y desactiva las transacciones implícitas. Es importante porque, si la conexión estuviera en modo implícito, el INSERT podría quedar pendiente y la prueba dejaría de ser una demostración de autocommit.

**Qué observaremos y cómo lo interpretaremos:** Aún no hemos registrado ninguna revisión; solo fijamos la conexión.

**Por qué se escribe así:** USE evita ejecutar en master; GO separa lotes en SSMS y no confirma datos. Con SET OFF, un INSERT aislado queda en autocommit.

**Qué cambia / qué veremos al ejecutar el script completo:** El contexto es la base de prácticas; todavía no se cambió ningún dato.

**Pregunta para predecir:** ¿GO ejecuta COMMIT?

**Respuesta razonada:** No, es un separador de lotes de SSMS.

**Siguiente decisión:** Antes de insertar verificaremos la existencia de la clave.

### Paso 2 — Consultar el estado anterior del evento

**Antes (explicación para entender lo que viene):** La tabla S10_EventosLab ya existe y conocemos su clave Codigo.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT Codigo,Descripcion FROM dbo.S10_EventosLab WHERE Codigo='EJ01';
```

**Objetivo de esta consulta o bloque:** Medir el punto de partida.

**Por qué lo necesitamos precisamente en este momento:** Leemos la tabla *antes* del cambio. La columna `Codigo` es la clave que distingue eventos: si `EJ01` ya existe, la práctica no debe interpretarse como una nueva inserción. Cero filas al comenzar indican que contamos con un caso limpio.

**Qué observaremos y cómo lo interpretaremos:** Cero filas tras preparación; si aparece una, el dato existía de una ejecución previa.

**Por qué se escribe así:** El SELECT filtra una sola fila por la clave primaria. En la primera ejecución, no debe existir EJ01.

**Qué cambia / qué veremos al ejecutar el script completo:** Se ven cero filas al empezar desde preparación.

**Pregunta para predecir:** ¿Qué ocurriría al repetir INSERT con la misma clave sin condición?

**Respuesta razonada:** Una violación de clave primaria.

**Siguiente decisión:** Ahora crearemos el evento solamente si no existe.

### Paso 3 — Guardar la marca mediante un INSERT autónomo

**Antes (explicación para entender lo que viene):** No hay fila EJ01 y no hemos abierto BEGIN TRANSACTION.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
IF NOT EXISTS(SELECT 1 FROM dbo.S10_EventosLab WHERE Codigo='EJ01')
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('EJ01',N'Revisión del cliente registrada');
```

**Objetivo de esta consulta o bloque:** Ejecutar la única escritura.

**Por qué lo necesitamos precisamente en este momento:** `IF NOT EXISTS` pregunta si el código falta; solo en ese caso el INSERT crea `EJ01`. Es importante que **no aparezca BEGIN TRANSACTION**: queremos observar qué hace SQL Server con una instrucción aislada. `VALUES` especifica tanto la clave como la descripción humana que guardaremos.

**Qué observaremos y cómo lo interpretaremos:** Se agrega una marca, no se cambia el país ni el contacto del cliente.

**Por qué se escribe así:** IF NOT EXISTS evita duplicar la clave; N permite textos Unicode. No hay COMMIT porque SQL Server autoconfirma el INSERT.

**Qué cambia / qué veremos al ejecutar el script completo:** La fila se inserta; volver a ejecutar no la duplica.

**Pregunta para predecir:** ¿IF NOT EXISTS agrupa varias instrucciones en una transacción?

**Respuesta razonada:** No, solo decide si ejecutar INSERT.

**Siguiente decisión:** Confirmaremos la persistencia y el estado de la conexión.

### Paso 4 — Verificar que el registro persiste

**Antes (explicación para entender lo que viene):** La instrucción de inserción ya terminó.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT Codigo,Descripcion FROM dbo.S10_EventosLab WHERE Codigo='EJ01';
SELECT @@TRANCOUNT AS TransaccionesPendientes;
```

**Objetivo de esta consulta o bloque:** Comprobar la afirmación.

**Por qué lo necesitamos precisamente en este momento:** Esta consulta está después del INSERT para responder **¿sigue aquí el registro?**. El segundo SELECT consulta el contador de transacciones de la conexión; es una prueba complementaria de que no quedó una operación abierta. Si solo viéramos “1 fila afectada”, aún no habríamos mostrado el dato almacenado.

**Qué observaremos y cómo lo interpretaremos:** La marca existe; el contador es cero.

**Por qué se escribe así:** El primer SELECT enseña el dato y el segundo detecta si quedó una transacción abierta.

**Qué cambia / qué veremos al ejecutar el script completo:** Una fila EJ01 y contador 0.

**Pregunta para predecir:** ¿Por qué EJ01 continúa visible?

**Respuesta razonada:** Porque fue confirmado automáticamente al completar INSERT.

**Siguiente decisión:** En EJ02 veremos el modo que exige confirmación manual.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SELECT Codigo,Descripcion FROM dbo.S10_EventosLab WHERE Codigo='EJ01';
IF NOT EXISTS(SELECT 1 FROM dbo.S10_EventosLab WHERE Codigo='EJ01')
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('EJ01',N'Revisión del cliente registrada');
SELECT Codigo,Descripcion FROM dbo.S10_EventosLab WHERE Codigo='EJ01';
SELECT @@TRANCOUNT AS TransaccionesPendientes;
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** Aparece exactamente la marca EJ01 y TransaccionesPendientes=0.

**Interpretación que debes poder explicar:** La instrucción INSERT se confirmó por sí misma porque el modo implícito está apagado y no se abrió una transacción explícita.

**Error habitual:** Confundir BEGIN...END de control con BEGIN TRANSACTION.

**Cómo corregirlo de forma segura:** No se necesita escribir COMMIT para una instrucción autoconfirmada; si quedó una transacción abierta de otro ejercicio, revisar esa conexión antes de continuar.

**Pregunta de cierre:** ¿Por qué la marca sigue allí si nunca escribimos COMMIT?

### T01 — Tarea espejo (resolver sin ver la solución)

**Reto:** T01: registra un evento diferente con código T01, sin BEGIN TRANSACTION ni COMMIT; hazlo idempotente con IF NOT EXISTS.

**Archivo a crear/editar:** `estudiante/tareas/T01_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Usa SET IMPLICIT_TRANSACTIONS OFF y busca primero la marca T01.

**Evidencia obligatoria:** SELECT sobre T01 debe devolver una sola fila y @@TRANCOUNT=0.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.

---

## EJ02 — Probar una transacción implícita y cancelarla

### Tema específico y objetivo de aprendizaje del EJ02

**Tema que trataremos:** Transacción implícita: un cambio que todavía no debe aprobarse.

**Objetivo del ejemplo (qué lograremos al terminar):** Activar el modo implícito, observar un país modificado **dentro de la transacción aún abierta** y cancelarlo; demostrar que el país final coincide con el original.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** Un operador propone cambiar el país de ALFKI, pero aún espera autorización. Queremos que el alumno distinga entre “veo un dato nuevo en la consulta” y “el dato quedó grabado de forma permanente”. Este ejemplo demuestra que el modo implícito mantiene pendiente una modificación hasta que se decide su destino.

**¿Qué datos vamos a observar y modificar?** `dbo.S10_ClientesLab`: solo el campo `Country` del cliente `ALFKI` se modifica temporalmente a `Peru` y luego vuelve a su valor inicial.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. `SELECT @PaisAntes=Country` | ¿Cuál era el país original? Guardarlo primero permite demostrar la reversión después. |
| 2. `UPDATE` y `SELECT ... @@TRANCOUNT` | ¿Cuál es el país provisional y sigue abierta la transacción? Se espera `Peru` y contador 1. |
| 3. `ROLLBACK` y lectura final | ¿Coincide el país después de cancelar con el valor inicial? Ese es el resultado real de la prueba. |

**Prueba de que alcanzamos el objetivo:** Dentro aparece `PaisProvisional=Peru`, `TransaccionesDurante=1`; al final, `PaisAntes=PaisDespues` y `TransaccionesPendientes=0`.

**Idea que no debemos confundir:** `SET IMPLICIT_TRANSACTIONS ON` no equivale a autocommit. Además, una lectura posterior con este modo aún activo puede abrir otra transacción; por eso se desactiva después de revertir.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** Un operador propone cambiar temporalmente el país del cliente ALFKI, pero todavía falta aprobación. El cambio no debe persistir.

**Qué haremos, en orden:** Activaremos SET IMPLICIT_TRANSACTIONS ON, actualizaremos el país, observaremos @@TRANCOUNT=1 y cancelaremos mediante ROLLBACK.

**Qué tenemos ANTES:** El país de ALFKI tiene un valor inicial obtenido del clon de Northwind.

**Qué debe existir DESPUÉS:** Durante la prueba se ve PaísProvisional=Peru y contador 1; al final, país idéntico al inicial y contador 0.

**Qué significa el resultado:** En modo implícito, UPDATE inicia una transacción que sigue pendiente hasta COMMIT o ROLLBACK. Es distinto de autocommit.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Preparar la conexión y capturar el país original

**Antes (explicación para entender lo que viene):** Estamos en una nueva consulta con las tablas de laboratorio disponibles.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
IF @@TRANCOUNT<>0 THROW 51102,N'Existe una transacción pendiente.',1;
DECLARE @PaisAntes NVARCHAR(15);
SELECT @PaisAntes=Country FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
```

**Objetivo de esta consulta o bloque:** Guardar el dato a recuperar.

**Por qué lo necesitamos precisamente en este momento:** La variable `@PaisAntes` captura el país tal como estaba antes de cualquier prueba. Lo hacemos **antes de activar** el modo implícito: consultar una tabla con ese modo activado también podría iniciar una transacción. El `THROW` evita acumular una nueva prueba sobre una transacción previa sin cerrar.

**Qué observaremos y cómo lo interpretaremos:** Un valor de texto queda en memoria de este lote; la tabla sigue igual.

**Por qué se escribe así:** La variable guarda el estado original antes de cambiar el modo; el guard impide comenzar encima de una transacción pendiente.

**Qué cambia / qué veremos al ejecutar el script completo:** Se guarda el país original en @PaisAntes; la tabla aún no cambia.

**Pregunta para predecir:** ¿Por qué leer antes de activar ON?

**Respuesta razonada:** Porque una lectura de tabla también puede iniciar una transacción implícita.

**Siguiente decisión:** Ahora activaremos explícitamente ese modo.

### Paso 2 — Activar la modalidad implícita

**Antes (explicación para entender lo que viene):** Todavía @@TRANCOUNT vale cero y el país original está guardado.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SET IMPLICIT_TRANSACTIONS ON;
BEGIN TRY
```

**Objetivo de esta consulta o bloque:** Cambiar el modo de trabajo.

**Por qué lo necesitamos precisamente en este momento:** `SET IMPLICIT_TRANSACTIONS ON` es una instrucción de configuración, **no una actualización de clientes**. Su objetivo es que el UPDATE posterior abra implícitamente una transacción. No necesitamos `BEGIN TRANSACTION`; eso es precisamente lo que vamos a demostrar.

**Qué observaremos y cómo lo interpretaremos:** El dato sigue original hasta el UPDATE siguiente.

**Por qué se escribe así:** El modo ON deja que la primera sentencia elegible abra una transacción automáticamente. BEGIN TRY inicia control de errores, no una transacción.

**Qué cambia / qué veremos al ejecutar el script completo:** El motor espera una sentencia que dispare la transacción.

**Pregunta para predecir:** ¿SET ON incrementa @@TRANCOUNT por sí mismo?

**Respuesta razonada:** No; lo hará la siguiente sentencia elegible.

**Siguiente decisión:** Provocaremos el UPDATE que abre la transacción.

### Paso 3 — Modificar el país y comprobar el estado pendiente

**Antes (explicación para entender lo que viene):** Dentro de TRY, todavía no escribimos BEGIN TRANSACTION.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    UPDATE dbo.S10_ClientesLab SET Country=N'Peru' WHERE CustomerID='ALFKI';
    SELECT Country AS PaisProvisional,@@TRANCOUNT AS TransaccionesDurante
    FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
```

**Objetivo de esta consulta o bloque:** Ver la diferencia entre provisional y permanente.

**Por qué lo necesitamos precisamente en este momento:** El `UPDATE` altera el campo `Country` de ALFKI. La lectura inmediata se hace **dentro de la misma transacción**: nos permite ver `Peru` y `@@TRANCOUNT=1`. La prueba es didáctica: *ver el valor nuevo no significa que esté definitivamente guardado*.

**Qué observaremos y cómo lo interpretaremos:** La vista provisional muestra Peru mientras la transacción continúa activa.

**Por qué se escribe así:** UPDATE inicia la transacción implícita. SELECT se ejecuta dentro de ella y permite ver el valor provisional y el contador.

**Qué cambia / qué veremos al ejecutar el script completo:** País provisional Peru; TransaccionesDurante=1.

**Pregunta para predecir:** ¿La fila ya es persistente en una transacción aún abierta?

**Respuesta razonada:** No, sigue pendiente de COMMIT o ROLLBACK.

**Siguiente decisión:** El caso no fue aprobado: anularemos este cambio.

### Paso 4 — Revertir y restaurar el modo de conexión

**Antes (explicación para entender lo que viene):** El país aparece modificado solo en la transacción abierta.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    ROLLBACK TRANSACTION;
    SET IMPLICIT_TRANSACTIONS OFF;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    SET IMPLICIT_TRANSACTIONS OFF;
    THROW;
END CATCH;
```

**Objetivo de esta consulta o bloque:** Rechazar la propuesta.

**Por qué lo necesitamos precisamente en este momento:** `ROLLBACK` cancela el trabajo que empezó el UPDATE. Solo **después** apagamos `IMPLICIT_TRANSACTIONS`, para evitar abrir accidentalmente otra transacción al consultar el resultado. El CATCH no es adorno: evita dejar la sesión en un estado confuso si hay un error de ejecución.

**Qué observaremos y cómo lo interpretaremos:** El valor vuelve a su estado anterior; deja de haber una transacción abierta.

**Por qué se escribe así:** ROLLBACK devuelve el valor anterior; SET OFF evita que una consulta posterior abra automáticamente otra transacción. CATCH limpia la conexión ante un error.

**Qué cambia / qué veremos al ejecutar el script completo:** El país se revierte y vuelve el modo autocommit.

**Pregunta para predecir:** ¿Se debe desactivar ON antes o después del ROLLBACK?

**Respuesta razonada:** Después: primero cerramos la transacción abierta.

**Siguiente decisión:** Haremos una comparación externa, ya sin transacción.

### Paso 5 — Comparar antes y después

**Antes (explicación para entender lo que viene):** La conexión está otra vez en autocommit.

**Acción en SSMS 22:** copia o escribe este **paso 5** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT @PaisAntes AS PaisAntes,Country AS PaisDespues,
       @@TRANCOUNT AS TransaccionesPendientes
FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
```

**Objetivo de esta consulta o bloque:** Confirmar la reversión con datos.

**Por qué lo necesitamos precisamente en este momento:** Comparamos en una misma fila el país capturado antes (`@PaisAntes`) y el país leído ahora (`Country`). Esa comparación es más fuerte que ver un mensaje de “comando completado”. El contador 0 confirma que la sesión no quedó con trabajo pendiente.

**Qué observaremos y cómo lo interpretaremos:** Las dos columnas de país deben coincidir; contador 0.

**Por qué se escribe así:** La comparación demuestra que no persistió ningún cambio. @@TRANCOUNT debe ser 0.

**Qué cambia / qué veremos al ejecutar el script completo:** PaisAntes y PaisDespues coinciden; contador 0.

**Pregunta para predecir:** ¿Qué habría ocurrido con COMMIT en lugar de ROLLBACK?

**Respuesta razonada:** El país Peru quedaría guardado.

**Siguiente decisión:** El siguiente ejemplo abre la transacción explícitamente.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
IF @@TRANCOUNT<>0 THROW 51102,N'Existe una transacción pendiente.',1;
DECLARE @PaisAntes NVARCHAR(15);
SELECT @PaisAntes=Country FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
SET IMPLICIT_TRANSACTIONS ON;
BEGIN TRY
    UPDATE dbo.S10_ClientesLab SET Country=N'Peru' WHERE CustomerID='ALFKI';
    SELECT Country AS PaisProvisional,@@TRANCOUNT AS TransaccionesDurante
    FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
    ROLLBACK TRANSACTION;
    SET IMPLICIT_TRANSACTIONS OFF;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    SET IMPLICIT_TRANSACTIONS OFF;
    THROW;
END CATCH;
SELECT @PaisAntes AS PaisAntes,Country AS PaisDespues,
       @@TRANCOUNT AS TransaccionesPendientes
FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** Durante la prueba se ve PaísProvisional=Peru y contador 1; al final, país idéntico al inicial y contador 0.

**Interpretación que debes poder explicar:** En modo implícito, UPDATE inicia una transacción que sigue pendiente hasta COMMIT o ROLLBACK. Es distinto de autocommit.

**Error habitual:** Ejecutar SELECT sobre una tabla mientras SET IMPLICIT_TRANSACTIONS ON y después del ROLLBACK puede iniciar otra transacción inesperada.

**Cómo corregirlo de forma segura:** Desactivar primero SET IMPLICIT_TRANSACTIONS OFF después de ROLLBACK; si ocurre error, el CATCH también apaga el modo.

**Pregunta de cierre:** ¿Basta con terminar UPDATE para dejar el país confirmado?

### T02 — Tarea espejo (resolver sin ver la solución)

**Reto:** T02: sobre ANATR cambia temporalmente Country a Chile en modo implícito, observa el contador, revierte y compara con el valor inicial.

**Archivo a crear/editar:** `estudiante/tareas/T02_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Captura el país antes de activar ON; desactívalo inmediatamente después de ROLLBACK.

**Evidencia obligatoria:** Country final igual a Country inicial; @@TRANCOUNT=0.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.

---

## EJ03 — Confirmar una actualización de cliente con COMMIT

### Tema específico y objetivo de aprendizaje del EJ03

**Tema que trataremos:** Transacción explícita confirmada con COMMIT.

**Objetivo del ejemplo (qué lograremos al terminar):** Actualizar de manera autorizada un contacto, verificar que se afectó **exactamente una fila** y confirmar el cambio usando `COMMIT TRANSACTION`.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** En EJ02 un cambio no aprobado se descartó. Ahora ocurre lo contrario: el responsable de la tienda **sí autorizó** actualizar el contacto de ALFKI. Debemos asegurarnos de que el ID existe antes de dejar el cambio como definitivo. Esto introduce la decisión empresarial “confirmar solo si pasó la validación”.

**¿Qué datos vamos a observar y modificar?** `dbo.S10_ClientesLab`: `ContactName` de `ALFKI` se establece como `Contacto S10`.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. `SELECT @ContactoAntes=ContactName` | ¿Cómo figuraba el contacto antes de intervenir? Necesitamos un punto inicial. |
| 2. `UPDATE ... WHERE CustomerID=ALFKI` | ¿Cuál es la única fila autorizada a cambiar? La cláusula WHERE limita el alcance. |
| 3. `SET @Filas=@@ROWCOUNT` | ¿Se encontró exactamente un cliente? Si no, lanzamos error y cancelamos. |
| 4. `SELECT` final | ¿Cuál era el contacto, cuál es el contacto confirmado y cuántas filas cambiaron? |

**Prueba de que alcanzamos el objetivo:** `ContactoConfirmado=Contacto S10`, `FilasAfectadas=1`, `TransaccionesPendientes=0`.

**Idea que no debemos confundir:** `COMMIT` no decide por sí mismo si el dato es correcto: nuestra validación `@Filas=1` justifica la confirmación.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** El responsable comercial aprobó un cambio en el nombre de contacto de ALFKI. Ya no queremos una modificación provisional.

**Qué haremos, en orden:** Abriremos una transacción explícita, actualizaremos ContactName, verificaremos @@ROWCOUNT y confirmaremos con COMMIT.

**Qué tenemos ANTES:** ContactName contiene el valor copiado de Northwind.

**Qué debe existir DESPUÉS:** El campo termina como Contacto S10 y hay exactamente una fila afectada; @@TRANCOUNT=0.

**Qué significa el resultado:** COMMIT hace persistente la actualización explícita cuando la validación confirmó que se afectó la fila esperada.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Capturar el contacto original

**Antes (explicación para entender lo que viene):** La tabla S10_ClientesLab conserva un cliente ALFKI con su contacto.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
DECLARE @ContactoAntes NVARCHAR(30),@Filas INT;
SELECT @ContactoAntes=ContactName FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
```

**Objetivo de esta consulta o bloque:** Registrar el contacto anterior.

**Por qué lo necesitamos precisamente en este momento:** Necesitamos saber **qué había antes** para enseñar que COMMIT produjo un cambio real. `@ContactoAntes` es una variable local, no una columna nueva; `@Filas` se usará luego para comprobar el alcance del UPDATE.

**Qué observaremos y cómo lo interpretaremos:** La lectura no modifica la fila del cliente.

**Por qué se escribe así:** Separamos el valor anterior para demostrar el cambio. @Filas guardará el efecto del UPDATE.

**Qué cambia / qué veremos al ejecutar el script completo:** No se ha modificado nada aún.

**Pregunta para predecir:** ¿Para qué guardar ContactName anterior?

**Respuesta razonada:** Para comparar el estado antes y después del COMMIT.

**Siguiente decisión:** Abriremos ahora la unidad explícita.

### Paso 2 — Abrir la transacción dentro de TRY

**Antes (explicación para entender lo que viene):** El dato original ya está en @ContactoAntes.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
```

**Objetivo de esta consulta o bloque:** Delimitar el trabajo aprobado.

**Por qué lo necesitamos precisamente en este momento:** `BEGIN TRANSACTION` indica desde qué punto el cambio debe considerarse pendiente de decisión. `BEGIN TRY` nos permite capturar problemas posteriores. Estas instrucciones preparan el contexto; por sí mismas aún no cambian datos.

**Qué observaremos y cómo lo interpretaremos:** Queda lista una transacción explícita para actualizar la fila correcta.

**Por qué se escribe así:** TRY permite dirigir errores al CATCH; BEGIN TRANSACTION agrupa las operaciones hasta COMMIT o ROLLBACK.

**Qué cambia / qué veremos al ejecutar el script completo:** @@TRANCOUNT pasa a 1 en esta conexión.

**Pregunta para predecir:** ¿TRY crea transacción por sí mismo?

**Respuesta razonada:** No. La crea BEGIN TRANSACTION.

**Siguiente decisión:** Modificaremos el contacto solo dentro de la unidad abierta.

### Paso 3 — Actualizar y asegurar una sola fila

**Antes (explicación para entender lo que viene):** La transacción está abierta; falta validar el resultado.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    UPDATE dbo.S10_ClientesLab SET ContactName=N'Contacto S10'
    WHERE CustomerID='ALFKI';
    SET @Filas=@@ROWCOUNT;
    IF @Filas<>1 THROW 51103,N'Se esperaba actualizar un cliente.',1;
```

**Objetivo de esta consulta o bloque:** Modificar solamente la fila solicitada.

**Por qué lo necesitamos precisamente en este momento:** `WHERE CustomerID=ALFKI` evita actualizar a todos los clientes. Justo después del UPDATE tomamos `@@ROWCOUNT`, porque otra sentencia podría modificar su valor. Si el resultado no es 1, `THROW` impide que se ejecute COMMIT: un cliente inexistente no debe informarse como una actualización exitosa.

**Qué observaremos y cómo lo interpretaremos:** El contacto de ALFKI cambia provisionalmente y la validación debe indicar 1 fila.

**Por qué se escribe así:** WHERE protege a otros clientes, @@ROWCOUNT informa las filas afectadas inmediatamente, e IF impide confirmar una operación ambigua.

**Qué cambia / qué veremos al ejecutar el script completo:** ALFKI tiene el nuevo contacto aún no confirmado.

**Pregunta para predecir:** ¿Qué pasa si el código de cliente no existe?

**Respuesta razonada:** @Filas será 0 y THROW llevará el flujo al CATCH.

**Siguiente decisión:** Solo después de validar confirmaremos.

### Paso 4 — Confirmar o revertir según el resultado

**Antes (explicación para entender lo que viene):** UPDATE afectó exactamente una fila, por lo que podemos confirmar.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
```

**Objetivo de esta consulta o bloque:** Convertir la modificación en definitiva.

**Por qué lo necesitamos precisamente en este momento:** La validación anterior justifica `COMMIT`: ahora sí queremos conservar el nuevo contacto. Si algo falló, CATCH consulta `XACT_STATE()` y revierte cuando haya una transacción activa. No se confirman cambios a ciegas.

**Qué observaremos y cómo lo interpretaremos:** La transacción se cierra; el contacto nuevo debe persistir.

**Por qué se escribe así:** COMMIT cierra la unidad con cambios persistentes. Si se produjo cualquier error capturado, CATCH revierte y THROW informa el problema.

**Qué cambia / qué veremos al ejecutar el script completo:** La transacción queda cerrada con actualización persistente.

**Pregunta para predecir:** ¿Por qué ROLLBACK solo si XACT_STATE()<>0?

**Respuesta razonada:** Para no revertir cuando el motor ya cerró la transacción.

**Siguiente decisión:** La consulta final probará persistencia.

### Paso 5 — Comparar el contacto anterior con el confirmado

**Antes (explicación para entender lo que viene):** No hay transacción abierta y el valor nuevo permanece.

**Acción en SSMS 22:** copia o escribe este **paso 5** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT @ContactoAntes AS ContactoAntes,ContactName AS ContactoConfirmado,
       @Filas AS FilasAfectadas,@@TRANCOUNT AS TransaccionesPendientes
FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
```

**Objetivo de esta consulta o bloque:** Comparar evidencia de antes y después.

**Por qué lo necesitamos precisamente en este momento:** El SELECT final coloca el nombre anterior, el confirmado y la cantidad de filas en una misma tabla de resultados. Esta es la evidencia del objetivo, mucho más informativa que ejecutar UPDATE sin comparar.

**Qué observaremos y cómo lo interpretaremos:** `ContactoConfirmado` debe ser `Contacto S10`, filas=1, pendientes=0.

**Por qué se escribe así:** La consulta es la evidencia de que el COMMIT hizo permanente el contacto y se afectó la fila correcta.

**Qué cambia / qué veremos al ejecutar el script completo:** ContactoConfirmado=Contacto S10, FilasAfectadas=1, contador 0.

**Pregunta para predecir:** ¿Repetir el script puede mostrar Antes y Después iguales?

**Respuesta razonada:** Sí: UPDATE sigue afectando la fila aunque el valor ya coincida.

**Siguiente decisión:** El siguiente ejercicio prueba la decisión opuesta: ROLLBACK.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
DECLARE @ContactoAntes NVARCHAR(30),@Filas INT;
SELECT @ContactoAntes=ContactName FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
BEGIN TRY
    BEGIN TRANSACTION;
    UPDATE dbo.S10_ClientesLab SET ContactName=N'Contacto S10'
    WHERE CustomerID='ALFKI';
    SET @Filas=@@ROWCOUNT;
    IF @Filas<>1 THROW 51103,N'Se esperaba actualizar un cliente.',1;
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
SELECT @ContactoAntes AS ContactoAntes,ContactName AS ContactoConfirmado,
       @Filas AS FilasAfectadas,@@TRANCOUNT AS TransaccionesPendientes
FROM dbo.S10_ClientesLab WHERE CustomerID='ALFKI';
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** El campo termina como Contacto S10 y hay exactamente una fila afectada; @@TRANCOUNT=0.

**Interpretación que debes poder explicar:** COMMIT hace persistente la actualización explícita cuando la validación confirmó que se afectó la fila esperada.

**Error habitual:** Confundir @@ROWCOUNT con una consulta posterior: su valor cambia con cada sentencia.

**Cómo corregirlo de forma segura:** Copiar @@ROWCOUNT inmediatamente después del UPDATE a @Filas; si no es 1, lanzar error para impedir COMMIT.

**Pregunta de cierre:** ¿En qué momento el cambio deja de ser provisional?

### T03 — Tarea espejo (resolver sin ver la solución)

**Reto:** T03: actualiza ContactName de ANATR a Responsable T03 mediante BEGIN TRANSACTION, comprueba una fila y confirma.

**Archivo a crear/editar:** `estudiante/tareas/T03_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Aísla la comprobación @@ROWCOUNT antes de ejecutar cualquier SELECT adicional.

**Evidencia obligatoria:** Contacto de ANATR igual a Responsable T03; @@TRANCOUNT=0.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.

---

## EJ04 — Descartar una modificación no aprobada con ROLLBACK

### Tema específico y objetivo de aprendizaje del EJ04

**Tema que trataremos:** Transacción explícita cancelada completamente.

**Objetivo del ejemplo (qué lograremos al terminar):** Cambiar temporalmente un país, comprobar el efecto **mientras la transacción está abierta** y demostrar que `ROLLBACK` restaura el valor anterior.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** El área comercial detecta una propuesta que no fue aprobada. Queremos practicar una decisión distinta de EJ03: incluso si el `UPDATE` se ejecutó técnicamente bien, el negocio puede exigir que el dato **no** permanezca. ROLLBACK sirve para rechazar una operación antes de confirmarla.

**¿Qué datos vamos a observar y modificar?** `dbo.S10_ClientesLab`: `Country` del cliente `ANATR` pasa provisionalmente a `Chile` y después se revierte.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. Primera lectura de `Country` | ¿De qué país partimos? La variable `@PaisAntes` lo recuerda. |
| 2. `UPDATE` y `SELECT` dentro de `TRY` | ¿Se observa el país provisional y la transacción sigue abierta? |
| 3. `ROLLBACK` y consulta final | ¿Volvimos al país inicial? ¿Está cerrada la transacción? |

**Prueba de que alcanzamos el objetivo:** Se ve `PaisProvisional=Chile` durante la prueba; la consulta posterior vuelve al valor original y muestra `TransaccionesPendientes=0`.

**Idea que no debemos confundir:** Un UPDATE exitoso puede ser una operación no autorizada: éxito técnico no es equivalente a aprobación funcional.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** Un operador cambió Country para ANATR, pero el área responsable rechazó la modificación antes de guardarla.

**Qué haremos, en orden:** Abriremos una transacción explícita, cambiaremos el país, observaremos el estado provisional y ejecutaremos ROLLBACK.

**Qué tenemos ANTES:** ANATR conserva un país inicial en la tabla auxiliar.

**Qué debe existir DESPUÉS:** El cambio provisional a Chile se ve dentro de la transacción, pero PaísDespues es igual a PaísAntes; contador cero.

**Qué significa el resultado:** ROLLBACK deshace todo lo realizado desde BEGIN TRANSACTION aunque el UPDATE se haya ejecutado sin error.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Guardar el país original de ANATR

**Antes (explicación para entender lo que viene):** Al iniciar la consulta, no conocemos aún el valor que necesitaremos restaurar.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
DECLARE @PaisAntes NVARCHAR(15);
SELECT @PaisAntes=Country FROM dbo.S10_ClientesLab WHERE CustomerID='ANATR';
```

**Objetivo de esta consulta o bloque:** Definir el valor que no queremos perder.

**Por qué lo necesitamos precisamente en este momento:** Capturamos el país de ANATR antes de la propuesta para poder demostrar que una reversión recuperó exactamente ese mismo valor. No cambia la tabla.

**Qué observaremos y cómo lo interpretaremos:** Se almacena el país real del cliente en una variable.

**Por qué se escribe así:** Guardamos el país original como referencia. No cambiamos la tabla todavía.

**Qué cambia / qué veremos al ejecutar el script completo:** La variable contiene el valor previo.

**Pregunta para predecir:** ¿Podemos conocer la reversión sin conservar el dato original?

**Respuesta razonada:** Sí mediante consulta, pero guardar el antes facilita evidencia clara.

**Siguiente decisión:** Haremos el cambio bajo una transacción.

### Paso 2 — Abrir la transacción controlada

**Antes (explicación para entender lo que viene):** El valor inicial quedó almacenado.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
```

**Objetivo de esta consulta o bloque:** Iniciar la zona de pruebas.

**Por qué lo necesitamos precisamente en este momento:** En EJ03 abrimos una transacción para confirmar; ahora la abrimos para **cancelar conscientemente**. Esta es una decisión de negocio: la propuesta no está aprobada, aunque el UPDATE pueda ejecutarse correctamente.

**Qué observaremos y cómo lo interpretaremos:** La operación posterior aún no es permanente.

**Por qué se escribe así:** BEGIN TRANSACTION delimita lo que se podrá deshacer de una sola vez; TRY maneja fallos inesperados.

**Qué cambia / qué veremos al ejecutar el script completo:** Hay una transacción abierta.

**Pregunta para predecir:** ¿Qué sentencia cierra esta unidad con cancelación?

**Respuesta razonada:** ROLLBACK TRANSACTION.

**Siguiente decisión:** Aplicaremos un cambio provisional.

### Paso 3 — Modificar y visualizar el cambio provisional

**Antes (explicación para entender lo que viene):** ANATR todavía conserva su país anterior.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    UPDATE dbo.S10_ClientesLab SET Country=N'Chile' WHERE CustomerID='ANATR';
    IF @@ROWCOUNT<>1 THROW 51104,N'Cliente no encontrado.',1;
    SELECT Country AS PaisProvisional,@@TRANCOUNT AS Durante
    FROM dbo.S10_ClientesLab WHERE CustomerID='ANATR';
```

**Objetivo de esta consulta o bloque:** Mostrar qué se habría cambiado.

**Por qué lo necesitamos precisamente en este momento:** El UPDATE pone `Chile` de forma provisional. `@@ROWCOUNT` debe indicar una fila; el SELECT siguiente muestra el país provisional mientras el bloque está abierto. De esta manera observamos el efecto real que habría tenido la modificación.

**Qué observaremos y cómo lo interpretaremos:** Durante la operación aparece Chile y el contador es 1.

**Por qué se escribe así:** El UPDATE afecta ANATR únicamente; la comprobación inmediata protege contra códigos erróneos. SELECT muestra el valor dentro de la transacción.

**Qué cambia / qué veremos al ejecutar el script completo:** Country provisional Chile, Durante=1.

**Pregunta para predecir:** ¿El SELECT confirma la modificación?

**Respuesta razonada:** No; solo la muestra en esta conexión.

**Siguiente decisión:** El responsable rechazó el cambio y debemos deshacerlo.

### Paso 4 — Cancelar todo el bloque abierto

**Antes (explicación para entender lo que viene):** El valor temporal está visible, pero no aprobado.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    ROLLBACK TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
```

**Objetivo de esta consulta o bloque:** Cancelar la operación completa.

**Por qué lo necesitamos precisamente en este momento:** En lugar de `COMMIT`, usamos `ROLLBACK` porque el cambio no fue autorizado. SQL Server deshace el UPDATE de esta transacción: no es necesario otro UPDATE manual para volver a escribir el país inicial.

**Qué observaremos y cómo lo interpretaremos:** El país deja de estar modificado al cerrar la transacción.

**Por qué se escribe así:** ROLLBACK deja los datos como estaban al abrir la transacción. CATCH también revierte ante un error técnico.

**Qué cambia / qué veremos al ejecutar el script completo:** No queda transacción abierta; ANATR recupera su país previo.

**Pregunta para predecir:** ¿La reversión borra cambios de ejercicios anteriores ya confirmados?

**Respuesta razonada:** No; solo los de esta transacción.

**Siguiente decisión:** Comprobaremos la restauración fuera del bloque.

### Paso 5 — Mostrar el país restaurado

**Antes (explicación para entender lo que viene):** ROLLBACK ya ha terminado.

**Acción en SSMS 22:** copia o escribe este **paso 5** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT @PaisAntes AS PaisAntes,Country AS PaisDespues,
       @@TRANCOUNT AS TransaccionesPendientes
FROM dbo.S10_ClientesLab WHERE CustomerID='ANATR';
```

**Objetivo de esta consulta o bloque:** Probar que no persistió.

**Por qué lo necesitamos precisamente en este momento:** El SELECT compara `@PaisAntes` con `Country` tras cerrar la operación. Si coinciden, hemos verificado la reversión. `@@TRANCOUNT=0` confirma que no dejamos una transacción abierta.

**Qué observaremos y cómo lo interpretaremos:** El país anterior y final coinciden, contador cero.

**Por qué se escribe así:** La comparación antes/después demuestra la reversión.

**Qué cambia / qué veremos al ejecutar el script completo:** Ambos países iguales; contador 0.

**Pregunta para predecir:** ¿Qué hubiera cambiado si ejecutábamos COMMIT?

**Respuesta razonada:** El valor Chile habría quedado persistido.

**Siguiente decisión:** En EJ05 la transacción protegerá dos tablas relacionadas.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
DECLARE @PaisAntes NVARCHAR(15);
SELECT @PaisAntes=Country FROM dbo.S10_ClientesLab WHERE CustomerID='ANATR';
BEGIN TRY
    BEGIN TRANSACTION;
    UPDATE dbo.S10_ClientesLab SET Country=N'Chile' WHERE CustomerID='ANATR';
    IF @@ROWCOUNT<>1 THROW 51104,N'Cliente no encontrado.',1;
    SELECT Country AS PaisProvisional,@@TRANCOUNT AS Durante
    FROM dbo.S10_ClientesLab WHERE CustomerID='ANATR';
    ROLLBACK TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
SELECT @PaisAntes AS PaisAntes,Country AS PaisDespues,
       @@TRANCOUNT AS TransaccionesPendientes
FROM dbo.S10_ClientesLab WHERE CustomerID='ANATR';
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** El cambio provisional a Chile se ve dentro de la transacción, pero PaísDespues es igual a PaísAntes; contador cero.

**Interpretación que debes poder explicar:** ROLLBACK deshace todo lo realizado desde BEGIN TRANSACTION aunque el UPDATE se haya ejecutado sin error.

**Error habitual:** Creer que una sentencia UPDATE exitosa obliga a usar COMMIT.

**Cómo corregirlo de forma segura:** Separar éxito técnico de aprobación del negocio; usar ROLLBACK si la operación no debe permanecer.

**Pregunta de cierre:** ¿Un UPDATE exitoso puede terminar deshecho?

### T04 — Tarea espejo (resolver sin ver la solución)

**Reto:** T04: realiza un cambio provisional de ContactName para ALFKI a No aprobado T04, verifica dentro y cancela; muestra la comparación.

**Archivo a crear/editar:** `estudiante/tareas/T04_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Guarda el nombre original y usa BEGIN TRANSACTION antes del UPDATE.

**Evidencia obligatoria:** El contacto de ALFKI queda igual al inicial; @@TRANCOUNT=0.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.

---

## EJ05 — Registrar una venta y descontar stock como una unidad

### Tema específico y objetivo de aprendizaje del EJ05

**Tema que trataremos:** Atomicidad: venta y descuento de inventario en la misma transacción.

**Objetivo del ejemplo (qué lograremos al terminar):** Registrar una venta de **2 unidades del producto 1** para ALFKI, reducir el stock y confirmar ambos cambios **juntos**; demostrar que existe una venta que corresponde al descuento.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** Esta es la situación central de la sesión. Si el stock baja, pero la venta no se registra, habrá existencias incorrectas; si se crea la venta sin descontar stock, habrá una venta que el almacén no respalda. No basta con dos SQL separados: ambos forman una sola operación del negocio.

**¿Qué datos vamos a observar y modificar?** `dbo.S10_ProductosLab`: `Stock` del producto 1 disminuye en 2. `dbo.S10_VentasLab`: aparece una nueva venta de 2 unidades vinculada al cliente ALFKI.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. `SELECT @StockAntes=Stock` | ¿Cuántas unidades teníamos antes del pedido? Necesitamos calcular la diferencia. |
| 2. `UPDATE ... Stock>=@Cantidad` | ¿Existe stock suficiente para reservar y descontar dos unidades? Si no, no debemos crear una venta. |
| 3. `INSERT ... SELECT Precio` | ¿Quedó registrada una venta que usa el precio del producto del laboratorio? |
| 4. Dos consultas finales | ¿El stock bajó exactamente dos unidades **y** existe la fila de venta? Deben verificarse ambas. |

**Prueba de que alcanzamos el objetivo:** Si el stock inicial era 20, termina en 18; aparece una nueva `VentaID` para ALFKI, producto 1, cantidad 2; `Pendientes=0`.

**Idea que no debemos confundir:** La primera vez después de preparación el stock es 20; si ya ejecutaste una venta, el valor anterior puede ser diferente. La comprobación correcta es **StockDespues = StockAntes − 2**, no asumir siempre 18.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** Un cliente compra dos unidades de Chai. La venta no debe existir si el inventario no pudo descontarse, ni el stock debe bajar sin una venta registrada.

**Qué haremos, en orden:** Agruparemos UPDATE del stock y INSERT en la tabla de ventas dentro de la misma transacción, comprobaremos filas afectadas y el ID de la venta.

**Qué tenemos ANTES:** El stock del producto 1 fue inicializado en 20; ALFKI figura en el laboratorio.

**Qué debe existir DESPUÉS:** Al iniciar desde preparación, StockAntes=20, StockDespues=18, Diferencia=-2 y aparece una venta de 2 unidades.

**Qué significa el resultado:** La atomicidad exige que stock y venta se confirmen juntos; si falla cualquiera, el CATCH revierte ambos.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Definir la venta y conocer el stock inicial

**Antes (explicación para entender lo que viene):** Se inició el laboratorio y existen ALFKI y el producto 1.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SET XACT_ABORT ON;
DECLARE @Cliente NCHAR(5)='ALFKI',@Producto INT=1,@Cantidad INT=2;
DECLARE @StockAntes INT,@VentaID INT;
SELECT @StockAntes=Stock FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
```

**Objetivo de esta consulta o bloque:** Preparar una orden real.

**Por qué lo necesitamos precisamente en este momento:** Las variables expresan el pedido: cliente ALFKI, producto 1, dos unidades. El SELECT de stock contesta **¿podemos medir cuántas unidades desaparecieron?** sin depender de un número fijo si ya ejecutaste otras ventas.

**Qué observaremos y cómo lo interpretaremos:** Stock inicial capturado; ninguna venta creada todavía.

**Por qué se escribe así:** @Cliente identifica al comprador; @Producto la mercancía; @Cantidad la demanda. Guardamos @StockAntes para verificar el descenso y @VentaID para buscar exactamente el pedido creado.

**Qué cambia / qué veremos al ejecutar el script completo:** StockAntes es 20 tras reiniciar el laboratorio.

**Pregunta para predecir:** ¿Por qué no usamos directamente las tablas originales de Products?

**Respuesta razonada:** Para practicar cambios de forma segura sobre S10_ProductosLab.

**Siguiente decisión:** Protegeremos las dos escrituras con una misma transacción.

### Paso 2 — Abrir la unidad indivisible

**Antes (explicación para entender lo que viene):** Tenemos los datos iniciales y todavía ninguna venta nueva.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
```

**Objetivo de esta consulta o bloque:** Abrir la operación indivisible.

**Por qué lo necesitamos precisamente en este momento:** Todo lo que ocurra entre `BEGIN TRANSACTION` y el COMMIT/ROLLBACK representa **un solo pedido**. Si el registro de ventas falla después de descontar stock, CATCH debe revertir ambos pasos.

**Qué observaremos y cómo lo interpretaremos:** Existe una transacción abierta, pero aún sin modificación de stock.

**Por qué se escribe así:** Este BEGIN TRANSACTION abarca inventario y registro de venta; si falla la segunda operación no se conservará la primera.

**Qué cambia / qué veremos al ejecutar el script completo:** Se abre una transacción activa.

**Pregunta para predecir:** ¿Cuándo debería ejecutarse COMMIT?

**Respuesta razonada:** Después de completar satisfactoriamente ambas escrituras.

**Siguiente decisión:** Primero debemos comprobar y reducir existencias.

### Paso 3 — Descontar solamente si hay stock suficiente

**Antes (explicación para entender lo que viene):** El stock actual debe ser mayor o igual a la cantidad solicitada.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    UPDATE dbo.S10_ProductosLab
       SET Stock=Stock-@Cantidad
     WHERE ProductID=@Producto AND Stock>=@Cantidad;
    IF @@ROWCOUNT<>1 THROW 51105,N'Producto inexistente o stock insuficiente.',1;
```

**Objetivo de esta consulta o bloque:** Comprobar disponibilidad antes de seguir.

**Por qué lo necesitamos precisamente en este momento:** El UPDATE resta `@Cantidad` **solo si** la fila del producto existe y `Stock>=@Cantidad`. Guardamos inmediatamente `@@ROWCOUNT`; si es 0, `THROW` cancela la intención de venta. Así evitamos existencias negativas o una venta sin unidades disponibles.

**Qué observaremos y cómo lo interpretaremos:** Si había al menos 2 unidades, el stock provisional se reduce en dos.

**Por qué se escribe así:** WHERE Stock>=@Cantidad impide valores negativos. @@ROWCOUNT=1 asegura que el producto existió y pudo descontarse. La restricción CHECK también protege el dato.

**Qué cambia / qué veremos al ejecutar el script completo:** El stock queda provisionalmente reducido 2 unidades.

**Pregunta para predecir:** ¿Qué indica @@ROWCOUNT=0?

**Respuesta razonada:** No hubo producto apto o no alcanzaron unidades.

**Siguiente decisión:** Como el descuento funcionó, registraremos la venta.

### Paso 4 — Insertar venta y guardar su identificador

**Antes (explicación para entender lo que viene):** Ya descontamos stock dentro de la misma transacción.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    INSERT INTO dbo.S10_VentasLab(CustomerID,ProductID,Cantidad,Precio)
    SELECT @Cliente,@Producto,@Cantidad,Precio
    FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
    IF @@ROWCOUNT<>1 THROW 51115,N'No se registró la venta.',1;
    SET @VentaID=CONVERT(INT,SCOPE_IDENTITY());
```

**Objetivo de esta consulta o bloque:** Asentar el hecho de negocio.

**Por qué lo necesitamos precisamente en este momento:** El INSERT construye la fila de venta usando el precio del mismo producto; no inventamos una cifra en el SQL. `SCOPE_IDENTITY()` guarda el ID creado, que después permitirá buscar la venta exacta. Si no se insertó una fila, no debemos confirmar el descuento de stock.

**Qué observaremos y cómo lo interpretaremos:** Se genera un ID de venta que corresponde al pedido de 2 unidades.

**Por qué se escribe así:** INSERT...SELECT toma el precio del producto del laboratorio y registra la cantidad. SCOPE_IDENTITY devuelve el IDENTITY de la venta recién insertada en este ámbito.

**Qué cambia / qué veremos al ejecutar el script completo:** Ahora hay una venta provisional asociada al descuento.

**Pregunta para predecir:** ¿Por qué guardar @VentaID?

**Respuesta razonada:** Para comprobar el registro exacto sin confundirlo con ventas anteriores.

**Siguiente decisión:** Confirmaremos las dos modificaciones juntas.

### Paso 5 — Confirmar el par stock/venta o revertir

**Antes (explicación para entender lo que viene):** Las dos operaciones se ejecutaron y se validaron.

**Acción en SSMS 22:** copia o escribe este **paso 5** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
```

**Objetivo de esta consulta o bloque:** Confirmar solo cuando ambos pasos pasaron.

**Por qué lo necesitamos precisamente en este momento:** El COMMIT ocurre **después** de comprobar que el stock se redujo y la venta se insertó. Si cualquiera falla, CATCH revierte la operación completa. Este es el principio de atomicidad aplicado al caso comercial.

**Qué observaremos y cómo lo interpretaremos:** O persisten ambos cambios, o no persiste ninguno.

**Por qué se escribe así:** COMMIT consolida stock y venta. Si falla stock o INSERT, CATCH cancela el conjunto. XACT_ABORT ON refuerza la cancelación ante errores de ejecución.

**Qué cambia / qué veremos al ejecutar el script completo:** Los dos cambios permanecen o ninguno permanece.

**Pregunta para predecir:** ¿Por qué THROW se coloca después de ROLLBACK?

**Respuesta razonada:** Para no ocultar el error real tras haber dejado la conexión limpia.

**Siguiente decisión:** Verificaremos stock y la venta recién creada.

### Paso 6 — Demostrar las dos consecuencias relacionadas

**Antes (explicación para entender lo que viene):** La transacción ya fue cerrada.

**Acción en SSMS 22:** copia o escribe este **paso 6** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT @StockAntes AS StockAntes,Stock AS StockDespues,
       Stock-@StockAntes AS Diferencia,@@TRANCOUNT AS Pendientes
FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
SELECT VentaID,CustomerID,ProductID,Cantidad
FROM dbo.S10_VentasLab WHERE VentaID=@VentaID;
```

**Objetivo de esta consulta o bloque:** Buscar evidencia en dos tablas.

**Por qué lo necesitamos precisamente en este momento:** El primer SELECT responde **¿cuántas unidades se descontaron?**; el segundo **¿qué venta quedó registrada con ese identificador?**. Las dos consultas se necesitan: una sola tabla no podría demostrar que ambos lados del negocio permanecen consistentes.

**Qué observaremos y cómo lo interpretaremos:** La diferencia será -2 y habrá una venta de cantidad 2, con pendientes 0.

**Por qué se escribe así:** La diferencia debe ser exactamente -2, debe existir la venta con Cantidad=2 y no quedar transacción abierta.

**Qué cambia / qué veremos al ejecutar el script completo:** Stock baja dos; se muestra una venta; Pendientes=0.

**Pregunta para predecir:** ¿Qué ocurre si repetimos la demostración muchas veces?

**Respuesta razonada:** Se acumulan ventas y disminuye stock; reinicia el laboratorio para volver a 20.

**Siguiente decisión:** El próximo caso provocará un fallo real y revisará la reversión.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SET XACT_ABORT ON;
DECLARE @Cliente NCHAR(5)='ALFKI',@Producto INT=1,@Cantidad INT=2;
DECLARE @StockAntes INT,@VentaID INT;
SELECT @StockAntes=Stock FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
BEGIN TRY
    BEGIN TRANSACTION;
    UPDATE dbo.S10_ProductosLab
       SET Stock=Stock-@Cantidad
     WHERE ProductID=@Producto AND Stock>=@Cantidad;
    IF @@ROWCOUNT<>1 THROW 51105,N'Producto inexistente o stock insuficiente.',1;
    INSERT INTO dbo.S10_VentasLab(CustomerID,ProductID,Cantidad,Precio)
    SELECT @Cliente,@Producto,@Cantidad,Precio
    FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
    IF @@ROWCOUNT<>1 THROW 51115,N'No se registró la venta.',1;
    SET @VentaID=CONVERT(INT,SCOPE_IDENTITY());
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
SELECT @StockAntes AS StockAntes,Stock AS StockDespues,
       Stock-@StockAntes AS Diferencia,@@TRANCOUNT AS Pendientes
FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
SELECT VentaID,CustomerID,ProductID,Cantidad
FROM dbo.S10_VentasLab WHERE VentaID=@VentaID;
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** Al iniciar desde preparación, StockAntes=20, StockDespues=18, Diferencia=-2 y aparece una venta de 2 unidades.

**Interpretación que debes poder explicar:** La atomicidad exige que stock y venta se confirmen juntos; si falla cualquiera, el CATCH revierte ambos.

**Error habitual:** Descontar antes y registrar la venta después sin transacción, dejando datos parciales si INSERT falla.

**Cómo corregirlo de forma segura:** Usar BEGIN TRANSACTION antes del primer UPDATE y COMMIT solo después del INSERT y validaciones.

**Pregunta de cierre:** ¿Qué pasaría si la venta no se insertara después de reducir stock?

### T05 — Tarea espejo (resolver sin ver la solución)

**Reto:** T05: registra una venta de tres unidades del producto 2 para ANATR, decrementa stock y valida venta y stock en la misma transacción.

**Archivo a crear/editar:** `estudiante/tareas/T05_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Reutiliza la secuencia de EJ05, cambia ID, cliente y cantidad; no omitas la validación de stock.

**Evidencia obligatoria:** El stock disminuye exactamente 3 y existe una nueva venta del producto 2.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.

---

## EJ06 — Detectar un error y evitar que un stock parcial quede guardado

### Tema específico y objetivo de aprendizaje del EJ06

**Tema que trataremos:** Manejo de errores: evitar que sobreviva un descuento parcial.

**Objetivo del ejemplo (qué lograremos al terminar):** Simular una instrucción inválida **después** de cambiar provisionalmente el stock y verificar que el error provoca un `ROLLBACK`, dejando las unidades iniciales intactas.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** El peligro más difícil es que el primer paso funcione y el segundo falle. Para verlo sin dañar datos reales, intentaremos insertar una clave `DUPLICADO` que el laboratorio ya tiene, produciendo una violación de clave primaria. La práctica debe mostrar cómo una falla posterior cancela también el descuento anterior.

**¿Qué datos vamos a observar y modificar?** `dbo.S10_ProductosLab`: stock del producto 2 se intenta reducir en 1; `dbo.S10_EventosLab`: la inserción duplicada falla. Ninguno de los dos cambios queda confirmado.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. `SELECT @StockAntes=Stock` | ¿Con qué cantidad empezamos? |
| 2. `UPDATE Stock=Stock-1` | ¿Qué operación puede quedar parcialmente hecha si no agrupamos ambas acciones? |
| 3. `INSERT` con clave `DUPLICADO` | ¿Qué error de prueba provoca el fallo dentro del mismo TRY? |
| 4. `SELECT ERROR_NUMBER(), ERROR_MESSAGE(), XACT_STATE()` | ¿Qué error ocurrió y en qué estado quedó la transacción? |
| 5. `SELECT` posterior al CATCH | ¿Se restauró el stock? Esperamos diferencia 0 y contador 0. |

**Prueba de que alcanzamos el objetivo:** Aparece información del error controlado; `StockAntes=StockDespues`, `Diferencia=0`, `Pendientes=0`.

**Idea que no debemos confundir:** El objetivo NO es lograr que el INSERT duplicado funcione. **Que falle controladamente es la prueba** de que la reversión protege los datos. Con `XACT_ABORT ON`, el manejo del estado transaccional es indispensable.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** Una operación de inventario reduce una unidad, pero después intenta insertar un evento con un Codigo que ya existe. Ese segundo paso es inválido.

**Qué haremos, en orden:** Provocaremos una infracción real de clave primaria, la capturaremos con TRY/CATCH, inspeccionaremos XACT_STATE y revertiremos el stock inicial.

**Qué tenemos ANTES:** La preparación dejó la marca DUPLICADO ya guardada y stock=20 en el producto 2 (salvo prácticas anteriores).

**Qué debe existir DESPUÉS:** Se muestra un error de duplicado en el CATCH; stock final igual al stock anterior y Pendientes=0.

**Qué significa el resultado:** Un error en una sentencia posterior no debe dejar confirmado el UPDATE previo; XACT_STATE orienta la cancelación segura.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Asegurar la marca duplicada y capturar stock

**Antes (explicación para entender lo que viene):** La fila DUPLICADO está garantizada por el script de preparación.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SET XACT_ABORT ON;
DECLARE @StockAntes INT;
SELECT @StockAntes=Stock FROM dbo.S10_ProductosLab WHERE ProductID=2;
```

**Objetivo de esta consulta o bloque:** Registrar el inventario inicial.

**Por qué lo necesitamos precisamente en este momento:** No podemos demostrar que un rollback restauró algo si antes no sabemos cuánto stock había. El valor se guarda en `@StockAntes` para compararlo con la lectura final.

**Qué observaremos y cómo lo interpretaremos:** Lectura sin cambios del producto 2.

**Por qué se escribe así:** El stock de referencia permitirá demostrar que el UPDATE provisional se revierte. XACT_ABORT ON hace más estricta la respuesta frente a determinados errores de ejecución.

**Qué cambia / qué veremos al ejecutar el script completo:** El producto 2 no ha cambiado todavía.

**Pregunta para predecir:** ¿Por qué es importante una clave ya existente?

**Respuesta razonada:** Porque el INSERT posterior fallará de forma controlada y reproducible.

**Siguiente decisión:** Iniciamos la transacción que contiene las dos acciones.

### Paso 2 — Abrir la transacción bajo manejo de errores

**Antes (explicación para entender lo que viene):** El escenario empieza con datos conocidos.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
```

**Objetivo de esta consulta o bloque:** Separar el bloque susceptible de fallar.

**Por qué lo necesitamos precisamente en este momento:** El par `TRY` + `BEGIN TRANSACTION` asegura que el descuento y el siguiente INSERT se traten como una operación. Activamos `XACT_ABORT ON` para ilustrar que los errores pueden impedir confirmar la transacción; CATCH deberá ocuparse de cerrar correctamente.

**Qué observaremos y cómo lo interpretaremos:** La transacción queda abierta y preparada para la prueba de error.

**Por qué se escribe así:** La transacción debe cubrir tanto inventario como auditoría; TRY capturará la violación de clave.

**Qué cambia / qué veremos al ejecutar el script completo:** @@TRANCOUNT=1 al entrar.

**Pregunta para predecir:** ¿Qué efecto queremos ante el segundo paso inválido?

**Respuesta razonada:** Cancelar también el primero.

**Siguiente decisión:** Realizaremos el descuento provisional.

### Paso 3 — Descontar una unidad aún sin confirmar

**Antes (explicación para entender lo que viene):** Existe stock disponible del producto 2.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    UPDATE dbo.S10_ProductosLab SET Stock=Stock-1
    WHERE ProductID=2 AND Stock>=1;
    IF @@ROWCOUNT<>1 THROW 51106,N'Sin stock para el experimento.',1;
```

**Objetivo de esta consulta o bloque:** Realizar el primer paso aparentemente exitoso.

**Por qué lo necesitamos precisamente en este momento:** Descontamos una unidad del producto 2 y comprobamos que sí se afectó una fila. Esto crea el estado parcial que queremos detectar: **una modificación válida seguida de una operación inválida**.

**Qué observaremos y cómo lo interpretaremos:** El stock se reduce provisionalmente en una unidad.

**Por qué se escribe así:** El UPDATE cambia provisionalmente el inventario. La condición evita valores negativos y THROW interrumpe si no pudo descontar.

**Qué cambia / qué veremos al ejecutar el script completo:** Stock provisional disminuye en una unidad.

**Pregunta para predecir:** ¿El stock está guardado para siempre ahora?

**Respuesta razonada:** No, depende del resultado de la transacción completa.

**Siguiente decisión:** El siguiente INSERT simulará el fallo.

### Paso 4 — Provocar un duplicado real de clave primaria

**Antes (explicación para entender lo que viene):** La marca DUPLICADO ya existe en S10_EventosLab.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('DUPLICADO',N'Intento indebido de repetir la marca');
    COMMIT TRANSACTION;
```

**Objetivo de esta consulta o bloque:** Provocar un fallo predecible y controlado.

**Por qué lo necesitamos precisamente en este momento:** La preparación creó la clave primaria `DUPLICADO` previamente. Intentar insertar ese mismo código viola una restricción real. El error no viene de un mensaje inventado: lo genera SQL Server al comprobar la clave.

**Qué observaremos y cómo lo interpretaremos:** El INSERT no se acepta; el flujo salta a CATCH.

**Por qué se escribe así:** El INSERT viola la PK de Codigo: COMMIT se omite en el flujo porque el control pasa al CATCH. Esta es una situación controlada, no corrupción de datos.

**Qué cambia / qué veremos al ejecutar el script completo:** Se produce una excepción SQL y el control pasa al CATCH.

**Pregunta para predecir:** ¿Por qué no se ejecuta COMMIT?

**Respuesta razonada:** El error interrumpe el bloque TRY antes de llegar a COMMIT.

**Siguiente decisión:** Necesitamos limpiar la transacción con CATCH.

### Paso 5 — Registrar el error y cancelar con XACT_STATE

**Antes (explicación para entender lo que viene):** Hay un error capturado; la transacción puede estar activa o no confirmable.

**Acción en SSMS 22:** copia o escribe este **paso 5** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
END TRY
BEGIN CATCH
    SELECT ERROR_NUMBER() AS NumeroError,ERROR_MESSAGE() AS MensajeError,
           XACT_STATE() AS EstadoAntesDeRollback;
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
END CATCH;
```

**Objetivo de esta consulta o bloque:** Observar el error y decidir qué hacer.

**Por qué lo necesitamos precisamente en este momento:** `ERROR_NUMBER` y `ERROR_MESSAGE` muestran la causa del fallo; `XACT_STATE()` indica el estado de la transacción. Si no es 0, hacemos un rollback total. El propósito es no dejar aplicado el descuento anterior cuando falló el registro posterior.

**Qué observaremos y cómo lo interpretaremos:** Verás datos del error; después de la reversión no debería quedar transacción activa.

**Por qué se escribe así:** ERROR_NUMBER y ERROR_MESSAGE muestran causa técnica. XACT_STATE=1 indica activa confirmable, -1 no confirmable, 0 ausente. Si aún existe transacción, se revierte completamente.

**Qué cambia / qué veremos al ejecutar el script completo:** El error queda visible en Resultados; no quedan cambios de esa unidad.

**Pregunta para predecir:** ¿Por qué no lanzamos COMMIT desde el CATCH?

**Respuesta razonada:** Porque el caso de negocio exige invalidar toda la operación.

**Siguiente decisión:** Comprobaremos el stock luego de la limpieza.

### Paso 6 — Verificar que no hubo descuento persistente

**Antes (explicación para entender lo que viene):** CATCH ha revertido la transacción abierta.

**Acción en SSMS 22:** copia o escribe este **paso 6** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT @StockAntes AS StockAntes,Stock AS StockDespues,
       Stock-@StockAntes AS Diferencia,@@TRANCOUNT AS Pendientes
FROM dbo.S10_ProductosLab WHERE ProductID=2;
```

**Objetivo de esta consulta o bloque:** Comprobar el stock restaurado.

**Por qué lo necesitamos precisamente en este momento:** El SELECT final compara la cifra capturada antes y la cifra que permanece ahora. La diferencia 0 y el contador 0 prueban que el descuento provisional **no** sobrevivió al error.

**Qué observaremos y cómo lo interpretaremos:** Stock antes=stock después; diferencia 0, pendientes 0.

**Por qué se escribe así:** Si stock antes/después coincide, el fallo posterior no dejó estado parcial. Pendientes debe ser 0.

**Qué cambia / qué veremos al ejecutar el script completo:** Diferencia=0 y Pendientes=0.

**Pregunta para predecir:** ¿El error elimina la fila DUPLICADO original?

**Respuesta razonada:** No, fue confirmada previamente en preparación.

**Siguiente decisión:** En el ejemplo siguiente desharemos solo parte de una transacción.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SET XACT_ABORT ON;
DECLARE @StockAntes INT;
SELECT @StockAntes=Stock FROM dbo.S10_ProductosLab WHERE ProductID=2;
BEGIN TRY
    BEGIN TRANSACTION;
    UPDATE dbo.S10_ProductosLab SET Stock=Stock-1
    WHERE ProductID=2 AND Stock>=1;
    IF @@ROWCOUNT<>1 THROW 51106,N'Sin stock para el experimento.',1;
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('DUPLICADO',N'Intento indebido de repetir la marca');
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    SELECT ERROR_NUMBER() AS NumeroError,ERROR_MESSAGE() AS MensajeError,
           XACT_STATE() AS EstadoAntesDeRollback;
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
END CATCH;
SELECT @StockAntes AS StockAntes,Stock AS StockDespues,
       Stock-@StockAntes AS Diferencia,@@TRANCOUNT AS Pendientes
FROM dbo.S10_ProductosLab WHERE ProductID=2;
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** Se muestra un error de duplicado en el CATCH; stock final igual al stock anterior y Pendientes=0.

**Interpretación que debes poder explicar:** Un error en una sentencia posterior no debe dejar confirmado el UPDATE previo; XACT_STATE orienta la cancelación segura.

**Error habitual:** Creer que todos los errores se resuelven con un COMMIT parcial o que el CATCH basta sin ROLLBACK.

**Cómo corregirlo de forma segura:** Revisar XACT_STATE: si es distinto de 0, hacer ROLLBACK completo; no intentar COMMIT de una transacción no confirmable.

**Pregunta de cierre:** ¿Qué prueba que el stock no quedó descontado?

### T06 — Tarea espejo (resolver sin ver la solución)

**Reto:** T06: intenta vender 9999 unidades del producto 1 con una transacción; usa THROW ante stock insuficiente, CATCH con XACT_STATE y demuestra que no se creó venta ni cambió el stock.

**Archivo a crear/editar:** `estudiante/tareas/T06_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Toma el stock y la cantidad de ventas antes del BEGIN TRANSACTION.

**Evidencia obligatoria:** Stock y conteo de ventas coinciden antes y después; @@TRANCOUNT=0.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.

---

## EJ07 — Conservar lo válido y revertir solo lo opcional con SAVE TRANSACTION

### Tema específico y objetivo de aprendizaje del EJ07

**Tema que trataremos:** Punto de guardado: deshacer solo el tramo opcional.

**Objetivo del ejemplo (qué lograremos al terminar):** Crear un evento obligatorio, registrar un punto de guardado, añadir un evento opcional y revertir únicamente este último, conservando el obligatorio.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** A veces el negocio aprueba una parte del trabajo, pero no una nota complementaria. Cancelar **toda** la transacción descartaría también la parte válida. `SAVE TRANSACTION` marca hasta dónde se desea retroceder si la transacción sigue siendo utilizable.

**¿Qué datos vamos a observar y modificar?** `dbo.S10_EventosLab`: `EJ07_BASE` permanece, mientras que `EJ07_OPCIONAL` se inserta temporalmente pero no queda en la tabla tras el rollback al punto.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. `DELETE` previo de dos marcas del ejercicio | ¿Cómo evitamos que una ejecución anterior nos distorsione la demostración? Se limpia únicamente este caso. |
| 2. `INSERT EJ07_BASE` | ¿Qué parte queremos conservar? |
| 3. `SAVE TRANSACTION` | ¿Cuál es el punto seguro al que podríamos regresar? |
| 4. `INSERT EJ07_OPCIONAL` y SELECT | ¿Qué aparece momentáneamente antes de cancelar lo opcional? |
| 5. `ROLLBACK ... PuntoDespuesBase` + `COMMIT` + SELECT | ¿Qué fila quedó tras la reversión parcial? Solo `EJ07_BASE`. |

**Prueba de que alcanzamos el objetivo:** Antes del rollback parcial se ven las dos marcas; al final solo existe `EJ07_BASE` y `TransaccionesPendientes=0`.

**Idea que no debemos confundir:** Un savepoint no equivale a una transacción independiente. Debemos cerrar la transacción exterior con `COMMIT` o `ROLLBACK`; si está irrecuperable, se requiere revertirla por completo.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** Un registro inicial de atención al cliente sí es obligatorio, pero una nota opcional agregada después no debe guardarse.

**Qué haremos, en orden:** Insertaremos una marca base, crearemos un SAVE TRANSACTION, insertaremos la nota opcional, haremos ROLLBACK al savepoint y COMMIT de la marca base.

**Qué tenemos ANTES:** Las marcas EJ07_BASE y EJ07_OPCIONAL no están presentes luego de preparación.

**Qué debe existir DESPUÉS:** Solo queda EJ07_BASE; EJ07_OPCIONAL no aparece; Pendientes=0.

**Qué significa el resultado:** SAVE TRANSACTION establece un punto dentro de la transacción; ROLLBACK al punto no deshace lo anterior y COMMIT final confirma el resto.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Limpiar solo las marcas de esta demostración

**Antes (explicación para entender lo que viene):** Puede que las marcas EJ07 existan de una ejecución anterior.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SET XACT_ABORT OFF;
DELETE FROM dbo.S10_EventosLab WHERE Codigo IN('EJ07_BASE','EJ07_OPCIONAL');
```

**Objetivo de esta consulta o bloque:** Partir de un caso limpio.

**Por qué lo necesitamos precisamente en este momento:** El DELETE inicial borra **solamente** marcas específicas de EJ07 que pudieron quedar de intentos anteriores. No borra ventas ni eventos de otros ejemplos. Es un paso de preparación, no la operación de reversión parcial que queremos demostrar.

**Qué observaremos y cómo lo interpretaremos:** Las dos marcas de esta demostración quedan ausentes.

**Por qué se escribe así:** El DELETE elimina exclusivamente marcas identificadas con EJ07, permitiendo repetir la actividad. XACT_ABORT OFF evita mezclar el ejemplo con un error que invalide la transacción, ya que el savepoint requiere una transacción utilizable.

**Qué cambia / qué veremos al ejecutar el script completo:** Las dos marcas quedan ausentes antes de la nueva demostración.

**Pregunta para predecir:** ¿Estamos borrando datos originales de Customers?

**Respuesta razonada:** No; solo filas de dbo.S10_EventosLab.

**Siguiente decisión:** Abriremos una transacción y guardaremos el primer evento.

### Paso 2 — Insertar el evento que sí queremos conservar

**Antes (explicación para entender lo que viene):** La tabla está limpia para EJ07.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('EJ07_BASE',N'Evento obligatorio aprobado');
```

**Objetivo de esta consulta o bloque:** Registrar la parte obligatoria.

**Por qué lo necesitamos precisamente en este momento:** La marca BASE representa un hecho ya aprobado. La insertamos antes del punto de guardado porque queremos conservarla si solo se cancela la parte opcional.

**Qué observaremos y cómo lo interpretaremos:** Hay un evento base provisional dentro de la transacción.

**Por qué se escribe así:** BEGIN TRANSACTION abre la unidad; el primer INSERT representa trabajo válido, todavía pendiente de confirmación.

**Qué cambia / qué veremos al ejecutar el script completo:** EJ07_BASE existe provisionalmente dentro de la transacción.

**Pregunta para predecir:** ¿Ya es permanente el evento base?

**Respuesta razonada:** Todavía no; falta COMMIT final.

**Siguiente decisión:** Marcaremos el límite entre trabajo obligatorio y opcional.

### Paso 3 — Crear el punto de restauración parcial

**Antes (explicación para entender lo que viene):** El registro obligatorio ya está dentro de la transacción.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    SAVE TRANSACTION PuntoDespuesBase;
```

**Objetivo de esta consulta o bloque:** Marcar el punto seguro.

**Por qué lo necesitamos precisamente en este momento:** `SAVE TRANSACTION PuntoDespuesBase` fija la posición a la que podremos regresar sin borrar lo que ocurrió antes. No guarda por sí solo nada permanentemente: la transacción exterior sigue abierta.

**Qué observaremos y cómo lo interpretaremos:** Se crea un punto de guardado y no se agrega ninguna fila nueva.

**Por qué se escribe así:** SAVE marca un punto lógico de retorno dentro de la misma transacción; no crea una transacción nueva ni confirma datos.

**Qué cambia / qué veremos al ejecutar el script completo:** El punto de guardado se coloca después de EJ07_BASE.

**Pregunta para predecir:** ¿Qué parte se deshará si volvemos a PuntoDespuesBase?

**Respuesta razonada:** Solamente las modificaciones posteriores al punto.

**Siguiente decisión:** Agregaremos una nota opcional que será descartada.

### Paso 4 — Insertar el evento opcional y verlo temporalmente

**Antes (explicación para entender lo que viene):** Ya tenemos una marca base y un savepoint.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('EJ07_OPCIONAL',N'Nota sin aprobación');
    SELECT Codigo FROM dbo.S10_EventosLab
    WHERE Codigo IN('EJ07_BASE','EJ07_OPCIONAL') ORDER BY Codigo;
```

**Objetivo de esta consulta o bloque:** Añadir una nota que podría ser rechazada.

**Por qué lo necesitamos precisamente en este momento:** Insertamos un evento adicional y consultamos los dos códigos mientras todavía están en la transacción. Mostrar ambos es importante: si no vimos la nota opcional, el rollback parcial posterior sería difícil de entender.

**Qué observaremos y cómo lo interpretaremos:** Durante el bloque aparecen la marca BASE y la OPCIONAL.

**Por qué se escribe así:** Ambos eventos aparecen temporalmente; el SELECT no confirma ni cierra el bloque.

**Qué cambia / qué veremos al ejecutar el script completo:** Dentro de la transacción se ven las dos marcas.

**Pregunta para predecir:** ¿Ambas deben persistir?

**Respuesta razonada:** No; solo la marca base fue aprobada.

**Siguiente decisión:** Volveremos al punto de guardado.

### Paso 5 — Deshacer lo opcional y confirmar el resto

**Antes (explicación para entender lo que viene):** La segunda inserción se realizó después del savepoint.

**Acción en SSMS 22:** copia o escribe este **paso 5** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    ROLLBACK TRANSACTION PuntoDespuesBase;
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
```

**Objetivo de esta consulta o bloque:** Retroceder hasta el punto y cerrar.

**Por qué lo necesitamos precisamente en este momento:** `ROLLBACK TRANSACTION PuntoDespuesBase` deshace solo las operaciones **posteriores** al savepoint, es decir, la nota opcional. El `COMMIT` posterior confirma el evento base que quedó vigente en la transacción.

**Qué observaremos y cómo lo interpretaremos:** Solo el evento base se confirma; la nota desaparece.

**Por qué se escribe así:** ROLLBACK al nombre descarta EJ07_OPCIONAL, no todo el trabajo. COMMIT es necesario para persistir EJ07_BASE y cerrar la transacción. Si ocurre un error real, CATCH revierte todo el bloque.

**Qué cambia / qué veremos al ejecutar el script completo:** Solo la marca base queda confirmada.

**Pregunta para predecir:** ¿@@TRANCOUNT queda en cero después del ROLLBACK al savepoint?

**Respuesta razonada:** No; baja a cero con el COMMIT final.

**Siguiente decisión:** Verificaremos cuáles códigos permanecen.

### Paso 6 — Comprobar la reversión parcial

**Antes (explicación para entender lo que viene):** La transacción terminó correctamente.

**Acción en SSMS 22:** copia o escribe este **paso 6** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
SELECT Codigo,Descripcion FROM dbo.S10_EventosLab
WHERE Codigo IN('EJ07_BASE','EJ07_OPCIONAL') ORDER BY Codigo;
SELECT @@TRANCOUNT AS TransaccionesPendientes;
```

**Objetivo de esta consulta o bloque:** Demostrar el retroceso selectivo.

**Por qué lo necesitamos precisamente en este momento:** Consultamos específicamente ambas marcas, no toda la bitácora; así se ve exactamente cuál sobrevivió. `@@TRANCOUNT=0` comprueba el cierre final.

**Qué observaremos y cómo lo interpretaremos:** Una fila `EJ07_BASE`, ninguna `EJ07_OPCIONAL`, contador 0.

**Por qué se escribe así:** El conjunto final es la prueba de que se conservaron las operaciones anteriores al savepoint y se revirtieron las posteriores.

**Qué cambia / qué veremos al ejecutar el script completo:** Solo EJ07_BASE, Pendientes=0.

**Pregunta para predecir:** ¿Reemplaza SAVE TRANSACTION un mecanismo de control de errores?

**Respuesta razonada:** No; sirve para retroceder parcialmente cuando la transacción es utilizable.

**Siguiente decisión:** El último ejemplo encapsula el caso en un procedimiento.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SET XACT_ABORT OFF;
DELETE FROM dbo.S10_EventosLab WHERE Codigo IN('EJ07_BASE','EJ07_OPCIONAL');
BEGIN TRY
    BEGIN TRANSACTION;
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('EJ07_BASE',N'Evento obligatorio aprobado');
    SAVE TRANSACTION PuntoDespuesBase;
    INSERT INTO dbo.S10_EventosLab(Codigo,Descripcion)
    VALUES('EJ07_OPCIONAL',N'Nota sin aprobación');
    SELECT Codigo FROM dbo.S10_EventosLab
    WHERE Codigo IN('EJ07_BASE','EJ07_OPCIONAL') ORDER BY Codigo;
    ROLLBACK TRANSACTION PuntoDespuesBase;
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
SELECT Codigo,Descripcion FROM dbo.S10_EventosLab
WHERE Codigo IN('EJ07_BASE','EJ07_OPCIONAL') ORDER BY Codigo;
SELECT @@TRANCOUNT AS TransaccionesPendientes;
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** Solo queda EJ07_BASE; EJ07_OPCIONAL no aparece; Pendientes=0.

**Interpretación que debes poder explicar:** SAVE TRANSACTION establece un punto dentro de la transacción; ROLLBACK al punto no deshace lo anterior y COMMIT final confirma el resto.

**Error habitual:** Pensar que SAVE TRANSACTION crea una transacción anidada independiente o que ROLLBACK a un savepoint deja la transacción ya cerrada.

**Cómo corregirlo de forma segura:** Ejecutar COMMIT después de ROLLBACK al punto si XACT_STATE lo permite; con transacción no confirmable (-1) se requiere ROLLBACK total.

**Pregunta de cierre:** ¿Por qué la marca base permanece aunque hubo un ROLLBACK?

### T07 — Tarea espejo (resolver sin ver la solución)

**Reto:** T07: registra T07_BASE, fija savepoint T07_Punto, agrega T07_OPCIONAL, revierte solo lo opcional y confirma lo base.

**Archivo a crear/editar:** `estudiante/tareas/T07_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Usa XACT_ABORT OFF para la demostración con savepoint sin error y revisa @@TRANCOUNT después.

**Evidencia obligatoria:** T07_BASE existe y T07_OPCIONAL no; @@TRANCOUNT=0.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.

---

## EJ08 — Crear un procedimiento que registre una venta completa y auditable

### Tema específico y objetivo de aprendizaje del EJ08

**Tema que trataremos:** Procedimiento almacenado con transacción completa y auditoría.

**Objetivo del ejemplo (qué lograremos al terminar):** Construir `dbo.usp_S10_RegistrarVentaLab` para recibir cliente, producto y cantidad; actualizar stock, crear venta y auditoría; devolver **éxito o rechazo** sin dejar cambios parciales.

**¿Por qué hacemos este ejercicio y no una consulta cualquiera?** En EJ05 escribimos manualmente los pasos de venta. Una aplicación real necesita reutilizar esa lógica para muchas solicitudes. El procedimiento encapsula validaciones, transacción y resultado de negocio, y además crea una marca de auditoría vinculada a la venta exitosa.

**¿Qué datos vamos a observar y modificar?** Afecta `dbo.S10_ProductosLab`, `dbo.S10_VentasLab` y `dbo.S10_AuditoriaLab` en una única confirmación. El caso inválido de 9999 unidades no debe cambiar ninguna de estas tablas.

**Ruta de consultas: qué pregunta responde cada una**

| Consulta o instrucción | Su finalidad dentro del problema |
|---|---|
| 1. `CREATE OR ALTER PROCEDURE` | ¿Qué operación repetible ofrecerá nuestra base? |
| 2. `IF` y existencia del cliente | ¿Qué entradas no son válidas antes de intentar la venta? |
| 3. `UPDATE` protegido por stock | ¿El pedido puede atenderse? |
| 4. `INSERT` de venta y auditoría | ¿Quedaron registrados el pedido y su trazabilidad? |
| 5. `EXEC` de caso válido y SELECT | ¿La rutina informa `Estado=1`, entrega una venta y reduce el stock? |
| 6. `EXEC` de cantidad 9999 y SELECT | ¿La rutina informa rechazo `-1` y conserva stock y número de ventas? |

**Prueba de que alcanzamos el objetivo:** Ejecución válida: estado 1, venta con ID, stock disminuye una unidad y auditoría con ese ID. Ejecución inválida: estado -1, `VentaID=NULL`, stock y cantidad de ventas sin cambios.

**Idea que no debemos confundir:** `CREATE OR ALTER` define el procedimiento; **no** registra una venta por sí mismo. La venta ocurre al ejecutar `EXEC`. El `GO` separa el lote de creación de las pruebas.

**Orden mental de trabajo:** identificar el dato antes → decidir qué operación representa el problema → ejecutarla dentro o fuera de una transacción según corresponda → confirmar o cancelar → comparar el dato al final. Esta última comprobación es obligatoria: de lo contrario, solo sabríamos que SSMS ejecutó comandos, no que el resultado comercial sea correcto.


### Antes de tocar el editor: ¿de qué trata el caso?

**Situación de negocio:** El sistema necesita reutilizar el registro de venta desde diferentes pantallas. No conviene copiar tres INSERT/UPDATE en cada operación.

**Qué haremos, en orden:** Crearemos dbo.usp_S10_RegistrarVentaLab con parámetros, controles, BEGIN TRANSACTION, descuento de stock, registro de venta, auditoría y salida; lo probaremos con éxito y rechazo.

**Qué tenemos ANTES:** Las tablas S10 de cliente, producto, ventas y auditoría están preparadas; el producto 1 cuenta con stock de laboratorio.

**Qué debe existir DESPUÉS:** Prueba válida devuelve Estado=1, VentaID no nulo, stock disminuye y se crea auditoría; prueba de 9999 unidades devuelve -1 y no cambia el stock ni el número de ventas.

**Qué significa el resultado:** Una operación de negocio reutilizable debe confirmar conjuntamente inventario, venta y auditoría; cualquier fallo revierte todo.

**Dónde trabajaremos:** SSMS 22 > **Nueva consulta** > `Northwind_S02_Lab`. Construye los fragmentos en una sola pestaña, uno debajo del anterior y en el orden mostrado. **No ejecutes fragmentos sueltos que dejen abierto TRY, BEGIN TRANSACTION o CREATE PROCEDURE.** Ejecuta con F5 solamente cuando hayas completado el paso final del ejemplo.

### Paso 1 — Seleccionar la base y empezar el lote de creación

**Antes (explicación para entender lo que viene):** Ya practicamos venta y auditoría por separado. Ahora necesitamos reutilizarlos.

**Acción en SSMS 22:** copia o escribe este **paso 1** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S10_RegistrarVentaLab
    @Cliente NCHAR(5),@Producto INT,@Cantidad INT,
    @VentaID INT OUTPUT,@Estado INT OUTPUT
AS
BEGIN
```

**Objetivo de esta consulta o bloque:** Declarar una rutina reusable.

**Por qué lo necesitamos precisamente en este momento:** `USE` selecciona la BD del laboratorio. `GO` asegura que `CREATE OR ALTER PROCEDURE` comience un lote nuevo, como exige SQL Server. Las variables de salida permitirán a quien llame al procedimiento saber si hubo éxito y cuál fue el ID generado.

**Qué observaremos y cómo lo interpretaremos:** Todavía no hay ninguna venta; solo se empieza la definición del procedimiento.

**Por qué se escribe así:** GO antes de CREATE OR ALTER deja el procedimiento como primera instrucción del lote. Los parámetros son entrada, y los OUTPUT reportan venta y estado de la operación. BEGIN abre cuerpo, no transacción. Aún no ejecutar: el cuerpo está incompleto.

**Qué cambia / qué veremos al ejecutar el script completo:** Existe la firma abierta del procedimiento, sin ejecución posible todavía.

**Pregunta para predecir:** ¿@VentaID OUTPUT es igual a RETURN?

**Respuesta razonada:** No: el OUTPUT entrega la identificación del pedido y se consulta luego del EXEC.

**Siguiente decisión:** Configuraremos y validaremos los parámetros antes de escribir datos.

### Paso 2 — Configurar el procedimiento y validar entrada

**Antes (explicación para entender lo que viene):** El cuerpo BEGIN sigue abierto después de la firma.

**Acción en SSMS 22:** copia o escribe este **paso 2** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    SET @VentaID=NULL; SET @Estado=-1;
    BEGIN TRY
        IF @Cantidad IS NULL OR @Cantidad<=0 OR @Cliente IS NULL OR @Producto IS NULL
            THROW 51180,N'Parámetros de venta inválidos.',1;
```

**Objetivo de esta consulta o bloque:** Validar el contrato de entrada.

**Por qué lo necesitamos precisamente en este momento:** La rutina se negará a vender cantidades 0, negativas o nulas, y requiere cliente y producto. `@Estado=-1` y `@VentaID=NULL` son los valores de rechazo iniciales; solo se reemplazarán si todo sale bien.

**Qué observaremos y cómo lo interpretaremos:** La definición incluye un control de parámetros antes de modificar tablas.

**Por qué se escribe así:** NOCOUNT evita mensajes de filas que interfieran con lectura; XACT_ABORT refuerza consistencia. Inicializamos las salidas en fallo por defecto y verificamos cantidad positiva y claves no nulas.

**Qué cambia / qué veremos al ejecutar el script completo:** Sin modificar tablas, el procedimiento ya puede rechazar entradas defectuosas.

**Pregunta para predecir:** ¿Por qué @Estado empieza en -1?

**Respuesta razonada:** Para no informar éxito antes de completar el COMMIT.

**Siguiente decisión:** Abriremos la transacción solo para una entrada aceptable.

### Paso 3 — Abrir unidad y comprobar existencia del cliente

**Antes (explicación para entender lo que viene):** La cantidad es positiva y hay claves suministradas.

**Acción en SSMS 22:** copia o escribe este **paso 3** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
        BEGIN TRANSACTION;
        IF NOT EXISTS(SELECT 1 FROM dbo.S10_ClientesLab WHERE CustomerID=@Cliente)
            THROW 51181,N'Cliente inexistente.',1;
```

**Objetivo de esta consulta o bloque:** Validar el cliente dentro de la transacción.

**Por qué lo necesitamos precisamente en este momento:** Comprobamos que el cliente solicitado esté presente en la copia de Northwind. Si no existe, el `THROW` conduce al CATCH y evita registrar una venta huérfana.

**Qué observaremos y cómo lo interpretaremos:** Si el cliente existe, el flujo puede evaluar el stock.

**Por qué se escribe así:** La transacción cubre cada escritura posterior; la consulta asegura que la venta esté asociada a un cliente de laboratorio existente.

**Qué cambia / qué veremos al ejecutar el script completo:** La transacción está abierta; aún no se descontó stock.

**Pregunta para predecir:** ¿Qué sucede si el cliente no existe?

**Respuesta razonada:** THROW va a CATCH, que revierte la transacción.

**Siguiente decisión:** Comprobaremos stock y lo descontaremos.

### Paso 4 — Descontar inventario bajo condición

**Antes (explicación para entender lo que viene):** El cliente es válido y ya hay transacción.

**Acción en SSMS 22:** copia o escribe este **paso 4** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
        UPDATE dbo.S10_ProductosLab SET Stock=Stock-@Cantidad
        WHERE ProductID=@Producto AND Stock>=@Cantidad;
        IF @@ROWCOUNT<>1 THROW 51182,N'Producto inexistente o stock insuficiente.',1;
```

**Objetivo de esta consulta o bloque:** Proteger el stock disponible.

**Por qué lo necesitamos precisamente en este momento:** El `UPDATE` solo descuenta cuando quedan unidades suficientes. `@@ROWCOUNT` comprueba que se encontró un producto válido y con capacidad para atender el pedido. Esta es la condición decisiva del caso de 9999 unidades.

**Qué observaremos y cómo lo interpretaremos:** Si no hay suficiente stock, se informa rechazo y se revierte.

**Por qué se escribe así:** La condición de stock protege contra ventas sin unidades y @@ROWCOUNT captura exactamente el resultado del UPDATE.

**Qué cambia / qué veremos al ejecutar el script completo:** Stock queda provisionalmente menor si había unidades.

**Pregunta para predecir:** ¿Podemos registrar la venta si no hubo UPDATE?

**Respuesta razonada:** No: THROW evita crear una venta sin inventario.

**Siguiente decisión:** Registraremos la operación comercial y su identificación.

### Paso 5 — Registrar la venta y construir la auditoría

**Antes (explicación para entender lo que viene):** El stock se descontó dentro de la misma transacción.

**Acción en SSMS 22:** copia o escribe este **paso 5** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
        INSERT INTO dbo.S10_VentasLab(CustomerID,ProductID,Cantidad,Precio)
        SELECT @Cliente,@Producto,@Cantidad,Precio
        FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
        IF @@ROWCOUNT<>1 THROW 51183,N'No pudo insertarse la venta.',1;
        SET @VentaID=CONVERT(INT,SCOPE_IDENTITY());
        INSERT INTO dbo.S10_AuditoriaLab(VentaID,Descripcion)
        VALUES(@VentaID,N'Venta confirmada mediante procedimiento S10');
```

**Objetivo de esta consulta o bloque:** Registrar venta y rastro de auditoría.

**Por qué lo necesitamos precisamente en este momento:** La venta recibe un `VentaID` por identidad. Ese valor se usa en la auditoría, uniendo el rastro al pedido real. Si alguna inserción no se completa, no habrá COMMIT y el procedimiento cancelará la operación.

**Qué observaremos y cómo lo interpretaremos:** Ambos registros pasan a formar parte de la misma transacción.

**Por qué se escribe así:** El INSERT...SELECT copia el precio del producto, SCOPE_IDENTITY toma el ID creado y auditoría referencia ese ID. Los tres cambios comparten una transacción.

**Qué cambia / qué veremos al ejecutar el script completo:** Venta y auditoría existen provisionalmente junto al descuento.

**Pregunta para predecir:** ¿Por qué no guardar auditoría después del COMMIT?

**Respuesta razonada:** Porque deseamos que también se revierta si falla la unidad.

**Siguiente decisión:** Falta decidir confirmación o cancelación definitiva.

### Paso 6 — Confirmar la venta y tratar el fallo

**Antes (explicación para entender lo que viene):** Todas las operaciones se completaron sin error.

**Acción en SSMS 22:** copia o escribe este **paso 6** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
        COMMIT TRANSACTION;
        SET @Estado=1;
    END TRY
    BEGIN CATCH
        IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
        SET @Estado=-1; SET @VentaID=NULL;
        SELECT ERROR_NUMBER() AS NumeroError,ERROR_MESSAGE() AS MensajeError;
    END CATCH;
END;
GO
```

**Objetivo de esta consulta o bloque:** Cerrar la operación con resultado explícito.

**Por qué lo necesitamos precisamente en este momento:** Solo después de insertar venta y auditoría llamamos a `COMMIT`. `@Estado=1` informa éxito al programa que invocó la rutina. En caso de error, CATCH revierte, deja `@Estado=-1` y limpia `@VentaID`; por eso no se comunica un pedido inexistente.

**Qué observaremos y cómo lo interpretaremos:** La definición completa queda lista; todavía debe llamarse con EXEC.

**Por qué se escribe así:** COMMIT hace persistentes los cambios; solo después asignamos éxito. CATCH revierte y devuelve estado -1 y mensaje informativo. END cierra cuerpo y GO finaliza lote. Ahora sí es ejecutable.

**Qué cambia / qué veremos al ejecutar el script completo:** El procedimiento fue definido; las tablas siguen sin cambiar hasta EXEC.

**Pregunta para predecir:** ¿El CREATE OR ALTER ejecuta automáticamente una venta?

**Respuesta razonada:** No: solo guarda la definición.

**Siguiente decisión:** Probaremos una operación válida y una rechazada.

### Paso 7 — Ejecutar el caso válido y observar la auditoría

**Antes (explicación para entender lo que viene):** El procedimiento ya existe y recibe parámetros concretos.

**Acción en SSMS 22:** copia o escribe este **paso 7** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
DECLARE @NuevaVenta INT,@EstadoVenta INT;
EXEC dbo.usp_S10_RegistrarVentaLab
    @Cliente='ALFKI',@Producto=1,@Cantidad=1,
    @VentaID=@NuevaVenta OUTPUT,@Estado=@EstadoVenta OUTPUT;
SELECT @EstadoVenta AS Estado,@NuevaVenta AS VentaID,@@TRANCOUNT AS Pendientes;
SELECT Stock FROM dbo.S10_ProductosLab WHERE ProductID=1;
SELECT VentaID,Descripcion FROM dbo.S10_AuditoriaLab WHERE VentaID=@NuevaVenta;
```

**Objetivo de esta consulta o bloque:** Ejecutar un pedido válido.

**Por qué lo necesitamos precisamente en este momento:** `EXEC` es distinto de `CREATE`: ahora sí se atiende una solicitud de compra de 1 unidad. El SELECT del estado prueba la respuesta del procedimiento; las consultas de stock y auditoría prueban el efecto real.

**Qué observaremos y cómo lo interpretaremos:** Estado 1, ID de venta válido, stock -1 y auditoría asociada.

**Por qué se escribe así:** EXEC llama a la rutina; OUTPUT permite leer el ID y Estado en este lote. SELECT de auditoría confirma que el evento se creó junto con la venta.

**Qué cambia / qué veremos al ejecutar el script completo:** Estado=1, VentaID no nulo, stock baja 1 y hay auditoría asociada.

**Pregunta para predecir:** ¿Qué evidencia demuestra que las tres acciones quedaron agrupadas?

**Respuesta razonada:** La venta existe, el stock bajó y la auditoría tiene el mismo VentaID.

**Siguiente decisión:** Finalmente, forzaremos una solicitud no admisible.

### Paso 8 — Probar rechazo sin dejar cambios parciales

**Antes (explicación para entender lo que viene):** La venta válida ya quedó confirmada y se conoce el stock que resta.

**Acción en SSMS 22:** copia o escribe este **paso 8** debajo del anterior, en la misma pestaña. Debes entenderlo antes de pasar al siguiente. No ejecutes todavía si el bloque global sigue incompleto; usa F5 sobre el script ensamblado al final del ejercicio.

```sql
DECLARE @StockPrevio INT,@VentasPrevias INT,@VentaRechazada INT,@EstadoRechazo INT;
SELECT @StockPrevio=Stock FROM dbo.S10_ProductosLab WHERE ProductID=1;
SELECT @VentasPrevias=COUNT(*) FROM dbo.S10_VentasLab;
EXEC dbo.usp_S10_RegistrarVentaLab
    @Cliente='ALFKI',@Producto=1,@Cantidad=9999,
    @VentaID=@VentaRechazada OUTPUT,@Estado=@EstadoRechazo OUTPUT;
SELECT @EstadoRechazo AS EstadoRechazo,@VentaRechazada AS VentaRechazada,
       @StockPrevio AS StockAntes,Stock AS StockDespues,@@TRANCOUNT AS Pendientes
FROM dbo.S10_ProductosLab WHERE ProductID=1;
SELECT @VentasPrevias AS VentasAntes,COUNT(*) AS VentasDespues FROM dbo.S10_VentasLab;
```

**Objetivo de esta consulta o bloque:** Simular rechazo y revisar invariantes.

**Por qué lo necesitamos precisamente en este momento:** La cantidad 9999 fuerza una venta imposible en el laboratorio. Antes de ejecutarla guardamos el stock y el número de ventas; después comparamos ambos valores. El éxito del test es que el procedimiento **rechace** y **no cambie** las tablas.

**Qué observaremos y cómo lo interpretaremos:** Estado -1, VentaID NULL, stock y total de ventas iguales antes/después.

**Por qué se escribe así:** La cantidad 9999 no supera el filtro de stock. El CATCH evita confirmación y el estado -1 identifica el rechazo. Las consultas prueban que inventario y conteo no cambiaron.

**Qué cambia / qué veremos al ejecutar el script completo:** EstadoRechazo=-1, VentaRechazada=NULL, stock y ventas iguales antes/después; Pendientes=0.

**Pregunta para predecir:** ¿Qué se debe hacer antes de repetir muchas ventas válidas?

**Respuesta razonada:** Reiniciar el laboratorio si se necesita volver al stock inicial.

**Siguiente decisión:** Terminamos demostrando confirmación y reversión dentro de una rutina reutilizable.

### Código completo para revisar antes de ejecutar

Este es **exactamente** el código formado al unir los pasos anteriores. Compáralo con tu pestaña, sin volver a copiarlo encima de los fragmentos que ya escribiste. Si optaste por empezar desde cero, reemplaza el contenido de esa pestaña por este script completo.

```sql
USE Northwind_S02_Lab;
GO
CREATE OR ALTER PROCEDURE dbo.usp_S10_RegistrarVentaLab
    @Cliente NCHAR(5),@Producto INT,@Cantidad INT,
    @VentaID INT OUTPUT,@Estado INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    SET @VentaID=NULL; SET @Estado=-1;
    BEGIN TRY
        IF @Cantidad IS NULL OR @Cantidad<=0 OR @Cliente IS NULL OR @Producto IS NULL
            THROW 51180,N'Parámetros de venta inválidos.',1;
        BEGIN TRANSACTION;
        IF NOT EXISTS(SELECT 1 FROM dbo.S10_ClientesLab WHERE CustomerID=@Cliente)
            THROW 51181,N'Cliente inexistente.',1;
        UPDATE dbo.S10_ProductosLab SET Stock=Stock-@Cantidad
        WHERE ProductID=@Producto AND Stock>=@Cantidad;
        IF @@ROWCOUNT<>1 THROW 51182,N'Producto inexistente o stock insuficiente.',1;
        INSERT INTO dbo.S10_VentasLab(CustomerID,ProductID,Cantidad,Precio)
        SELECT @Cliente,@Producto,@Cantidad,Precio
        FROM dbo.S10_ProductosLab WHERE ProductID=@Producto;
        IF @@ROWCOUNT<>1 THROW 51183,N'No pudo insertarse la venta.',1;
        SET @VentaID=CONVERT(INT,SCOPE_IDENTITY());
        INSERT INTO dbo.S10_AuditoriaLab(VentaID,Descripcion)
        VALUES(@VentaID,N'Venta confirmada mediante procedimiento S10');
        COMMIT TRANSACTION;
        SET @Estado=1;
    END TRY
    BEGIN CATCH
        IF XACT_STATE()<>0 ROLLBACK TRANSACTION;
        SET @Estado=-1; SET @VentaID=NULL;
        SELECT ERROR_NUMBER() AS NumeroError,ERROR_MESSAGE() AS MensajeError;
    END CATCH;
END;
GO
DECLARE @NuevaVenta INT,@EstadoVenta INT;
EXEC dbo.usp_S10_RegistrarVentaLab
    @Cliente='ALFKI',@Producto=1,@Cantidad=1,
    @VentaID=@NuevaVenta OUTPUT,@Estado=@EstadoVenta OUTPUT;
SELECT @EstadoVenta AS Estado,@NuevaVenta AS VentaID,@@TRANCOUNT AS Pendientes;
SELECT Stock FROM dbo.S10_ProductosLab WHERE ProductID=1;
SELECT VentaID,Descripcion FROM dbo.S10_AuditoriaLab WHERE VentaID=@NuevaVenta;
DECLARE @StockPrevio INT,@VentasPrevias INT,@VentaRechazada INT,@EstadoRechazo INT;
SELECT @StockPrevio=Stock FROM dbo.S10_ProductosLab WHERE ProductID=1;
SELECT @VentasPrevias=COUNT(*) FROM dbo.S10_VentasLab;
EXEC dbo.usp_S10_RegistrarVentaLab
    @Cliente='ALFKI',@Producto=1,@Cantidad=9999,
    @VentaID=@VentaRechazada OUTPUT,@Estado=@EstadoRechazo OUTPUT;
SELECT @EstadoRechazo AS EstadoRechazo,@VentaRechazada AS VentaRechazada,
       @StockPrevio AS StockAntes,Stock AS StockDespues,@@TRANCOUNT AS Pendientes
FROM dbo.S10_ProductosLab WHERE ProductID=1;
SELECT @VentasPrevias AS VentasAntes,COUNT(*) AS VentasDespues FROM dbo.S10_VentasLab;
```

### Predicción, ejecución e interpretación

**Antes de F5**, contesta: 1) ¿qué dato cambiará temporalmente?, 2) ¿qué terminará persistiendo?, 3) ¿qué valor tendrá `@@TRANCOUNT` al concluir? Después ejecuta **todo el ejemplo** y consulta las pestañas **Resultados** y **Mensajes**.

**Resultado esperado:** Prueba válida devuelve Estado=1, VentaID no nulo, stock disminuye y se crea auditoría; prueba de 9999 unidades devuelve -1 y no cambia el stock ni el número de ventas.

**Interpretación que debes poder explicar:** Una operación de negocio reutilizable debe confirmar conjuntamente inventario, venta y auditoría; cualquier fallo revierte todo.

**Error habitual:** Crear procedimiento con GO entre CREATE y AS/BEGIN, o aceptar cantidades inválidas sin validación.

**Cómo corregirlo de forma segura:** CREATE OR ALTER debe comenzar su propio lote; el cuerpo TRY/CATCH debe cerrar la transacción en todas las rutas; comprobar resultado con dos pruebas.

**Pregunta de cierre:** ¿Qué tres tablas cambian en la ejecución exitosa?

### T08 — Tarea espejo (resolver sin ver la solución)

**Reto:** T08: crea dbo.usp_S10_CambiarPaisLab con parámetros @Cliente, @NuevoPais y @Estado OUTPUT; actualiza una fila y registra auditoría en una sola transacción, prueba éxito e ID inexistente.

**Archivo a crear/editar:** `estudiante/tareas/T08_ParaResolver.sql`. En la FASE 13 puedes hacer una pestaña nueva y resolverlo sin descargar otro material.

**Pista y orden:** Reutiliza TRY/CATCH, valida @NuevoPais, exige @@ROWCOUNT=1 y agrega auditoría solo si el UPDATE funcionó.

**Evidencia obligatoria:** Éxito: país cambia y auditoría crece en una fila. Fallo: ninguna fila cambia y @Estado=-1.

**Criterio de aceptación:** el SQL solo escribe en `dbo.S10_*Lab`; hay un SELECT anterior y posterior cuando el caso lo necesita; el resultado coincide con la predicción; al terminar `@@TRANCOUNT=0`.


## 6. Síntesis y comprobación al cerrar la sesión

Ejecuta en una pestaña limpia, tras confirmar que no tienes transacciones pendientes, este SELECT de solo lectura:

```sql
USE Northwind_S02_Lab;
GO
SET IMPLICIT_TRANSACTIONS OFF;
SELECT @@TRANCOUNT AS TransaccionesPendientes;
SELECT CustomerID,ContactName,Country FROM dbo.S10_ClientesLab ORDER BY CustomerID;
SELECT ProductID,Stock FROM dbo.S10_ProductosLab ORDER BY ProductID;
SELECT TOP(10) VentaID,CustomerID,ProductID,Cantidad FROM dbo.S10_VentasLab ORDER BY VentaID DESC;
SELECT Codigo FROM dbo.S10_EventosLab ORDER BY Codigo;
```

**Evidencia individual:** captura del resultado de una confirmación (EJ03 o EJ05), una reversión total (EJ04 o EJ06), una reversión parcial (EJ07) y explicación escrita de por qué los datos quedaron así. El valor final de `@@TRANCOUNT` debe ser cero.

**Si los saldos o conteos no coinciden:** recuerda que las prácticas de venta acumulan registros y disminuyen stock. Es normal que ejecutar EJ05 varias veces produzca nuevas ventas. Solo el script de preparación restablece el estado inicial, y al hacerlo elimina evidencias previas de S10.

**Criterio de dominio final:** no basta con ejecutar F5; cada estudiante debe mostrar qué estaba antes, qué sentencia produjo el cambio, dónde se decidió COMMIT/ROLLBACK y qué dato persistió.


## Autoevaluación razonada: ¿comprendí el propósito de las consultas?

Al terminar, responde con tus propias palabras, no solamente con nombres de comandos: ¿por qué EJ01 necesita un SELECT anterior y otro posterior? ¿Qué evidencia diferencia el país temporal del país confirmado en EJ02? ¿Por qué EJ03 consulta `@@ROWCOUNT` inmediatamente después del UPDATE? ¿Por qué EJ05 necesita dos lecturas finales de tablas distintas? ¿Por qué que aparezca un error en EJ06 puede ser una prueba exitosa? ¿Qué parte sobrevive en EJ07 y por qué? ¿Cómo diferencia EJ08 un procedimiento definido de uno ejecutado?

**Criterio de aprobación de la práctica:** puedes señalar la sentencia que cambió el dato, la sentencia que decidió conservarlo o revertirlo, y una consulta que **demuestre** el estado final. Si en cualquier ejemplo el contador de transacciones termina distinto de cero, revisa el estado de tu conexión antes de avanzar.
