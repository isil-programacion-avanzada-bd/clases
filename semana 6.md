---
course_id: PABD
session_id: S06
source_origin: PPT
status: draft
---
# Guía del Estudiante - Vistas e índices en SQL Server

**Curso:** Programación Avanzada de Base de Datos  
**Sesión:** 06  
**Base de datos:** Northwind  
**Herramienta principal:** SQL Server Management Studio (SSMS)

## 1. Propósito de la sesión
En esta clase-laboratorio aprenderás a usar una vista como tabla virtual, administrar su ciclo de vida, comprobar qué ocurre cuando una vista actualizable modifica datos de la tabla origen, inspeccionar índices y crear índices de laboratorio sin alterar los objetos canónicos de Northwind. También observarás un plan de ejecución real para conectar el diseño de índices con el trabajo que realiza SQL Server.

La evidencia final será concreta: vistas creadas y consultadas, cambios verificados y revertidos, índices identificados y creados en copias seguras, un error UNIQUE reproducido de forma controlada y un plan de ejecución observado en SSMS.

## 2. Resultado observable
### A. Lectura sugerida del docente
Al terminar esta sesión no bastará con recordar que una vista es una consulta almacenada o que un índice puede acelerar búsquedas. Debes poder construir una vista, consultarla, modificarla, eliminarla y explicar dónde residen realmente los datos. Después deberás revisar los índices existentes antes de crear nuevos, distinguir un índice clustered de uno nonclustered en un caso de laboratorio y justificar por qué no conviene crear índices sin observar primero la estructura disponible. Finalmente, usarás el plan de ejecución actual para reconocer que SQL Server decide una estrategia de acceso en función de sus objetos, estadísticas y condiciones de consulta. El objetivo es que puedas pasar de sintaxis aislada a una lectura técnica del estado de la base de datos.

### B. Desempeños observables
- Crear y consultar vistas.
- Modificar y eliminar vistas.
- Verificar una actualización a través de una vista simple.
- Interpretar la salida de `sp_helpindex`.
- Crear índices clustered, nonclustered y unique en un entorno de laboratorio.
- Reproducir un error controlado de unicidad.
- Activar y observar un plan de ejecución real.
- Explicar el trade-off entre lectura y mantenimiento de índices.

### C. Criterio de dominio
Dominas la sesión si puedes explicar por qué existe cada objeto, predecir el efecto de cada sentencia, validar el resultado con evidencia y corregir los errores controlados sin limitarte a copiar el script.

## 3. Antes de iniciar
### Debe saber
SELECT, WHERE, JOIN, alias, subconsultas, INSERT/UPDATE básicos y uso de Northwind, desarrollados previamente.

### Debe tener disponible
- SQL Server funcionando.
- SQL Server Management Studio o cliente institucional equivalente.
- Base de datos `Northwind` accesible.
- Permisos para crear vistas, tablas de laboratorio e índices dentro del entorno de práctica.

### No se asumirá todavía
Tuning avanzado, estadísticas manuales, índices columnstore, particionamiento ni estrategias avanzadas de administración.

## 4. Preparación del entorno
1. Abre SSMS y conéctate a tu instancia de laboratorio.
2. En Object Explorer confirma que `Northwind` existe.
3. Abre una nueva ventana de consulta.
4. Ejecuta:
```sql
USE Northwind;
GO
SELECT DB_NAME() AS CurrentDatabase;
SELECT TOP (3) EmployeeID, FirstName, LastName FROM dbo.Employees;
GO
```
5. Debes observar `Northwind` como base actual y al menos algunas filas de Employees. Si falla, no continúes hasta corregir conexión, nombre de base o permisos.

### Nota técnica importante de esta edición
La PPT contiene varios fragmentos útiles, pero algunos requieren corrección operativa para ser ejecutables de forma segura: el `ORDER BY` de una vista se traslada a la consulta que consume la vista; antes de crear un clustered index se inspecciona si ya existe uno; y la validación del índice UNIQUE debe provocar el error en la clave UNIQUE, no en la PK. La práctica conserva los conceptos de la sesión y corrige solo la ejecución.

## 5. Secuencia de clase-laboratorio


# EJ01 - Crear y consultar una vista simple

## Qué vamos a construir / resolver
Evitar repetir una consulta de empleados y disponer de una tabla virtual reutilizable.

## Qué aprenderás aquí
**Habilidad:** CREATE VIEW + SELECT sobre una vista.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
CREATE VIEW + SELECT sobre una vista.

### Por qué lo necesitamos
Evitar repetir una consulta de empleados y disponer de una tabla virtual reutilizable.

## Paso 2 - Predicción antes de ejecutar
**Predice:** La vista devolverá una fila por empleado con nombre, apellido y fecha de nacimiento; no se copiarán físicamente esas filas a una tabla nueva.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `USE Northwind`  
**Por qué existe:** Selecciona la base donde se crearán y consultarán los objetos; evita crear la vista en otra base por error.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `CREATE OR ALTER VIEW dbo.HappybirthdayEmployeeView`  
**Por qué existe:** Materializa la definición lógica de la vista. Se usa CREATE OR ALTER para que el laboratorio pueda repetirse sin fallar si la vista ya existe.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `SELECT FirstName, LastName, BirthDate FROM dbo.Employees`  
**Por qué existe:** Define las columnas visibles desde la vista; los datos siguen residiendo en Employees.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 4
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `SELECT ... FROM dbo.HappybirthdayEmployeeView`  
**Por qué existe:** Consulta la vista como si fuera una tabla y permite comprobar que la definición quedó disponible.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

CREATE OR ALTER VIEW dbo.HappybirthdayEmployeeView
AS
SELECT FirstName, LastName, BirthDate
FROM dbo.Employees;
GO

SELECT FirstName, LastName, BirthDate
FROM dbo.HappybirthdayEmployeeView;
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa CREATE VIEW + SELECT sobre una vista | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** Comprobar que la vista aparece bajo Views/Vistas y que el SELECT devuelve empleados sin error.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Agregar un filtro en la consulta de lectura, no en la definición, por ejemplo WHERE BirthDate IS NOT NULL, para comprobar que una vista puede volver a filtrarse.

## Paso 9 - Error o caso límite controlado
**Error probable:** Crear la vista en otra base por no ejecutar USE Northwind.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T01 - Tarea espejo de EJ01
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Crear `dbo.EmployeeContactView` con `EmployeeID`, `FirstName`, `LastName`, `City` y luego consultar solo empleados cuya ciudad no sea NULL.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ01, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ01 y resolver T01, continúa con el siguiente ID.


# EJ02 - Modificar la definición de una vista

## Qué vamos a construir / resolver
Una vista puede necesitar exponer nuevos campos sin obligar al consumidor a reescribir toda la consulta base.

## Qué aprenderás aquí
**Habilidad:** ALTER VIEW / evolución de una vista.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
ALTER VIEW / evolución de una vista.

### Por qué lo necesitamos
Una vista puede necesitar exponer nuevos campos sin obligar al consumidor a reescribir toda la consulta base.

## Paso 2 - Predicción antes de ejecutar
**Predice:** La vista conservará su nombre pero ahora mostrará cinco columnas.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `ALTER VIEW`  
**Por qué existe:** Cambia la definición del objeto existente sin eliminarlo previamente.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `Title, City`  
**Por qué existe:** Amplían la proyección de la vista con cargo y ciudad, tal como plantea la sesión.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `SELECT sobre la vista`  
**Por qué existe:** Valida que los nuevos campos quedaron expuestos por el objeto.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

ALTER VIEW dbo.HappybirthdayEmployeeView
AS
SELECT FirstName, LastName, BirthDate, Title, City
FROM dbo.Employees;
GO

SELECT FirstName, LastName, BirthDate, Title, City
FROM dbo.HappybirthdayEmployeeView;
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa ALTER VIEW / evolución de una vista | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** Ejecutar el SELECT y confirmar que aparecen Title y City.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Retirar temporalmente BirthDate de la definición y observar cómo cambia el contrato de columnas; después restaurarla.

## Paso 9 - Error o caso límite controlado
**Error probable:** Ejecutar ALTER VIEW antes de haber creado la vista; el laboratorio evita esto ejecutando EJ01 primero.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T02 - Tarea espejo de EJ02
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Modificar `dbo.EmployeeContactView` para incluir `Title` y `Country`, y verificar el resultado.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ02, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ02 y resolver T02, continúa con el siguiente ID.


# EJ03 - Actualizar datos a través de una vista y verificar la tabla origen

## Qué vamos a construir / resolver
La sesión debe demostrar que, cuando una vista es actualizable, el cambio afecta la tabla base y no una copia independiente.

## Qué aprenderás aquí
**Habilidad:** UPDATE mediante vista + verificación de estado.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
UPDATE mediante vista + verificación de estado.

### Por qué lo necesitamos
La sesión debe demostrar que, cuando una vista es actualizable, el cambio afecta la tabla base y no una copia independiente.

## Paso 2 - Predicción antes de ejecutar
**Predice:** Durante la transacción, la ciudad de Nancy aparecerá como Lima tanto en la vista como en Employees; después del ROLLBACK volverá al valor original.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `BEGIN TRANSACTION`  
**Por qué existe:** Aísla la demostración para poder revertirla y no dejar alterados los datos de práctica.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `SELECT previo`  
**Por qué existe:** Registra el estado antes del cambio.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `UPDATE dbo.HappybirthdayEmployeeView`  
**Por qué existe:** Escribe mediante la vista simple; SQL Server dirige el cambio a la tabla base cuando la operación es válida.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 4
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `SELECT sobre vista y Employees`  
**Por qué existe:** Permite comprobar que ambos reflejan la misma actualización mientras la transacción sigue abierta.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 5
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `ROLLBACK TRANSACTION`  
**Por qué existe:** Devuelve Northwind al estado inicial y convierte el ejemplo en una práctica segura y repetible.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

BEGIN TRANSACTION;

SELECT EmployeeID, FirstName, LastName, City
FROM dbo.Employees
WHERE FirstName = 'Nancy' AND LastName = 'Davolio';

UPDATE dbo.HappybirthdayEmployeeView
SET City = 'Lima'
WHERE FirstName = 'Nancy' AND LastName = 'Davolio';

SELECT FirstName, LastName, City
FROM dbo.HappybirthdayEmployeeView
WHERE FirstName = 'Nancy' AND LastName = 'Davolio';

SELECT EmployeeID, FirstName, LastName, City
FROM dbo.Employees
WHERE FirstName = 'Nancy' AND LastName = 'Davolio';

ROLLBACK TRANSACTION;
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa UPDATE mediante vista + verificación de estado | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** Comparar los SELECT antes/durante/después y verificar que el estado final se restaura.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Cambiar el valor temporal a `Cusco`, ejecutar la misma secuencia y volver a revertir.

## Paso 9 - Error o caso límite controlado
**Error probable:** Omitir el ROLLBACK y dejar modificado el dato de laboratorio.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T03 - Tarea espejo de EJ03
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Dentro de una transacción, actualizar temporalmente `City` a `Arequipa` para Andrew Fuller mediante `dbo.HappybirthdayEmployeeView`, verificar en la vista y en Employees, y hacer ROLLBACK.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ03, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ03 y resolver T03, continúa con el siguiente ID.


# EJ04 - Eliminar una vista de forma controlada

## Qué vamos a construir / resolver
Un objeto que ya no se necesita debe retirarse explícitamente; la práctica también enseña a comprobar su existencia antes y después.

## Qué aprenderás aquí
**Habilidad:** DROP VIEW + ciclo de vida.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
DROP VIEW + ciclo de vida.

### Por qué lo necesitamos
Un objeto que ya no se necesita debe retirarse explícitamente; la práctica también enseña a comprobar su existencia antes y después.

## Paso 2 - Predicción antes de ejecutar
**Predice:** ViewIdBefore tendrá un valor si T01/T02 se ejecutaron; ViewIdAfter será NULL.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `OBJECT_ID(...,'V')`  
**Por qué existe:** Busca el identificador del objeto solo si es una vista; un valor no nulo antes demuestra que existe.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `DROP VIEW IF EXISTS`  
**Por qué existe:** Elimina el objeto si existe y permite repetir el script sin error cuando ya fue eliminado.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `OBJECT_ID después`  
**Por qué existe:** Debe devolver NULL, evidencia observable de que la vista ya no existe.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

SELECT OBJECT_ID('dbo.EmployeeContactView', 'V') AS ViewIdBefore;
GO

DROP VIEW IF EXISTS dbo.EmployeeContactView;
GO

SELECT OBJECT_ID('dbo.EmployeeContactView', 'V') AS ViewIdAfter;
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa DROP VIEW + ciclo de vida | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** Confirmar NULL después del DROP y ausencia de la vista en el explorador de objetos tras refrescar.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Recrear la vista con la solución de T01 y comprobar que OBJECT_ID vuelve a ser no nulo.

## Paso 9 - Error o caso límite controlado
**Error probable:** Intentar consultar la vista después de eliminarla y confundir el error de objeto inexistente con un error de SELECT.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T04 - Tarea espejo de EJ04
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Eliminar `dbo.EmployeeContactView` con `DROP VIEW IF EXISTS` y demostrar con `OBJECT_ID` que ya no existe.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ04, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ04 y resolver T04, continúa con el siguiente ID.


# EJ05 - Construir una vista compleja de clientes, pedidos y productos

## Qué vamos a construir / resolver
La fuente propone encapsular una consulta extensa para reutilizarla. Se corrigen únicamente errores operativos de sintaxis y ordenamiento para que sea válida en SQL Server.

## Qué aprenderás aquí
**Habilidad:** VIEW con JOIN + subconsulta + UNION.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
VIEW con JOIN + subconsulta + UNION.

### Por qué lo necesitamos
La fuente propone encapsular una consulta extensa para reutilizarla. Se corrigen únicamente errores operativos de sintaxis y ordenamiento para que sea válida en SQL Server.

## Paso 2 - Predicción antes de ejecutar
**Predice:** La consulta final mostrará filas con pedidos y, para clientes sin pedidos, columnas de pedido/producto en NULL.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `Primera SELECT con JOIN`  
**Por qué existe:** Reúne clientes que sí tienen pedidos y sus productos asociados.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `od.Quantity * od.UnitPrice AS TotalPrice`  
**Por qué existe:** Calcula un valor derivado por detalle; se elimina el paréntesis sobrante de la fuente.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `Subconsulta COUNT(*) correlacionada`  
**Por qué existe:** Cuenta pedidos del cliente actual utilizando el alias c.CustomerID como referencia.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 4
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `UNION + segunda SELECT`  
**Por qué existe:** Añade clientes sin órdenes con columnas nulas para los datos que no existen.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 5
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `ORDER BY fuera de CREATE VIEW`  
**Por qué existe:** En SQL Server, el ORDER BY no garantiza orden dentro de una vista y no se admite en la definición salvo casos específicos; por eso se aplica al SELECT que consume la vista.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

CREATE OR ALTER VIEW dbo.CustomerOrderSummaryView
AS
SELECT
    c.CompanyName,
    c.CustomerID,
    o.OrderID,
    o.OrderDate,
    p.ProductName,
    od.Quantity,
    od.UnitPrice,
    od.Quantity * od.UnitPrice AS TotalPrice,
    (SELECT COUNT(*)
     FROM dbo.Orders AS ox
     WHERE ox.CustomerID = c.CustomerID) AS TotalOrders
FROM dbo.Customers AS c
INNER JOIN dbo.Orders AS o ON c.CustomerID = o.CustomerID
INNER JOIN dbo.[Order Details] AS od ON o.OrderID = od.OrderID
INNER JOIN dbo.Products AS p ON od.ProductID = p.ProductID
WHERE YEAR(o.OrderDate) BETWEEN 1996 AND 2000

UNION

SELECT
    c.CompanyName, c.CustomerID,
    NULL, NULL, NULL, NULL, NULL, NULL, 0
FROM dbo.Customers AS c
WHERE c.CustomerID NOT IN (SELECT DISTINCT CustomerID FROM dbo.Orders);
GO

SELECT TOP (30) *
FROM dbo.CustomerOrderSummaryView
ORDER BY CompanyName, OrderDate;
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa VIEW con JOIN + subconsulta + UNION | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** Comprobar que la vista se crea sin error y que el SELECT ordenado devuelve ambos tipos de filas cuando existan.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Cambiar TOP (30) por TOP (10) en la consulta de lectura sin alterar la definición de la vista.

## Paso 9 - Error o caso límite controlado
**Error probable:** Copiar literalmente el ORDER BY dentro de la definición de la vista o conservar el paréntesis extra de TotalPrice.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T05 - Tarea espejo de EJ05
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Crear `dbo.CustomerOrderCountView` con `CustomerID`, `CompanyName` y una subconsulta que cuente pedidos por cliente; luego consultar ordenando de mayor a menor por TotalOrders.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ05, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ05 y resolver T05, continúa con el siguiente ID.


# EJ06 - Inspeccionar los índices existentes

## Qué vamos a construir / resolver
Antes de crear un índice hay que saber qué existe. La fuente usa sp_helpindex y este paso evita intentar crear un segundo índice clustered sobre una tabla que ya tiene uno.

## Qué aprenderás aquí
**Habilidad:** sp_helpindex + lectura del estado actual.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
sp_helpindex + lectura del estado actual.

### Por qué lo necesitamos
Antes de crear un índice hay que saber qué existe. La fuente usa sp_helpindex y este paso evita intentar crear un segundo índice clustered sobre una tabla que ya tiene uno.

## Paso 2 - Predicción antes de ejecutar
**Predice:** Las tablas de Northwind normalmente mostrarán índices asociados a claves primarias y posiblemente otros índices auxiliares.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `sys.sp_helpindex`  
**Por qué existe:** Devuelve nombre, descripción y columnas clave de índices de una tabla.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `Employees / Orders / Products`  
**Por qué existe:** Se revisan tablas usadas por la sesión para identificar PK, clustered y nonclustered existentes.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `Lectura previa`  
**Por qué existe:** La existencia de un clustered index condiciona la creación de otro, porque una tabla solo puede tener uno.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

EXEC sys.sp_helpindex N'dbo.Employees';
EXEC sys.sp_helpindex N'dbo.Orders';
EXEC sys.sp_helpindex N'dbo.Products';
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa sp_helpindex + lectura del estado actual | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** Registrar para cada tabla al menos `index_name`, `index_description` e `index_keys`.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Ejecutar sp_helpindex sobre Customers y comparar sus índices con los de Orders.

## Paso 9 - Error o caso límite controlado
**Error probable:** Crear índices sin inspección previa y recibir errores por nombres duplicados o por un clustered index ya existente.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T06 - Tarea espejo de EJ06
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Ejecutar `sp_helpindex` sobre `Customers` y `Order Details`; clasificar cada índice visible como clustered o nonclustered según la descripción.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ06, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ06 y resolver T06, continúa con el siguiente ID.


# EJ07 - Crear índices físicos en copias seguras de laboratorio

## Qué vamos a construir / resolver
La fuente pide crear índices clustered y nonclustered. Para que el ejercicio sea repetible sin alterar índices canónicos de Northwind, se trabaja sobre copias de laboratorio de las tablas.

## Qué aprenderás aquí
**Habilidad:** CLUSTERED + NONCLUSTERED + trade-off.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
CLUSTERED + NONCLUSTERED + trade-off.

### Por qué lo necesitamos
La fuente pide crear índices clustered y nonclustered. Para que el ejercicio sea repetible sin alterar índices canónicos de Northwind, se trabaja sobre copias de laboratorio de las tablas.

## Paso 2 - Predicción antes de ejecutar
**Predice:** Orders_IndexLab tendrá un clustered index por OrderDate; OrderDetails_IndexLab tendrá un nonclustered index por OrderID, ProductID.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `SELECT ... INTO`  
**Por qué existe:** Crea copias de laboratorio con datos reales de Northwind sin modificar las tablas originales.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `CREATE CLUSTERED INDEX ... OrderDate`  
**Por qué existe:** Organiza la estructura de almacenamiento de la copia según la clave del índice; solo puede existir uno por tabla.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `CREATE NONCLUSTERED INDEX ... (OrderID, ProductID)`  
**Por qué existe:** Crea una estructura separada de acceso para esas columnas sin reorganizar la tabla base como clustered.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 4
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `sp_helpindex final`  
**Por qué existe:** Demuestra el tipo y las columnas de cada índice creado.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

DROP TABLE IF EXISTS dbo.Orders_IndexLab;
SELECT TOP (300) OrderID, CustomerID, OrderDate, ShipCountry
INTO dbo.Orders_IndexLab
FROM dbo.Orders;
GO

CREATE CLUSTERED INDEX IX_Orders_IndexLab_OrderDate
ON dbo.Orders_IndexLab (OrderDate);
GO

DROP TABLE IF EXISTS dbo.OrderDetails_IndexLab;
SELECT TOP (500) OrderID, ProductID, UnitPrice, Quantity, Discount
INTO dbo.OrderDetails_IndexLab
FROM dbo.[Order Details];
GO

CREATE NONCLUSTERED INDEX IX_OrderDetails_IndexLab_Order_Product
ON dbo.OrderDetails_IndexLab (OrderID, ProductID);
GO

EXEC sys.sp_helpindex N'dbo.Orders_IndexLab';
EXEC sys.sp_helpindex N'dbo.OrderDetails_IndexLab';
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa CLUSTERED + NONCLUSTERED + trade-off | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** La salida de sp_helpindex debe describir los índices con el tipo esperado.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Agregar un nonclustered index a Orders_IndexLab sobre CustomerID y volver a inspeccionar la tabla.

## Paso 9 - Error o caso límite controlado
**Error probable:** Ejecutar CREATE CLUSTERED INDEX directamente sobre Orders sin revisar si ya existe un clustered index.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T07 - Tarea espejo de EJ07
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Sobre `Products_IndexLab` (copia de ProductID, ProductName, CategoryID), crear un índice nonclustered en `ProductName` y validar con sp_helpindex.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ07, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ07 y resolver T07, continúa con el siguiente ID.


# EJ08 - Validar un índice UNIQUE y observar un plan de ejecución

## Qué vamos a construir / resolver
La sesión combina índices únicos y planes de ejecución. El ejemplo corrige la validación de la fuente para provocar realmente una violación del índice UNIQUE y no solo de la clave primaria.

## Qué aprenderás aquí
**Habilidad:** UNIQUE + error controlado + Actual Execution Plan.

## Archivos, recursos o artefactos involucrados
Base `Northwind`, ventana de consulta de SSMS y objetos `dbo` indicados en el script.

## Punto de partida
Debes haber completado la preparación del entorno. Si el ejemplo depende de un objeto anterior, respeta el orden de EJ01 a EJ08.

## Paso 1 - Comprender la necesidad
### Problema actual
Una sentencia aislada resuelve una acción, pero todavía necesitamos un objeto/estructura que pueda reutilizarse, modificarse o validarse de acuerdo con la habilidad de este ejemplo.

### Qué necesitamos crear o modificar
UNIQUE + error controlado + Actual Execution Plan.

### Por qué lo necesitamos
La sesión combina índices únicos y planes de ejecución. El ejemplo corrige la validación de la fuente para provocar realmente una violación del índice UNIQUE y no solo de la clave primaria.

## Paso 2 - Predicción antes de ejecutar
**Predice:** El INSERT duplicado fallará por la combinación UNIQUE y la consulta de Employees mostrará un plan real al tener activada la opción correspondiente.

Escribe tu predicción en una frase antes de continuar.

## Paso 3 - Construcción por microbloques


### Microbloque 1
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `CREATE UNIQUE INDEX`  
**Por qué existe:** Impide repetir la combinación de columnas que forman la clave del índice.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 2
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `CustomerID nuevo`  
**Por qué existe:** Evita que el error sea causado por una PK duplicada; así la evidencia corresponde al índice UNIQUE que se quiere validar.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 3
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `TRY/CATCH`  
**Por qué existe:** Mantiene el error controlado y permite registrar el mensaje sin interrumpir el resto del laboratorio.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 4
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `Ctrl+M / Actual Execution Plan`  
**Por qué existe:** Hace que SSMS muestre el plan real de la consulta junto con Results y Messages.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


### Microbloque 5
**Antes:** identifica el estado actual del objeto o de los datos.  
**Elemento:** `ORDER BY HireDate`  
**Por qué existe:** La fuente utiliza esta consulta para observar que el motor puede incorporar un operador Sort u otra estrategia según los índices y estadísticas disponibles.  
**Estado después:** al ejecutarse correctamente, este elemento deja preparada la siguiente parte del ejemplo.


## Paso 4 - Artefacto completo del ejemplo
```sql
USE Northwind;
GO

DROP TABLE IF EXISTS dbo.Customers_UniqueLab;
SELECT CustomerID, CompanyName, Country
INTO dbo.Customers_UniqueLab
FROM dbo.Customers;
GO

CREATE UNIQUE INDEX IX_Customers_UniqueLab_Country_Company
ON dbo.Customers_UniqueLab (Country, CompanyName);
GO

SELECT TOP (10) CustomerID, CompanyName, Country
FROM dbo.Customers_UniqueLab
ORDER BY Country, CompanyName;
GO

-- ERROR CONTROLADO: usar un CustomerID nuevo pero repetir Country + CompanyName.
DECLARE @CompanyName nvarchar(40), @Country nvarchar(15);
SELECT TOP (1) @CompanyName = CompanyName, @Country = Country
FROM dbo.Customers_UniqueLab
WHERE CompanyName IS NOT NULL AND Country IS NOT NULL;

BEGIN TRY
    INSERT INTO dbo.Customers_UniqueLab (CustomerID, CompanyName, Country)
    VALUES ('ZZZZZ', @CompanyName, @Country);
END TRY
BEGIN CATCH
    SELECT ERROR_NUMBER() AS ErrorNumber, ERROR_MESSAGE() AS ErrorMessage;
END CATCH;
GO

-- Para el plan de ejecución: activar Include Actual Execution Plan / Ctrl+M en SSMS
SELECT *
FROM dbo.Employees
ORDER BY HireDate;
GO
```

## Paso 5 - Auditoría antes de ejecutar
| Elemento | Qué hace | Por qué existe | Efecto esperado |
|---|---|---|---|
| Contexto de base | Sitúa la ejecución en Northwind | Evita operar en otra base | Objetos en la base correcta |
| Objeto principal | Implementa UNIQUE + error controlado + Actual Execution Plan | Cumple el objetivo del ejemplo | Cambio visible en metadatos/resultados |
| Validación | Consulta o inspecciona el estado | Evita asumir que funcionó | Evidencia observable |

## Paso 6 - Ejecutar y validar
**Cómo validarlo:** Confirmar el mensaje de índice UNIQUE y revisar los operadores del plan sin asumir un costo fijo idéntico en todos los entornos.

## Paso 7 - Interpretar el resultado
No te quedes con “ejecutó sin error”. Explica qué cambió en la base, qué permaneció igual y qué objeto produjo el cambio.

## Paso 8 - Modificación controlada
Consultar Customers_UniqueLab filtrando por Country y observar qué operador utiliza el plan en tu entorno.

## Paso 9 - Error o caso límite controlado
**Error probable:** Duplicar el mismo CustomerID de Northwind y concluir que se validó el índice UNIQUE cuando el error provino de la clave primaria.

Explica qué evidencia te permitiría distinguir este error de un error de sintaxis.

## Paso 10 - Corrección
Revisa el contexto, el objeto indicado, el orden de ejecución y vuelve a ejecutar solo cuando puedas explicar la causa.

# T08 - Tarea espejo de EJ08
**Objetivo:** aplicar la misma habilidad en una variación que requiere el mismo razonamiento.

Crear `EmployeeName_UniqueLab` con EmployeeID, FirstName y LastName; crear un índice UNIQUE sobre (LastName, FirstName), intentar insertar un EmployeeID nuevo repitiendo un nombre existente dentro de TRY/CATCH y registrar el error.

### Antes de ejecutar, predice
1. ¿Qué objeto o dato cambiará?
2. ¿Qué debe permanecer igual?
3. ¿Qué resultado observarás?
4. ¿Cómo sabrás que tu solución es correcta?

### Evidencia que debes entregar
- Script ejecutado.
- Resultado o mensaje relevante.
- Explicación de 3 a 5 líneas sobre lo observado.

### Pista
Reutiliza el patrón de EJ08, pero no copies nombres u objetos sin revisar qué cambia en esta tarea.

## Transición al siguiente ejemplo
Cuando puedas explicar el resultado de EJ08 y resolver T08, continúa con el siguiente ID.


## 6. Checkpoint conceptual: índice de texto completo
La sesión también menciona índices de texto completo. Este contenido se mantiene como concepto de la unidad, pero no se convierte aquí en práctica ejecutable porque la sintaxis resumida de la fuente no contiene todos los requisitos actuales y el esquema Northwind mostrado no evidencia una columna `Description` en `Products`. En SQL Server, un full-text index requiere una clave única válida mediante `KEY INDEX` y que la característica correspondiente esté disponible. Por ello, esta sesión no inventa una columna ni una configuración ausente.

## 7. Cierre de sesión
Durante esta práctica construiste y consultaste vistas, modificaste su definición, comprobaste que una actualización hecha a través de una vista simple puede reflejarse en la tabla origen y eliminaste una vista de manera controlada. Después pasaste al segundo eje de la sesión: antes de crear índices inspeccionaste los existentes, lo que te permitió entender por qué un clustered index no debe agregarse sin revisar la estructura actual. Para practicar de forma segura, creaste copias de laboratorio y sobre ellas materializaste índices clustered y nonclustered. Finalmente validaste un índice UNIQUE con un error realmente asociado a la combinación única y activaste el plan de ejecución actual para observar cómo SQL Server resuelve una consulta.

El punto más importante no es memorizar sentencias: debes poder relacionar una vista con sus tablas subyacentes, un índice con su propósito de acceso y un plan de ejecución con el trabajo realizado por el motor. Repasa especialmente la diferencia entre “consultar una vista” y “almacenar datos en una vista”, así como el costo de mantener índices cuando se insertan o actualizan filas. La siguiente sesión podrá asumir que ya sabes inspeccionar objetos y validar resultados antes de optimizar.

Puedes reforzar lo trabajado en el canal de **Lideratec Academy** y utilizar el blog como material de consulta académica.

- Blog: https://lideratecacademy.com/blog/
- YouTube: https://www.youtube.com/@LideratecAcademy

Si deseas comparar tus archivos con una versión materializada, puedes utilizar el laboratorio de FASE 15 como recurso opcional posterior.
