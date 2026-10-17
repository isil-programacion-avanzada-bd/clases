# FASE 13 — GUÍA DEL ESTUDIANTE
## Sesión 07 · El asistente de optimización de consultas · Programación Avanzada de Base de Datos (ISIL)

**Modalidad:** clase-laboratorio guiada · **Tecnología:** Microsoft SQL Server y SQL Server Management Studio (SSMS) · **Caso de trabajo:** ventas ficticias de una tienda.

## 1. Propósito de la sesión

En esta práctica aprenderás a **interpretar planes de ejecución reales y estimados**, a conectar sus operadores con sentencias SQL y a comprobar cómo influyen la proyección, los filtros, las relaciones entre tablas, las agrupaciones y los datos de prueba. Comenzarás desde una base creada especialmente para el laboratorio, para evitar depender de Northwind o de información personal. El objetivo no es forzar un icono concreto, sino razonar sobre el plan que tu motor elige de verdad.

Al finalizar tendrás una base de laboratorio, ocho ejemplos resueltos, ocho tareas para demostrar comprensión y evidencias verificables de resultados y propiedades de operadores.

## 2. Resultado observable

### Lectura inicial sugerida del docente

Hoy no vamos a limitarnos a escribir una consulta y verificar que aparecen datos. Una consulta puede ser lógicamente correcta y, aun así, recorrer más información de la que esperábamos. SQL Server elabora un plan para ejecutarla; ese plan permite identificar las operaciones físicas que eligió el optimizador. Primero compararemos el plan estimado, que se obtiene sin ejecutar la consulta, con el plan real, que incorpora información de la ejecución. Después veremos cómo el orden, los filtros, las columnas solicitadas y las uniones entre tablas aparecen representados en distintos operadores. No aprenderemos una lista de iconos de memoria: relacionaremos cada operador con una parte de la sentencia y analizaremos su contexto.

Utilizaremos una base de datos totalmente ficticia, con clientes, productos y pedidos. Así podrás repetir los ejercicios sin depender de archivos externos ni tocar información institucional. También construiremos registros de prueba y comprobaremos restricciones para entender por qué un laboratorio debe ser reproducible. Algunas consultas pueden mostrar operadores distintos en otros equipos. Eso no invalida el ejercicio: precisamente aprenderemos a interpretar lo observado en lugar de asumir que SQL Server siempre elige el mismo recorrido. Al terminar, deberás justificar por qué tu consulta obtiene las filas correctas y qué evidencia aporta el plan.

### Desempeños observables

1. Crear y validar un entorno SQL aislado.
2. Obtener un plan estimado y uno real en SSMS.
3. Interpretar operadores de acceso, ordenación, unión y agregación sin atribuirles tiempos falsos.
4. Analizar filtros y columnas solicitadas, conservando la semántica SQL.
5. Generar datos ficticios y comprobar conteo, unicidad y rango.
6. Resolver ocho tareas de transferencia explicando resultados y diferencias.

**Criterio de dominio:** resolver una tarea sin copiar el ejemplo literalmente, predecir un resultado, verificarlo y justificar tanto la salida como el plan elegido. No basta con adjuntar capturas sin interpretación.

## 3. Antes de iniciar

**Debe saber:** SELECT, WHERE, ORDER BY, INNER JOIN, GROUP BY, HAVING, subconsultas y conceptos iniciales de índices de las semanas anteriores.

**Debe tener disponible:** un SQL Server institucional/local de laboratorio, acceso mediante SSMS, permisos para crear una base de datos o una base vacía autorizada, permisos para ejecutar sentencias y permiso SHOWPLAN para observar planes. Debe existir espacio suficiente para unas decenas de miles de filas ficticias. No se necesitan credenciales publicadas ni internet durante la práctica si los programas ya están instalados.

**No se asumirá:** administración de instancias productivas, tuning exhaustivo, estadísticas avanzadas, particionamiento, vistas indexadas o técnicas de seguridad ajenas a los objetivos de esta sesión.

## 4. Preparar y verificar el entorno

### Ruta A — SQL Server y SSMS ya instalados

1. Inicia **SQL Server Management Studio** y conecta con una instancia SQL Server de laboratorio autorizada por tu institución. La autenticación y el servidor dependen de tu centro; no compartas contraseñas en capturas ni repositorios.
2. Abre **Nueva consulta** y ejecuta `SELECT @@VERSION AS VersionMotor, DB_NAME() AS BaseActual;`. Debes obtener una fila con la edición del servidor y el nombre de la base activa.
3. Comprueba que tienes derecho a crear `Lab_ISIL_S07_Planes` o solicita al docente una base de datos vacía autorizada. Si no tienes permiso SHOWPLAN, la consulta puede funcionar, pero no podrás visualizar el plan: solicita el acceso necesario sin elevar privilegios por tu cuenta.
4. **Antes de ejecutar el script inicial**, confirma que no estás en producción. El script crea una nueva base si no existe y agrega tablas solo cuando faltan. No elimina bases ni tablas.
5. Ejecuta la preparación completa indicada más abajo. Verifica los cuatro conteos. Si aparecen otros valores, detén la práctica y revisa con el docente si ese nombre de base ya fue utilizado.

### Ruta B — Equipo sin preparar

1. Solicita al responsable institucional acceso a **SQL Server Database Engine**, que es el motor que ejecuta las sentencias. SSMS es el programa cliente que edita consultas y muestra planes; instalar SSMS no implica instalar el motor.
2. Si el equipo no dispone de los programas, usa los instaladores oficiales elegidos por la institución: https://www.microsoft.com/sql-server/sql-server-downloads y https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms . Para un laboratorio, una edición gratuita de desarrollo autorizada puede ser apropiada; elige según compatibilidad y licencia institucional, no por un número de versión supuesto.
3. Completa la instalación con el técnico o docente autorizado, identifica el nombre real de la instancia y conéctate desde SSMS. Prueba `SELECT @@VERSION;` antes de crear objetos.
4. No necesitas Azure, extensiones de optimización de terceros, herramientas de generación de datos ni una copia de Northwind.

### Herramientas y errores frecuentes

| Componente | Función | Prueba | Si falla |
|---|---|---|---|
| SQL Server (motor) | Ejecuta T-SQL y optimiza consultas | `SELECT @@VERSION;` devuelve una fila | Verifica servidor/servicio/autorización |
| SSMS (cliente) | Edita SQL, muestra resultados y planes | Nueva consulta y F5 | Verifica conexión y base seleccionada |
| Permiso SHOWPLAN | Permite obtener planes gráficos | Ctrl+L sobre SELECT | Solicita permiso al administrador |
| Base de laboratorio | Aísla todas las prácticas | `SELECT DB_NAME();` | Selecciona la base autorizada |

### Preparación completa: crea el conjunto de tablas y datos

**Qué construiremos primero:** cuatro tablas relacionadas mediante claves y restricciones: `Clientes` (500), `Productos` (50), `Pedidos` (10 000) y `DetallePedido` (30 000). La clave primaria identifica filas, la foránea protege las relaciones, y el `CHECK` impide cantidades inválidas. Primero se crean tablas principales y después las dependientes para cumplir las FK. A continuación se generan filas mediante combinaciones de diez dígitos y `ROW_NUMBER`.

**Antes de pegar el script:** comprueba que el editor SSMS está conectado a una instancia de pruebas. Cada sección `GO` indica al cliente que envíe el lote anterior al servidor. No modifiques datos de otras bases.

```sql
/*
LABORATORIO SESIÓN 07 — PROGRAMACIÓN AVANZADA DE BASE DE DATOS
Entorno didáctico aislado. NO ejecutar sobre bases con datos reales.
No elimina ni modifica bases ajenas. Ejecutar una sola vez al comenzar.
Si no se puede crear una base, solicitar al docente una base vacía autorizada
antes de adaptar la sentencia USE.
*/
USE master;
GO
IF DB_ID(N'Lab_ISIL_S07_Planes') IS NULL
BEGIN
    CREATE DATABASE Lab_ISIL_S07_Planes;
END;
GO
USE Lab_ISIL_S07_Planes;
GO
-- Todas las relaciones son ficticias y permanecen dentro de esta base.
IF OBJECT_ID(N'dbo.Clientes', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.Clientes
    (
        ClienteID int NOT NULL PRIMARY KEY CLUSTERED,
        Nombre nvarchar(80) NOT NULL,
        Ciudad nvarchar(40) NOT NULL
    );
END;
IF OBJECT_ID(N'dbo.Productos', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.Productos
    (
        ProductoID int NOT NULL PRIMARY KEY CLUSTERED,
        Nombre nvarchar(80) NOT NULL,
        Precio decimal(10,2) NOT NULL CHECK (Precio >= 0)
    );
END;
IF OBJECT_ID(N'dbo.Pedidos', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.Pedidos
    (
        PedidoID int NOT NULL PRIMARY KEY CLUSTERED,
        ClienteID int NOT NULL,
        FechaPedido date NOT NULL,
        Estado varchar(12) NOT NULL,
        Total decimal(12,2) NOT NULL,
        CONSTRAINT FK_Pedidos_Clientes FOREIGN KEY (ClienteID)
            REFERENCES dbo.Clientes(ClienteID)
    );
END;
IF OBJECT_ID(N'dbo.DetallePedido', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.DetallePedido
    (
        DetalleID int NOT NULL PRIMARY KEY CLUSTERED,
        PedidoID int NOT NULL,
        ProductoID int NOT NULL,
        Cantidad int NOT NULL CHECK (Cantidad > 0),
        CONSTRAINT FK_Detalle_Pedido FOREIGN KEY (PedidoID)
            REFERENCES dbo.Pedidos(PedidoID),
        CONSTRAINT FK_Detalle_Producto FOREIGN KEY (ProductoID)
            REFERENCES dbo.Productos(ProductoID)
    );
END;
GO
-- Generador reproducible 1..10000: 10 x 10 x 10 x 10 combinaciones.
-- La clave n es única; el orden físico en que se inserta no es relevante.
IF NOT EXISTS (SELECT 1 FROM dbo.Clientes)
BEGIN
    WITH D AS (
        SELECT v FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS a(v)
    ), N AS (
        SELECT ROW_NUMBER() OVER (ORDER BY a.v,b.v,c.v,d.v) AS n
        FROM D AS a CROSS JOIN D AS b CROSS JOIN D AS c CROSS JOIN D AS d
    )
    INSERT INTO dbo.Clientes (ClienteID, Nombre, Ciudad)
    SELECT CAST(n AS int),
           CONCAT(N'Cliente ', RIGHT(CONCAT(N'000', n), 3)),
           CASE n % 4 WHEN 0 THEN N'Lima' WHEN 1 THEN N'Arequipa'
                      WHEN 2 THEN N'Cusco' ELSE N'Trujillo' END
    FROM N WHERE n <= 500;
END;
IF NOT EXISTS (SELECT 1 FROM dbo.Productos)
BEGIN
    WITH D AS (
        SELECT v FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS a(v)
    ), N AS (
        SELECT ROW_NUMBER() OVER (ORDER BY a.v,b.v) AS n
        FROM D AS a CROSS JOIN D AS b
    )
    INSERT INTO dbo.Productos (ProductoID, Nombre, Precio)
    SELECT CAST(n AS int), CONCAT(N'Producto ', n),
           CAST(10 + (n % 40) * 3.25 AS decimal(10,2))
    FROM N WHERE n <= 50;
END;
IF NOT EXISTS (SELECT 1 FROM dbo.Pedidos)
BEGIN
    WITH D AS (
        SELECT v FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS a(v)
    ), N AS (
        SELECT ROW_NUMBER() OVER (ORDER BY a.v,b.v,c.v,d.v) AS n
        FROM D AS a CROSS JOIN D AS b CROSS JOIN D AS c CROSS JOIN D AS d
    )
    INSERT INTO dbo.Pedidos
        (PedidoID, ClienteID, FechaPedido, Estado, Total)
    SELECT CAST(n AS int), CAST(((n-1) % 500)+1 AS int),
           DATEADD(day, -CAST(n % 720 AS int), CONVERT(date,'20260930',112)),
           CASE WHEN n % 10 = 0 THEN 'Cancelado'
                WHEN n % 3 = 0 THEN 'Pendiente' ELSE 'Pagado' END,
           CAST(50 + (n % 190) * 4.50 AS decimal(12,2))
    FROM N;
END;
IF NOT EXISTS (SELECT 1 FROM dbo.DetallePedido)
BEGIN
    WITH D AS (
        SELECT v FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS a(v)
    ), N AS (
        SELECT ROW_NUMBER() OVER (ORDER BY a.v,b.v,c.v,d.v) AS n
        FROM D AS a CROSS JOIN D AS b CROSS JOIN D AS c CROSS JOIN D AS d
    )
    INSERT INTO dbo.DetallePedido (DetalleID, PedidoID, ProductoID, Cantidad)
    SELECT CAST(3*n-2 AS int), CAST(n AS int),
           CAST(((n-1)%50)+1 AS int), 1+CAST(n%4 AS int)
    FROM N
    UNION ALL
    SELECT CAST(3*n-1 AS int), CAST(n AS int),
           CAST((n%50)+1 AS int), 1+CAST((n+1)%4 AS int)
    FROM N
    UNION ALL
    SELECT CAST(3*n AS int), CAST(n AS int),
           CAST(((n+1)%50)+1 AS int), 1+CAST((n+2)%4 AS int)
    FROM N;
END;
GO
-- Control inicial, antes de crear el índice didáctico del EJ03.
SELECT 'Clientes' AS Tabla, COUNT(*) AS Filas FROM dbo.Clientes
UNION ALL SELECT 'Productos', COUNT(*) FROM dbo.Productos
UNION ALL SELECT 'Pedidos', COUNT(*) FROM dbo.Pedidos
UNION ALL SELECT 'DetallePedido', COUNT(*) FROM dbo.DetallePedido;
SELECT @@VERSION AS VersionMotor, DB_NAME() AS BaseActual;
GO
```

**Resultado de la preparación:** la tabla de control muestra exactamente `Clientes=500`, `Productos=50`, `Pedidos=10000` y `DetallePedido=30000`. La consulta final confirma `BaseActual=Lab_ISIL_S07_Planes`. Si la base ya tenía objetos incompatibles o recuentos distintos, detente: no fuerces la práctica sobre datos desconocidos.

**Traza de construcción:** se crean primero los clientes y productos porque todavía no tienen dependencias; después pedidos, cuya FK necesita clientes; finalmente detalle, cuya FK necesita pedidos y productos. El generador `D` aporta 10 filas; cuatro cruces producen 10 000 números. Tres selecciones unidas por `UNION ALL` materializan 30 000 detalles, tres por cada pedido. No existe aleatoriedad en los identificadores de la base base; esta característica permite comprobar resultados de forma estable.

**Primer control de confianza:** ejecuta `SELECT COUNT(*) FROM dbo.Pedidos;` y explica por qué devuelve 10 000. No continúes si el resultado es distinto.

## 5. Ocho ejemplos resueltos con tareas espejo

Sigue el orden; **EJ03 añade un índice**, de manera que algunos planes de EJ01/EJ02 pueden cambiar si los repites después. Los resultados lógicos deben conservarse. Los iconos exactos, costos porcentuales y algoritmos físicos **no están garantizados**: el optimizador depende de estadísticas, versión y volumen.

---

## EJ01 — Del SELECT al plan estimado y al plan real

**Qué vamos a resolver.** Un SELECT puede devolver filas correctas sin mostrar cómo el motor las obtuvo. Necesitamos observar ambos planes de la misma consulta para separar predicción del optimizador y ejecución.

**Aprendizaje principal.** Distinguir estimación de ejecución real y leer filas estimadas/reales.

**Artefactos:** `estudiante/ejemplos/EJ01.sql` y `estudiante/tareas/T01.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Un SELECT puede devolver filas correctas sin mostrar cómo el motor las obtuvo. Necesitamos observar ambos planes de la misma consulta para separar predicción del optimizador y ejecución.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Delimitar la fuente y proyectar columnas

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
SELECT p.PedidoID, p.FechaPedido, p.Estado
FROM dbo.Pedidos AS p
```

**Explicación detallada:** SELECT indica las tres columnas necesarias; `p` es el alias de Pedidos y `dbo` el esquema. Elegir columnas concretas impide cargar valores no solicitados. Aún no hay predicado, por lo que se considerarían todos los pedidos.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Precisar las filas requeridas

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
WHERE p.PedidoID BETWEEN 101 AND 110;
```

**Explicación detallada:** BETWEEN incluye ambos extremos y define exactamente diez identificadores consecutivos del conjunto generado. Como PedidoID es clave primaria, los registros son únicos; el optimizador decide el acceso apropiado.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Solicitar los dos tipos de plan

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
-- En SSMS: Ctrl+L (estimado); luego Ctrl+M y F5 (real).
```

**Explicación detallada:** Ctrl+L solicita un plan sin ejecutar SELECT. Ctrl+M habilita Incluir plan de ejecución real; F5 ejecuta la consulta y añade la pestaña Plan de ejecución. Los valores de filas reales solamente aparecen después de ejecutar.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. La cuadrícula Results prueba los datos devueltos; el plan indica cómo SQL Server eligió obtenerlos. El costo porcentual mostrado es estimado y relativo al plan, no un cronómetro.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ01: primero observa un plan ESTIMADO con Ctrl+L.
-- Luego activa Ctrl+M y ejecuta (F5) para ver el plan REAL.
SELECT p.PedidoID, p.FechaPedido, p.Estado
FROM dbo.Pedidos AS p
WHERE p.PedidoID BETWEEN 101 AND 110;
GO
```

### Antes de ejecutar: predice

1. ¿Cuántas filas devolverá la consulta?
2. ¿Qué plan podrá indicar filas reales antes de ejecutar?

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** 10 pedidos (101–110); plan estimado y plan real accesibles, con operadores y propiedades. El tipo de operador físico puede cambiar por versión o estadísticas.

**Interpretación:** La cuadrícula Results prueba los datos devueltos; el plan indica cómo SQL Server eligió obtenerlos. El costo porcentual mostrado es estimado y relativo al plan, no un cronómetro.

### Variación controlada

Reemplaza BETWEEN 101 AND 110 por BETWEEN 101 AND 105 y predice la cantidad de filas. Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Intentar ver filas reales después de Ctrl+L, sin ejecutar. Es un plan estimado: usar Ctrl+M y F5 para comprobar propiedades reales.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Por qué un SELECT correcto no demuestra por sí solo el trabajo interno del motor?

### T01 — Tarea espejo de EJ01

**Enunciado:** recupera los pedidos del 201 al 220, solamente -- PedidoID, ClienteID y Estado. Observa Ctrl+L y Ctrl+M/F5. -- Escribe la consulta a continuación.

**Qué modificar:** `estudiante/tareas/T01.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** Usa BETWEEN y conserva una proyección de solo tres columnas.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** La cuadrícula contiene 20 pedidos distintos; explicar por qué el estimado no tiene métricas reales.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T01: recupera los pedidos del 201 al 220, solamente
-- PedidoID, ClienteID y Estado. Observa Ctrl+L y Ctrl+M/F5.
-- Escribe la consulta a continuación.
-- TODO T01: proyectar columnas y añadir predicado inclusivo.
GO
```


---

## EJ02 — Detectar una operación Sort al ordenar resultados

**Qué vamos a resolver.** Una tabla puede tener un índice por su identificador, pero el usuario solicita ordenar por fecha. Se comparará una consulta sin ORDER BY y otra con ORDER BY para identificar el efecto lógico y un posible Sort.

**Aprendizaje principal.** Diferenciar orden lógico solicitado y ordenamiento físico del plan.

**Artefactos:** `estudiante/ejemplos/EJ02.sql` y `estudiante/tareas/T02.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Una tabla puede tener un índice por su identificador, pero el usuario solicita ordenar por fecha. Se comparará una consulta sin ORDER BY y otra con ORDER BY para identificar el efecto lógico y un posible Sort.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Fijar el conjunto de comparación

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
SELECT p.PedidoID, p.FechaPedido
FROM dbo.Pedidos AS p
WHERE p.PedidoID BETWEEN 1 AND 200;
```

**Explicación detallada:** Se eligen 200 filas conocidas. La ausencia de ORDER BY significa que SQL no promete un orden lógico aunque en ocasiones la pantalla parezca estar ordenada.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Exigir un orden reproducible

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
ORDER BY p.FechaPedido ASC, p.PedidoID ASC;
```

**Explicación detallada:** FechaPedido es la primera clave de ordenación, y PedidoID rompe empates. El segundo criterio evita depender del orden accidental de registros con la misma fecha.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Inspeccionar el plan con la misma selección

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
-- Ctrl+M, F5 sobre cada SELECT por separado.
```

**Explicación detallada:** Al comparar planes, busca Sort en la segunda consulta y comprueba su entrada/salida. La presencia de Sort no es obligatoria: SQL Server puede resolver el orden con otro acceso disponible.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. ORDER BY modifica el requisito de presentación, no el conjunto de filas. Sort representa trabajo de ordenación si el optimizador lo necesita.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ02-A: sin garantía de orden de salida.
SELECT p.PedidoID, p.FechaPedido
FROM dbo.Pedidos AS p
WHERE p.PedidoID BETWEEN 1 AND 200;
GO
-- EJ02-B: orden definido por fecha e identificador.
SELECT p.PedidoID, p.FechaPedido
FROM dbo.Pedidos AS p
WHERE p.PedidoID BETWEEN 1 AND 200
ORDER BY p.FechaPedido ASC, p.PedidoID ASC;
GO
```

### Antes de ejecutar: predice

1. ¿Cambiará la cantidad de filas por añadir ORDER BY?
2. ¿Cuál consulta garantiza que la fecha sea ascendente?

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** 200 filas en ambas consultas. La segunda queda ordenada por FechaPedido y PedidoID; es posible ver Sort en su plan.

**Interpretación:** ORDER BY modifica el requisito de presentación, no el conjunto de filas. Sort representa trabajo de ordenación si el optimizador lo necesita.

### Variación controlada

Invierte ASC a DESC en ambos criterios y compara primeros/últimos registros. Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Suponer que una consulta sin ORDER BY tiene orden garantizado porque se observan claves ascendentes. Corregir añadiendo ORDER BY explícito.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Por qué un índice clustered por PedidoID no asegura que FechaPedido ya esté ordenada?

### T02 — Tarea espejo de EJ02

**Enunciado:** muestra PedidoID, FechaPedido y Total de los pedidos 301..360. -- Primera consulta sin ordenar; segunda ordenada por Total DESC -- y PedidoID ASC para desempatar.

**Qué modificar:** `estudiante/tareas/T02.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** La segunda consulta conserva el mismo filtro e incorpora dos criterios de ORDER BY.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** Ambas consultas entregan 60 filas; la segunda es descendente por Total y consistente en los empates.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T02: muestra PedidoID, FechaPedido y Total de los pedidos 301..360.
-- Primera consulta sin ordenar; segunda ordenada por Total DESC
-- y PedidoID ASC para desempatar.
-- TODO T02-A y T02-B.
GO
```


---

## EJ03 — Comparar Scan, Seek y cobertura de un índice

**Qué vamos a resolver.** Las búsquedas por fechas pueden leer numerosas filas cuando no hay acceso apropiado. Se añade un índice específico y se distingue consulta cubierta de consulta que solicita una columna adicional.

**Aprendizaje principal.** Observar efecto de crear un índice no clustered y proyectar columnas cubiertas.

**Artefactos:** `estudiante/ejemplos/EJ03.sql` y `estudiante/tareas/T03.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Las búsquedas por fechas pueden leer numerosas filas cuando no hay acceso apropiado. Se añade un índice específico y se distingue consulta cubierta de consulta que solicita una columna adicional.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Definir índice, clave y columnas incluidas

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
CREATE NONCLUSTERED INDEX IX_Pedidos_FechaPedido
ON dbo.Pedidos (FechaPedido)
INCLUDE (Estado);
```

**Explicación detallada:** FechaPedido es la clave buscable del índice; Estado es columna incluida solo para devolver resultados sin consultar el registro base. En un índice no clustered sobre tabla clustered, PedidoID se incorpora como localizador. La protección IF NOT EXISTS del script integral hace seguro repetir la demostración.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Filtrar con límites de fecha

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
WHERE p.FechaPedido >= '20260301'
  AND p.FechaPedido <  '20260401';
```

**Explicación detallada:** Este intervalo semiabierto reúne todos los días de marzo de 2026 y evita funciones sobre FechaPedido. El formato de fecha YYYYMMDD es inequívoco en SQL Server.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Contrastar la proyección

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
SELECT p.PedidoID, p.FechaPedido, p.Estado -- cubierta
SELECT p.PedidoID, p.FechaPedido, p.Total  -- Total no incluida
```

**Explicación detallada:** Primera proyección: clave de agrupación, clave clustered e INCLUDE cubren la consulta. Segunda: Total exige obtener información adicional, que puede manifestarse como Key Lookup o elegirse otro acceso; nunca asumir un operador fijo.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. Seek no siempre es más rápido que Scan; debe evaluarse volumen, costo total, estadísticas y columnas requeridas. Lookup no es automáticamente un error.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ03: el índice de fecha no debe existir antes de EJ01/EJ02.
IF NOT EXISTS (
    SELECT 1 FROM sys.indexes
    WHERE object_id = OBJECT_ID(N'dbo.Pedidos')
      AND name = N'IX_Pedidos_FechaPedido'
)
BEGIN
    CREATE NONCLUSTERED INDEX IX_Pedidos_FechaPedido
        ON dbo.Pedidos (FechaPedido)
        INCLUDE (Estado);
END;
GO
-- Consulta cubierta por el índice (PedidoID es clave clustered).
SELECT p.PedidoID, p.FechaPedido, p.Estado
FROM dbo.Pedidos AS p
WHERE p.FechaPedido >= '20260301'
  AND p.FechaPedido <  '20260401';
GO
-- La columna Total no se incluyó en este índice.
SELECT p.PedidoID, p.FechaPedido, p.Total
FROM dbo.Pedidos AS p
WHERE p.FechaPedido >= '20260301'
  AND p.FechaPedido <  '20260401';
GO
```

### Antes de ejecutar: predice

1. ¿Cuál consulta puede resolverse completamente desde el índice?
2. ¿Aparecerá inevitablemente Key Lookup en la segunda? Justifica.

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** Ambas consultas devuelven pedidos del mismo mes. En el primer plan puede observarse Index Seek; en el segundo puede existir Key Lookup o un acceso alternativo.

**Interpretación:** Seek no siempre es más rápido que Scan; debe evaluarse volumen, costo total, estadísticas y columnas requeridas. Lookup no es automáticamente un error.

### Variación controlada

Cambiar al intervalo de abril de 2026 y comparar número de filas y operadores. Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Crear otro índice idéntico al ejecutar el script repetidamente. El guard IF NOT EXISTS evita el error de nombre duplicado.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Por qué INCLUDE(Estado) mejora cobertura sin que Estado sea una clave de búsqueda?

### T03 — Tarea espejo de EJ03

**Enunciado:** consultar pedidos de mayo de 2026 con PedidoID, FechaPedido, -- Estado; luego repetir añadiendo Total para observar cobertura. -- El índice EJ03 ya está creado. No crear uno adicional.

**Qué modificar:** `estudiante/tareas/T03.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** Usa `>=` para el comienzo de mayo y `<` para junio; compara las propiedades del operador seleccionado.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** Ambas consultas devuelven el mismo número de filas, pero la segunda tiene una columna adicional; describir el acceso realmente observado.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T03: consultar pedidos de mayo de 2026 con PedidoID, FechaPedido,
-- Estado; luego repetir añadiendo Total para observar cobertura.
-- El índice EJ03 ya está creado. No crear uno adicional.
-- TODO T03-A y T03-B: intervalo [mayo, junio).
GO
```


---

## EJ04 — Leer los operadores físicos al combinar tablas

**Qué vamos a resolver.** Un INNER JOIN expresa la relación deseada, mientras el optimizador decide cómo ejecutarla. Un conjunto reducido y un conjunto completo permitirán examinar diferencias en los operadores.

**Aprendizaje principal.** Relacionar INNER JOIN lógico con algoritmos físicos variables.

**Artefactos:** `estudiante/ejemplos/EJ04.sql` y `estudiante/tareas/T04.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Un INNER JOIN expresa la relación deseada, mientras el optimizador decide cómo ejecutarla. Un conjunto reducido y un conjunto completo permitirán examinar diferencias en los operadores.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Identificar claves relacionadas

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
FROM dbo.Pedidos AS p
INNER JOIN dbo.Clientes AS c
    ON c.ClienteID = p.ClienteID
```

**Explicación detallada:** La FK Pedidos.ClienteID apunta a la PK Clientes.ClienteID. ON expresa la igualdad que vincula cada pedido a su dueño; falta de ON o desigualdad cambiaría el resultado lógico.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Limitar un conjunto de entrada

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
WHERE p.PedidoID BETWEEN 1 AND 20;
```

**Explicación detallada:** Limitar pedidos produce veinte filas y normalmente favorece operaciones físicas apropiadas a entradas pequeñas, sin imponer ningún algoritmo.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Agrupar el conjunto grande

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
SELECT c.Ciudad, COUNT_BIG(*) AS CantidadPedidos
... 
GROUP BY c.Ciudad;
```

**Explicación detallada:** COUNT_BIG cuenta pedidos por Ciudad; la agregación reduce muchas filas a cuatro ciudades. Al ejecutar, distinguir un operador físico JOIN de uno de agregación como Hash Match (Aggregate).

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. Nested Loops, Hash Match y Merge Join son alternativas físicas; ni la presencia de Hash Match ni la de Nested Loops demuestra por sí sola un problema.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ04-A: veinte pedidos y sus respectivos clientes.
SELECT p.PedidoID, c.Nombre, p.Estado
FROM dbo.Pedidos AS p
INNER JOIN dbo.Clientes AS c
    ON c.ClienteID = p.ClienteID
WHERE p.PedidoID BETWEEN 1 AND 20;
GO
-- EJ04-B: misma relación, mayor volumen; salida agregada.
SELECT c.Ciudad, COUNT_BIG(*) AS CantidadPedidos
FROM dbo.Pedidos AS p
INNER JOIN dbo.Clientes AS c
    ON c.ClienteID = p.ClienteID
GROUP BY c.Ciudad;
GO
```

### Antes de ejecutar: predice

1. ¿Qué diferencia existe entre INNER JOIN y Nested Loops?
2. ¿Cuántas filas devolverá la consulta A? ¿Cuántos grupos devolverá la B?

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** A: 20 filas. B: 4 grupos de ciudad; suma de CantidadPedidos = 10 000. SQL Server puede elegir Nested Loops, Hash Match u otro algoritmo.

**Interpretación:** Nested Loops, Hash Match y Merge Join son alternativas físicas; ni la presencia de Hash Match ni la de Nested Loops demuestra por sí sola un problema.

### Variación controlada

Filtra el grupo A a 5 pedidos y describe las diferencias con la selección de 20. Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Leer “Hash Match (Aggregate)” como si fuera un algoritmo JOIN. Inspecciona el texto del operador y sus propiedades lógicas.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Qué parte de la consulta describe el resultado lógico y cuál pertenece a la decisión del optimizador?

### T04 — Tarea espejo de EJ04

**Enunciado:** listar PedidoID, FechaPedido, Nombre de los pedidos 401..425. -- Después contar pedidos agrupados por Ciudad, filtrando Estado='Pagado'.

**Qué modificar:** `estudiante/tareas/T04.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** Coloca filtro de Estado antes de GROUP BY en la misma consulta; usa COUNT_BIG(*) para contar pedidos.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** La consulta A devuelve 25 filas; la B devuelve agrupaciones por ciudad cuya suma coincide con `SELECT COUNT(*) FROM dbo.Pedidos WHERE Estado = Pagado`.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T04: listar PedidoID, FechaPedido, Nombre de los pedidos 401..425.
-- Después contar pedidos agrupados por Ciudad, filtrando Estado='Pagado'.
-- TODO T04-A (JOIN pequeño) y T04-B (JOIN + GROUP BY + WHERE).
GO
```


---

## EJ05 — Agrupar, filtrar grupos y ordenar sin confundir operadores

**Qué vamos a resolver.** Los reportes suelen agrupar y ordenar datos. El plan puede mostrar varios operadores: deben relacionarse con la necesidad de contar filas, filtrar grupos y ordenar la salida, sin exigir un operador único.

**Aprendizaje principal.** Interpretar GROUP BY, HAVING y ORDER BY junto a operaciones físicas.

**Artefactos:** `estudiante/ejemplos/EJ05.sql` y `estudiante/tareas/T05.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Los reportes suelen agrupar y ordenar datos. El plan puede mostrar varios operadores: deben relacionarse con la necesidad de contar filas, filtrar grupos y ordenar la salida, sin exigir un operador único.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Construir cantidades por grupo

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
SELECT p.Estado, COUNT_BIG(*) AS Cantidad
FROM dbo.Pedidos AS p
GROUP BY p.Estado
```

**Explicación detallada:** Estado contiene las etiquetas Pagado, Pendiente y Cancelado. GROUP BY crea un grupo por estado; COUNT_BIG(*) cuenta sus pedidos, no clientes.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Agregar una medida calculada

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
CAST(AVG(p.Total) AS decimal(12,2)) AS Promedio
```

**Explicación detallada:** AVG calcula el promedio de Total y CAST lo presenta con dos decimales. Puede surgir Compute Scalar para alguna expresión, aunque el plan puede ubicar el cálculo de otra forma.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Distinguir filtro de grupos y orden

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
HAVING COUNT_BIG(*) > 1000
ORDER BY Cantidad DESC, p.Estado ASC;
```

**Explicación detallada:** HAVING actúa sobre grupos ya formados, distinto de WHERE sobre filas individuales. ORDER BY ordena por cantidad y usa Estado para desempatar.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. Hash Match (Aggregate) calcula agrupaciones cuando se elige esa estrategia; Filter puede implementar HAVING; Sort puede ordenar, pero no todos son obligatorios visualmente.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ05: agrupación por estado, filtro de grupos y orden de salida.
SELECT p.Estado, COUNT_BIG(*) AS Cantidad,
       CAST(AVG(p.Total) AS decimal(12,2)) AS Promedio
FROM dbo.Pedidos AS p
GROUP BY p.Estado
HAVING COUNT_BIG(*) > 1000
ORDER BY Cantidad DESC, p.Estado ASC;
GO
```

### Antes de ejecutar: predice

1. ¿Cuántos grupos diferentes de Estado existen?
2. ¿Qué ocurre con los grupos que no superan 1 000 pedidos?

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** Grupos provenientes de tres estados; solo aparecen los que superan 1 000. Los operadores físicos de agregación, filtro y orden dependen del plan elegido.

**Interpretación:** Hash Match (Aggregate) calcula agrupaciones cuando se elige esa estrategia; Filter puede implementar HAVING; Sort puede ordenar, pero no todos son obligatorios visualmente.

### Variación controlada

Sube HAVING a > 3000 y compara cuáles grupos se conservan. Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Colocar COUNT_BIG(*) en WHERE: SQL Server no acepta agregados directamente en WHERE. Mover la condición a HAVING.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Por qué HAVING y WHERE no son intercambiables cuando hablamos de COUNT_BIG?

### T05 — Tarea espejo de EJ05

**Enunciado:** agrupa pedidos por ClienteID. Muestra ClienteID, COUNT_BIG(*) -- y MAX(Total); conserva clientes con al menos 20 pedidos -- y ordena por MAX(Total) DESC y ClienteID ASC.

**Qué modificar:** `estudiante/tareas/T05.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** COUNT_BIG(*) se evalúa por ClienteID; aplica el umbral con HAVING.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** 500 grupos; cada grupo tiene 20 pedidos; la salida está ordenada según MayorTotal descendente.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T05: agrupa pedidos por ClienteID. Muestra ClienteID, COUNT_BIG(*)
-- y MAX(Total); conserva clientes con al menos 20 pedidos
-- y ordena por MAX(Total) DESC y ClienteID ASC.
-- TODO T05: GROUP BY + HAVING + ORDER BY.
GO
```


---

## EJ06 — Escribir filtros sargables y seleccionar solo lo necesario

**Qué vamos a resolver.** Aplicar funciones sobre una columna filtrada puede dificultar el uso eficiente de un índice. Compararemos dos expresiones equivalentes por año y estudiaremos la diferencia entre proyectar pocas columnas y usar SELECT *.

**Aprendizaje principal.** Reescribir filtros sin alterar resultados y observar cambios potenciales en el acceso.

**Artefactos:** `estudiante/ejemplos/EJ06.sql` y `estudiante/tareas/T06.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Aplicar funciones sobre una columna filtrada puede dificultar el uso eficiente de un índice. Compararemos dos expresiones equivalentes por año y estudiaremos la diferencia entre proyectar pocas columnas y usar SELECT *.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Establecer un filtro semántico por año

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
WHERE YEAR(p.FechaPedido) = 2026;
```

**Explicación detallada:** YEAR extrae el año y devuelve las filas correctas. Sin embargo, aplica una función a cada FechaPedido, lo que puede dificultar el aprovechamiento directo de la clave del índice.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Reformular como rango equivalente

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
WHERE p.FechaPedido >= '20260101'
  AND p.FechaPedido <  '20270101';
```

**Explicación detallada:** El rango semiabierto incluye todas las fechas de 2026, sin transformar la columna. La igualdad del resultado es verificable comparando conteos y diferencias de conjuntos.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Disminuir las columnas devueltas

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
SELECT * FROM dbo.Clientes AS c WHERE c.Nombre LIKE N'Cliente 01%';
SELECT c.ClienteID, c.Nombre FROM dbo.Clientes AS c
WHERE c.Nombre LIKE N'Cliente 01%';
```

**Explicación detallada:** SELECT * incluye Ciudad aunque no se necesita. Proyección explícita reduce anchura de salida. `LIKE N'Cliente 01%'` busca un prefijo; el patrón con `%` al inicio no tendría las mismas oportunidades de búsqueda indexada.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. Sargabilidad implica posibilidad de aprovechar rutas de acceso; no garantiza un Seek en toda circunstancia. La consulta con funciones no debe calificarse automáticamente como lenta sin medición.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ06-A: la función está aplicada a la columna filtrada.
SELECT p.PedidoID, p.FechaPedido
FROM dbo.Pedidos AS p
WHERE YEAR(p.FechaPedido) = 2026;
GO
-- EJ06-B: intervalo sobre la columna sin transformarla.
SELECT p.PedidoID, p.FechaPedido
FROM dbo.Pedidos AS p
WHERE p.FechaPedido >= '20260101'
  AND p.FechaPedido <  '20270101';
GO
-- EJ06-C/D: mismo filtro, distinto ancho de proyección.
SELECT * FROM dbo.Clientes AS c WHERE c.Nombre LIKE N'Cliente 01%';
SELECT c.ClienteID, c.Nombre
FROM dbo.Clientes AS c WHERE c.Nombre LIKE N'Cliente 01%';
GO
```

### Antes de ejecutar: predice

1. ¿Deberían devolver igual número de pedidos EJ06-A y B?
2. ¿El SELECT * cambia las filas coincidentes o solo las columnas?

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** EJ06-A y B devuelven el mismo conjunto de pedidos 2026. Los dos SELECT a Clientes devuelven las mismas filas, pero distintas columnas.

**Interpretación:** Sargabilidad implica posibilidad de aprovechar rutas de acceso; no garantiza un Seek en toda circunstancia. La consulta con funciones no debe calificarse automáticamente como lenta sin medición.

### Variación controlada

Prueba `LIKE N'%01'` y describe qué cambia en el patrón (no afirmes índices nuevos). Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Usar `BETWEEN '20260101' AND '20261231'` con datetime puede omitir parte del 31 de diciembre. Mantener intervalo semiabierto incluso cuando hoy se usa date.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Por qué reescribir WHERE puede mantener las filas y permitir otro plan físico?

### T06 — Tarea espejo de EJ06

**Enunciado:** compara MONTH(FechaPedido)=6 AND YEAR(FechaPedido)=2026 -- con un rango semiabierto junio 2026. Proyecta PedidoID y FechaPedido. -- Añade una consulta a Clientes que devuelva solo ClienteID y Nombre -- para nombres cuyo prefijo sea 'Cliente 02'.

**Qué modificar:** `estudiante/tareas/T06.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** Junio comienza en 20260601 y termina justo antes de 20260701.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** T06-A y T06-B presentan los mismos PedidoID; T06-C contiene solo las dos columnas solicitadas.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T06: compara MONTH(FechaPedido)=6 AND YEAR(FechaPedido)=2026
-- con un rango semiabierto junio 2026. Proyecta PedidoID y FechaPedido.
-- Añade una consulta a Clientes que devuelva solo ClienteID y Nombre
-- para nombres cuyo prefijo sea 'Cliente 02'.
-- TODO T06-A, T06-B y T06-C.
GO
```


---

## EJ07 — Validar EXISTS frente a JOIN sin duplicar entidades

**Qué vamos a resolver.** Sustituir siempre una subconsulta por JOIN es arriesgado: una coincidencia múltiple puede duplicar filas. Compararemos EXISTS con JOIN y mostraremos una reformulación equivalente con DISTINCT solo cuando corresponda.

**Aprendizaje principal.** Conservar semántica de existencia frente a multiplicación de filas en JOIN.

**Artefactos:** `estudiante/ejemplos/EJ07.sql` y `estudiante/tareas/T07.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Sustituir siempre una subconsulta por JOIN es arriesgado: una coincidencia múltiple puede duplicar filas. Compararemos EXISTS con JOIN y mostraremos una reformulación equivalente con DISTINCT solo cuando corresponda.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Formular una pregunta de existencia

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
WHERE EXISTS (
    SELECT 1 FROM dbo.Pedidos AS p
    WHERE p.ClienteID = c.ClienteID
      AND p.Estado = 'Pagado'
);
```

**Explicación detallada:** EXISTS responde si existe por lo menos un pedido pagado del cliente externo c. SELECT 1 no transporta una columna; la prueba es la existencia de filas. La condición de correlación impide preguntar por pedidos de otro cliente.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Examinar un JOIN con múltiples coincidencias

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
INNER JOIN dbo.Pedidos AS p ON p.ClienteID = c.ClienteID
WHERE p.Estado = 'Pagado';
```

**Explicación detallada:** JOIN construye coincidencias cliente-pedido. Un cliente con varios pedidos pagados aparece varias veces, por lo que el resultado no representa el mismo conjunto de clientes distintos.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Restablecer unicidad cuando la semántica lo exige

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
SELECT DISTINCT c.ClienteID, c.Nombre
```

**Explicación detallada:** DISTINCT elimina filas duplicadas en la proyección. Para una pregunta de mera existencia, EXISTS puede expresar mejor la intención sin solicitar una deduplicación adicional.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. Un cambio de IN/EXISTS a JOIN es válido solo si se mantiene la cardinalidad y la semántica. No afirmar mejoras de rendimiento sin comparar planes y resultados.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ07-A: un registro por cliente con algún pedido pagado.
SELECT c.ClienteID, c.Nombre
FROM dbo.Clientes AS c
WHERE EXISTS (
    SELECT 1 FROM dbo.Pedidos AS p
    WHERE p.ClienteID = c.ClienteID
      AND p.Estado = 'Pagado'
);
GO
-- EJ07-B: el JOIN devuelve una fila por coincidencia pedido-cliente.
SELECT c.ClienteID, c.Nombre
FROM dbo.Clientes AS c
INNER JOIN dbo.Pedidos AS p ON p.ClienteID = c.ClienteID
WHERE p.Estado = 'Pagado';
GO
-- EJ07-C: elimina duplicados para equiparar semántica de existencia.
SELECT DISTINCT c.ClienteID, c.Nombre
FROM dbo.Clientes AS c
INNER JOIN dbo.Pedidos AS p ON p.ClienteID = c.ClienteID
WHERE p.Estado = 'Pagado';
GO
```

### Antes de ejecutar: predice

1. ¿Qué consulta puede repetir ClienteID?
2. ¿Por qué EXISTS no necesita DISTINCT para devolver una fila por cliente?

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** EJ07-A y EJ07-C contienen los mismos clientes únicos. EJ07-B contiene tantas filas como coincidencias pagadas, por tanto normalmente más que clientes.

**Interpretación:** Un cambio de IN/EXISTS a JOIN es válido solo si se mantiene la cardinalidad y la semántica. No afirmar mejoras de rendimiento sin comparar planes y resultados.

### Variación controlada

Cambiar Estado a Cancelado y comparar qué clientes cumplen la condición. Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Quitar el predicado `p.ClienteID = c.ClienteID` del EXISTS: la subconsulta puede hallar pedidos de cualquier cliente y devolver resultados incorrectos. Reponer correlación.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Cuándo un JOIN cambia la cardinalidad, aun si enlaza correctamente dos tablas?

### T07 — Tarea espejo de EJ07

**Enunciado:** obtener ClienteID y Nombre de los clientes que tienen -- al menos un pedido Pendiente: primero EXISTS, luego JOIN con -- DISTINCT para conservar una fila por cliente.

**Qué modificar:** `estudiante/tareas/T07.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** Mantén la correlación por ClienteID y usa DISTINCT únicamente en la versión JOIN.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** Ambas consultas contienen el mismo conjunto de ClienteID, sin duplicados; pueden devolver números de filas idénticos.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T07: obtener ClienteID y Nombre de los clientes que tienen
-- al menos un pedido Pendiente: primero EXISTS, luego JOIN con
-- DISTINCT para conservar una fila por cliente.
-- TODO T07-A y T07-B. Compara ambos conjuntos.
GO
```


---

## EJ08 — Generar y comprobar datos de prueba en SQL Server

**Qué vamos a resolver.** Para probar consultas necesitamos lotes de datos controlados. ROW_NUMBER crea una secuencia y CHECKSUM(NEWID()) genera valores de prueba variables por fila. RAND() en INSERT no debe suponerse aleatorio por cada fila.

**Aprendizaje principal.** Generar datos ficticios en lote y validar integridad, rangos y variación.

**Artefactos:** `estudiante/ejemplos/EJ08.sql` y `estudiante/tareas/T08.sql` (opcional para practicar; el código completo de ejemplo aparece abajo).

**Punto de partida:** conectado a `Lab_ISIL_S07_Planes`, con el script de preparación ya ejecutado; sigue la secuencia EJ01–EJ08. Para mostrar un plan real en SSMS activa **Consulta > Incluir plan de ejecución real**, atajo **Ctrl+M**, y luego **Ejecutar (F5)**. Para estimado usa **Ctrl+L**.

### Paso 1 — Comprender la necesidad

**Problema actual:** Para probar consultas necesitamos lotes de datos controlados. ROW_NUMBER crea una secuencia y CHECKSUM(NEWID()) genera valores de prueba variables por fila. RAND() en INSERT no debe suponerse aleatorio por cada fila.

**Qué necesitamos crear o modificar:** las cláusulas SQL o los objetos descritos en la secuencia, usando exclusivamente la base aislada del laboratorio. **Por qué:** se requiere una evidencia observable y comparable. **Responsabilidad:** cada consulta debe producir un resultado SQL y una explicación de su plan o de sus datos.

### Paso 2 — Crear una tabla con restricciones de calidad

**Estado antes:** los objetos de laboratorio ya existen, pero no se ha agregado esta parte de la consulta.

**Escribe / configura ahora:**

```sql
CREATE TABLE dbo.MuestrasS07
(
 MuestraID int NOT NULL PRIMARY KEY,
 Etiqueta nvarchar(50) NOT NULL,
 Puntaje decimal(6,2) NOT NULL CHECK (Puntaje BETWEEN 0 AND 99.99)
);
```

**Explicación detallada:** MuestraID impide repeticiones de clave; Etiqueta permite identificar filas, y CHECK protege el rango. La condición IF OBJECT_ID evita recrear la tabla, de modo que el laboratorio es repetible.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 3 — Construir cien números de fila

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
WITH D AS (
 SELECT v FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS a(v)
), N AS (
 SELECT ROW_NUMBER() OVER (ORDER BY a.v,b.v) AS n
 FROM D AS a CROSS JOIN D AS b
)
```

**Explicación detallada:** D representa los diez dígitos; CROSS JOIN cruza diez por diez y produce cien combinaciones. ROW_NUMBER asigna identificadores 1..100 bajo el ORDER BY mostrado.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 4 — Generar un valor ficticio por fila

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
CAST((ABS(CAST(CHECKSUM(NEWID()) AS bigint)) % 10000) / 100.0
     AS decimal(6,2))
```

**Explicación detallada:** NEWID genera un GUID nuevo por fila; CHECKSUM obtiene un entero, convertido a bigint antes de ABS para evitar desbordamiento del entero mínimo. Módulo 10000 y división por 100 sitúan el valor en 0..99.99. No garantiza unicidad de Puntaje.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Paso 5 — Validar el lote insertado

**Estado antes:** el microbloque anterior define el contexto; falta introducir la nueva decisión técnica.

**Escribe / configura ahora:**

```sql
SELECT COUNT(*) AS Filas, COUNT(DISTINCT MuestraID) AS IDsDistintos,
       MIN(Puntaje) AS MinPuntaje, MAX(Puntaje) AS MaxPuntaje,
       COUNT(DISTINCT Puntaje) AS PuntajesDistintos
FROM dbo.MuestrasS07;
```

**Explicación detallada:** El control exige 100 filas y 100 identificadores distintos; el rango de Puntaje debe respetarse. PuntajesDistintos es diagnóstico de variación, no una promesa de 100 valores diferentes.

**Elementos nuevos y su función:** observa los identificadores y operadores presentes en el microbloque; `dbo` identifica el esquema, los alias designan las tablas, y las cláusulas cambian el conjunto de filas o la forma de calcularlo según la explicación anterior.

**Estado después:** esta parte de la consulta ya expresa la nueva condición o estructura. **Por qué avanzamos:** ahora se puede integrar con los otros microbloques, predecir y ejecutar la consulta completa.

### Traza manual y auditoría de lo construido

Antes de ejecutar, sigue mentalmente la ruta **conjunto fuente → proyección/filtro/relación → resultado → análisis del plan**. Son datos ficticios para probar consultas, no datos personales. ROW_NUMBER asegura IDs de la secuencia y CHECKSUM(NEWID()) aporta variación sin garantizar unicidad ni seguridad criptográfica.

**Auditoría de responsabilidades:** la fuente `FROM` identifica tablas; `SELECT` determina columnas o agregados; `WHERE`, `JOIN`, `GROUP BY`, `HAVING` y `ORDER BY` (solo los presentes) determinan filtros, vínculos, agrupación y orden. Los comentarios `--` son documentación y `GO` separa lotes en SSMS; no es una instrucción T-SQL del motor.

### Script completo al terminar el ejemplo

```sql
USE Lab_ISIL_S07_Planes;
GO
-- EJ08: tabla didáctica de datos ficticios, separada del negocio.
IF OBJECT_ID(N'dbo.MuestrasS07', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.MuestrasS07
    (
        MuestraID int NOT NULL PRIMARY KEY,
        Etiqueta nvarchar(50) NOT NULL,
        Puntaje decimal(6,2) NOT NULL CHECK (Puntaje BETWEEN 0 AND 99.99)
    );
END;
GO
-- Repetible: inserta solo la primera vez.
IF NOT EXISTS (SELECT 1 FROM dbo.MuestrasS07)
BEGIN
    WITH D AS (
        SELECT v FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9)) AS a(v)
    ), N AS (
        SELECT ROW_NUMBER() OVER (ORDER BY a.v,b.v) AS n
        FROM D AS a CROSS JOIN D AS b
    )
    INSERT INTO dbo.MuestrasS07 (MuestraID, Etiqueta, Puntaje)
    SELECT CAST(n AS int), CONCAT(N'Muestra ', n),
           CAST((ABS(CAST(CHECKSUM(NEWID()) AS bigint)) % 10000) / 100.0
               AS decimal(6,2))
    FROM N;
END;
GO
SELECT COUNT(*) AS Filas,
       COUNT(DISTINCT MuestraID) AS IDsDistintos,
       MIN(Puntaje) AS MinPuntaje,
       MAX(Puntaje) AS MaxPuntaje,
       COUNT(DISTINCT Puntaje) AS PuntajesDistintos
FROM dbo.MuestrasS07;
GO
```

### Antes de ejecutar: predice

1. ¿Habrá exactamente 100 filas?
2. ¿Deben ser únicos los 100 puntajes generados?

### Ejecuta y comprueba

En SSMS, selecciona el lote del ejemplo (sin abarcar otros ejercicios), activa **Ctrl+M** y pulsa **F5**. Si hay varias consultas, ejecútalas separadamente para identificar qué plan corresponde a cada una. Observa **Results**, **Messages** y **Execution plan**, abre el panel de propiedades de los operadores; registra en una captura la operación física realmente elegida.

**Resultado esperado:** 100 filas, 100 MuestraID únicos, Puntaje entre 0 y 99.99. Los puntajes no necesariamente son todos distintos.

**Interpretación:** Son datos ficticios para probar consultas, no datos personales. ROW_NUMBER asegura IDs de la secuencia y CHECKSUM(NEWID()) aporta variación sin garantizar unicidad ni seguridad criptográfica.

### Variación controlada

Consulta cuántos puntajes son mayores o iguales a 50. No esperes un número exacto constante entre laboratorios. Antes de ejecutar, anticipa qué filas, columnas u operadores podrían cambiar. Ejecuta y explica qué observaste; no confundas un cambio de plan posible con un resultado garantizado.

### Error o caso límite controlado

**Situación:** Usar ROUND(RAND()*100,2) directamente en INSERT SELECT esperando un precio distinto por fila: SQL Server asigna el mismo valor RAND a las filas afectadas por la sentencia. Emplea NEWID/CHECKSUM con controles de rango.

**Corrección razonada:** vuelve al último script completo funcional, aplica la corrección indicada y explica por qué el comportamiento cambia. No ejecutes operaciones destructivas ni cambies bases ajenas.

### Compruébalo con tus palabras

¿Por qué la comprobación de datos generados debe validar restricciones y no exigir aleatoriedad perfecta?

### T08 — Tarea espejo de EJ08

**Enunciado:** crea dbo.MuestrasT08 con columnas ID int PK, -- Codigo nvarchar(40) NOT NULL y Nivel int CHECK entre 1 y 5. -- Inserta 100 datos si la tabla está vacía usando D (0..9), -- CROSS JOIN, ROW_NUMBER, CHECKSUM(NEWID()) y una conversión -- segura a bigint antes de ABS. Comprueba COUNT y MIN/MAX.

**Qué modificar:** `estudiante/tareas/T08.sql` o una ventana nueva de SSMS. **Restricción:** trabajar solo en la base del laboratorio y respetar los nombres de tablas y columnas establecidos.

**Pista:** `1 + (valor % 5)` produce 1..5; no exijas que haya las cinco categorías en todas las muestras, aunque es muy probable.

**Evidencia para entregar:** archivo SQL de la tarea, captura o exportación del resultado, captura del plan cuando corresponda y explicación breve de 3–5 oraciones.

**Criterio de logro:** 100 filas y 100 IDs únicos; MIN(Nivel) >= 1 y MAX(Nivel) <= 5.

**Estructura inicial de tarea:**

```sql
USE Lab_ISIL_S07_Planes;
GO
-- T08: crea dbo.MuestrasT08 con columnas ID int PK,
-- Codigo nvarchar(40) NOT NULL y Nivel int CHECK entre 1 y 5.
-- Inserta 100 datos si la tabla está vacía usando D (0..9),
-- CROSS JOIN, ROW_NUMBER, CHECKSUM(NEWID()) y una conversión
-- segura a bigint antes de ABS. Comprueba COUNT y MIN/MAX.
-- TODO T08: creación, generación y validación.
GO
```

---

## 6. Cierre docente y conexión académica

Durante esta sesión construiste un entorno de trabajo reproducible y lo usaste para observar una misma consulta desde dos perspectivas: el conjunto de datos que devuelve y la estrategia que eligió SQL Server para producirlo. Al comparar un plan estimado con un plan real reconociste qué información solo puede obtenerse después de ejecutar. También comprobaste cómo una cláusula ORDER BY expresa un requisito de ordenación que puede implicar una operación Sort y por qué un índice no garantiza siempre un operador Seek.

Después interpretaste accesos, relaciones entre tablas y agregaciones, separando la operación lógica del algoritmo físico. Aprendiste a analizar claves, filtros y columnas incluidas en un índice; a reescribir condiciones sobre fechas sin transformar la columna; y a decidir entre EXISTS y JOIN sin modificar inadvertidamente la cantidad de entidades devueltas. Finalmente generaste información ficticia con una numeración estable y valores de prueba variables, comprobando integridad y rangos en lugar de confiar únicamente en que el script terminó sin errores.

El siguiente paso es revisar tus ocho tareas y ser capaz de defender cada decisión: qué se pretendía obtener, por qué la sentencia está escrita de esa forma, qué resultado esperabas, qué observaste y qué harías si no coincidiera. Repasa especialmente las propiedades del plan y las diferencias entre costo estimado, filas reales y orden de salida. No evalúes el rendimiento con una sola captura ni confundas ausencia de Scan con éxito universal.

Para reforzar los conceptos, utiliza los recursos formativos de **Lideratec Academy**: el blog https://lideratecacademy.com/blog/ y el canal público https://www.youtube.com/@LideratecAcademy. El laboratorio materializado de FASE 15 puede servir después como apoyo opcional, pero esta guía contiene todos los pasos necesarios para realizar la práctica desde cero.
