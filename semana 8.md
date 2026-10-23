# FASE 13 AMPLIADA | Guía del estudiante - repaso aplicado S05, S06 y S07

**Lideratec Academy · ISIL · Programación Avanzada de Base de Datos (30627)**  
**Tiempo presencial:** 120 minutos exactos · **Herramientas:** Motor SQL Server + SQL Server Management Studio 22 (SSMS 22) · **Datos fuente:** Northwind (solo lectura).

> **Naturaleza académica:** repaso integrador **propuesto** a partir de los temas del curso. Los contenidos oficiales de S05, S06 y S07 se han respetado; el caso, los ejercicios y sus datos de trabajo son nuevos. No es un cambio del sílabo ni un examen PA2.

## 1. Propósito y objetivo medible de la clase

**Situación problema:** un analista debe preparar información confiable de pedidos por país y año, corregir de forma controlada errores detectados, publicar reportes reutilizables y **justificar técnicamente** si un índice ayuda a consultas de un cliente. ¿Cómo demuestra que sus datos no se alteraron accidentalmente y que su conclusión sobre rendimiento está sustentada en evidencia?

**OBJETIVO DE APRENDIZAJE:** *Al finalizar 120 minutos, el estudiante construirá y justificará un miniinforme técnico de pedidos por mercado, aplicando operaciones DML seguras, comprobaciones de consistencia, dos vistas de resumen, un índice no agrupado y la lectura de planes reales y estadísticas de E/S en SQL Server; entregará cuatro evidencias observables en una base de práctica derivada de Northwind, sin escribir en Northwind.*

**Producto final concreto:** una **ficha de diagnóstico de cuatro evidencias**, elaborada durante la clase y cerrada en EJ06:

1. **Datos fiables (S05):** capturas o cifras de filas insertadas, actualizadas y eliminadas, con controles de `WHERE` y de repetición.
2. **Reporte reutilizable (S06):** resultados de dos vistas (`por país` y `por país/año`) y conciliación con `PedidosLab`.
3. **Decisión sobre índice (S06-S07):** mismo `SELECT` antes y después de crear índice, más operadores y lecturas lógicas observadas (sin inventar valores).
4. **Conclusión válida (S07):** comparación de predicados, selectividad y reconocimiento de que los registros multiplicados 60 veces son datos **simulados**.

**Criterio de logro:** podrá explicar **qué hizo, por qué, qué objeto cambió, qué valor observó y qué decisión recomienda**. Cumplir las cuatro evidencias, no ejecutar muchas consultas sin interpretación, significa lograr el objetivo.

### Qué se recupera de cada sesión

| Sesión | Conocimientos verificados del curso | Aplicación en el caso |
|---|---|---|
| **05** | `INSERT`, `INSERT ... SELECT`, `UPDATE`, `DELETE`. | Cargar muestra, corregir pedidos ficticios y auditar filas. |
| **06** | Vistas como tablas virtuales e índices. | Reportes por país/año, conciliación e índice compuesto. |
| **07** | Plan real, interpretación de resultados, buenas prácticas y datos de prueba. | Comparar lecturas, filtros, selectividad y origen simulado. |

**Plan de 12 ejercicios con propósito:** conservamos los seis EJ01-EJ06 del material anterior y agregamos seis EJ07-EJ12. El orden en el aula se define por **prerrequisitos**, no por número. Cada ejercicio tiene una tarea espejo `Txx`; **T01-T06 son retos cortos dentro de la demostración** y **T07-T12 quedan como refuerzo autónomo**, para no exceder 120 minutos. Los scripts docentes contienen las soluciones.

---

## 2. Requisitos y preparación: primero confirma qué tienes instalado

- **SQL Server** es el motor donde residen las bases de datos; **SSMS 22** es la herramienta gráfica con la que te conectas a ese motor. Instalar SSMS no significa que Northwind haya sido instalada automáticamente.
- **Northwind** es la base de ejemplo usada en las primeras clases, con `dbo.Orders` y `dbo.Customers`. El nombre debe aparecer en **Explorador de objetos > Bases de datos** de tu instancia.
- Un archivo **`.sql`** (por ejemplo `Instala_Northwind.sql`) se abre en el editor para ejecutarlo según las instrucciones del curso; un **`.mdf`** es un archivo físico que puede adjuntarse si está completo y tienes permisos; un **`.bak`** debe restaurarse. No confundas **adjuntar** con **ejecutar un script**.

> **Regla de seguridad:** `Northwind` solo se usa en consultas `SELECT`. Toda escritura (`CREATE`, `INSERT`, `UPDATE`, `DELETE`, `CREATE INDEX`) se realiza en **`Lideratec_Repaso_S05_S07`**, una base adicional de prácticas. No ejecutes los scripts de esta guía en `master` ni en una base productiva. Los archivos se pueden leer antes de ejecutarlos; evita modificar sus nombres de base de forma automática.

### Preparación en SSMS 22 — clic por clic

1. **Abre SSMS 22** y conéctate al **Motor de base de datos** usando la misma instancia que utilizaste en las primeras clases. El nombre del servidor depende de tu instalación; `localhost` no siempre es correcto.
2. En **Explorador de objetos**, despliega **Bases de datos**. Comprueba que ves `Northwind`. Si no aparece, clic derecho en **Bases de datos > Actualizar**.
3. Haz clic en **Nueva consulta**. Desde **Archivo > Abrir > Archivo**, abre `SQL/00_Verificar_Northwind.sql`. Ejecuta con **F5**.
4. Mira la primera cuadrícula: `IdBaseNorthwind` debe **tener un número**, no `NULL`. La segunda cuadrícula debe mostrar pedidos y nombres de empresas. Si aparece un error, **no sigas** hasta verificar instancia y base.
5. Abre `SQL/01_Preparar_Laboratorio.sql`. Observa que crea `Lideratec_Repaso_S05_S07` y sus tablas `dbo.PedidosLab` y `dbo.PedidosPractica`; **no escribe en Northwind**. Ejecuta con **F5**.
6. En la salida final verifica `BaseActual = Lideratec_Repaso_S05_S07`; `FilasParaAnalisis` debe ser un número positivo y `FilasParaMantenimiento` normalmente será **0** la primera vez.
7. Actualiza **Bases de datos** y ubica `Lideratec_Repaso_S05_S07 > Tablas`. Abre los scripts en el **orden del cronograma ampliado**; los números EJ identifican una versión previa de seis ejemplos y seis incorporaciones nuevas, no el orden temporal; si cambias manualmente de base, comprueba `SELECT DB_NAME()`.
8. Antes de una prueba de rendimiento, activa **Consulta > Incluir plan de ejecución real** o presiona **Ctrl+M**. Al ejecutar con **F5**, observa **Resultados**, **Mensajes** y **Plan de ejecución**. Las **lecturas lógicas** se muestran en **Mensajes** cuando activamos `SET STATISTICS IO ON`.

**Si Northwind no está instalada:** detén la ejecución y vuelve al recurso de instalación de la primera clase o solicita el script/archivo original. Los enlaces de Microsoft SQL Server, SSMS y Attach explican la tecnología y el procedimiento, pero **no entregan por sí solos la base Northwind**.

### ¿Qué construye el archivo de preparación y por qué?

El script crea **dos tablas distintas**, ambas fuera de Northwind:

- `dbo.PedidosPractica`: tabla de tamaño reducido para entrenar `INSERT`, `UPDATE` y `DELETE` sin consecuencias sobre datos fuente.
- `dbo.PedidosLab`: copia ampliada, aproximadamente **60 veces** los pedidos válidos de la Northwind conectada. La ampliación facilita observar el plan de ejecución; **no representa sesenta ventas reales** ni permite concluir que los tiempos mejorarán en cualquier servidor.

El identificador de la copia se genera con `o.OrderID + (n.N - 1) * 100000`, de modo que cada repetición obtenga un `PedidoID` distinto. Se conserva `ClienteID`, la fecha, el costo de envío, la empresa y el país para practicar filtros. La **clave primaria agrupada** protege la unicidad de `PedidoID`. La carga se ejecuta **solo si `PedidosLab` está vacía**, de modo que volver a abrir el archivo no duplique datos.

**Ejemplo explicado del núcleo de preparación:**

```sql
;WITH Numeros AS
(
    SELECT TOP (60)
           ROW_NUMBER() OVER (ORDER BY a.object_id, b.object_id) AS N
    FROM sys.all_objects AS a
    CROSS JOIN sys.all_objects AS b
)
INSERT INTO dbo.PedidosLab
    (PedidoID, ClienteID, FechaPedido, CostoEnvio, Empresa, Pais)
SELECT o.OrderID + CAST((n.N - 1) * 100000 AS INT),
       o.CustomerID, CAST(o.OrderDate AS DATE),
       CAST(COALESCE(o.Freight, 0) AS DECIMAL(12,2)),
       c.CompanyName, c.Country
FROM Northwind.dbo.Orders AS o
INNER JOIN Northwind.dbo.Customers AS c
        ON c.CustomerID = o.CustomerID
CROSS JOIN Numeros AS n
WHERE o.OrderDate IS NOT NULL AND o.CustomerID IS NOT NULL
  AND c.Country IS NOT NULL;
```

**Explicación esencial:** `WITH Numeros` crea una expresión de tabla común (CTE) con números del 1 al 60; `ROW_NUMBER()` enumera las filas generadas, no crea tablas permanentes. `INSERT INTO dbo.PedidosLab` identifica el **destino**; `SELECT ... FROM Northwind...` identifica el **origen de lectura**. `INNER JOIN` añade los datos del cliente vinculado al pedido. `CROSS JOIN Numeros` replica cada fila del origen; la expresión que añade un múltiplo de 100000 evita que la clave primaria se repita. `COALESCE` reemplaza un costo nulo por cero solamente durante la copia. Las condiciones `IS NOT NULL` excluyen registros que no cumplen los campos obligatorios de la tabla de laboratorio.

> **Predicción antes de F5:** ¿cuál será la base que cambia? `Lideratec_Repaso_S05_S07`. ¿Cuál no cambia? `Northwind`. El número final de filas depende de la versión de Northwind que tengas; no memorices una cifra sin comprobarla.

## 3. Cronograma EXACTO de 120 minutos con 12 ejercicios

| Minuto | Sesión | Ejercicio / habilidad | Min |
|---|---|---|---:|
| 00–08 | Apertura | Objetivo, diagnóstico rápido, confirmar preparación | 8 |
| 08–15 | S05 | **EJ01:** insertar desde SELECT sin tocar Northwind | 7 |
| 15–23 | S05 | **EJ02:** UPDATE/DELETE con WHERE y @@ROWCOUNT | 8 |
| 23–30 | S05 | **EJ07:** auditoría del origen/destino tras INSERT | 7 |
| 30–38 | S05 | **EJ08:** actualización condicionada e idempotente | 8 |
| 38–46 | S06 | **EJ03:** primera vista de resumen por país | 8 |
| 46–54 | S06 | **EJ09:** vista por país y año (granularidad) | 8 |
| 54–62 | S06 | **EJ10:** conciliación de reportes vs tabla física | 8 |
| 62–70 | S06 + S07 | **EJ04:** índice y medición antes/después | 8 |
| 70–78 | S07 | **EJ05:** función YEAR vs rango, mismo resultado | 8 |
| 78–86 | S07 | **EJ11:** selectividad y pregunta de negocio | 8 |
| 86–93 | S07 | **EJ12:** auditoría de los datos simulados | 7 |
| 93–100 | Integración | **EJ06:** reporte final de tres sesiones | 7 |
| 100–114 | Evaluación | Preguntas de razonamiento + entrega de ficha | 14 |
| 114–120 | Cierre | Retroalimentación y decisión técnica | 6 |
| **Total** | | | **120** |

**Organización temporal:** los ejercicios EJ01-EJ06 contienen sus microretos T01-T06 (1-2 minutos cuando corresponda); si el grupo no alcanza a terminarlos, la plantilla queda para completar fuera de clase. Las tareas nuevas T07-T12 son ampliaciones **fuera de esos 120 min**. No se agregan minutos ocultos de laboratorio ni una evaluación al final sin tiempo reservado.

**Preparación docente antes de entrar al aula:** ejecutar y comprobar `00_Verificar_Northwind.sql` y `01_Preparar_Laboratorio.sql` en una instancia autorizada. Si deben hacerlo estudiantes durante clase, usa parte de los **8 minutos iniciales** y facilita previamente los archivos. Si preparar 60 copias toma más tiempo en la instancia, usar la base docente ya preparada y explicar qué se hizo; no simules resultados.

---

## 4. Secuencia práctica - de datos controlados a una decisión documentada

**Orden de archivos que debes abrir:** EJ01 → EJ02 → EJ07 → EJ08 → EJ03 → EJ09 → EJ10 → EJ04 (ANTES, índice, DESPUÉS) → EJ05 → EJ11 → EJ12 → EJ06. Cada bloque indica por qué se ejecuta, código nuevo, explicación, predicción y comprobación.


## EJ01 — S05 | Importar pedidos seleccionados mediante `INSERT ... SELECT` (7 min)

### A. ¿Qué problema necesitamos resolver?

Queremos analizar una lista pequeña de pedidos de clientes de Alemania. Northwind contiene los pedidos originales, pero **no debemos editarlos**. Necesitamos una **tabla de trabajo** en la que podamos copiar solamente los registros que vamos a revisar.

### B. ¿Por qué necesitamos esta operación?

`SELECT` solo devuelve información. `INSERT ... SELECT` permite **guardar** el resultado de una consulta en una tabla destino. Es útil cuando queremos crear un subconjunto para validaciones, pruebas o una etapa de procesamiento. Aquí los nombres de las columnas del `INSERT` deben corresponder en **cantidad, posición y tipo** a las columnas del `SELECT`.

### C. Archivo, ubicación y pasos exactos

**Archivo:** `SQL/EJ01_Insert_Select.sql`. Requisito previo: haber ejecutado correctamente `00` y `01`. Abre el archivo en SSMS 22; confirma que la primera instrucción es `USE Lideratec_Repaso_S05_S07;`. **Antes de F5**, identifica la tabla de destino `dbo.PedidosPractica` y la tabla de origen `Northwind.dbo.Orders`.

```sql
IF NOT EXISTS (SELECT 1 FROM dbo.PedidosPractica)
BEGIN
    INSERT INTO dbo.PedidosPractica
        (PedidoID, ClienteID, FechaPedido, CostoEnvio, Empresa, Pais)
    SELECT TOP (12)
           o.OrderID, o.CustomerID, CAST(o.OrderDate AS DATE),
           CAST(COALESCE(o.Freight, 0) AS DECIMAL(12,2)),
           c.CompanyName, c.Country
    FROM Northwind.dbo.Orders AS o
    INNER JOIN Northwind.dbo.Customers AS c
            ON c.CustomerID = o.CustomerID
    WHERE c.Country = N'Germany' AND o.OrderDate IS NOT NULL
    ORDER BY o.OrderID;
END;
```

### D. Explicación instrucción por instrucción

- `IF NOT EXISTS (SELECT 1 FROM dbo.PedidosPractica)` comprueba si la tabla de práctica está vacía. Si ya se ejecutó EJ01, **no vuelve a añadir los mismos datos**; así evitamos errores de clave duplicada y alteraciones accidentales de la demostración.
- `BEGIN ... END` delimita el conjunto de instrucciones que solo se ejecuta si se cumple la condición anterior.
- `INSERT INTO dbo.PedidosPractica (columnas...)` indica **dónde** se guardarán los datos y en qué orden. Escribir `Northwind.dbo.Orders` aquí sería un error grave: se modificaría el origen.
- `SELECT TOP (12)` limita a doce filas del conjunto que cumple el filtro. `ORDER BY o.OrderID` hace determinista la selección para una versión fija de Northwind.
- `o.OrderID`, `o.CustomerID`, `o.OrderDate` pertenecen a `Orders`. `CAST(... AS DATE)` convierte la fecha al tipo usado por `PedidosPractica`.
- `COALESCE(o.Freight, 0)` sustituye un valor nulo por cero **en el resultado copiado**; `CAST(... AS DECIMAL(12,2))` ajusta el tipo de costo.
- `INNER JOIN ... ON c.CustomerID = o.CustomerID` relaciona cada pedido con su empresa. Si omitiéramos la condición de relación, aparecerían asociaciones incorrectas o un producto cartesiano.
- `WHERE c.Country = N'Germany'` selecciona pedidos alemanes y `o.OrderDate IS NOT NULL` cumple la restricción de fecha de la tabla destino.

### E. Seguimiento del estado y predicción

**Antes:** `PedidosPractica` vacía → **se ejecuta `INSERT ... SELECT`** → **después:** hasta 12 pedidos alemanes. Si ejecutas la tarea T01 luego, ese total crecerá. Northwind permanece idéntica.

**Pregunta previa:** si elimino `TOP (12)`, ¿qué ocurriría? Se intentarían copiar **todos** los pedidos que cumplan el filtro, no solo doce. ¿Qué efecto tiene quitar `ORDER BY`? No puedes asegurar cuáles son los primeros doce.

### F. Ejecución, resultados y comprobación

Pulsa **F5** y localiza `FilasAntes`, `FilasDespues` y las cinco filas de ejemplo que ya incluye el script. En Northwind estándar habrá suficientes pedidos de Alemania para llegar a **12** en el primer intento; si tu copia es distinta, el máximo es 12. Comprueba también que `SELECT DB_NAME()` es la base de práctica.

### G. Error controlado

**Problema habitual:** `Cannot insert duplicate key` al copiar nuevamente IDs iguales. **Causa:** repetir el `INSERT` sin condición de control. **Solución:** conservar el `IF NOT EXISTS` o reiniciar únicamente la tabla de práctica con el script de reinicio, después de advertir que ese script borra los cambios del laboratorio. No borres datos de Northwind.

### H. Tarea espejo T01

**Enunciado:** importar **cinco** pedidos de clientes de **Francia** a `dbo.PedidosPractica`, evitando repetir la inserción si ya hay pedidos franceses. **Archivo:** `SQL/T01_Alumno.sql`. Usa `INSERT ... SELECT`, `TOP (5)`, `INNER JOIN`, `WHERE` y `ORDER BY`. **Restricción:** no tocar `Northwind`. **Pista:** sustituye `Germany` por `France` y protege la operación con `IF NOT EXISTS`. **Criterio:** primera ejecución agrega hasta cinco filas y una segunda no agrega nuevas filas. **No consultes el solucionario docente antes de resolverla.**

### I. Conexión

Ya sabemos copiar registros, pero todavía necesitamos **corregir o retirar** un dato de práctica sin afectar a los demás: eso veremos en EJ02.

---

## EJ02 — S05 | Actualizar y eliminar exactamente el registro deseado (8 min)

### A. Problema y propósito

Una empresa detecta que el costo de envío de un pedido de prueba es incorrecto. Necesitamos editarlo y, cuando termine la validación, retirarlo sin afectar otros pedidos. Para poder verificar el proceso, **crearemos primero un registro identificable**.

### B. ¿Por qué estos componentes?

`INSERT ... VALUES` crea una fila inventada solo para el ejercicio; `UPDATE ... WHERE` cambia un dato de esa fila; `DELETE ... WHERE` elimina esa misma fila. `@@ROWCOUNT` permite comprobar cuántas filas afectó el comando anterior. **Sin `WHERE`, un `UPDATE` o `DELETE` puede afectar toda la tabla.**

### C. Archivo y ejecución

**Archivo:** `SQL/EJ02_Update_Delete.sql`. Se ejecuta en `Lideratec_Repaso_S05_S07`, tabla `dbo.PedidosPractica`. Antes de F5, localiza y señala los tres `WHERE PedidoID = 90000001`.

```sql
IF NOT EXISTS (SELECT 1 FROM dbo.PedidosPractica WHERE PedidoID = 90000001)
    INSERT INTO dbo.PedidosPractica
        (PedidoID, ClienteID, FechaPedido, CostoEnvio, Empresa, Pais)
    VALUES (90000001, N'PRB01', '20261001', 10.00,
            N'Cliente de prueba', N'Germany');

UPDATE dbo.PedidosPractica
   SET CostoEnvio = 25.00
 WHERE PedidoID = 90000001;
SELECT @@ROWCOUNT AS FilasActualizadas;

DELETE FROM dbo.PedidosPractica WHERE PedidoID = 90000001;
SELECT @@ROWCOUNT AS FilasEliminadas;
```

### D. Explicación línea por línea y cambio de estado

- `IF NOT EXISTS ... WHERE PedidoID = 90000001`: evita crear dos registros con la misma clave de prueba.
- `INSERT ... VALUES (...)`: escribe directamente los seis valores, en el mismo orden de las columnas. El ID **90000001** no proviene de Northwind; está reservado para esta demostración.
- `UPDATE dbo.PedidosPractica`: señala la tabla que cambia. `SET CostoEnvio = 25.00` reemplaza el valor previo **10.00** por **25.00**; **no** suma 25 al anterior.
- `WHERE PedidoID = 90000001`: hace que el cambio alcance solo la fila identificada. Si lo omites, afectarías todas las filas de `PedidosPractica`.
- `SELECT @@ROWCOUNT AS FilasActualizadas`: recoge inmediatamente la cantidad afectada por **el `UPDATE` anterior**. Si intercalas otra sentencia antes, ya no estarías midiendo ese `UPDATE`.
- `DELETE ... WHERE PedidoID = 90000001`: retira exclusivamente la fila creada para la prueba; un `SELECT COUNT(*) ...` final debe mostrar **cero** filas con ese ID.

### E. Predicción, comprobación y error controlado

**Estados:** *no existe 90000001* → *INSERT: cuesta 10* → *UPDATE: cuesta 25* → *DELETE: ya no existe*. **Predice:** ¿cuántas filas se actualizan? 1. ¿Cuántas se borran? 1. Si el archivo se ejecuta completo sin errores y sin cambios externos, `FilasActualizadas = 1`, `FilasEliminadas = 1`, `RestantesDelIDPrueba = 0`.

**Error a discutir, NO ejecutar:** `DELETE FROM dbo.PedidosPractica;` eliminaría **todas las filas** de práctica. La corrección consiste en añadir un `WHERE` específico y **verificar el ID mediante un `SELECT` antes de modificar**. Nunca uses este ejemplo sobre Northwind.

### F. Tarea espejo T02

**Enunciado:** repetir la operación sobre `PedidoID = 90000002` y `ClienteID = PRB02`. Insertar envío **8.00**, actualizarlo a **12.00** y eliminar solo ese ID. **Archivo:** `SQL/T02_Alumno.sql`. **Criterios:** `@@ROWCOUNT` debe ser 1 tras actualizar y eliminar, y la comprobación final para el ID debe ser 0. **Restricción:** usar `WHERE` en ambos cambios.

### G. Conexión

Sabemos mantener filas, pero todavía estamos consultando detalles de forma repetida. En EJ07 auditaremos primero la muestra y luego convertiremos una consulta de resumen en un objeto reutilizable para otros usuarios.

---

## EJ07 — S05 | Diagnosticar si la carga realmente es correcta (7 min)

### A. Problema y propósito

No basta con que `INSERT ... SELECT` termine sin errores: podríamos haber copiado pedidos del país equivocado, más filas de las previstas o haber repetido un identificador. El área comercial necesita una **comprobación objetiva** antes de usar esa muestra en un análisis. En EJ01 copiamos pedidos de Alemania y quizá resolvimos T01 con Francia. **Ahora auditaremos esa tabla sin alterar una sola fila.**

**Objetivo observable:** demostrar el tamaño de la muestra, la unicidad de sus IDs y la distribución por país; explicar cuándo un conteo no coincide con lo esperado.

### B. Abrir el archivo y anticipar el resultado

1. En SSMS 22 abre `SQL/EJ07_Control_Calidad_Importacion.sql` con **Archivo > Abrir > Archivo**. Comprueba que la primera instrucción es `USE Lideratec_Repaso_S05_S07`.
2. Antes de ejecutar, predice: si EJ01 se ejecutó una sola vez, la tabla tendrá **hasta 12** pedidos. Si realizaste T01, puede haber hasta cinco de Francia; si hiciste ya EJ02, el registro ficticio `90000001` se habrá eliminado.
3. Comprueba que ambas consultas son `SELECT`: este ejemplo **no modifica registros**.

```sql
SELECT COUNT_BIG(*) AS TotalImportado,
       COUNT(DISTINCT PedidoID) AS IdsDiferentes,
       MIN(FechaPedido) AS PrimeraFecha,
       MAX(FechaPedido) AS UltimaFecha
FROM dbo.PedidosPractica;

SELECT Pais, COUNT_BIG(*) AS Pedidos,
       CAST(SUM(CostoEnvio) AS DECIMAL(18,2)) AS CostosEnvio
FROM dbo.PedidosPractica
GROUP BY Pais
ORDER BY Pedidos DESC, Pais;
```

### C. ¿Qué significa cada instrucción y qué evidencia produce?

- `COUNT_BIG(*)` cuenta filas sin cambiar la tabla y devuelve un entero grande. **Razón:** necesitamos el volumen exacto para comparar con el límite `TOP (12)` del primer ejercicio.
- `COUNT(DISTINCT PedidoID)` cuenta **identificadores distintos**, no filas distintas. **Razón:** si el total y este conteo no coincidieran, habría un problema de duplicados. En nuestra tabla la `PRIMARY KEY` impide IDs repetidos; por eso deberían coincidir.
- `MIN(FechaPedido)` y `MAX(FechaPedido)` indican el intervalo temporal de la muestra. **Razón:** un reporte sobre envíos debe saber qué período representa.
- `GROUP BY Pais` crea un grupo por país; `SUM(CostoEnvio)` calcula la suma dentro de cada grupo. **Razón:** dos mercados no deben mezclarse al construir un diagnóstico comercial.
- `ORDER BY Pedidos DESC, Pais` muestra primero el país con más pedidos y resuelve empates por nombre.

### D. Estado, ejecución y diagnóstico de error

**Antes y después:** la tabla contiene las mismas filas; solo aparece información resumida en **Resultados**. Pulsa **F5** y anota `TotalImportado`, `IdsDiferentes` y los países devueltos. El total dependerá de las tareas que hayas ejecutado: **no hay un valor universal fuera de un laboratorio reiniciado**.

**Error controlado:** si `COUNT(DISTINCT PedidoID)` es menor que `COUNT_BIG(*)` en una tabla sin clave, sospecharíamos duplicados; **aquí** eso no puede suceder mientras la PK esté activa. Si el país que esperabas no aparece, revisa el `WHERE` de EJ01/T01 antes de volver a insertar datos.

### E. Tarea espejo T07 y conexión

**T07 (práctica autónoma):** escribe una sola consulta que muestre por país la cantidad y suma de costos, ordenando de mayor a menor cantidad. Archivo: `SQL/T07_Alumno.sql`. **Aceptación:** una fila por país, sumas numéricas y ningún `INSERT`, `UPDATE` o `DELETE`. La solución razonada queda reservada al docente.

**Transición:** si el análisis encuentra un costo de envío que no satisface una regla de negocio, no basta con conocer el dato: debemos ajustarlo **solo si cumple una condición**, y demostrar que repetir el ajuste no lo acumula.

---

## EJ08 — S05 | Aplicar una regla de actualización idempotente (8 min)

### A. Problema y objetivo

Una organización establece para una **fila ficticia de prueba** que el costo de envío mínimo será **25.00**. Si el costo registrado es 20.00 debemos corregirlo; si ya vale 25.00, ejecutar otra vez el script no debe volver a cambiarlo. Esta vez el `WHERE` protege no solo el ID, sino también la **condición comercial**.

**Habilidad nueva:** formular un `UPDATE` condicionado por **dos predicados**, registrar las filas alcanzadas y explicar por qué una segunda ejecución puede afectar cero filas.

### B. Pasos en SSMS y código

1. Abre `SQL/EJ08_Regla_Actualizacion_Condicionada.sql`; constata el nombre de la base. No afecta las filas copiadas de Northwind: utiliza el ID ficticio `90000003`.
2. Antes de F5, señala qué fila creará el `INSERT` y cuál es la condición de cambio. Predice el resultado de **dos ejecuciones sucesivas**.
3. Ejecuta **F5**; consulta las cuadrículas **Antes**, `FilasRealmenteAfectadasPorPredicado` y **Despues**. Ejecuta **F5** una segunda vez y compara el contador.

```sql
IF NOT EXISTS (SELECT 1 FROM dbo.PedidosPractica WHERE PedidoID = 90000003)
BEGIN
    INSERT INTO dbo.PedidosPractica
        (PedidoID, ClienteID, FechaPedido, CostoEnvio, Empresa, Pais)
    VALUES (90000003, N'PRB03', '20261001', 20.00,
            N'Pedido de demostracion', N'Canada');
END;

UPDATE dbo.PedidosPractica
   SET CostoEnvio = 25.00
 WHERE PedidoID = 90000003
   AND CostoEnvio < 25.00;
SELECT @@ROWCOUNT AS FilasRealmenteAfectadasPorPredicado;
```

### C. Análisis causal de las instrucciones

- `IF NOT EXISTS ... PedidoID = 90000003` impide insertar dos veces la misma fila de prueba. Sin esta protección, la clave primaria podría rechazar la segunda inserción.
- `VALUES (..., 20.00, ..., N'France')` establece el estado **inicial**, un costo menor al umbral. Esta fila se agrega **solo a `PedidosPractica`**.
- `UPDATE ... SET CostoEnvio = 25.00` asigna el mínimo establecido; **no suma** 25.00 al valor anterior.
- `WHERE PedidoID = 90000003 AND CostoEnvio < 25.00` combina identificación **y** regla de negocio. Si ya vale 25.00, la fila queda fuera del conjunto que se actualiza.
- `@@ROWCOUNT` se consulta inmediatamente después del `UPDATE` para informar cuántas filas coincidieron con el predicado del cambio. No lo confundas con un contador universal de diferencias entre valores viejos y nuevos.

### D. Traza, resultado y error que debemos evitar

**Ejecución 1:** no existe ID → `INSERT` lo crea con 20.00 → `UPDATE` lo lleva a 25.00 → contador de actualización esperado: **1**. **Ejecución 2:** ID ya existe en 25.00 → `INSERT` se omite → condición `CostoEnvio < 25` es falsa → contador esperado: **0**. Comprobarlo es el propósito del ejercicio.

**Error a discutir, no ejecutar:** `UPDATE dbo.PedidosPractica SET CostoEnvio = CostoEnvio + 5;` modificaría todas las filas y volvería a sumar en cada ejecución. La solución es restringir el `WHERE` y asignar un valor coherente con la regla, no aplicar un incremento ilimitado.

### E. Tarea espejo T08 y conexión

**T08 (autónoma):** crea el ID `90000004` con costo 20.00, país Germany, y ajústalo a **28.00** solo si es menor; acredita qué ocurre al repetir. Archivo: `SQL/T08_Alumno.sql`. **Criterio:** en ejecución 1 alcanza una fila y en ejecución 2 ninguna, sin cambiar las filas de otros pedidos. La solución está en el archivo docente.

**Transición:** mantuvimos datos de manera segura. En las siguientes actividades crearemos **reportes reutilizables** sobre `PedidosLab` y verificaremos que sus totales sean confiables.

---

## EJ03 — S06 | Crear una vista para consultar pedidos por país (8 min)

### A. Problema

El equipo comercial solicita con frecuencia **total de pedidos** y **suma de costos de envío** por país. Reescribir `GROUP BY` en cada consulta genera repetición y riesgo de inconsistencias. Una **vista** permite reutilizar la definición.

### B. ¿Por qué una vista?

`dbo.vw_PedidosPorPais` es una consulta almacenada como objeto de SQL Server; no es una segunda tabla con datos copiados. Cuando consultas la vista, el motor obtiene los datos de `dbo.PedidosLab` de acuerdo con su definición. **La vista no garantiza por sí misma mayor velocidad:** su beneficio principal en este ejemplo es la reutilización lógica.

### C. Archivo, pasos y código

**Archivo:** `SQL/EJ03_Vista.sql`. Haz clic en **Nueva consulta**, abre el archivo y confirma que `USE Lideratec_Repaso_S05_S07` está antes de `GO`. **Importante:** `CREATE OR ALTER VIEW` debe estar al inicio de su lote; por eso incluimos `GO`.

```sql
CREATE OR ALTER VIEW dbo.vw_PedidosPorPais
AS
SELECT Pais,
       COUNT_BIG(*) AS TotalPedidos,
       CAST(SUM(CostoEnvio) AS DECIMAL(18,2)) AS TotalEnvios
FROM dbo.PedidosLab
GROUP BY Pais;
GO
SELECT Pais, TotalPedidos, TotalEnvios
FROM dbo.vw_PedidosPorPais
WHERE Pais IN (N'Germany', N'France')
ORDER BY Pais;
```

### D. Explicación instrucción por instrucción

- `CREATE OR ALTER VIEW dbo.vw_PedidosPorPais`: crea la vista si no existe o actualiza su definición si ya existe. **No duplica pedidos**.
- `AS SELECT Pais`: comienza la consulta que define los datos virtuales de la vista.
- `COUNT_BIG(*) AS TotalPedidos`: cuenta filas de la tabla por país, devolviendo un entero grande. Si elimináramos `GROUP BY`, el conjunto no estaría correctamente agrupado por país.
- `SUM(CostoEnvio)` suma costos; `CAST(... AS DECIMAL(18,2))` presenta el agregado con dos decimales.
- `FROM dbo.PedidosLab` refiere a la **tabla ampliada de análisis**, no a `PedidosPractica` ni a Northwind. `GROUP BY Pais` produce una fila por país.
- `GO` cierra el lote de creación. El `SELECT` posterior **consume** la vista como cualquier otro origen de consulta.
- `WHERE ... IN` filtra dos países y `ORDER BY` ordena el resultado de la consulta externa. **No colocamos `ORDER BY` como regla de presentación dentro de la vista.**

### E. Predicción, comprobación y error

**Predicción:** ¿cuántas filas devuelve el `SELECT` final? Hasta **dos** (Alemania y Francia), si ambos países están en la fuente. ¿Se habrán duplicado físicamente los datos? **No.**

**Comprobación:** ejecuta **F5** y compara `TotalPedidos` de la vista con la consulta `SELECT Pais, COUNT_BIG(*) ... FROM dbo.PedidosLab GROUP BY Pais` incluida al final del archivo: ambos conteos deben coincidir para cada país.

**Error controlado:** si aparece `CREATE VIEW must be the first statement in a query batch`, no significa que falte Northwind; normalmente se olvidó el `GO` después de `USE`. Inserta el separador y vuelve a ejecutar.

### F. Tarea espejo T03

**Enunciado:** crea `dbo.vw_EnviosPorEmpresa`, que devuelva `Empresa`, `TotalPedidos` y `TotalEnvios`. Después consulta las **cinco empresas con más pedidos**, ordenadas de mayor a menor. **Archivo:** `SQL/T03_Alumno.sql`. **Restricción:** `ORDER BY` se usa en el `SELECT` externo. **Criterio:** definición ejecutable, consulta que devuelve hasta cinco empresas y totales consistentes.

### G. Conexión

La vista soluciona la reutilización, pero una consulta a muchos registros todavía puede requerir examinar demasiadas páginas. En EJ09 añadiremos la dimensión anual, validaremos totales y en EJ04 observaremos cómo un índice cambia las posibilidades de acceso.

---

## EJ09 — S06 | Cambiar la granularidad de un reporte con otra vista (8 min)

### A. Necesidad del negocio

La vista de EJ03 resume pedidos **por país**, pero el área gerencial pregunta: *¿en qué año se concentraron los pedidos de Alemania y Francia?* La vista actual no puede responderlo porque perdió la dimensión temporal al agrupar. **Necesitamos una nueva definición que agrupe por país y por año**, sin duplicar físicamente la tabla de pedidos.

### B. Crear la vista paso a paso

1. Abre `SQL/EJ09_Vista_Pais_Anio.sql`. Comprueba que `dbo.vw_PedidosPorPais` ya existe de EJ03; la nueva vista tendrá **otro nombre** y no reemplazará a la anterior.
2. Lee la diferencia central antes de ejecutar: hay **dos expresiones en `GROUP BY`**, una por cada dimensión del análisis.
3. Pulsa **F5**, actualiza **Vistas** en el Explorador de objetos si no aparece, y observa los años disponibles para Germany y France.

```sql
CREATE OR ALTER VIEW dbo.vw_PedidosPorPaisAnio
AS
SELECT Pais,
       YEAR(FechaPedido) AS Anio,
       COUNT_BIG(*) AS TotalPedidos,
       CAST(SUM(CostoEnvio) AS DECIMAL(18,2)) AS TotalEnvios
FROM dbo.PedidosLab
GROUP BY Pais, YEAR(FechaPedido);
GO
SELECT Pais, Anio, TotalPedidos, TotalEnvios
FROM dbo.vw_PedidosPorPaisAnio
WHERE Pais IN (N'Germany', N'France')
ORDER BY Pais, Anio;
```

### C. Significado de cada bloque

- `CREATE OR ALTER VIEW` guarda la **definición**, no una copia materializada de filas de la tabla de laboratorio.
- `YEAR(FechaPedido) AS Anio` extrae el año de cada fecha y le asigna un nombre a la columna de resultado. Es apropiado aquí para **agrupar**, aunque en EJ05 veremos por qué aplicarlo en `WHERE` puede perjudicar un acceso por rango.
- `COUNT_BIG` y `SUM` responden dos preguntas distintas: cantidad de pedidos y costo total de envío.
- `GROUP BY Pais, YEAR(FechaPedido)` crea una fila por **combinación país-año**; omitir el año mezclaría períodos, omitir el país mezclaría mercados.
- `GO` separa el lote `CREATE VIEW` del `SELECT` que la utiliza. `WHERE Pais IN (...)` filtra solo la salida solicitada; no modifica la tabla.

### D. Resultado y caso de error

**Predicción:** una sola fila de EJ03 para Germany puede convertirse en varias (una por año) en EJ09, porque el reporte es **más detallado**. El total de Germany sumado entre años debe ser el mismo que su total en `vw_PedidosPorPais` mientras ambas vistas consulten `PedidosLab` sin cambios. **No inventes cantidades**: revisa la pestaña Resultados.

**Error común:** `Column 'FechaPedido' is invalid in the select list because it is not contained in ... GROUP BY`. Se produce al seleccionar una columna no agregada que no coincide con las expresiones agrupadas. Aquí seleccionamos `YEAR(FechaPedido)` y agrupamos por esa misma expresión.

### E. T09 y conexión

**T09 (autónoma):** con la vista nueva determina el año con más pedidos de Francia, ordenando por cantidad y, en caso de empate, año. **Criterio:** devuelve una sola fila explicando qué representa; archivo `SQL/T09_Alumno.sql`.

**Transición:** hemos creado dos reportes con granularidades diferentes. En EJ10 demostraremos que **no se contradicen** al sumar los mismos pedidos.

---

## EJ10 — S06 | Conciliar cifras entre dos vistas y la tabla base (8 min)

### A. Problema y propósito

Cuando dos cuadros de mando entregan cifras distintas, los usuarios pierden confianza. Tenemos un resumen por país (EJ03), otro por país-año (EJ09) y una tabla física de la cual ambos dependen. La evidencia de calidad será que **las sumas de pedidos de los reportes coincidan con la cantidad de filas de la tabla base**.

### B. Guía de ejecución

1. Abre `SQL/EJ10_Conciliacion_Vista_Tabla.sql`. Se requieren las vistas creadas en EJ03 y EJ09.
2. Antes de F5, escribe qué significa cada una de las tres columnas de la primera consulta. Predice **igualdad de totales**, pero no una cifra fija.
3. Ejecuta **F5**. Comprueba primero la igualdad y después el ranking de cinco países. Si difiere, detén las conclusiones y verifica la base/vistas.

```sql
SELECT (SELECT SUM(TotalPedidos) FROM dbo.vw_PedidosPorPais)
           AS SumaVistaPais,
       (SELECT COUNT_BIG(*) FROM dbo.PedidosLab)
           AS FilasTabla,
       (SELECT SUM(TotalPedidos) FROM dbo.vw_PedidosPorPaisAnio)
           AS SumaVistaPaisAnio;

SELECT TOP (5) Pais, TotalPedidos, TotalEnvios
FROM dbo.vw_PedidosPorPais
ORDER BY TotalEnvios DESC, Pais ASC;
```

### C. Interpretar las líneas

- Las tres subconsultas escalares devuelven **un valor por expresión**. Podemos colocarlas una al lado de otra para revisar la consistencia rápidamente.
- `SUM(TotalPedidos)` acumula cantidades que ya fueron agrupadas; es válido porque cada pedido pertenece a **un país y un año** de la muestra.
- `COUNT_BIG(*) FROM dbo.PedidosLab` mide las filas físicas, el **patrón de referencia** contra el que conciliaremos.
- `TOP (5) ... ORDER BY TotalEnvios DESC` cambia el punto de vista: ahora priorizamos mercados con mayor costo de envío. **El ranking no valida integridad por sí solo**; la primera consulta sí.

### D. Resultado esperado, predicción y error

**Estado:** sin cambios de datos ni de vistas. **Esperado:** `SumaVistaPais = FilasTabla = SumaVistaPaisAnio`, con números que dependen de tu Northwind y del laboratorio. **Error controlado:** comparar una vista que incluye todos los años con otra que filtra 1997 generaría diferencias **explicables por el filtro**. Antes de concluir que hay un defecto, confirma que comparas el mismo conjunto de filas.

### E. T10 y conexión

**T10 (autónoma):** valida solo Alemania, comparando `TotalPedidos` de `vw_PedidosPorPais` con `COUNT_BIG(*)` de `PedidosLab WHERE Pais = N'Germany'`. **Criterio:** ambos valores coinciden y puedes explicar el porqué. Archivo: `SQL/T10_Alumno.sql`.

**Transición:** una cifra correcta no demuestra que la consulta sea eficiente. En EJ04 introduciremos un índice para medir la estrategia de acceso.

---

## EJ04 — S06 + S07 | Justificar un índice comparando «antes y después» (8 min)

### A. Problema

Se consulta con frecuencia el historial de un cliente (`ALFKI`) dentro de un año. Nuestra `PedidosLab` contiene muchas filas derivadas de Northwind; queremos saber si un índice sobre **cliente y fecha** ayuda a resolver esa búsqueda. **Un índice es una estructura física adicional** que facilita ciertos accesos, pero también ocupa espacio y requiere mantenimiento.

### B. Por qué esas columnas y no otras

El predicado usa `ClienteID = valor` y un **rango** en `FechaPedido`. Por eso se propone un índice cuyo orden de claves es `(ClienteID, FechaPedido)`. Las columnas `CostoEnvio` y `Empresa` se incluyen para posibilitar una consulta cubierta; no forman parte de las claves de búsqueda. Esto no obliga al optimizador a usarlo: la decisión depende de la cardinalidad y de las estadísticas.

### C. Pasos: nunca mezclar la evidencia previa y posterior

1. Antes de crear nada, asegúrate de que no exista `IX_PedidosLab_Cliente_Fecha` en `dbo.PedidosLab`. Si ya practicaste, lee `SQL/02_Reiniciar_Solo_Demostracion.sql`: **borra datos de la tabla de práctica y elimina objetos docentes**; ejecútalo solo si realmente vas a comenzar de cero.
2. Pulsa **Ctrl+M** para incluir el **plan real**.
3. Abre **`SQL/EJ04_Antes.sql`**, ejecuta **F5**, guarda una captura del **plan**, el número de filas y las lecturas lógicas de `PedidosLab` en **Mensajes**.
4. Abre **`SQL/EJ04_Crear_Indice.sql`** y ejecuta. Lee el mensaje de finalización.
5. Abre **`SQL/EJ04_Despues.sql`**; ejecuta con Ctrl+M activado. Registra la misma información y compárala con la captura previa.

**Consulta base (misma antes y después):**

```sql
SET STATISTICS IO ON;
SELECT PedidoID, ClienteID, FechaPedido, CostoEnvio, Empresa
FROM dbo.PedidosLab
WHERE ClienteID = N'ALFKI'
  AND FechaPedido >= '19970101'
  AND FechaPedido <  '19980101'
OPTION (RECOMPILE);
SET STATISTICS IO OFF;
```

**Creación del índice (solo después de registrar «ANTES»):**

```sql
CREATE NONCLUSTERED INDEX IX_PedidosLab_Cliente_Fecha
ON dbo.PedidosLab (ClienteID, FechaPedido)
INCLUDE (CostoEnvio, Empresa);
```

El archivo ejecutable añade `IF NOT EXISTS` para no intentar crear el mismo índice dos veces.

### D. Explicación técnica de cada elemento

- `SET STATISTICS IO ON`: ordena al motor presentar estadísticas de acceso a páginas en la pestaña **Mensajes**. No cambia el contenido de la tabla.
- `WHERE ClienteID = ...`: igualdad sobre la primera columna clave propuesta. `FechaPedido >= ... AND < ...` fija un intervalo **incluyente al inicio y excluyente al final**; así obtenemos el año completo sin depender de horas.
- `OPTION (RECOMPILE)`: solicita un plan de la consulta para esa ejecución; se usa aquí para mejorar la comparabilidad de la demostración y no como recomendación universal de producción.
- `CREATE NONCLUSTERED INDEX`: añade un índice secundario; **no convierte** `FechaPedido` en una nueva clave primaria. `INCLUDE` agrega columnas para devolver datos sin tener que buscarlos necesariamente en la tabla principal.
- **Plan real:** observa, por ejemplo, `Clustered Index Scan` o `Index Seek`, pero **no supongas** que uno aparecerá siempre: el optimizador elige según datos y versión del motor.
- **Lecturas lógicas:** aproximan cuántas páginas de datos fueron accedidas en caché; una cifra menor para el mismo resultado puede apoyar la decisión, aunque también debes considerar el costo de mantener el índice.

### E. Predicción, estado, resultado y error

**Predicción antes del índice:** ¿el motor podrá llegar directamente a cliente+fecha mediante el índice propuesto? Todavía no, porque ese índice **no existe**. **Estado después:** `dbo.PedidosLab` conserva sus filas y gana `IX_PedidosLab_Cliente_Fecha` como estructura física adicional.

**Comprobación obligatoria:** el **conjunto de resultados** antes y después debe ser el mismo. Anota tus mediciones en esta ficha, que **no tiene valores prefijados**:

| Variable | Antes | Después |
|---|---|---|
| Filas devueltas | ____ | ____ |
| Operador de acceso predominante | ____ | ____ |
| Lecturas lógicas de `PedidosLab` | ____ | ____ |
| ¿Mismo resultado? | — | Sí / No |

**Error frecuente:** crear el índice antes de guardar el plan «ANTES». **Corrección:** repetir el escenario en la base de laboratorio tras leer el script de reinicio, guardando ambas evidencias en el orden correcto. **No elimines índices de Northwind.**

### F. Tarea espejo T04

**Enunciado:** repite la consulta con `ClienteID = N'BERGS'`, mismo rango anual; antes de ejecutar predice si el índice puede ayudar. **Archivo:** `SQL/T04_Alumno.sql`. **Criterio:** consulta correcta y registro de operador, filas y lecturas; conclusión coherente aunque el plan no cambie.

### G. Conexión

El índice ya existe, pero hay consultas que bloquean parte de sus ventajas al transformar una columna dentro del filtro. Esa distinción será el núcleo de EJ05.

---

## EJ05 — S07 | Dos filtros correctos: ¿cuál favorece la búsqueda? (8 min)

### A. Problema

Dos consultas pueden mostrar **exactamente el mismo resultado** y, aun así, provocar diferentes accesos físicos. Compararemos el filtro `YEAR(FechaPedido) = 1997` con un rango sobre `FechaPedido`. La capacidad de aprovechar el índice es parte de las buenas prácticas de consulta (**SARGabilidad**).

### B. Por qué comparar ambas formas

`YEAR(FechaPedido)` aplica una función a cada valor de la columna. Un predicado de rango directo suele ser más adecuado para utilizar la segunda columna de un índice `(ClienteID, FechaPedido)`. No afirmaremos que cualquier ejecución se acelera, sino que **observaremos** qué elige el optimizador de tu instancia.

### C. Archivo y código

**Archivo:** `SQL/EJ05_Sargabilidad.sql`. Asegúrate de haber ejecutado EJ04 y de tener habilitado **Ctrl+M**. Este archivo ejecuta dos consultas y coloca sus lecturas en **Mensajes**.

```sql
-- Consulta A: función sobre la columna.
SELECT COUNT_BIG(*) AS CantidadA
FROM dbo.PedidosLab
WHERE ClienteID = N'ALFKI'
  AND YEAR(FechaPedido) = 1997
OPTION (RECOMPILE);

-- Consulta B: rango directo equivalente para 1997.
SELECT COUNT_BIG(*) AS CantidadB
FROM dbo.PedidosLab
WHERE ClienteID = N'ALFKI'
  AND FechaPedido >= '19970101'
  AND FechaPedido <  '19980101'
OPTION (RECOMPILE);
```

### D. Explicación detallada

- `COUNT_BIG(*)` hace comparable el número de registros que satisface cada condición.
- `ClienteID = N'ALFKI'` mantiene constante el cliente para aislar la forma del filtro de fecha.
- `YEAR(FechaPedido) = 1997` obtiene el año desde cada fecha para comparar; la función puede dificultar el uso del rango de fecha del índice.
- `FechaPedido >= '19970101'` incluye el primer día de 1997; `FechaPedido < '19980101'` excluye el primero de 1998 y todos los años posteriores. El rango funciona también si la columna fuera `datetime` con horas (aunque aquí usamos `DATE`).
- `SET STATISTICS IO ON` aparece en el script completo, antes de ambos `SELECT`; **lee por separado** los mensajes correspondientes a cada consulta.

### E. Seguimiento, predicción, validación y error

**Estado de los datos:** ambas consultas son `SELECT`; ninguna cambia las filas. **Predice:** `CantidadA` y `CantidadB` deben ser **iguales** si los datos no cambian entre consultas. ¿Y sus lecturas? Es posible que difieran: **hay que medirlo**.

Ejecuta **F5** y mira las dos pestañas de plan asociadas. Comprueba resultados iguales, compara `Index Seek`, rangos de búsqueda, posibles predicados residuales y lecturas lógicas. Si ambos planes coinciden o la mejora no es visible, **documenta esa observación**: tamaño, selectividad y estadísticas pueden hacer que el optimizador elija alternativas parecidas.

**Error frecuente:** filtrar un año con `BETWEEN '19970101' AND '19971231'` cuando la columna es `datetime`: podría excluir horas posteriores a medianoche del último día. **Corrección recomendada:** límite superior abierto `< '19980101'`.

### F. Tarea espejo T05

**Enunciado:** repite A y B para el año **1996**, manteniendo cliente `ALFKI`. **Archivo:** `SQL/T05_Alumno.sql`. **Pista:** rango desde `'19960101'` inclusive hasta `'19970101'` exclusivo. **Criterio:** conteos iguales y comparación argumentada de los planes/lecturas; no inventar medidas.

### G. Conexión

Ya sabemos qué modificar, cómo presentar resultados y qué observar al consultar. Primero analizaremos la selectividad y la calidad de los datos; después demostraremos que esas habilidades forman **un solo procedimiento técnico** comprensible para otro analista.

---

## EJ11 — S07 | Selectividad: no todas las preguntas requieren el mismo acceso (8 min)

### A. ¿Qué queremos decidir?

El equipo comercial consulta el historial completo de un cliente y también sus pedidos de un año particular. La pregunta anual es **más selectiva**: añade un filtro de fecha. Queremos observar si esa diferencia modifica el número de filas y las lecturas. **Las dos consultas no son equivalentes**; por eso no deben utilizarse como demostración automática de que una versión es «mejor» que otra.

### B. Archivo y pasos SSMS 22

1. Verifica que ejecutaste EJ04: el índice `IX_PedidosLab_Cliente_Fecha` debe existir.
2. Abre `SQL/EJ11_Selectividad_Indice.sql`. Activa **Ctrl+M** para obtener **planes de ejecución reales** y asegúrate de que el script incluye `SET STATISTICS IO ON`.
3. Antes de ejecutar, predice: ¿puede la consulta de un año devolver más filas que la consulta de todos los años? **No**, porque añade una restricción.
4. Ejecuta **F5**. Registra resultados, operador predominante de cada plan y mensajes de lecturas correspondientes a cada consulta.

```sql
SET STATISTICS IO ON;
SELECT COUNT_BIG(*) AS PedidosClienteTodosLosAnios
FROM dbo.PedidosLab
WHERE ClienteID = N'ALFKI'
OPTION (RECOMPILE);

SELECT COUNT_BIG(*) AS PedidosClienteEn1997
FROM dbo.PedidosLab
WHERE ClienteID = N'ALFKI'
  AND FechaPedido >= '19970101'
  AND FechaPedido < '19980101'
OPTION (RECOMPILE);
SET STATISTICS IO OFF;
```

### C. ¿Por qué usamos cada instrucción?

- La primera consulta responde **cuántos pedidos tiene ALFKI en todos los años**. La segunda responde **cuántos tuvo durante 1997**. No deben tener necesariamente el mismo resultado.
- `COUNT_BIG(*)` evita trasladar miles de filas al panel Resultados y nos permite concentrarnos en el conteo.
- `FechaPedido >= ... AND < ...` expresa un intervalo completo, con límite inferior incluido y superior excluido.
- `STATISTICS IO ON` hace visibles las lecturas lógicas; `OPTION (RECOMPILE)` solicita un plan específico para esta ejecución. Ni una ni otra instrucción modifica filas.
- La clave del índice `(ClienteID, FechaPedido)` coincide conceptualmente con igualdad por cliente y, cuando existe, rango por fecha. **SQL Server podría elegir distintos operadores** según estadísticas y distribución.

### D. Registro de pruebas y error conceptual

| Medida que anotarás | Todos los años | Solo 1997 |
|---|---|---|
| Filas contabilizadas | ____ | ____ |
| Operador en el plan real | ____ | ____ |
| Lecturas lógicas en `PedidosLab` | ____ | ____ |
| Pregunta de negocio que responde | Historial completo | Historial anual |

**Error habitual:** «El segundo `COUNT` es menor, por lo tanto mi SQL es más eficiente». Es un razonamiento incompleto: **ambas consultas responden requisitos diferentes**. La decisión sobre eficiencia exige examinar además plan, lecturas y costo de mantener índices.

### E. T11 y conexión

**T11 (autónoma):** repite el contraste usando `BERGS`, registra las lecturas y explica la diferencia entre **menor conjunto de salida** y **mejor acceso físico**. Archivo `SQL/T11_Alumno.sql`. **Criterio:** conteo anual no supera al total y conclusión basada en datos observados.

**Transición:** incluso con buenos planes, las pruebas podrían inducir a error si su volumen de datos es artificial y eso no se documenta. Ahora verificaremos la procedencia de la muestra.

---

## EJ12 — S07 | Auditar la validez de los datos de prueba (7 min)

### A. Problema y propósito

Para que un plan de ejecución sea representativo necesitamos saber **sobre qué datos se midió**. En la preparación copiamos registros reales de Northwind **60 veces**, variando `PedidoID`; creamos una muestra artificial para enseñanza, no sesenta ventas reales por pedido. El resultado de rendimiento puede diferir al ejecutar una base comercial auténtica.

### B. Pasos concretos

1. Abre `SQL/EJ12_Auditar_Datos_Simulados.sql` en SSMS. Señala los dos orígenes: **Northwind solo lectura** y `dbo.PedidosLab` de laboratorio.
2. Antes de ejecutar, predice la relación entre cantidad de pedidos fuente válidos y filas de laboratorio: **60 a 1** si nadie alteró la carga.
3. Ejecuta **F5**. Lee `PedidosValidosEnNorthwind`, `FilasDeLaboratorio`, `FilasEsperadas` y `Validacion`.

```sql
SELECT src.PedidosValidosEnNorthwind,
       lab.FilasDeLaboratorio,
       src.PedidosValidosEnNorthwind * CONVERT(BIGINT, 60) AS FilasEsperadas,
       CASE WHEN lab.FilasDeLaboratorio =
                 src.PedidosValidosEnNorthwind * CONVERT(BIGINT, 60)
            THEN N'COINCIDE' ELSE N'REVISAR CARGA' END AS Validacion
FROM
(
    SELECT COUNT_BIG(*) AS PedidosValidosEnNorthwind
    FROM Northwind.dbo.Orders AS o
    INNER JOIN Northwind.dbo.Customers AS c ON c.CustomerID = o.CustomerID
    WHERE o.OrderDate IS NOT NULL AND o.CustomerID IS NOT NULL
      AND c.Country IS NOT NULL
) AS src
CROSS JOIN
(
    SELECT COUNT_BIG(*) AS FilasDeLaboratorio
    FROM dbo.PedidosLab
) AS lab;
```

### C. ¿Qué cambia, qué significa y qué debemos observar?

- La subconsulta `src` cuenta **únicamente** pedidos del origen que cumplen los filtros de la carga inicial. Sin replicar la condición, la comparación podría ser inválida.
- La subconsulta `lab` cuenta las filas presentes en `PedidosLab`, creada para experimentar.
- `CROSS JOIN` aquí une **dos filas resumen**, no multiplica pedidos: cada subconsulta devuelve exactamente un registro.
- Multiplicar por `CONVERT(BIGINT, 60)` calcula la cantidad **esperada** de filas; `CASE` traduce la comprobación numérica a un estado legible. Si dice `REVISAR CARGA`, no significa automáticamente que Northwind esté dañada: quizá la tabla didáctica fue modificada o la fuente cambió.

**Predicción verificada:** `Validacion = COINCIDE` si la carga anterior está íntegra y su fuente no cambió. Ningún dato se modifica. **Error a prevenir:** presentar las sumas de `PedidosLab` como ingresos reales de la empresa; son cifras de una **ampliación sintética**.

### D. T12 y conexión

**T12 (autónoma):** calcula la relación original/ampliación **solo para Alemania** y documenta una frase sobre por qué esos registros no equivalen a ventas reales. Archivo `SQL/T12_Alumno.sql`. **Aceptación:** factor 60 en una muestra intacta y explicación del origen de datos.

**Transición:** ya tenemos cambios seguros, reportes conciliados, planes medidos y un control del volumen de prueba. En EJ06 consolidaremos la evidencia en una presentación técnica de decisión.

---

## EJ06 — Integración S05 + S06 + S07 | Informe SQL con tres evidencias (7 min)

### A. Problema

El docente quiere revisar tu práctica y necesita encontrar, **en un único archivo**, evidencia de: datos mantenidos de forma controlada; una vista reutilizable; y una consulta en la que justificas el acceso mediante el índice y el plan real.

### B. Por qué integrar y qué archivo usar

**Archivo:** `SQL/EJ06_Reporte_Integrado.sql`. No introduce una técnica nueva; **conecta** lo aprendido en EJ01–EJ05. Los primeros dos `SELECT` no modifican datos; el último activa `STATISTICS IO` y se debe ejecutar con Ctrl+M habilitado.

```sql
-- S05: estado de la tabla usada para mantenimiento.
SELECT Pais, COUNT(*) AS PedidosDePractica
FROM dbo.PedidosPractica GROUP BY Pais ORDER BY Pais;

-- S06: consulta a un objeto reutilizable.
SELECT Pais, TotalPedidos, TotalEnvios
FROM dbo.vw_PedidosPorPais
WHERE Pais = N'Germany';

-- S07: consulta medible que aprovecha el criterio de acceso de EJ04.
SELECT TOP (5) PedidoID, ClienteID, FechaPedido, CostoEnvio, Empresa
FROM dbo.PedidosLab
WHERE ClienteID = N'ALFKI'
  AND FechaPedido >= '19970101' AND FechaPedido < '19980101'
ORDER BY FechaPedido, PedidoID
OPTION (RECOMPILE);
```

### C. Explicación y seguimiento

- **Primer bloque:** `COUNT(*)` agrupa el número de filas que permanecen después de las operaciones en `PedidosPractica`. El número puede variar si completaste la tarea T01. **No representa todo Northwind.**
- **Segundo bloque:** la vista obtiene el resumen de `PedidosLab`. `WHERE Pais = N'Germany'` entrega solo la fila de Alemania, sin reescribir agregaciones.
- **Tercer bloque:** `TOP (5)` presenta una muestra limitada; `WHERE ClienteID = ...` y el rango de fecha conectan con el índice de EJ04; `ORDER BY FechaPedido, PedidoID` controla la presentación de las primeras filas.
- La medición solo tiene sentido si guardas: **consulta, plan real, valores observados y una conclusión**. El mismo número de filas no basta para afirmar que un índice es mejor.

### D. Predicción, ejecución y error controlado

**Predicción:** ejecutar el informe no cambia el número de registros de ninguna tabla. **Verificación:** apunta qué bloque corresponde a S05, S06 y S07 y toma al menos una evidencia por bloque. **Error frecuente:** interpretar los conteos ampliados de `PedidosLab` como ventas auténticas; la copia ampliada es **exclusivamente didáctica**.

### E. Tarea espejo T06 — entrega mínima individual

**Enunciado:** crea un informe alternativo en `SQL/T06_Alumno.sql` con: (a) número de pedidos de práctica de `France` (S05), (b) resumen `France` desde la vista (S06), y (c) cinco pedidos de `BERGS` en 1997 con plan real y lecturas (S07). **Restricción:** no crear/alterar objetos de Northwind; cada consulta debe tener un comentario que explique por qué existe. **Criterio:** tres resultados coherentes más una explicación escrita de qué se midió y qué no puede concluirse de esos datos.

---


## 5. Evaluación de 14 minutos - no solo recordar sintaxis

**Actividad:** responde de forma breve y con evidencia (12 preguntas; aproximadamente un minuto cada una más dos minutos de intercambio):

1. ¿Cuál es la diferencia entre un `SELECT` desde Northwind y un `INSERT ... SELECT` hacia `PedidosPractica`?
2. ¿Qué evita `IF NOT EXISTS` al ejecutar EJ01 otra vez?
3. ¿Qué limita exactamente `WHERE PedidoID = 90000001` en un `DELETE`?
4. ¿Por qué `@@ROWCOUNT` se lee justo después del `UPDATE`?
5. ¿Por qué EJ08 afecta una fila la primera vez y ninguna la segunda?
6. ¿En qué se diferencian `vw_PedidosPorPais` y `vw_PedidosPorPaisAnio`?
7. ¿Qué igualdad debe mostrar la conciliación de EJ10?
8. ¿Por qué se elige `(ClienteID, FechaPedido)` para el índice de EJ04?
9. ¿Qué datos reales registras para defender que ese índice ayudó, no ayudó o no puede juzgarse?
10. ¿Por qué `YEAR(FechaPedido) = 1997` puede tener un acceso distinto al rango de fechas?
11. ¿Por qué EJ11 no compara consultas equivalentes?
12. ¿Por qué no debemos llamar «pedidos reales» a la cantidad ampliada de EJ12?

**Microentrega individual (2 min incluidos en el bloque):** completa la **ficha del objetivo**: (a) una fila modificada/validada; (b) un total conciliado; (c) operador y lecturas del plan antes/después o indicar expresamente «no pude medir»; (d) una decisión técnica de dos oraciones con una limitación.

## 6. Cierre de 6 minutos y autoevaluación

**Reflexión:** ¿qué harías diferente para validar un sistema productivo? No basta con copiar el índice de la demostración: hay que medirlo en sus consultas y volúmenes reales, evaluar escritura/almacenamiento y mantener datos íntegros.

| Evidencia observada | Respuesta / valor propio |
|---|---|
| **S05**: filtro usado en un cambio; filas efectivamente alcanzadas | __________________________ |
| **S06**: total de filas vs. suma de vistas | __________________________ |
| **S07**: plan/lecturas antes y después del índice | __________________________ |
| **S07**: advertencia sobre datos multiplicados 60x | __________________________ |
| **Decisión**: conservar, estudiar o descartar el índice y por qué | __________________________ |

### Errores frecuentes y ruta de corrección

- `Invalid object name`: verifica primero `SELECT DB_NAME()` y la ejecución de `SQL/01_Preparar_Laboratorio.sql`. Revisa también la secuencia de vistas.
- `Northwind` no existe: comprueba la **instancia** en SSMS; no improvises tablas llamadas Northwind.
- No aparece plan real: activa Ctrl+M **antes** de F5. No aparece E/S: comprueba que ejecutaste `SET STATISTICS IO ON` y mira **Mensajes**.
- Un índice ya existe al intentar medir «antes»: el experimento está contaminado; **no borres objetos ajenos**. Usa únicamente el archivo de reinicio de laboratorio después de guardar el trabajo o prepara una copia nueva.
- Falló la conciliación de los totales: confirma que ambas vistas usan `PedidosLab` y agrupan todo el universo sin restricciones accidentales.
- Una medición no mejora: es un resultado **válido** si se documenta sin inventar diferencias.

**Advertencia:** los scripts se han revisado de manera estática. Los tipos de operadores, tiempos y lecturas lógicas requieren ejecutar cada consulta en el motor real; **SSMS 22 es el cliente de administración**.
