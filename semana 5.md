---
course_id: ISIL-PABD
session_id: S05
source_origin: PPT
status: draft
---

# Guía del estudiante - Sesión 05

## Sentencias de mantenimiento de datos en SQL Server

**Curso:** Programación Avanzada de Base de Datos  
**Sesión:** 05  
**Tema:** Sentencias de mantenimiento de datos: INSERT, UPDATE, DELETE, INSERT...SELECT e importación de tablas

## 1. Propósito de la sesión

En esta sesión trabajarás el mantenimiento de datos en SQL Server sobre la base **Northwind**. El objetivo no es memorizar tres palabras reservadas, sino aprender a **predecir qué filas serán afectadas, ejecutar una operación de forma controlada, comprobar el resultado e identificar qué condición produjo el cambio**.

Trabajarás con `INSERT`, `UPDATE` y `DELETE`, además de la inserción basada en resultados de `SELECT` y el flujo de importación de tablas entre bases de datos. La práctica se realiza con transacciones de laboratorio y consultas de verificación para evitar modificar de forma permanente los datos de Northwind.

La evidencia final será observable: podrás mostrar filas insertadas, valores modificados, registros eliminados en copias de laboratorio, tablas creadas a partir de `SELECT` y una importación comprobada mediante consultas.

## 2. Resultado observable

### A. Lectura sugerida del docente

Hoy vamos a trabajar con operaciones que cambian el estado de una base de datos. Antes de ejecutar una sentencia de mantenimiento, necesitamos responder tres preguntas: **qué filas serán afectadas, por qué esas filas y cómo comprobaremos el cambio**. Por eso, en cada ejemplo primero veremos o buscaremos el conjunto de registros objetivo con `SELECT`; después realizaremos la operación; finalmente volveremos a consultar los datos para verificar el efecto. Cuando la operación pueda modificar información existente, utilizaremos una transacción y terminaremos con `ROLLBACK`, de modo que la práctica sea repetible. También aprenderemos a copiar datos mediante `INSERT ... SELECT` y a recorrer el asistente de importación de SQL Server, distinguiendo origen, destino, selección de tablas y validación final. La meta es que puedas explicar cada cláusula y no solo copiar una sentencia.

### B. Desempeños observables

Al finalizar podrás:

1. construir sentencias `INSERT` de uno y varios registros;
2. insertar datos a partir de un `SELECT`;
3. actualizar registros usando condiciones y expresiones;
4. utilizar una subconsulta para decidir qué filas actualizar;
5. crear una tabla de destino e insertar resultados de otra tabla;
6. eliminar registros de forma controlada y verificable;
7. respetar el orden lógico cuando existen datos relacionados;
8. validar una importación de tablas mediante consultas.

### C. Criterio de dominio

Demuestras dominio cuando puedes **explicar antes de ejecutar qué filas cambiarán**, justificar la cláusula `WHERE` o el `SELECT` que determina el conjunto afectado, ejecutar la sentencia y mostrar evidencia de que el resultado coincide con tu predicción.

## 3. Antes de iniciar

### Debes saber

- abrir SQL Server Management Studio (SSMS) y conectarte a una instancia autorizada;
- ejecutar consultas `SELECT` básicas;
- reconocer tablas, columnas y claves de Northwind;
- interpretar condiciones con `WHERE`, `LIKE`, `IN`, `IS NULL` y operadores lógicos ya usados en consultas.

### Debes tener disponible

- una instancia de SQL Server de laboratorio;
- SSMS o el cliente institucional equivalente;
- la base `Northwind` restaurada y accesible;
- permisos para consultar y, en laboratorio, ejecutar sentencias de mantenimiento;
- una base de datos destino autorizada para el ejercicio de importación, si el docente realizará EJ10.

### No se asumirá todavía

No se requiere estudiar procedimientos almacenados, triggers, MERGE, índices avanzados ni administración de producción.

## 4. Preparación del entorno

### Ruta A - El entorno ya existe

1. Abre SSMS y conecta con tu instancia de laboratorio.
2. En el Explorador de objetos, comprueba que existe `Northwind`.
3. Abre **New Query**.
4. Ejecuta:

```sql
USE Northwind;
GO
```

5. Valida el punto de partida:

```sql
SELECT DB_NAME() AS BaseActual;
SELECT TOP (5) CustomerID, CompanyName, Country
FROM Customers
ORDER BY CustomerID;
```

**Resultado esperado:** `BaseActual` debe ser `Northwind` y la segunda consulta debe devolver filas de `Customers`.

### Ruta B - El equipo no está preparado

Esta sesión no instala SQL Server ni SSMS desde cero. Si alguno de los componentes no está disponible, detén la práctica y utiliza la instalación institucional o la preparación realizada en sesiones anteriores. No continúes sobre una base de producción.

## 5. Protocolo de seguridad de la práctica

En operaciones que modifican datos usaremos este patrón:

```sql
BEGIN TRAN;

-- operación de laboratorio

ROLLBACK;
```

`BEGIN TRAN` inicia una transacción. `ROLLBACK` deshace los cambios realizados desde el inicio de esa transacción. Así podemos observar el efecto sin dejar alterada la base al terminar el ejemplo.

> Regla de clase: antes de un `UPDATE` o `DELETE`, ejecuta primero un `SELECT` con la misma condición para comprobar qué filas serán afectadas.

---
# EJ01 - INSERT básico: agregar un cliente y comprobar la fila creada

## Qué vamos a realizar

Insertar un registro en `Customers` con una lista explícita de columnas y verificarlo antes de deshacer la transacción.

## Qué vamos a buscar o comprobar antes de escribir

Antes de escribir la sentencia vamos a **buscar si el identificador ya existe**. Si devuelve cero filas, tendremos un punto de partida válido. Luego insertaremos el cliente y repetiremos la búsqueda.

## Qué aprenderás aquí

Correspondencia columnas-valores, uso de `VALUES`, validación de una clave y efecto de la inserción.

## Archivos, recursos o artefactos involucrados

Base `Northwind`, tabla `Customers`, editor SQL.

## Punto de partida

La tabla `Customers` contiene los clientes originales de Northwind. Utilizaremos el identificador de laboratorio `LAC01`, de cinco caracteres.

## Paso 1 - Comprobar el estado antes

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT CustomerID, CompanyName, ContactName, Country
FROM Customers
WHERE CustomerID = 'LAC01';
```

### Explicación detallada

La condición limita la búsqueda al identificador que pensamos insertar. El resultado esperado antes del `INSERT` es cero filas.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Iniciar una transacción de laboratorio

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;
```

### Explicación detallada

La transacción permite observar el cambio y revertirlo al final con `ROLLBACK`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Insertar el registro

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
INSERT INTO Customers (CustomerID, CompanyName, ContactName, Country)
VALUES ('LAC01', 'Lideratec Demo', 'Ana Torres', 'Peru');
```

### Explicación detallada

`INSERT INTO` identifica la tabla destino. La lista de columnas fija el orden; `VALUES` aporta un valor para cada columna indicada. No se proporcionan columnas opcionales que Northwind puede aceptar como `NULL`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Comprobar el estado después

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT CustomerID, CompanyName, ContactName, Country
FROM Customers
WHERE CustomerID = 'LAC01';
```

### Explicación detallada

Ahora debe aparecer exactamente una fila con los valores insertados.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Deshacer la práctica

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
ROLLBACK;
```

### Explicación detallada

El registro desaparece porque toda la inserción ocurrió dentro de la transacción.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO

SELECT CustomerID, CompanyName, ContactName, Country
FROM Customers
WHERE CustomerID = 'LAC01';

BEGIN TRAN;

INSERT INTO Customers (CustomerID, CompanyName, ContactName, Country)
VALUES ('LAC01', 'Lideratec Demo', 'Ana Torres', 'Peru');

SELECT CustomerID, CompanyName, ContactName, Country
FROM Customers
WHERE CustomerID = 'LAC01';

ROLLBACK;

SELECT CustomerID, CompanyName
FROM Customers
WHERE CustomerID = 'LAC01';
```

## Antes de ejecutar o validar: predicción

Antes del `INSERT`: 0 filas. Después del `INSERT`: 1 fila. Después de `ROLLBACK`: 0 filas.

## Resultado esperado

Una fila visible durante la transacción y ninguna después del rollback.

## Cómo interpretarlo

El cambio fue producido únicamente por `INSERT`; `ROLLBACK` demuestra que la operación estaba contenida en la transacción.

## Variación A

Prueba omitir `ContactName`. Si la columna acepta `NULL`, la inserción sigue siendo válida; si existe una restricción distinta en tu versión de la base, SQL Server la reportará.

## Error o caso límite controlado

Usar un `CustomerID` ya existente produce una violación de clave primaria.

## Por qué ocurre y corrección razonada

Elige un identificador de cinco caracteres que no exista y repite la inserción. No elimines ni modifiques el registro original que generó el conflicto.

## Qué debes poder explicar con tus palabras

Debes poder explicar por qué el orden de columnas debe corresponder con el orden de valores y por qué conviene comprobar primero la clave.

## T01 - Tarea espejo de EJ01

### Enunciado

Inserta temporalmente un proveedor en `Suppliers` con `CompanyName = 'XYZ Supplier'`, `ContactName = 'Maria Lopez'` y `Country = 'Spain'`. Usa una transacción, muestra la fila insertada y termina con `ROLLBACK`. **Evidencia:** consulta que muestre la fila durante la transacción. **Restricción:** no uses `INSERT` sin lista de columnas. **Pista:** `SupplierID` es generado por la tabla; no necesitas proporcionarlo.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ02 - INSERT múltiple: agregar varias filas con una sola sentencia

## Qué vamos a realizar

Insertar más de un empleado usando una sola sentencia `INSERT` y varios grupos de `VALUES`.

## Qué vamos a buscar o comprobar antes de escribir

Antes de escribir vamos a **buscar si ya existen apellidos de laboratorio**. Después insertaremos dos filas dentro de una transacción y contaremos cuántas aparecieron.

## Qué aprenderás aquí

Separación de filas por comas, columnas comunes a todas las filas y validación por predicado.

## Archivos, recursos o artefactos involucrados

Tabla `Employees`.

## Punto de partida

Partimos de `Employees`. Insertaremos dos filas de laboratorio y las identificaremos por apellido.

## Paso 1 - Buscar coincidencias previas

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT EmployeeID, LastName, FirstName
FROM Employees
WHERE LastName IN ('LabMartinez', 'LabGarcia');
```

### Explicación detallada

La consulta debe devolver cero filas para evitar confundir datos nuevos con datos existentes.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Iniciar la transacción

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;
```

### Explicación detallada

Los dos registros formarán parte de la misma práctica reversible.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Insertar dos registros

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
INSERT INTO Employees (LastName, FirstName, Title, Country)
VALUES
  ('LabMartinez', 'Maria', 'Sales Representative', 'Peru'),
  ('LabGarcia', 'Pedro', 'Sales Manager', 'Peru');
```

### Explicación detallada

Cada paréntesis representa una fila. Las cuatro posiciones de cada fila corresponden, en orden, a las cuatro columnas declaradas.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Verificar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT EmployeeID, LastName, FirstName, Title, Country
FROM Employees
WHERE LastName IN ('LabMartinez', 'LabGarcia');
```

### Explicación detallada

Deben verse dos filas y cada una tendrá un `EmployeeID` asignado por SQL Server.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
ROLLBACK;
```

### Explicación detallada

Ambas filas insertadas quedan deshechas.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
BEGIN TRAN;

INSERT INTO Employees (LastName, FirstName, Title, Country)
VALUES
  ('LabMartinez', 'Maria', 'Sales Representative', 'Peru'),
  ('LabGarcia', 'Pedro', 'Sales Manager', 'Peru');

SELECT EmployeeID, LastName, FirstName, Title, Country
FROM Employees
WHERE LastName IN ('LabMartinez', 'LabGarcia');

ROLLBACK;
```

## Antes de ejecutar o validar: predicción

Deben aparecer dos filas nuevas durante la transacción.

## Resultado esperado

Dos empleados identificables por sus apellidos de laboratorio.

## Cómo interpretarlo

Una sola sentencia `INSERT` puede crear varias filas cuando cada grupo de valores tiene la misma estructura.

## Variación A

Añade una tercera fila con las mismas cuatro columnas y comprueba que la sentencia sigue teniendo un único `INSERT`.

## Error o caso límite controlado

Omitir un valor en uno de los grupos provoca que el número de valores no coincida con las columnas.

## Por qué ocurre y corrección razonada

Revisa cada grupo de paréntesis y confirma que todos contienen cuatro valores en el mismo orden.

## Qué debes poder explicar con tus palabras

Debes poder explicar qué parte se repite por fila y qué parte se declara una sola vez.

## T02 - Tarea espejo de EJ02

### Enunciado

Dentro de una transacción, inserta dos empleados de prueba con los apellidos `LabRojas` y `LabQuispe`, indicando `FirstName`, `Title` y `Country`. Verifica que aparezcan dos filas y termina con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ03 - INSERT con SELECT: crear un pedido a partir de un cliente existente

## Qué vamos a realizar

Insertar en `Orders` utilizando el resultado de un `SELECT` sobre `Customers`.

## Qué vamos a buscar o comprobar antes de escribir

Primero **buscaremos el cliente que alimentará la inserción**. Si el `SELECT` no devuelve una fila, el `INSERT ... SELECT` tampoco insertará nada.

## Qué aprenderás aquí

`INSERT ... SELECT`, transferencia de valores entre tablas y condición que determina si se inserta una fila.

## Archivos, recursos o artefactos involucrados

Tablas `Customers` y `Orders`.

## Punto de partida

Usaremos el cliente `ALFKI`, presente en Northwind, y la fecha actual mediante `GETDATE()`.

## Paso 1 - Verificar la fuente

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT CustomerID, CompanyName
FROM Customers
WHERE CustomerID = 'ALFKI';
```

### Explicación detallada

Esta es la fila fuente. Su `CustomerID` será utilizado por el `INSERT`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Iniciar la transacción

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;
```

### Explicación detallada

El nuevo pedido será temporal.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Insertar desde SELECT

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
INSERT INTO Orders (CustomerID, OrderDate)
SELECT CustomerID, GETDATE()
FROM Customers
WHERE CustomerID = 'ALFKI';
```

### Explicación detallada

El `SELECT` devuelve dos columnas: `CustomerID` y una fecha. Ese orden coincide con las dos columnas destino de `Orders`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Encontrar el pedido creado

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT TOP (1) OrderID, CustomerID, OrderDate
FROM Orders
WHERE CustomerID = 'ALFKI'
ORDER BY OrderID DESC;
```

### Explicación detallada

El pedido recién insertado debe aparecer entre los de mayor `OrderID` para ese cliente durante la transacción.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
ROLLBACK;
```

### Explicación detallada

El pedido temporal deja de existir.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
SELECT CustomerID, CompanyName
FROM Customers
WHERE CustomerID = 'ALFKI';

BEGIN TRAN;

INSERT INTO Orders (CustomerID, OrderDate)
SELECT CustomerID, GETDATE()
FROM Customers
WHERE CustomerID = 'ALFKI';

SELECT TOP (1) OrderID, CustomerID, OrderDate
FROM Orders
WHERE CustomerID = 'ALFKI'
ORDER BY OrderID DESC;

ROLLBACK;
```

## Antes de ejecutar o validar: predicción

Si `ALFKI` existe, se insertará un pedido; si no existe, se insertarán cero filas.

## Resultado esperado

Un pedido temporal para `ALFKI` con fecha cercana a la ejecución.

## Cómo interpretarlo

`INSERT ... SELECT` depende del conjunto devuelto por la consulta fuente; no existe un bloque `VALUES`.

## Variación A

Cambia temporalmente el filtro a un `CustomerID` inexistente y observa que el `SELECT` devuelve cero filas; no dejes cambios permanentes.

## Error o caso límite controlado

Usar una consulta que devuelva más columnas o tipos incompatibles con el destino provoca error.

## Por qué ocurre y corrección razonada

Alinea cantidad, orden y tipos de datos entre columnas destino y expresiones del `SELECT`.

## Qué debes poder explicar con tus palabras

Debes poder explicar de dónde sale cada valor insertado.

## T03 - Tarea espejo de EJ03

### Enunciado

Inserta temporalmente un empleado `John Doe` cuyo `ReportsTo` sea el `EmployeeID` de `Andrew Fuller`. Antes de insertar, comprueba que Andrew Fuller existe. Usa una subconsulta escalar para obtener el identificador del supervisor, muestra el registro insertado y termina con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ04 - UPDATE seguro: modificar una fila y luego varias filas con WHERE

## Qué vamos a realizar

Aplicar `UPDATE` con una condición precisa, previsualizar las filas afectadas y comprobar el cambio.

## Qué vamos a buscar o comprobar antes de escribir

Antes de escribir `UPDATE` vamos a **buscar exactamente la fila objetivo**. La consulta de previsualización debe usar la misma condición que la actualización.

## Qué aprenderás aquí

`SET`, `WHERE`, actualización de uno o varios registros y riesgo de omitir la condición.

## Archivos, recursos o artefactos involucrados

Tablas `Employees` y `Customers`.

## Punto de partida

Primero actualizaremos un empleado identificado por `EmployeeID`; luego veremos una variante de actualización múltiple por patrón de código postal.

## Paso 1 - Previsualizar la fila

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT EmployeeID, FirstName, LastName
FROM Employees
WHERE EmployeeID = 1;
```

### Explicación detallada

Debe aparecer como máximo una fila porque `EmployeeID` identifica al empleado.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Iniciar la transacción

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;
```

### Explicación detallada

La modificación será reversible.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Actualizar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
UPDATE Employees
SET FirstName = 'John',
    LastName = 'Smith'
WHERE EmployeeID = 1;
```

### Explicación detallada

`SET` expresa el nuevo estado. `WHERE` decide qué fila recibe ese estado.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Verificar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT EmployeeID, FirstName, LastName
FROM Employees
WHERE EmployeeID = 1;
```

### Explicación detallada

Ahora deben verse los valores `John` y `Smith` mientras la transacción esté activa.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
ROLLBACK;
```

### Explicación detallada

El empleado recupera sus valores originales.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
SELECT EmployeeID, FirstName, LastName
FROM Employees
WHERE EmployeeID = 1;

BEGIN TRAN;

UPDATE Employees
SET FirstName = 'John',
    LastName = 'Smith'
WHERE EmployeeID = 1;

SELECT EmployeeID, FirstName, LastName
FROM Employees
WHERE EmployeeID = 1;

ROLLBACK;
```

## Antes de ejecutar o validar: predicción

Cambiará únicamente la fila con `EmployeeID = 1`.

## Resultado esperado

La fila muestra los nuevos nombres durante la transacción y vuelve al estado original después.

## Cómo interpretarlo

La precisión de `WHERE` controla el alcance de `UPDATE`.

## Variación A

Como variante, previsualiza clientes con `PostalCode LIKE '101%'` y analiza cuántas filas cambiarían antes de ejecutar cualquier actualización.

## Error o caso límite controlado

Omitir `WHERE` haría que `UPDATE` afecte todas las filas de la tabla.

## Por qué ocurre y corrección razonada

Detén la ejecución antes del `UPDATE`, escribe primero el `SELECT` de previsualización y confirma el número de filas objetivo.

## Qué debes poder explicar con tus palabras

Debes poder explicar qué ocurriría si se elimina la cláusula `WHERE`.

## T04 - Tarea espejo de EJ04

### Enunciado

Actualiza temporalmente el `Title` del empleado con `EmployeeID = 5` a `Senior Sales Representative`. Previsualiza la fila, ejecuta el `UPDATE`, verifica el nuevo valor y finaliza con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ05 - UPDATE con expresión aritmética y condición

## Qué vamos a realizar

Incrementar precios en 10 % para una categoría específica y validar el cálculo.

## Qué vamos a buscar o comprobar antes de escribir

Antes de escribir el `UPDATE` vamos a **calcular qué precio esperamos obtener** para varias filas. Así podremos comparar predicción y resultado.

## Qué aprenderás aquí

Asignaciones basadas en el valor actual de la columna, filtros y predicción del resultado.

## Archivos, recursos o artefactos involucrados

Tabla `Products`.

## Punto de partida

Trabajaremos con los productos de `CategoryID = 2` sin conservar cambios.

## Paso 1 - Predecir precios

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT TOP (5) ProductID, ProductName, UnitPrice AS PrecioAntes,
       UnitPrice * 1.10 AS PrecioEsperado
FROM Products
WHERE CategoryID = 2
ORDER BY ProductID;
```

### Explicación detallada

`PrecioEsperado` no modifica nada; solo calcula el valor que esperamos después.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Iniciar transacción

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;
```

### Explicación detallada

Aísla el cambio.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Aplicar el incremento

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
UPDATE Products
SET UnitPrice = UnitPrice * 1.10
WHERE CategoryID = 2;
```

### Explicación detallada

La expresión usa el precio actual de cada fila y lo multiplica por `1.10`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Comprobar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT TOP (5) ProductID, ProductName, UnitPrice
FROM Products
WHERE CategoryID = 2
ORDER BY ProductID;
```

### Explicación detallada

Los valores deben coincidir, considerando el tipo de dato y redondeo de la columna, con la predicción previa.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
ROLLBACK;
```

### Explicación detallada

Los precios originales se restauran.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
SELECT TOP (5) ProductID, ProductName, UnitPrice AS PrecioAntes,
       UnitPrice * 1.10 AS PrecioEsperado
FROM Products
WHERE CategoryID = 2
ORDER BY ProductID;

BEGIN TRAN;
UPDATE Products
SET UnitPrice = UnitPrice * 1.10
WHERE CategoryID = 2;

SELECT TOP (5) ProductID, ProductName, UnitPrice
FROM Products
WHERE CategoryID = 2
ORDER BY ProductID;
ROLLBACK;
```

## Antes de ejecutar o validar: predicción

Cada precio de categoría 2 aumenta aproximadamente 10 %.

## Resultado esperado

Los productos fuera de `CategoryID = 2` permanecen iguales.

## Cómo interpretarlo

`SET` puede usar el valor anterior de la propia columna para calcular el nuevo estado.

## Variación A

Añade una segunda condición, por ejemplo `AND UnitPrice > 10`, y vuelve a previsualizar antes de ejecutar.

## Error o caso límite controlado

Confundir `1.10` con `10` multiplicaría los precios por diez en vez de aumentarlos 10 %.

## Por qué ocurre y corrección razonada

Traduce el porcentaje a factor: 10 % de aumento = valor actual × 1.10.

## Qué debes poder explicar con tus palabras

Debes poder justificar la expresión matemática y la condición.

## T05 - Tarea espejo de EJ05

### Enunciado

Actualiza temporalmente todos los precios de `Products` para incrementarlos en 10 %. Antes de hacerlo, cuenta cuántas filas existen y muestra una muestra de precios. Después del `UPDATE`, valida una muestra y termina con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ06 - UPDATE con subconsulta: decidir filas usando otra tabla

## Qué vamos a realizar

Actualizar productos a partir de los proveedores obtenidos por una subconsulta.

## Qué vamos a buscar o comprobar antes de escribir

Antes de escribir el `UPDATE` vamos a **resolver la subconsulta por separado**. Después buscaremos qué productos usan esos identificadores. Solo entonces ejecutaremos la actualización.

## Qué aprenderás aquí

`IN (SELECT ...)`, separación entre conjunto de proveedores y conjunto de productos a actualizar.

## Archivos, recursos o artefactos involucrados

Tablas `Products` y `Suppliers`.

## Punto de partida

Buscaremos el `SupplierID` de `Exotic Liquids` y luego los productos asociados a ese proveedor.

## Paso 1 - Resolver la subconsulta

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT SupplierID, CompanyName
FROM Suppliers
WHERE CompanyName = 'Exotic Liquids';
```

### Explicación detallada

Obtiene el conjunto de proveedores que servirá como filtro.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Previsualizar productos

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT ProductID, ProductName, SupplierID, CategoryID
FROM Products
WHERE SupplierID IN (
    SELECT SupplierID
    FROM Suppliers
    WHERE CompanyName = 'Exotic Liquids'
);
```

### Explicación detallada

Estas son exactamente las filas que recibirán el nuevo `CategoryID`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Iniciar transacción y actualizar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;

UPDATE Products
SET CategoryID = 2
WHERE SupplierID IN (
    SELECT SupplierID
    FROM Suppliers
    WHERE CompanyName = 'Exotic Liquids'
);
```

### Explicación detallada

La subconsulta no modifica datos; produce identificadores. El `UPDATE` modifica las filas de `Products` que coinciden.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Verificar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT ProductID, ProductName, SupplierID, CategoryID
FROM Products
WHERE SupplierID IN (
    SELECT SupplierID
    FROM Suppliers
    WHERE CompanyName = 'Exotic Liquids'
);
```

### Explicación detallada

Durante la transacción los productos seleccionados deben mostrar `CategoryID = 2`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
ROLLBACK;
```

### Explicación detallada

Restaura las categorías originales.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
SELECT SupplierID, CompanyName
FROM Suppliers
WHERE CompanyName = 'Exotic Liquids';

SELECT ProductID, ProductName, SupplierID, CategoryID
FROM Products
WHERE SupplierID IN (
    SELECT SupplierID
    FROM Suppliers
    WHERE CompanyName = 'Exotic Liquids'
);

BEGIN TRAN;
UPDATE Products
SET CategoryID = 2
WHERE SupplierID IN (
    SELECT SupplierID
    FROM Suppliers
    WHERE CompanyName = 'Exotic Liquids'
);

SELECT ProductID, ProductName, SupplierID, CategoryID
FROM Products
WHERE SupplierID IN (
    SELECT SupplierID
    FROM Suppliers
    WHERE CompanyName = 'Exotic Liquids'
);
ROLLBACK;
```

## Antes de ejecutar o validar: predicción

Cambiarán solo productos cuyo `SupplierID` pertenezca al conjunto devuelto por la subconsulta.

## Resultado esperado

Filas filtradas por proveedor con `CategoryID = 2` durante la transacción.

## Cómo interpretarlo

La subconsulta determina el conjunto; la sentencia externa realiza el cambio.

## Variación A

Cambia la empresa buscada por otro proveedor existente y repite solo la previsualización.

## Error o caso límite controlado

Si la subconsulta devuelve cero identificadores, el `UPDATE` afectará cero filas; si el filtro es demasiado amplio, podría afectar más filas de las previstas.

## Por qué ocurre y corrección razonada

Ejecuta y revisa primero la subconsulta y luego la consulta de previsualización.

## Qué debes poder explicar con tus palabras

Debes poder explicar qué consulta decide y qué sentencia modifica.

## T06 - Tarea espejo de EJ06

### Enunciado

Incrementa temporalmente 10 % el precio de los productos cuya categoría sea `Beverages`. Obtén el `CategoryID` desde `Categories` mediante una subconsulta, previsualiza los productos, ejecuta la actualización, valida y termina con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ07 - INSERT ... SELECT: crear una tabla de destino e insertar resultados

## Qué vamos a realizar

Crear una copia vacía de `Customers` y luego insertar en ella únicamente clientes de Londres.

## Qué vamos a buscar o comprobar antes de escribir

Primero **buscaremos cuántos clientes de Londres existen**. Después crearemos una tabla vacía con la misma estructura básica y finalmente insertaremos solo esas filas.

## Qué aprenderás aquí

`SELECT ... INTO ... WHERE 1=0`, tabla destino preexistente para `INSERT ... SELECT`, filtro de filas.

## Archivos, recursos o artefactos involucrados

Tabla `Customers`; tabla de laboratorio `CustomersLondon_S05` creada dentro de una transacción.

## Punto de partida

Queremos practicar la copia selectiva sin dejar una tabla permanente en la base.

## Paso 1 - Contar las filas fuente

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT COUNT(*) AS ClientesLondon
FROM Customers
WHERE City = 'London';
```

### Explicación detallada

Este conteo es la referencia para validar la inserción posterior.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Iniciar transacción

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;
```

### Explicación detallada

La creación de la tabla y la inserción quedarán dentro de la práctica.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Crear una copia vacía

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT *
INTO CustomersLondon_S05
FROM Customers
WHERE 1 = 0;
```

### Explicación detallada

`SELECT INTO` crea la tabla. La condición `1 = 0` nunca se cumple, por lo que se copia la estructura de columnas pero ninguna fila.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Insertar filas seleccionadas

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
INSERT INTO CustomersLondon_S05
SELECT *
FROM Customers
WHERE City = 'London';
```

### Explicación detallada

La tabla destino ya existe y recibe solo las filas cuyo `City` es `London`.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Validar y revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT COUNT(*) AS FilasCopiadas
FROM CustomersLondon_S05;

SELECT CustomerID, CompanyName, City
FROM CustomersLondon_S05
ORDER BY CustomerID;

ROLLBACK;
```

### Explicación detallada

El número de filas copiadas debe coincidir con el conteo del paso 1. `ROLLBACK` elimina también la tabla creada dentro de la transacción.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
SELECT COUNT(*) AS ClientesLondon
FROM Customers
WHERE City = 'London';

BEGIN TRAN;
SELECT *
INTO CustomersLondon_S05
FROM Customers
WHERE 1 = 0;

INSERT INTO CustomersLondon_S05
SELECT *
FROM Customers
WHERE City = 'London';

SELECT COUNT(*) AS FilasCopiadas
FROM CustomersLondon_S05;

SELECT CustomerID, CompanyName, City
FROM CustomersLondon_S05
ORDER BY CustomerID;
ROLLBACK;
```

## Antes de ejecutar o validar: predicción

El conteo inicial y el conteo de la tabla destino deben ser iguales.

## Resultado esperado

Una tabla temporal de práctica con solo clientes de Londres durante la transacción.

## Cómo interpretarlo

`SELECT INTO` crea el destino; `INSERT INTO ... SELECT` llena ese destino a partir de una consulta.

## Variación A

En lugar de `SELECT *`, crea una tabla solo con `CustomerID`, `CompanyName` y `City` y ajusta el `INSERT` a esas columnas.

## Error o caso límite controlado

Si el `SELECT` devuelve un conjunto de columnas incompatible con la tabla destino, el `INSERT` falla.

## Por qué ocurre y corrección razonada

Compara número, orden y tipos de columnas del destino con la consulta fuente.

## Qué debes poder explicar con tus palabras

Debes poder explicar por qué `WHERE 1=0` crea estructura sin copiar filas.

## T07 - Tarea espejo de EJ07

### Enunciado

Crea dentro de una transacción una tabla `ProductPrices_S05` con las columnas `ProductName` y `UnitPrice`, inicialmente vacía. Luego inserta esas dos columnas desde `Products`, cuenta las filas y termina con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ08 - DELETE controlado: eliminar filas de una copia de laboratorio

## Qué vamos a realizar

Eliminar clientes sin teléfono de una copia para practicar `DELETE` sin afectar la tabla original.

## Qué vamos a buscar o comprobar antes de escribir

Antes de escribir `DELETE` vamos a **buscar cuántas filas cumplen la condición**. Luego aplicaremos exactamente esa condición sobre la copia.

## Qué aprenderás aquí

Previsualización, `DELETE ... WHERE`, conteo antes/después y separación entre tabla original y copia.

## Archivos, recursos o artefactos involucrados

Tabla `Customers` y copia `CustomersDeleteLab_S05` dentro de transacción.

## Punto de partida

No eliminaremos clientes de la tabla original porque podrían existir relaciones con pedidos. Primero construiremos una copia de laboratorio.

## Paso 1 - Iniciar transacción y crear copia

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;

SELECT *
INTO CustomersDeleteLab_S05
FROM Customers;
```

### Explicación detallada

La copia contiene las filas de `Customers`, pero se usa únicamente para practicar la eliminación.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Previsualizar objetivo

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT CustomerID, CompanyName, Phone
FROM CustomersDeleteLab_S05
WHERE Phone IS NULL;
```

### Explicación detallada

Estas son las filas que desaparecerán de la copia.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Eliminar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
DELETE FROM CustomersDeleteLab_S05
WHERE Phone IS NULL;
```

### Explicación detallada

`DELETE FROM` identifica la tabla y `WHERE Phone IS NULL` limita el conjunto.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Confirmar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT CustomerID, CompanyName, Phone
FROM CustomersDeleteLab_S05
WHERE Phone IS NULL;
```

### Explicación detallada

El resultado debe ser cero filas en la copia.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
ROLLBACK;
```

### Explicación detallada

La copia desaparece y `Customers` nunca fue modificada directamente.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
BEGIN TRAN;
SELECT * INTO CustomersDeleteLab_S05 FROM Customers;

SELECT CustomerID, CompanyName, Phone
FROM CustomersDeleteLab_S05
WHERE Phone IS NULL;

DELETE FROM CustomersDeleteLab_S05
WHERE Phone IS NULL;

SELECT CustomerID, CompanyName, Phone
FROM CustomersDeleteLab_S05
WHERE Phone IS NULL;
ROLLBACK;
```

## Antes de ejecutar o validar: predicción

Después de borrar, ninguna fila de la copia debe conservar `Phone IS NULL`.

## Resultado esperado

La tabla original queda intacta y la copia deja de mostrar esas filas.

## Cómo interpretarlo

La eliminación debe evaluarse sobre el conjunto exacto previamente observado.

## Variación A

Cambia la condición por `Country <> 'USA'` sobre otra copia y compara cuántas filas serían eliminadas.

## Error o caso límite controlado

Omitir `WHERE` elimina todas las filas de la tabla objetivo.

## Por qué ocurre y corrección razonada

Nunca ejecutes el `DELETE` hasta haber validado con un `SELECT` la misma condición.

## Qué debes poder explicar con tus palabras

Debes poder explicar por qué practicamos sobre una copia y cómo demostramos que la tabla original no cambió.

## T08 - Tarea espejo de EJ08

### Enunciado

Dentro de una transacción crea `ProductsDeleteLab_S05` como copia de `Products`. Elimina de la copia los productos con `UnitPrice < 10`, muestra el conteo antes y después, y finaliza con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ09 - DELETE con datos relacionados: respetar el orden de eliminación

## Qué vamos a realizar

Eliminar, en copias de laboratorio, detalles y pedidos de un cliente respetando la dependencia lógica hijo -> padre.

## Qué vamos a buscar o comprobar antes de escribir

Antes de escribir los `DELETE` vamos a **buscar los OrderID del cliente y contar cuántos detalles dependen de ellos**. La eliminación se hará primero en los detalles y luego en los pedidos.

## Qué aprenderás aquí

Orden de operaciones, subconsulta de `OrderID`, tablas relacionadas y validación por conteos.

## Archivos, recursos o artefactos involucrados

Tablas `Orders`, `[Order Details]` y copias de laboratorio.

## Punto de partida

Usaremos el cliente `ALFKI`. Primero construiremos dos copias limitadas a sus pedidos y detalles.

## Paso 1 - Iniciar transacción y crear copias

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
BEGIN TRAN;

SELECT *
INTO OrdersDeleteLab_S05
FROM Orders
WHERE CustomerID = 'ALFKI';

SELECT od.*
INTO OrderDetailsDeleteLab_S05
FROM [Order Details] AS od
WHERE od.OrderID IN (
    SELECT OrderID
    FROM OrdersDeleteLab_S05
);
```

### Explicación detallada

La primera copia contiene pedidos del cliente. La segunda contiene los detalles de esos pedidos.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Contar antes

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT COUNT(*) AS PedidosAntes FROM OrdersDeleteLab_S05;
SELECT COUNT(*) AS DetallesAntes FROM OrderDetailsDeleteLab_S05;
```

### Explicación detallada

Los conteos permiten comprobar el cambio.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Eliminar hijos primero

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
DELETE FROM OrderDetailsDeleteLab_S05
WHERE OrderID IN (
    SELECT OrderID
    FROM OrdersDeleteLab_S05
);
```

### Explicación detallada

Los detalles son dependientes de los pedidos. En un esquema con integridad referencial, este orden evita intentar borrar primero el registro padre.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Eliminar pedidos

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
DELETE FROM OrdersDeleteLab_S05
WHERE CustomerID = 'ALFKI';
```

### Explicación detallada

Una vez retirados los detalles de la copia, eliminamos los pedidos objetivo.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Validar y revertir

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
SELECT COUNT(*) AS PedidosDespues FROM OrdersDeleteLab_S05;
SELECT COUNT(*) AS DetallesDespues FROM OrderDetailsDeleteLab_S05;

ROLLBACK;
```

### Explicación detallada

Ambos conteos deben quedar en cero dentro de las copias y luego todo se revierte.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
USE Northwind;
GO
BEGIN TRAN;
SELECT * INTO OrdersDeleteLab_S05
FROM Orders
WHERE CustomerID = 'ALFKI';

SELECT od.* INTO OrderDetailsDeleteLab_S05
FROM [Order Details] AS od
WHERE od.OrderID IN (SELECT OrderID FROM OrdersDeleteLab_S05);

SELECT COUNT(*) AS PedidosAntes FROM OrdersDeleteLab_S05;
SELECT COUNT(*) AS DetallesAntes FROM OrderDetailsDeleteLab_S05;

DELETE FROM OrderDetailsDeleteLab_S05
WHERE OrderID IN (SELECT OrderID FROM OrdersDeleteLab_S05);

DELETE FROM OrdersDeleteLab_S05
WHERE CustomerID = 'ALFKI';

SELECT COUNT(*) AS PedidosDespues FROM OrdersDeleteLab_S05;
SELECT COUNT(*) AS DetallesDespues FROM OrderDetailsDeleteLab_S05;
ROLLBACK;
```

## Antes de ejecutar o validar: predicción

Primero deben desaparecer los detalles y después los pedidos de las copias.

## Resultado esperado

Los conteos finales de ambas copias son cero.

## Cómo interpretarlo

Cuando existen dependencias, el orden importa: primero los registros dependientes y después los registros referenciados.

## Variación A

Invierte el orden solo como análisis teórico: explica por qué en tablas originales con claves foráneas podría fallar. No ejecutes esa variante sobre las tablas originales.

## Error o caso límite controlado

Intentar borrar primero el padre puede fallar por una restricción `FOREIGN KEY` o dejar una secuencia lógica incorrecta.

## Por qué ocurre y corrección razonada

Identifica la tabla dependiente y elimina primero sus filas relacionadas; usa una transacción y valida conteos.

## Qué debes poder explicar con tus palabras

Debes poder explicar por qué el orden hijo -> padre protege la integridad.

## T09 - Tarea espejo de EJ09

### Enunciado

En copias de laboratorio de `Orders` y `[Order Details]`, identifica pedidos cuyo total sea mayor a 1000 usando `GROUP BY` y `HAVING SUM(Quantity * UnitPrice) > 1000`. Elimina primero sus detalles y después los pedidos seleccionados. Valida conteos y termina con `ROLLBACK`.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# EJ10 - Importación de tablas entre bases de datos con el asistente de SQL Server

## Qué vamos a realizar

Recorrer el flujo de importación desde una base origen hacia una base destino autorizada y verificar el resultado.

## Qué vamos a buscar o comprobar antes de escribir

Antes de abrir el asistente vamos a **definir qué tabla se copiará y cómo comprobaremos que llegó al destino**. La validación no termina cuando el asistente muestra éxito: también consultaremos la tabla en la base destino.

## Qué aprenderás aquí

Origen, destino, opción de copia, selección de tablas, ejecución inmediata, revisión de mensajes y validación por consulta.

## Archivos, recursos o artefactos involucrados

SSMS, SQL Server Import and Export Wizard, base origen `Northwind` y una base destino de laboratorio autorizada.

## Punto de partida

La base origen contiene las tablas que se copiarán. La base destino debe existir y estar autorizada por el docente.

## Paso 1 - Identificar origen y destino

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

**Origen:** instancia y base `Northwind`.  
**Destino:** base de laboratorio indicada por el docente.  
**Tabla de demostración:** `Categories`.

### Explicación detallada

No uses una base de producción ni una base cuyo contenido no tengas permiso de modificar.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 2 - Abrir el asistente

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

En el Explorador de objetos, sobre la base destino, abre el menú contextual, entra a **Tasks/Tareas** y elige la opción de **Import Data/Importar datos** o su equivalente visible en tu versión de SSMS.

### Explicación detallada

El asistente guía la transferencia; la interfaz puede variar, pero deben aparecer etapas equivalentes de origen, destino y selección.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 3 - Configurar el origen

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

Selecciona SQL Server como origen, indica el servidor autorizado, autenticación correspondiente y la base `Northwind`. Continúa solo cuando la conexión sea válida.

### Explicación detallada

Aquí se define de dónde se leerán los datos.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 4 - Configurar el destino

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

Selecciona el servidor y la base destino autorizada. Revisa cuidadosamente el nombre de la base antes de continuar.

### Explicación detallada

Aquí se define dónde se crearán o llenarán las tablas.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 5 - Elegir copia de tablas

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

Selecciona la opción equivalente a **Copy data from one or more tables or views**. Marca `Categories` para la demostración y revisa la tabla destino propuesta.

### Explicación detallada

La fuente presenta dos rutas: copiar tablas/vistas o escribir una consulta. Esta sesión practica la primera.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 6 - Ejecutar y revisar

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

Selecciona ejecución inmediata, revisa el resumen y finaliza. Si aparece un error, lee la columna de mensajes antes de cambiar configuraciones.

### Explicación detallada

El estado final debe mostrar éxito para la tabla copiada.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Paso 7 - Validar en SQL

### Estado antes / intención

Antes de ejecutar este microbloque, identifica qué filas o estado esperas observar.

### Escribe / configura ahora

```sql
USE [BASE_DESTINO_AUTORIZADA];
GO
SELECT COUNT(*) AS FilasCategories
FROM dbo.Categories;

SELECT TOP (5) *
FROM dbo.Categories;
```

### Explicación detallada

Reemplaza el marcador por el nombre real de la base destino. La tabla debe existir y devolver filas.

### Estado después

Comprueba el resultado inmediatamente antes de continuar.

### Por qué podemos avanzar

El microbloque deja verificado el supuesto necesario para el siguiente paso.

## Auditoría del artefacto construido

| Elemento | Qué hace | Por qué existe | Efecto |
|---|---|---|---|
| Tabla/consulta objetivo | Identifica dónde ocurre la operación | Evita trabajar sobre una tabla equivocada | Define el alcance |
| Condición o fuente | Determina las filas | Controla el conjunto afectado | Reduce cambios no deseados |
| Sentencia de mantenimiento | Cambia el estado | Materializa el objetivo del ejemplo | Inserta, actualiza, elimina o copia |
| Consulta de validación | Observa el resultado | Permite comparar predicción y realidad | Confirma el cambio |
| `ROLLBACK` cuando aplica | Revierte la práctica | Mantiene Northwind reutilizable | Restaura el estado |

## Archivo / artefacto completo al terminar este ejemplo

```sql
-- EJ10 se ejecuta principalmente en el asistente gráfico.
-- Validación posterior:
USE [BASE_DESTINO_AUTORIZADA];
GO
SELECT COUNT(*) AS FilasCategories
FROM dbo.Categories;

SELECT TOP (5) *
FROM dbo.Categories;
```

## Antes de ejecutar o validar: predicción

El asistente debe finalizar sin errores para la tabla seleccionada y la consulta de destino debe devolver filas.

## Resultado esperado

Tabla `Categories` visible en la base destino con un conteo coherente respecto del origen.

## Cómo interpretarlo

Una importación se considera completa cuando el asistente reporta éxito y una consulta independiente confirma que la tabla está disponible en el destino.

## Variación A

Importa una segunda tabla pequeña autorizada y compara los conteos origen/destino.

## Error o caso límite controlado

Seleccionar por error una base destino equivocada puede copiar datos al lugar incorrecto; errores de mapeo o permisos pueden detener la operación.

## Por qué ocurre y corrección razonada

Revisa origen, destino, tabla y mensajes antes de reintentar. Si el destino no es de laboratorio, detén la práctica.

## Qué debes poder explicar con tus palabras

Debes poder explicar la diferencia entre origen y destino y cómo validar una importación más allá del mensaje del asistente.

## T10 - Tarea espejo de EJ10

### Enunciado

Importa `Suppliers` desde `Northwind` hacia la base destino de laboratorio indicada por el docente. Antes de ejecutar, registra el conteo de filas en origen. Después de la importación, registra el conteo en destino y compáralos. Si la tabla ya existe, no sobrescribas datos sin autorización.

### Archivos/artefactos a modificar

Trabaja en una nueva ventana de consulta o en el artefacto indicado por el docente.

### Restricción

No dejes cambios permanentes en `Northwind`; usa `BEGIN TRAN` y `ROLLBACK` cuando la tarea modifique datos.

### Pista

Repite el patrón del ejemplo, pero no copies valores de la solución docente.

### Evidencia que debes mostrar

Captura o resultado de consulta donde se observe el estado antes y después.

### Cómo saber si está correcta

La evidencia coincide con tu predicción y el conjunto afectado es exactamente el indicado por la tarea.

# 6. Cierre de la sesión

En esta sesión construiste una secuencia completa de mantenimiento de datos: primero identificaste el conjunto objetivo; después insertaste, actualizaste, eliminaste o copiaste datos; finalmente verificaste el efecto y, cuando correspondía, deshiciste la práctica. La idea central no es ejecutar sentencias “a ciegas”, sino controlar el alcance con consultas previas, condiciones precisas y validaciones posteriores.

También comprobaste que `INSERT` puede recibir valores directos o resultados de `SELECT`, que `UPDATE` puede calcular nuevos valores o depender de una subconsulta, y que `DELETE` exige especial cuidado cuando existen datos relacionados. En la importación, el éxito del asistente no sustituye la validación: una consulta sobre la base destino debe confirmar que la tabla y sus filas realmente están disponibles.

Antes de la siguiente sesión repasa especialmente: correspondencia entre columnas y valores, uso de `WHERE`, lectura de subconsultas, comparación del estado antes/después y orden de eliminación en datos relacionados. Si deseas validar tu resultado o trabajar desde una versión ya materializada, puedes utilizar opcionalmente el laboratorio de FASE 15.

**Lideratec Academy**  
Blog: https://lideratecacademy.com/blog/  
YouTube: https://www.youtube.com/@LideratecAcademy
