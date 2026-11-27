# FASE 13 — Sesión 13: Implementando la seguridad en SQL Server

**Curso:** Programación Avanzada de Base de Datos · **Unidad:** Seguridad, copias de seguridad y restauración (UA4) · **Duración sugerida:** 120 minutos · **Entorno:** SQL Server y SQL Server Management Studio (SSMS).

**Base de práctica:** `S13_Seguridad_Lab`, aislada de `Northwind_S02_Lab` para no modificar los datos que se usan en sesiones anteriores. Cuando no se tengan permisos para crear una base de datos, el docente debe preparar previamente la base de práctica; no ejecutar scripts de seguridad en producción.

## Propósito y objetivo de la sesión

En una organización, no todas las personas deben poder consultar, modificar y administrar los mismos datos. El problema de esta sesión consiste en **distinguir la autenticación de la autorización y materializar el principio de privilegio mínimo**. Primero identificaremos la forma de autenticación disponible, después prepararemos objetos de prueba, crearemos identidades de base de datos, agruparemos permisos en roles, probaremos lo que ocurre cuando una operación está permitida o prohibida y examinaremos los metadatos de seguridad. Finalmente, crearemos una demostración controlada de claves y certificados sin tratar el ejercicio como una estrategia lista para producción.

Al finalizar, cada participante podrá: (1) explicar la diferencia entre inicio de sesión (`LOGIN`) y usuario de base de datos (`USER`); (2) consultar la configuración de autenticación de la instancia; (3) crear y verificar usuarios de laboratorio sin acceso de conexión; (4) crear roles personalizados; (5) conceder y comprobar permisos específicos; (6) diagnosticar restricciones con un contexto de ejecución simulado; (7) auditar las asignaciones mediante vistas de catálogo; (8) reconocer claves y certificados y validar un cifrado/descifrado educativo.

## Delimitación respecto de la presentación fuente

La presentación aborda **control de acceso, autenticación Windows y SQL Server, roles de servidor y de base de datos, privilegios mínimos, segregación de funciones, auditoría y monitoreo, cifrado, claves, certificados y protección de material criptográfico**. Los ejercicios T-SQL siguientes son una **ampliación didáctica construida para practicar esos conceptos**: la presentación no proporciona estos scripts ni sugiere operar sobre una base real. Los ejemplos de permisos y claves son para una instancia de laboratorio y requieren privilegios de preparación.

## Requisitos y reglas de seguridad

- SSMS conectado a una instancia SQL Server de práctica; permisos de `CREATE DATABASE` para EJ01 o base precreada por el docente.
- EJ02 requiere consultar metadatos; resultados de visibilidad dependen de permisos. EJ03–EJ08 requieren que el preparador posea `db_owner` en la base aislada; las pruebas `EXECUTE AS USER` también requieren autorización para impersonar al usuario (habitualmente incluida en este montaje).
- **No habilitar el modo mixto, no crear cuentas `sa`, no otorgar `sysadmin` ni usar contraseñas compartidas para una actividad académica**. La autenticación SQL se explica y se compara, pero no se obliga a activarla.
- La contraseña de la clave maestra en EJ08 se escribe localmente **reemplazando un marcador** por una frase robusta elegida por el docente; no publicar ni entregar ese valor. El laboratorio criptográfico **no constituye una configuración de producción**.
- `GO` separa lotes en SSMS. Ejecuta cada archivo del ejemplo completo **una sola vez** siguiendo el orden; reiniciar el laboratorio requiere una instancia aislada y la intervención del docente.

## Agenda de 120 minutos

| Bloque | Minutos | Actividad |
|---|---:|---|
| Apertura y diagnóstico | 10 | Acceso vs. autorización; panorama de la fuente. |
| EJ01–EJ02 | 20 | Entorno aislado y modalidades de autenticación. |
| EJ03–EJ04 | 25 | Usuarios y roles. |
| EJ05–EJ06 | 30 | Pruebas de privilegios y segregación. |
| EJ07 | 15 | Inventario y trazabilidad de permisos. |
| EJ08 | 15 | Certificado, clave simétrica y lectura crítica. |
| Cierre | 5 | Evidencias y criterios de seguridad. |

**Modalidad de tareas espejo:** se enuncian durante la clase y se completan como comprobación rápida o trabajo autónomo. Si el grupo necesita más tiempo, las ocho tareas constituyen la práctica posterior, no se elimina la explicación de los ejemplos.


## Método común: construir → explicar → predecir → ejecutar → interpretar → transferir

Trabaja con una pestaña de SSMS por ejemplo. Antes de presionar **Ejecutar**, lee la predicción. Ejecuta el bloque completo respetando las instrucciones `GO`; compara los resultados con la expectativa y registra una captura o salida de texto. Al terminar cada ejemplo encontrarás una tarea espejo de aplicación.


---


## Mapa de dependencia: por qué los ejemplos tienen este orden

| Primero | Después | Relación causal |
|---|---|---|
| EJ01: base y dos tablas | EJ02: identidad de conexión | Primero tenemos un entorno seguro, después averiguamos quién realiza las operaciones. |
| EJ02: login y usuario actual | EJ03: usuarios sin login | La distinción conceptual permite crear identidades para simular permisos sin abrir conexiones nuevas. |
| EJ03: usuarios | EJ04: rol y `GRANT` | No puede agregarse al rol un usuario que aún no existe; el rol se define antes de asignarle permisos. |
| EJ04: autorización declarada | EJ05: autorización efectiva | Una concesión en catálogo necesita contrastarse mediante una operación bajo el usuario limitado. |
| EJ05: lector | EJ06: auditor | La separación se entiende al comparar dos usuarios con permisos diferentes sobre dos objetos. |
| EJ06: separación efectiva | EJ07: inventario | Tras las pruebas operativas, registramos qué concesiones y membresías explican el resultado. |
| EJ07: control de acceso | EJ08: cifrado | Se diferencia quién puede acceder de cómo se protege criptográficamente un dato. |

**Regla de lectura:** No ejecutes todas las instrucciones sin interpretarlas. En cada paso, formula la hipótesis, explica cada línea, ejecuta, observa y justifica por qué el resultado se relaciona con el objetivo.

## EJ01 — Preparar una base aislada y objetos con distinta sensibilidad (12 min)

**Vínculo con la presentación:** diapositivas 4–9. **Objetivo observable:** Necesitamos objetos con propósitos diferentes para demostrar por qué el acceso a una tabla no debe equivaler al acceso indiscriminado a toda la base.

**Antes:** No existe la base `S13_Seguridad_Lab`; conectarse con una cuenta administradora a una instancia de práctica.


### Paso 1 — Crear el ámbito

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Antes de hablar de permisos tenemos que saber dónde estamos trabajando. Una instrucción SQL no actúa sobre una idea abstracta de ‘mi base’, sino sobre un contexto concreto. Si hoy ejecutamos `CREATE USER` en otra base, SQL Server podría aceptar la orden y aun así estaríamos protegiendo el lugar equivocado. Por eso nuestra primera operación no es crear un usuario: es delimitar el escenario. Escribimos `USE master` para partir de un contexto conocido y consultamos `DB_ID` como pregunta: ‘¿ya existe la base de práctica?’. Si la respuesta es `NULL`, creamos la base; si existe y es visible, la conservamos. Así evitamos el error de intentar crear dos veces la misma base.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

Miren ahora los `GO`: son separadores que interpreta SSMS, no órdenes de seguridad ni sentencias que ejecuta SQL Server. El `GO` posterior a `CREATE DATABASE` permite que el siguiente lote seleccione la nueva base. Al final escribimos `USE S13_Seguridad_Lab`. Esa línea es decisiva: las tablas, los usuarios y los roles que crearemos después serán objetos de esa base. El mensaje ‘Comandos completados correctamente’ no demuestra todavía dónde quedamos; lo comprobamos con `SELECT DB_NAME() AS BaseActiva;`. Si no aparece `S13_Seguridad_Lab`, no seguimos. Esta consulta de verificación sirve para evitar errores silenciosos de contexto, no para conceder acceso.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no crear las tablas directamente en `master`? Porque mezclar objetos de enseñanza con una base de sistema aumenta el riesgo y hace confusa la administración. ¿Qué pasaría si omitiéramos el último `USE`? El siguiente `CREATE TABLE` se ejecutaría en el contexto que tenga la ventana; podríamos obtener un resultado aparentemente correcto en el lugar incorrecto.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Establecer una base aislada antes de crear tablas, usuarios o permisos.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Primero delimitamos dónde trabajaremos; sin esta separación los próximos CREATE USER y GRANT podrían aplicarse a una base equivocada.

**Explicación detallada, instrucción por instrucción:**

1. `USE master` cambia temporalmente a la base de sistema para ejecutar la comprobación de existencia desde un contexto conocido. No modifica por sí solo otras bases.
2. `GO` indica a SSMS el fin del lote; no es una instrucción T-SQL enviada al motor.
3. `DB_ID(N'S13_Seguridad_Lab')` obtiene el identificador de la base si existe; devuelve `NULL` si no está disponible en el contexto de visibilidad adecuado.
4. `IF ... IS NULL CREATE DATABASE` evita intentar recrear la base en una segunda ejecución. `CREATE DATABASE` necesita permiso específico de instancia.
5. `USE S13_Seguridad_Lab` selecciona el destino de las instrucciones siguientes.

**Qué debes observar después de ejecutarla:** En SSMS ejecuta `SELECT DB_NAME() AS BaseActiva;`: debe indicar `S13_Seguridad_Lab`.

**Cómo interpretar lo observado:** Crear la base no otorga todavía permisos a usuarios de laboratorio; solo prepara el contenedor.

**Precaución y error frecuente:** Si omites el segundo `USE`, el ejemplo siguiente podría crear tablas en otra base.


**Qué vamos a lograr y por qué se hace ahora.** `DB_ID` permite crear la base solo si falta; `USE` fija el contexto antes de definir objetos. `GO` evita mezclar cambios de contexto y DDL en un mismo lote.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE master;
GO
IF DB_ID(N'S13_Seguridad_Lab') IS NULL
    CREATE DATABASE S13_Seguridad_Lab;
GO
USE S13_Seguridad_Lab;
GO
```



### Paso 2 — Construir datos públicos y sensibles

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Ya hemos asegurado el contenedor, pero una base vacía no permite demostrar qué información debería recibir cada persona. Vamos a representar un caso de negocio mínimo con dos clases de datos. `PedidosLab` es información operativa que podría necesitar un lector de pedidos. `AuditoriaLab` representa información de seguimiento y queremos reservarla para otra función. La finalidad de escribir dos `CREATE TABLE` es fabricar dos objetos sobre los cuales podremos tomar decisiones de autorización distintas; no estamos activando la característica de auditoría de SQL Server.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

Leamos la primera condición: `OBJECT_ID(N'dbo.PedidosLab', N'U') IS NULL` pregunta si existe una tabla de usuario con ese nombre. Solo si falta entramos al `BEGIN...END`, creamos sus columnas e insertamos dos filas. La clave primaria impide identificadores repetidos, `NOT NULL` exige valores y `decimal(10,2)` representa importes con dos decimales. Repetimos la estrategia para `AuditoriaLab`, esta vez con una fila. Colocar `INSERT` dentro del `IF` es importante: si corremos dos veces el ejemplo, no duplicamos el conjunto inicial.

**3. ¿Por qué usar esta consulta y no otra?**

Detengámonos en la decisión de seguridad: que dos tablas estén bajo `dbo` no significa que debamos dar el mismo permiso sobre ambas. Tampoco implica que ya haya restricciones; todavía no existe nuestra política de roles. El objetivo de este paso es preparar recursos observables para que un `GRANT` posterior tenga sentido. Si creáramos una sola tabla, no podríamos mostrar de forma convincente que un usuario puede leer un objeto y no otro.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Crear dos recursos con distinto nivel de acceso: pedidos y eventos de auditoría.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Solo podemos demostrar privilegios mínimos si tenemos al menos dos objetos que deban separarse.

**Explicación detallada, instrucción por instrucción:**

1. `OBJECT_ID(..., N'U')` busca una tabla de usuario. La condición `IS NULL` evita duplicar su definición en una repetición normal del ejercicio.
2. `CREATE TABLE dbo.PedidosLab` define una tabla operativa. `PedidoID int PRIMARY KEY` identifica de forma única el pedido; `Cliente nvarchar(60)` guarda texto Unicode; `Total decimal(10,2)` conserva precisión monetaria de demostración.
3. `INSERT` incorpora exactamente dos pedidos de ejemplo; la inserción está dentro del `BEGIN...END` para no duplicarlos al ejecutar nuevamente.
4. `CREATE TABLE dbo.AuditoriaLab` define un segundo objeto con su clave `EventoID` y la descripción del evento; su fila inicial sirve para comprobar permisos de lectura más adelante.
5. Los nombres bajo `dbo` son de la misma base, pero no deben asumir el mismo nivel de acceso.

**Qué debes observar después de ejecutarla:** Comprueba en el Explorador de objetos las dos tablas y sus columnas. Las inserciones iniciales deben efectuarse una sola vez por tabla.

**Cómo interpretar lo observado:** La separación de objetos no restringe todavía el acceso: la seguridad la establecerán los roles y `GRANT` de EJ04 y EJ06.

**Precaución y error frecuente:** No confundir el nombre `AuditoriaLab` con un sistema de auditoría activado; solo es una tabla de muestra.


**Qué vamos a lograr y por qué se hace ahora.** `PedidosLab` representa datos operativos; `AuditoriaLab` representa un registro de evidencia que no debe estar disponible para cualquier lector.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
IF OBJECT_ID(N'dbo.PedidosLab', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.PedidosLab
    (PedidoID int PRIMARY KEY, Cliente nvarchar(60) NOT NULL, Total decimal(10,2) NOT NULL);
    INSERT dbo.PedidosLab VALUES (1,N'Ana',120.00),(2,N'Luis',245.50);
END;
IF OBJECT_ID(N'dbo.AuditoriaLab', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.AuditoriaLab
    (EventoID int PRIMARY KEY, Descripcion nvarchar(100) NOT NULL);
    INSERT dbo.AuditoriaLab VALUES (1,N'Inicio de laboratorio');
END;
```



### Paso 3 — Comprobar las bases del caso

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Antes de crear usuarios, comprobemos el estado de partida. Si en la siguiente práctica un lector hace `SELECT` y no devuelve filas, existen al menos dos explicaciones posibles: la tabla está vacía o la consulta se ejecutó con un filtro. No debemos deducir de una cuadrícula vacía que el permiso falló. Por eso ahora hacemos una lectura administrativa del contenido inicial, cuando todavía controlamos el contexto de preparación.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

El primer `SELECT` cuenta las filas de `PedidosLab` y rotula el resultado como `Pedidos`. `COUNT(*)` devuelve una cantidad; no consulta permisos. El segundo `SELECT` cuenta `AuditoriaLab`. `UNION ALL` coloca ambos resúmenes uno debajo del otro sin exigir que los nombres de las tablas sean iguales: exige que las expresiones de ambas ramas sean compatibles. Esperamos `Pedidos = 2` y `Auditoria = 1` en un laboratorio nuevo. Si vemos otro número, debemos investigar si la base existía con datos previos, no alterar la cifra para que coincida con la guía.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no usar `SELECT *` de las dos tablas? Podríamos hacerlo, pero el conteo nos da una línea base breve y suficiente para verificar que la práctica no comenzó con objetos vacíos. Guardaremos mentalmente esta distinción: un resultado correcto obtenido con una cuenta administradora prueba que los datos existen, no que un usuario limitado puede consultarlos.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Medir que los datos base están cargados antes de probar autorizaciones.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Sin datos de referencia, un `SELECT` posterior podría devolver cero filas aunque el usuario tenga permiso, provocando una interpretación incorrecta.

**Explicación detallada, instrucción por instrucción:**

1. La primera consulta `SELECT N'Pedidos' AS Objeto, COUNT(*) AS Filas FROM dbo.PedidosLab` crea una fila resumen con etiqueta y cantidad de pedidos.
2. `UNION ALL` apila el resumen de auditoría sin eliminar duplicados; ambas consultas devuelven dos columnas compatibles.
3. `COUNT(*)` cuenta filas, no permisos. El usuario preparador consulta ambas tablas con su propio contexto.

**Qué debes observar después de ejecutarla:** La cuadrícula muestra `Pedidos | 2` y `Auditoria | 1` en una instalación nueva.

**Cómo interpretar lo observado:** Esta salida prueba la preparación de datos, no la restricción de acceso. La diferencia se verificará con identidades limitadas en EJ05 y EJ06.

**Precaución y error frecuente:** Si no coincide el conteo, comprueba qué tablas se habían creado y evita añadir registros aleatorios para forzar el resultado.


**Qué vamos a lograr y por qué se hace ahora.** El conteo confirma dos filas operativas y un evento de auditoría. No es aún una prueba de permisos.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
SELECT N'Pedidos' AS Objeto, COUNT(*) AS Filas FROM dbo.PedidosLab
UNION ALL
SELECT N'Auditoria', COUNT(*) FROM dbo.AuditoriaLab;
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** Dos filas en PedidosLab y una en AuditoriaLab; la base aislada queda disponible.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T01 — Tarea espejo de EJ01

**Situación y encargo:** Crear `dbo.ConfiguracionLab(Clave nvarchar(30) PRIMARY KEY, Valor nvarchar(60))` con una fila `(Entorno,Practica)`, de forma idempotente, y verificar el conteo.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** Filas = 1.


---

## EJ02 — Diferenciar autenticación Windows, SQL y autorización (8 min)

**Vínculo con la presentación:** diapositivas 10–15. **Objetivo observable:** Antes de otorgar permisos debemos saber cómo una identidad llega a la instancia. Una conexión válida no concede automáticamente privilegios sobre las tablas.

**Antes:** EJ01 disponible; nueva ventana de consulta en SSMS con la conexión actual.


### Paso 1 — Examinar identidad y modo de instancia

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Ya tenemos datos. Ahora la pregunta cambia: ¿quién está ejecutando estas instrucciones? No podemos interpretar los futuros permisos sin distinguir la identidad de la conexión y el usuario efectivo. Este `SELECT` es de observación: no crea cuentas ni modifica la configuración del servidor. Nos entrega cuatro indicadores que usaremos para explicar autenticación, contexto y autorización.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`ORIGINAL_LOGIN()` devuelve el inicio de sesión que abrió la conexión. `SUSER_SNAME()` identifica el inicio de sesión del contexto actual, que en escenarios con cambios de contexto puede diferir del original. `DB_NAME()` nos dice dónde estamos ejecutando el bloque. Finalmente `SERVERPROPERTY('IsIntegratedSecurityOnly')` consulta si la instancia está configurada únicamente para autenticación Windows: normalmente `1` indica solo Windows y `0` permite modo mixto. Observar el modo no activa ni desactiva nada. En una instalación con permisos restringidos ciertos metadatos pueden ser inaccesibles o variar.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué empezar por preguntar en vez de ejecutar `CREATE LOGIN`? Porque modificar la autenticación del servidor excede la necesidad de esta lección. Queremos aprender a interpretar el mecanismo disponible sin crear credenciales reales. Que la conexión haya funcionado prueba autenticación, pero no prueba que esa misma identidad tenga derecho a leer `PedidosLab`. La autorización la comprobamos después, con un usuario limitado.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Identificar quién se conectó y qué modalidades de autenticación admite la instancia.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Antes de crear usuarios, distinguimos cómo entra una identidad a SQL Server de qué puede hacer una vez dentro.

**Explicación detallada, instrucción por instrucción:**

1. `ORIGINAL_LOGIN()` identifica el principal con el que se inició la conexión; puede diferir de la identidad efectiva tras una suplantación de contexto.
2. `SUSER_SNAME()` informa el principal actual a nivel de servidor; sirve para comparar el contexto presente con el original.
3. `DB_NAME()` informa la base activa, para prevenir interpretaciones fuera del lugar esperado.
4. `SERVERPROPERTY('IsIntegratedSecurityOnly')` devuelve `1` cuando solo se admite autenticación Windows y `0` en modo mixto; no cambia la configuración.

**Qué debes observar después de ejecutarla:** Observa nombres de login y base, y uno de los valores `0` o `1`; no se exige un valor universal.

**Cómo interpretar lo observado:** Autenticación es la comprobación de identidad al entrar; no equivale a `SELECT` o `UPDATE` sobre las tablas.

**Precaución y error frecuente:** El modo mixto no se debe activar por este ejercicio; solo se consulta.


**Qué vamos a lograr y por qué se hace ahora.** `ORIGINAL_LOGIN()` muestra el login que inició la conexión y `SUSER_SNAME()` el login actual. La propiedad 1 indica solo Windows; 0 indica modo mixto (Windows y SQL). No es permiso para cambiar esa configuración.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
SELECT ORIGINAL_LOGIN() AS LoginOriginal,
       SUSER_SNAME() AS LoginEnContexto,
       DB_NAME() AS BaseActual,
       SERVERPROPERTY('IsIntegratedSecurityOnly') AS SoloAutenticacionWindows;
```



### Paso 2 — Distinguir login y usuario

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

En el paso anterior vimos la identidad de conexión. Ahora entra un segundo nombre que suele confundirse con ella: el usuario de una base de datos. SQL Server distingue un principal de nivel servidor —el `LOGIN`— y un principal de nivel base —el `USER`. Es habitual que estén relacionados, pero no significan lo mismo; además, nuestro laboratorio utilizará usuarios `WITHOUT LOGIN` para practicar permisos sin abrir sesiones externas.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

Primero ejecutamos `USE S13_Seguridad_Lab` porque `USER_NAME()` se interpreta dentro de una base concreta. Después mostramos `SUSER_SNAME()` como `LoginInstancia`, `USER_NAME()` como `UsuarioEnBase` y `DB_NAME()` como prueba del contexto. Las tres columnas no compiten: responden preguntas distintas. `SUSER_SNAME()` responde ‘¿bajo qué identidad de servidor opero?’; `USER_NAME()` responde ‘¿qué principal represento aquí?’; `DB_NAME()` responde ‘¿dónde se evalúan mis permisos?’.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no mostrar solo `SYSTEM_USER` o solo `USER_NAME()`? Porque una sola función ocultaría la diferencia de niveles. Esta comparación prepara EJ03: cuando creemos `lector_s13 WITHOUT LOGIN`, estaremos creando una identidad utilizable dentro de la base para pruebas, no una cuenta que pueda iniciar sesión por sí sola.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Comparar el principal de instancia con el usuario de base de datos.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Después de reconocer el acceso al servidor, identificamos la identidad que SQL Server emplea para autorizar operaciones dentro de la base.

**Explicación detallada, instrucción por instrucción:**

1. `USE S13_Seguridad_Lab` fija la base en la que se resolverá `USER_NAME()`.
2. `SUSER_SNAME()` muestra el nombre de login o principal de servidor en contexto.
3. `USER_NAME()` devuelve el usuario del contexto actual en esa base; el resultado puede ser `dbo` para el preparador y no coincidir literalmente con el login.
4. `DB_NAME()` confirma que los datos se interpretan en el laboratorio.

**Qué debes observar después de ejecutarla:** Se obtiene una fila con login, usuario y `S13_Seguridad_Lab`.

**Cómo interpretar lo observado:** El modelo es identidad autenticada en instancia → usuario/mapeo en base → membresías y permisos. Este ejercicio aún no concede acceso.

**Precaución y error frecuente:** Un login y un usuario de base no tienen por qué compartir nombre.


**Qué vamos a lograr y por qué se hace ahora.** El mismo contexto de conexión puede corresponder a una identidad de servidor y a un usuario de base. Esto prepara el modelo LOGIN → USER → ROL → PERMISO.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE S13_Seguridad_Lab;
GO
SELECT SUSER_SNAME() AS LoginInstancia,
       USER_NAME() AS UsuarioEnBase,
       DB_NAME() AS BaseActual;
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** Se observa el login original, el usuario de base actual y un valor 0 o 1 según configuración.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T02 — Tarea espejo de EJ02

**Situación y encargo:** Ejecutar una consulta que informe la autenticación configurada con etiquetas textuales, sin modificar la instancia.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** Una fila con la modalidad de la instancia.


---

## EJ03 — Crear usuarios de laboratorio sin LOGIN y entender su límite (12 min)

**Vínculo con la presentación:** diapositivas 11–15. **Objetivo observable:** Un usuario de base de datos no equivale por sí solo a una cuenta con la que alguien pueda conectarse. Probaremos permisos sin crear credenciales ni abrir el acceso a la instancia.

**Antes:** Conectado como preparador en `S13_Seguridad_Lab`.


### Paso 1 — Crear identidades de prueba

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Identificamos la diferencia entre `LOGIN` y `USER`. Es momento de disponer de dos identidades de práctica sin abrir accesos de red ni distribuir contraseñas. Necesitaremos una persona lógica que lea pedidos y otra que examine registros de auditoría. No creamos roles todavía, porque primero conviene que existan los sujetos a quienes vamos a asociar las funciones.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`DATABASE_PRINCIPAL_ID(N'lector_s13')` busca el principal dentro de la base activa. Si no existe, `CREATE USER lector_s13 WITHOUT LOGIN` lo registra. Hacemos lo mismo para `auditor_s13`. La condición evita que una segunda ejecución intente crear el usuario con un nombre repetido. `WITHOUT LOGIN` significa que este usuario no tiene un inicio de sesión de servidor asociado; no dispone de credenciales para conectarse directamente. Para las pruebas, el preparador autorizado podrá cambiar temporalmente de contexto mediante `EXECUTE AS USER`.

**3. ¿Por qué usar esta consulta y no otra?**

Es importante no confundir creación con privilegio: estos usuarios existen, pero no les hemos concedido `SELECT` sobre ninguna tabla de negocio. Tampoco hemos abierto una puerta al servidor. Si alguien pregunta por qué no crear `LOGIN` reales, la respuesta es que nuestro objetivo es estudiar autorización sobre objetos, no administrar contraseñas ni el acceso remoto a la instancia.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Crear identidades sin credenciales que sirvan para simular permisos de menor privilegio.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Tras observar el contexto real en EJ02, necesitamos usuarios distintos para comparar qué puede consultar cada uno sin crear cuentas de acceso a la instancia.

**Explicación detallada, instrucción por instrucción:**

1. `USE ...; GO` ubica la creación en la base aislada.
2. `DATABASE_PRINCIPAL_ID(N'lector_s13')` comprueba si existe el usuario o principal nombrado y permite que el bloque pueda repetirse sin un error de nombre duplicado.
3. `CREATE USER lector_s13 WITHOUT LOGIN` crea un usuario solo en la base; no crea un `LOGIN` ni habilita iniciar sesión directamente.
4. El segundo par `IF/CREATE USER` repite el patrón para `auditor_s13`, separando los dos perfiles que recibirán roles distintos.

**Qué debes observar después de ejecutarla:** Las dos instrucciones `CREATE USER` no producen una tabla de resultados; confirma su creación en el siguiente paso.

**Cómo interpretar lo observado:** Se crean identidades, NO permisos sobre tablas. Los permisos se incorporarán después a través de roles.

**Precaución y error frecuente:** Estos usuarios no pueden conectarse por sí solos; `EXECUTE AS USER` requerirá un preparador autorizado.


**Qué vamos a lograr y por qué se hace ahora.** `WITHOUT LOGIN` crea identidades confinadas al contexto de la base, apropiadas para `EXECUTE AS USER`; **no pueden iniciar sesión directamente** en SQL Server.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE S13_Seguridad_Lab;
GO
IF DATABASE_PRINCIPAL_ID(N'lector_s13') IS NULL
    CREATE USER lector_s13 WITHOUT LOGIN;
IF DATABASE_PRINCIPAL_ID(N'auditor_s13') IS NULL
    CREATE USER auditor_s13 WITHOUT LOGIN;
```



### Paso 2 — Comprobar su tipo

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

La instrucción anterior no devolvió una tabla con las identidades creadas. En SQL Server, el mensaje de éxito de un `CREATE USER` informa que la sentencia terminó, pero para explicar qué tipo de principal existe preferimos consultar el catálogo. Por eso esta segunda consulta es verificadora, no repetición del alta.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`sys.database_principals` expone metadatos de los principales de la base. Seleccionamos `name`, `type_desc` y `authentication_type_desc` para identificar el nombre, la clase de principal y el tipo de autenticación registrado. `WHERE name IN (...)` limita la salida a los dos usuarios del ejercicio, mientras `ORDER BY name` facilita compararlos. No veremos contraseñas porque no se han creado credenciales de conexión.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no basta con verlos en el Explorador de objetos? Porque una consulta reproducible deja evidencia concreta, filtra exactamente lo que interesa y se puede guardar junto con el trabajo. Si no aparece alguno, primero comprobaremos la base activa y si el bloque anterior se ejecutó; no crearemos un login para intentar ‘arreglar’ algo que no lo requiere.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Comprobar en el catálogo que los usuarios de prueba fueron creados.

**¿Por qué ejecutamos esta consulta ahora y no otra?** No asignaremos roles sobre nombres supuestos: primero corroboramos que existen los dos principales.

**Explicación detallada, instrucción por instrucción:**

1. `sys.database_principals` contiene metadatos sobre usuarios, roles y otros principales de la base activa.
2. `name` identifica el principal; `type_desc` explica su tipo; `authentication_type_desc` describe cómo se autentica.
3. `WHERE name IN (...)` limita el informe a las identidades creadas por EJ03.
4. `ORDER BY name` ordena para comparar las dos filas fácilmente.

**Qué debes observar después de ejecutarla:** Deben aparecer `auditor_s13` y `lector_s13`, típicamente como usuarios SQL sin login con autenticación `NONE`.

**Cómo interpretar lo observado:** Ver un usuario en el catálogo solo demuestra existencia; no demuestra que tenga SELECT sobre PedidosLab.

**Precaución y error frecuente:** No consultar `sys.server_principals` para demostrar la existencia de estos usuarios: son principales de base.


**Qué vamos a lograr y por qué se hace ahora.** La vista de catálogo verifica los principales creados. Contrastar con el inicio de sesión (entidad de nivel servidor) que se explicó en EJ02.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
SELECT name, type_desc, authentication_type_desc
FROM sys.database_principals
WHERE name IN (N'lector_s13',N'auditor_s13')
ORDER BY name;
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** Dos usuarios presentes, sin credenciales de conexión propias.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T03 — Tarea espejo de EJ03

**Situación y encargo:** Crear `supervisor_s13 WITHOUT LOGIN` y verificar su existencia en `sys.database_principals`.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** Una fila supervisor_s13.


---

## EJ04 — Crear un rol lector con privilegios mínimos (13 min)

**Vínculo con la presentación:** diapositivas 5–7, 17–19. **Objetivo observable:** Asignar privilegios a cada usuario vuelve difícil administrar equipos. Un rol personalizado reúne el derecho estrictamente necesario: leer pedidos.

**Antes:** EJ01 y EJ03 terminados.


### Paso 1 — Declarar rol y membresía

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Tenemos dos usuarios de base. Si fuéramos asignando permisos tabla por tabla a cada persona, terminaríamos con una configuración difícil de revisar. Introducimos el rol como agrupación de permisos: el lector no recibirá cada permiso de manera directa; pertenecerá a un rol que representa una función del negocio. Así, cuando en el futuro tengamos más lectores, podremos incorporarlos al mismo rol.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`DATABASE_PRINCIPAL_ID` comprueba si `rol_lectura_pedidos` ya existe. `CREATE ROLE` lo da de alta si es necesario. Luego `IS_ROLEMEMBER('rol_lectura_pedidos','lector_s13')` pregunta si el usuario es miembro; solo si devuelve `0` agregamos la membresía con `ALTER ROLE ... ADD MEMBER`. Observa el orden: el usuario fue creado en EJ03, el rol se crea ahora y la relación de pertenencia se establece después. Si la función devuelve `NULL`, puede haber un nombre no resoluble o una condición de visibilidad: no equivale automáticamente a ‘no es miembro’.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no usar `db_datareader`? Porque ese rol fijo tiene un alcance de lectura mucho más amplio del que necesita nuestro caso. Aquí buscamos explicar el principio de mínimo privilegio: que el rol exista no lo autoriza todavía a leer pedidos. El permiso concreto aparecerá en el siguiente paso, donde completaremos la cadena usuario → rol → derecho sobre objeto.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Crear un rol y asociar el usuario lector a ese rol.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Ya tenemos a `lector_s13`; ahora definimos una agrupación administrativa a la que le otorgaremos un permiso concreto en el paso siguiente.

**Explicación detallada, instrucción por instrucción:**

1. `DATABASE_PRINCIPAL_ID` evita crear de nuevo un rol existente.
2. `CREATE ROLE rol_lectura_pedidos` registra un principal de tipo rol dentro de la base.
3. `IS_ROLEMEMBER(rol, usuario)` comprueba la pertenencia: `0` significa que no es miembro, `1` que sí; `NULL` indica una condición que debe diagnosticarse (por ejemplo nombre no resoluble).
4. `ALTER ROLE ... ADD MEMBER lector_s13` incorpora al usuario en el rol y permite que herede sus permisos cuando estos se concedan.

**Qué debes observar después de ejecutarla:** El rol y la relación de membresía quedan registrados, pero todavía no se ha ejecutado el `GRANT`.

**Cómo interpretar lo observado:** Crear un rol no significa que los integrantes puedan leer tablas automáticamente.

**Precaución y error frecuente:** Si `IS_ROLEMEMBER` devuelve `NULL`, investiga el nombre o la visibilidad; no asumas que equivale a `0`.


**Qué vamos a lograr y por qué se hace ahora.** El rol es un principal de base. `ALTER ROLE ... ADD MEMBER` evita asignar permisos repetidamente por usuario.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE S13_Seguridad_Lab;
GO
IF DATABASE_PRINCIPAL_ID(N'rol_lectura_pedidos') IS NULL
    CREATE ROLE rol_lectura_pedidos;
IF IS_ROLEMEMBER(N'rol_lectura_pedidos',N'lector_s13') = 0
    ALTER ROLE rol_lectura_pedidos ADD MEMBER lector_s13;
```



### Paso 2 — Conceder acceso al objeto exacto

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Ahora sí concedemos la capacidad que justifica al rol. La consulta anterior creó una estructura organizativa, no una autorización específica. Si ejecutáramos la prueba de lectura en ese punto, el rol aún no tendría nuestra concesión de `SELECT`. Por eso el paso siguiente es `GRANT`: transforma una agrupación de usuarios en una responsabilidad funcional sobre una tabla.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

Leamos la instrucción de izquierda a derecha. `GRANT SELECT` indica la operación autorizada: consultar filas. `ON OBJECT::dbo.PedidosLab` delimita el recurso: una tabla, no toda la base ni todas las tablas del esquema `dbo`. `TO rol_lectura_pedidos` indica el destinatario del permiso; los miembros del rol podrán beneficiarse de él. No hemos otorgado `INSERT`, `UPDATE` ni `DELETE`. Tampoco se ha concedido lectura de `AuditoriaLab`.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no utilizar `GRANT SELECT ON SCHEMA::dbo`? Porque permitiría leer también otros objetos del esquema, ampliando la superficie de datos expuestos. Este ejemplo enseña a conceder el derecho mínimo y a comprobar luego qué ocurre en el contexto limitado. El mensaje de ejecución correcta indica que el permiso fue declarado, no sustituye la comprobación operativa que haremos en EJ05.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Otorgar únicamente lectura de pedidos al rol definido.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Ahora que existe la relación usuario→rol, añadimos el eslabón rol→permiso. Esta secuencia explica cómo llega el permiso al usuario.

**Explicación detallada, instrucción por instrucción:**

1. `GRANT SELECT` autoriza la operación de lectura, no inserciones ni actualizaciones.
2. `ON OBJECT::dbo.PedidosLab` limita el permiso a una tabla específica; no cubre todo el esquema `dbo` ni otras tablas.
3. `TO rol_lectura_pedidos` establece al rol como destinatario; sus miembros se benefician de la concesión.

**Qué debes observar después de ejecutarla:** SQL Server debe confirmar ejecución correcta; no se espera una cuadrícula de datos.

**Cómo interpretar lo observado:** El derecho concedido es `SELECT` sobre `PedidosLab`. No se ha otorgado permiso sobre `AuditoriaLab`.

**Precaución y error frecuente:** No sustituir por `db_datareader` en este caso: daría lectura más amplia de la necesaria.


**Qué vamos a lograr y por qué se hace ahora.** `GRANT SELECT` da lectura sobre **una tabla**; no concede `INSERT`, `UPDATE` ni lectura de `AuditoriaLab`.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
GRANT SELECT ON OBJECT::dbo.PedidosLab TO rol_lectura_pedidos;
```



### Paso 3 — Verificar la pertenencia y el permiso registrado

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Después de modificar la seguridad, necesitamos evidencia. Vamos a revisar específicamente la relación de pertenencia que declaramos, sin confundirla con el permiso mismo. Esta distinción es importante: un usuario puede pertenecer a un rol y, sin embargo, ese rol no tener permisos útiles; o puede obtener permisos por otra vía. La pregunta de esta consulta es: ‘¿quién pertenece a nuestro rol lector?’.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`sys.database_role_members` guarda pares de identificadores: rol y miembro. La primera unión a `sys.database_principals` traduce el identificador del rol a su nombre; la segunda traduce el identificador del miembro. `WHERE r.name = N'rol_lectura_pedidos'` limita la salida al rol del ejercicio. Deberíamos obtener una fila con el rol y `lector_s13`. Esta consulta no verifica por sí sola el `GRANT SELECT`: solo prueba la membresía; el permiso se revisará en el inventario de EJ07 y se probará operativamente en EJ05.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué usar las vistas de catálogo y no repetir `ALTER ROLE`? Porque repetir el cambio no aporta información. Verificar es observar el estado que quedó. Si no aparece la fila esperada, no sigamos a `EXECUTE AS` hasta revisar el usuario, el rol, el contexto y las condiciones de ejecución del paso previo.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Verificar que el usuario pertenece al rol lector.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Antes de intentar la lectura con una identidad limitada, comprobamos el vínculo que transmite la autorización.

**Explicación detallada, instrucción por instrucción:**

1. `sys.database_role_members` guarda identificadores de rol y miembro.
2. El primer `JOIN` con `sys.database_principals r` traduce el identificador del rol a su nombre.
3. El segundo `JOIN` con `sys.database_principals m` traduce el miembro a `lector_s13`.
4. `WHERE r.name=...` restringe el resultado al rol que acabamos de preparar.

**Qué debes observar después de ejecutarla:** La cuadrícula debe mostrar `rol_lectura_pedidos | lector_s13`.

**Cómo interpretar lo observado:** Esta consulta confirma membresía, no demuestra por sí sola el `GRANT` ni el acceso efectivo: eso se contrasta en EJ05 y EJ07.

**Precaución y error frecuente:** Si el resultado está vacío, comprueba que EJ03 y EJ04-Paso 1 se ejecutaron en esta base.


**Qué vamos a lograr y por qué se hace ahora.** La unión muestra el rol y su miembro. La autorización efectiva se comprobará en EJ05, no solo por el catálogo.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
SELECT r.name AS Rol, m.name AS Miembro
FROM sys.database_role_members rm
JOIN sys.database_principals r ON r.principal_id=rm.role_principal_id
JOIN sys.database_principals m ON m.principal_id=rm.member_principal_id
WHERE r.name=N'rol_lectura_pedidos';
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** Existe rol_lectura_pedidos y su miembro lector_s13; el único GRANT explícito de este rol es SELECT sobre PedidosLab.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T04 — Tarea espejo de EJ04

**Situación y encargo:** Crear `rol_supervision_s13`, agregar `supervisor_s13` y conceder SELECT únicamente sobre `dbo.AuditoriaLab`.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** Rol y miembro; permiso SELECT en AuditoriaLab.


---

## EJ05 — Demostrar autorización positiva y negativa con EXECUTE AS (18 min)

**Vínculo con la presentación:** diapositivas 4–7. **Objetivo observable:** No basta con afirmar que alguien tiene permisos. Vamos a ejecutar la misma consulta bajo una identidad limitada y observar qué permite SQL Server.

**Antes:** EJ04 completado; usuario `lector_s13` sin permisos administrativos; ejecución por preparador autorizado.


### Paso 1 — Comprobar lectura permitida

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Hasta ahora hemos trabajado como preparadores, con privilegios suficientes para administrar el laboratorio. Si consultamos `PedidosLab` con esa cuenta, veremos filas, pero no demostraremos que el rol lector funciona. Para probar la política debemos cambiar momentáneamente de perspectiva: ejecutar una consulta como el usuario de práctica. Por eso `EXECUTE AS USER` es el núcleo de este ejemplo.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`USE S13_Seguridad_Lab` confirma dónde se evaluarán los permisos. `EXECUTE AS USER = N'lector_s13'` cambia temporalmente el contexto de ejecución dentro de la base; no abre una conexión real de ese usuario. `SELECT USER_NAME()` nos permite verificar que efectivamente estamos bajo `lector_s13`, y solo entonces ejecutamos `SELECT PedidoID, Cliente, Total FROM dbo.PedidosLab`. Si aparece la lista de pedidos, la operación de lectura fue permitida en ese contexto. Finalmente `REVERT` restablece el contexto anterior. Este cierre no es decorativo: evita que el resto del laboratorio siga ejecutándose como un usuario limitado.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no bastaría con consultar el catálogo de permisos? Porque un registro de `GRANT` no sustituye una prueba del comportamiento real; los permisos efectivos dependen también de membresías y otras condiciones. Si ocurre un error antes de `REVERT`, ejecutaremos `REVERT` en la misma sesión antes de seguir. Para cerrar la demostración, compararemos el nombre de usuario antes y después del cambio de contexto.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Probar una operación real bajo la identidad del lector.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Después de registrar roles y permisos, ya podemos observar el efecto de la autorización en una consulta de negocio.

**Explicación detallada, instrucción por instrucción:**

1. `EXECUTE AS USER = N'lector_s13'` sustituye temporalmente el usuario efectivo para probar seguridad de la base actual.
2. `SELECT USER_NAME()` comprueba quién está ejecutando, evitando atribuir el resultado a la cuenta administradora.
3. `SELECT PedidoID, Cliente, Total FROM dbo.PedidosLab` solicita filas reales. Debe funcionar por el `GRANT SELECT` concedido al rol lector.
4. `REVERT` devuelve la ejecución al contexto anterior. Se coloca al final para no dejar la sesión bajo el usuario simulado.

**Qué debes observar después de ejecutarla:** Se visualiza `lector_s13`, seguido de los dos pedidos iniciales. Verifica después `SELECT USER_NAME()` con la identidad original.

**Cómo interpretar lo observado:** La lectura exitosa bajo `lector_s13` es evidencia más fuerte que la mera presencia de un permiso en metadatos.

**Precaución y error frecuente:** Si la consulta falla antes de `REVERT`, restaura el contexto en esa conexión antes de continuar; los permisos de impersonación requieren autorización.


**Qué vamos a lograr y por qué se hace ahora.** `EXECUTE AS USER` cambia temporalmente el principal efectivo dentro de la base. El `SELECT` funciona gracias al rol; `REVERT` devuelve la identidad original.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE S13_Seguridad_Lab;
GO
EXECUTE AS USER = N'lector_s13';
SELECT USER_NAME() AS UsuarioSimulado;
SELECT PedidoID, Cliente, Total FROM dbo.PedidosLab;
REVERT;
```



### Paso 2 — Comprobar modificación denegada sin alterar datos

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Acabamos de demostrar lo que el lector sí puede hacer. Pero el mínimo privilegio se entiende mejor cuando comprobamos también lo que no debería hacer. No queremos que una actividad pedagógica altere las filas del laboratorio ni provoque un error de inserción innecesario; por eso recurrimos a una función de inspección de permisos en el contexto simulado.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`EXECUTE AS USER = N'lector_s13'` reproduce la identidad usada en la prueba anterior. `HAS_PERMS_BY_NAME(N'dbo.PedidosLab',N'OBJECT',N'SELECT')` pregunta si la identidad actual tiene permiso de lectura y la segunda llamada pregunta por `INSERT`. Los alias `PuedeLeer` y `PuedeInsertar` convierten los indicadores en un resultado fácil de discutir. Normalmente esperamos `1` para lectura y `0` para inserción, siempre que no existan otras concesiones. `NULL` exigiría revisar la validez del objeto, el permiso consultado y la visibilidad; no significa automáticamente denegación.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no ejecutar un `INSERT` real y esperar que falle? Porque nuestra pregunta se responde sin modificar información y sin transformar la clase en una práctica de errores. La función ayuda a diagnosticar el permiso; si necesitáramos certificar un escenario más complejo, complementaríamos con pruebas operativas controladas. Cerramos de nuevo con `REVERT`, ya que el contexto simulado no debe extenderse a otros ejercicios.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Comprobar lectura y falta de inserción sin modificar datos.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Primero demostramos una acción permitida. Ahora contrastamos una acción que no debe estar autorizada, sin provocar un INSERT fallido ni alterar tablas.

**Explicación detallada, instrucción por instrucción:**

1. `EXECUTE AS USER` vuelve a fijar el usuario con menos privilegios.
2. `HAS_PERMS_BY_NAME(N'dbo.PedidosLab', N'OBJECT', N'SELECT')` devuelve una evaluación del permiso SELECT efectivo en ese objeto.
3. La segunda llamada cambia solo el permiso solicitado a `INSERT`; como no se ha concedido, se espera `0` en el montaje limpio.
4. Los alias `PuedeLeer` y `PuedeInsertar` hacen explícita la comparación; `REVERT` devuelve la sesión.

**Qué debes observar después de ejecutarla:** Esperado: `PuedeLeer = 1` y `PuedeInsertar = 0`. Algunos contextos o nombres no resolubles pueden producir `NULL`: diagnosticar, no convertirlo automáticamente en denegación.

**Cómo interpretar lo observado:** Una cuenta puede ser autorizada para consultar datos y no para crearlos; eso aplica el principio de privilegio mínimo.

**Precaución y error frecuente:** No ejecutar un `INSERT` innecesario para “probar” una negativa: el chequeo de permisos es suficiente para el objetivo.


**Qué vamos a lograr y por qué se hace ahora.** `HAS_PERMS_BY_NAME` prueba acceso efectivo sin provocar errores ni insertar información. Para este usuario se espera 1 en SELECT y 0 en INSERT.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
EXECUTE AS USER = N'lector_s13';
SELECT HAS_PERMS_BY_NAME(N'dbo.PedidosLab',N'OBJECT',N'SELECT') AS PuedeLeer,
       HAS_PERMS_BY_NAME(N'dbo.PedidosLab',N'OBJECT',N'INSERT') AS PuedeInsertar;
REVERT;
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** Consulta de pedidos correcta; PuedeLeer = 1 y PuedeInsertar = 0; identidad restaurada al final.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T05 — Tarea espejo de EJ05

**Situación y encargo:** Evaluar desde `lector_s13` si tiene UPDATE sobre PedidosLab y SELECT sobre AuditoriaLab, sin ejecutar modificaciones.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** En montaje limpio: 0 y 0.


---

## EJ06 — Separar responsabilidades entre lector y auditor (12 min)

**Vínculo con la presentación:** diapositivas 7–8, 17–19. **Objetivo observable:** El principio de segregación de funciones impide que un lector de operaciones vea automáticamente la evidencia de auditoría. La separación se materializa por roles distintos.

**Antes:** EJ03 y EJ04 completos; rol lector creado.


### Paso 1 — Crear rol exclusivo de auditoría

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

El lector de pedidos ya tiene un papel definido. Ahora imaginemos que otra persona debe examinar evidencias de control, pero no la información comercial. Si diéramos al auditor el mismo rol que al lector, ambas funciones quedarían mezcladas. Para enseñar segregación de funciones crearemos una segunda ruta de permisos totalmente distinta.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

Primero confirmamos `USE S13_Seguridad_Lab`; luego comprobamos si existe `rol_auditoria_s13` y lo creamos cuando falta. La consulta a `IS_ROLEMEMBER` evita incorporar repetidamente a `auditor_s13`; `ALTER ROLE ... ADD MEMBER` conecta al auditor con el rol. Finalmente `GRANT SELECT ON OBJECT::dbo.AuditoriaLab TO rol_auditoria_s13` concede lectura solo sobre el registro de auditoría ficticio. En este bloque se ven juntos los tres eslabones: rol, membresía y permiso, porque ya aprendimos cada operación por separado en EJ04.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no agregar `auditor_s13` también a `rol_lectura_pedidos`? Porque destruiríamos la separación que queremos demostrar. Y recordemos: `AuditoriaLab` es una tabla educativa, no un mecanismo de auditoría de acciones realmente habilitado en la instancia. El resultado de este paso será una configuración, cuya eficacia mediremos con el contexto del auditor en el siguiente bloque.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Crear un segundo rol cuya única finalidad sea leer la evidencia de auditoría.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Ya tenemos un lector de pedidos; ahora incorporamos un perfil diferente para demostrar segregación de funciones.

**Explicación detallada, instrucción por instrucción:**

1. `CREATE ROLE rol_auditoria_s13` registra un rol dedicado si todavía no existe.
2. `IS_ROLEMEMBER` y `ALTER ROLE ... ADD MEMBER` incorporan a `auditor_s13` en la agrupación específica.
3. `GRANT SELECT ON OBJECT::dbo.AuditoriaLab` autoriza solo lectura de la tabla evidencial.
4. No existe en este bloque ninguna concesión sobre `dbo.PedidosLab`.

**Qué debes observar después de ejecutarla:** Ejecución correcta; el contraste se verifica en el siguiente paso.

**Cómo interpretar lo observado:** Los perfiles quedan separados por objeto: el lector de pedidos no recibe auditoría, y el auditor no recibe pedidos.

**Precaución y error frecuente:** Un nombre como “auditor” no da permiso por sí mismo; solo lo hacen membresías y concesiones efectivas.


**Qué vamos a lograr y por qué se hace ahora.** La pertenencia al rol de auditoría no atribuye permisos sobre PedidosLab; se restringe al objeto evidencial.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE S13_Seguridad_Lab;
GO
IF DATABASE_PRINCIPAL_ID(N'rol_auditoria_s13') IS NULL
 CREATE ROLE rol_auditoria_s13;
IF IS_ROLEMEMBER(N'rol_auditoria_s13',N'auditor_s13')=0
 ALTER ROLE rol_auditoria_s13 ADD MEMBER auditor_s13;
GRANT SELECT ON OBJECT::dbo.AuditoriaLab TO rol_auditoria_s13;
```



### Paso 2 — Medir accesos diferenciados

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Tenemos dos rutas funcionales: el rol lector apunta a `PedidosLab` y el rol auditor apunta a `AuditoriaLab`. Ahora necesitamos comprobar que no se han cruzado. Para responderlo sin modificar datos, nos ponemos temporalmente en la identidad del auditor y preguntamos por dos recursos en una misma consulta.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`EXECUTE AS USER = N'auditor_s13'` activa la identidad limitada; las dos expresiones `HAS_PERMS_BY_NAME` consultan el permiso `SELECT` sobre las tablas de auditoría y pedidos. Los alias `Auditoria` y `Pedidos` hacen visible la comparación en una sola fila. En el caso previsto, el auditor obtiene `1` para `AuditoriaLab` y `0` para `PedidosLab`. Esto no es una sentencia `DENY`: estamos observando la ausencia de una concesión de lectura en una configuración limpia. `REVERT` devuelve la identidad preparadora.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué comparar los dos permisos al mismo tiempo? Porque una política de segregación no se demuestra diciendo ‘el auditor sí puede leer auditoría’; también debe mostrar qué queda fuera de su función. Si ambas columnas fueran `1`, habría que investigar membresías o permisos adicionales antes de afirmar que la separación está lograda.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Comparar dos permisos bajo la identidad del auditor.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Crear roles sin comprobar acceso sería insuficiente; medimos ahora que la separación funciona en dirección inversa al lector.

**Explicación detallada, instrucción por instrucción:**

1. `EXECUTE AS USER = N'auditor_s13'` activa el contexto de prueba del segundo perfil.
2. `HAS_PERMS_BY_NAME` sobre `AuditoriaLab` contrasta el permiso que acabamos de conceder.
3. La llamada sobre `PedidosLab` contrasta un recurso que nunca autorizamos a ese rol.
4. `REVERT` devuelve el control a la identidad original.

**Qué debes observar después de ejecutarla:** Esperado en la base limpia: `Auditoria = 1`, `Pedidos = 0`.

**Cómo interpretar lo observado:** La seguridad se demuestra por el contraste de los dos recursos. Un resultado 1/1 indicaría permisos más amplios o membresías adicionales que deben revisarse.

**Precaución y error frecuente:** Si aparecen valores inesperados, inspecciona otras concesiones o roles del usuario en lugar de asumir un error de SQL.


**Qué vamos a lograr y por qué se hace ahora.** El usuario auditor debería poder leer AuditoriaLab, pero no PedidosLab. Este contraste verifica segregación sin operar sobre registros reales.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
EXECUTE AS USER = N'auditor_s13';
SELECT HAS_PERMS_BY_NAME(N'dbo.AuditoriaLab',N'OBJECT',N'SELECT') AS Auditoria,
       HAS_PERMS_BY_NAME(N'dbo.PedidosLab',N'OBJECT',N'SELECT') AS Pedidos;
REVERT;
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** Para auditor_s13: Auditoria = 1 y Pedidos = 0.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T06 — Tarea espejo de EJ06

**Situación y encargo:** Probar los accesos de `supervisor_s13` que obtuvo el rol en T04, verificando que puede leer AuditoriaLab pero no PedidosLab.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** Después de T04: 1 y 0.


---

## EJ07 — Inventariar permisos y distinguir catálogo de auditoría (15 min)

**Vínculo con la presentación:** diapositivas 8–9, 17–19. **Objetivo observable:** Una organización necesita saber qué permisos se asignaron y a quién. El inventario del catálogo ayuda a revisar configuración, aunque no reemplaza SQL Server Audit ni un historial de actividad.

**Antes:** EJ04–EJ06 completados.


### Paso 1 — Listar asignaciones explícitas sobre objetos

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Ya comprobamos las operaciones desde dos identidades. Ahora actuaremos como responsables de revisar la configuración. No vamos a cambiar permisos: consultaremos los metadatos que explican qué derechos explícitos quedaron registrados. Esta distinción importa porque ver una fila en el catálogo de permisos no significa que estemos viendo un historial de accesos de usuarios.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`sys.database_permissions` contiene concesiones y denegaciones explícitas a nivel de base y de objetos. Lo unimos con `sys.database_principals` para obtener el nombre del destinatario y con `sys.objects` para obtener el nombre del objeto. `state_desc` indica el estado de la asignación; `permission_name` identifica el derecho; `class_desc = 'OBJECT_OR_COLUMN'` limita el tipo de elemento observado. El `IN` reduce el inventario a `PedidosLab` y `AuditoriaLab`, y el `ORDER BY` ayuda a comparar resultados. Normalmente veremos permisos de `SELECT` otorgados a los roles del laboratorio.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué esta consulta no sirve para saber quién consultó un pedido ayer? Porque el catálogo describe configuración, no eventos históricos. Para registrar uso real se requieren mecanismos específicos de auditoría y monitoreo. Esta consulta responde ‘¿qué configuramos?’; las pruebas de EJ05 y EJ06 respondieron ‘¿qué puede hacer la identidad?’. Son evidencias complementarias.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Localizar concesiones explícitas sobre los dos objetos.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Tras las pruebas de comportamiento, necesitamos evidencia de configuración: quién recibió qué derecho y sobre qué tabla.

**Explicación detallada, instrucción por instrucción:**

1. `sys.database_permissions perm` registra permisos declarados; no es un historial de consultas ejecutadas.
2. El `JOIN` con `sys.database_principals p` traduce `grantee_principal_id` al nombre del rol o usuario destinatario.
3. El `JOIN` con `sys.objects o` relaciona el identificador `major_id` con el objeto afectado.
4. `WHERE perm.class_desc = N'OBJECT_OR_COLUMN'` selecciona permisos de objeto/columna y el filtro de nombres limita la vista a tablas de práctica.
5. `state_desc` informa `GRANT`, `DENY`, etc.; `permission_name` identifica `SELECT`, `INSERT`, etc.; `ORDER BY` facilita revisión.

**Qué debes observar después de ejecutarla:** Debe verse como mínimo `GRANT SELECT` sobre `PedidosLab` al rol lector y sobre `AuditoriaLab` al rol auditor; pueden aparecer más filas si se añadieron tareas.

**Cómo interpretar lo observado:** Este catálogo explica permisos explícitos, no agota todos los permisos efectivos ni acredita que un usuario haya consultado datos.

**Precaución y error frecuente:** No denominar “registro de auditoría histórica” a `sys.database_permissions`.


**Qué vamos a lograr y por qué se hace ahora.** `sys.database_permissions` muestra asignaciones declaradas; no reporta por sí sola quién ejecutó una consulta ayer.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE S13_Seguridad_Lab;
GO
SELECT p.name AS Principal, perm.state_desc AS Estado,
       perm.permission_name AS Permiso, o.name AS Objeto
FROM sys.database_permissions perm
JOIN sys.database_principals p ON p.principal_id=perm.grantee_principal_id
JOIN sys.objects o ON o.object_id=perm.major_id
WHERE perm.class_desc=N'OBJECT_OR_COLUMN'
  AND o.name IN(N'PedidosLab',N'AuditoriaLab')
ORDER BY p.name,o.name,perm.permission_name;
```



### Paso 2 — Listar membresías para explicar el camino del permiso

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

La consulta anterior mostró permisos asociados a roles. Sin embargo, al responsable de seguridad también le interesa saber qué usuarios reciben esos permisos a través de sus membresías. Si vemos que `rol_lectura_pedidos` posee `SELECT`, aún falta una pieza para justificar por qué `lector_s13` pudo consultar pedidos. Esa pieza es la relación rol → miembro.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`sys.database_role_members` contiene la relación numérica entre cada rol y sus miembros. Las dos uniones a `sys.database_principals` convierten esos identificadores en nombres legibles, esta vez con los alias `Rol` y `Usuario`. El `WHERE` selecciona los roles del laboratorio y `ORDER BY` presenta el inventario de forma consistente. La cláusula `LIKE N'rol_%s13'` usa `_` como comodín de un solo carácter en SQL, no como guion bajo literal; para este laboratorio funciona junto con el filtro explícito de `rol_lectura_pedidos`, pero en entornos amplios convendría filtrar nombres exactos.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué ejecutar dos inventarios separados en EJ07? Porque un permiso registrado para un rol no revela por sí mismo qué usuario lo hereda. Debemos unir mentalmente la salida anterior (rol → permiso → objeto) con la actual (usuario → rol). Si una prueba de EJ05 no coincide con la expectativa, esa cadena permite localizar si falló la membresía, el permiso o la evaluación del contexto.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Reconstruir quién recibe indirectamente los permisos asignados a roles.

**¿Por qué ejecutamos esta consulta ahora y no otra?** El catálogo anterior identifica al rol como destinatario. Falta unirlo conceptualmente con la persona/usuario que pertenece a ese rol.

**Explicación detallada, instrucción por instrucción:**

1. `sys.database_role_members` contiene la relación rol-miembro.
2. Los dos `JOIN` a `sys.database_principals` convierten IDs en nombre del rol y del usuario.
3. `WHERE r.name LIKE N'rol_%s13' OR r.name=N'rol_lectura_pedidos'` busca roles de la práctica; en `LIKE`, el guion bajo `_` funciona como comodín de un carácter, así que el filtro no es una coincidencia literal estricta.
4. `ORDER BY` presenta roles y usuarios de forma organizada.

**Qué debes observar después de ejecutarla:** Se listan al menos las relaciones rol lector→lector y rol auditor→auditor.

**Cómo interpretar lo observado:** Para justificar un acceso debemos considerar ambos listados: membresía y permiso del rol. Ninguno sustituye probar el permiso efectivo.

**Precaución y error frecuente:** No concluir que todos los miembros consultaron las tablas; el catálogo solo describe su configuración.


**Qué vamos a lograr y por qué se hace ahora.** El inventario de roles y el de permisos se complementan: la concesión puede ser al rol y el beneficio llega a sus miembros.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
SELECT r.name AS Rol, u.name AS Usuario
FROM sys.database_role_members m
JOIN sys.database_principals r ON r.principal_id=m.role_principal_id
JOIN sys.database_principals u ON u.principal_id=m.member_principal_id
WHERE r.name LIKE N'rol_%s13' OR r.name=N'rol_lectura_pedidos'
ORDER BY r.name,u.name;
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** Se muestran GRANT SELECT a roles específicos y los miembros correspondientes. No se genera un registro de acciones pasadas.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T07 — Tarea espejo de EJ07

**Situación y encargo:** Construir un informe filtrado únicamente a los permisos SELECT otorgados a roles de este laboratorio.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** Dos filas mínimas de ejemplos; tres tras completar T04.


---

## EJ08 — Crear certificado y clave simétrica para cifrado demostrativo (15 min)

**Vínculo con la presentación:** diapositivas 20–28. **Objetivo observable:** Los permisos protegen operaciones autorizadas, pero la sesión también presenta claves y certificados como objetos criptográficos. Construiremos una jerarquía mínima en la base aislada y verificaremos que el texto cifrado puede recuperarse.

**Antes:** EJ01 completado; base de laboratorio; db_owner; contraseña local robusta no compartida. El instructor reemplaza el marcador en el bloque 1 en su propia estación y no entrega el archivo editado.


### Paso 1 — Crear la clave maestra y el certificado

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Hasta ahora hemos controlado quién puede leer objetos. La presentación también introduce la protección criptográfica: aunque una aplicación gestione autorizaciones, ciertos valores pueden necesitar cifrado y custodia de claves. No vamos a confundir ambas estrategias: el cifrado no decide quién tiene derecho de consultar una tabla. En esta demostración construiremos una jerarquía mínima de objetos criptográficos sin tocar datos de negocio reales.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

La primera condición consulta `sys.symmetric_keys` por el nombre especial `##MS_DatabaseMasterKey##` para determinar si existe la clave maestra de la base. Si falta, `CREATE MASTER KEY ENCRYPTION BY PASSWORD` prepara el material de protección. La contraseña del archivo es un MARCADOR, no una contraseña para usar: el docente debe sustituirla localmente y no distribuir el valor. La segunda condición revisa `sys.certificates` y crea `cert_s13_laboratorio` cuando no existe. Un certificado es un objeto de clave pública y privada que puede proteger otra clave; `SUBJECT` lo describe, pero no cifra datos por sí mismo.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no comenzar con `EncryptByKey`? Porque aún no existe la clave simétrica con la que operará la función. Primero preparamos la protección de claves; luego creamos la clave de datos; por último realizamos la operación. En producción habría exigencias adicionales de copias de seguridad, recuperación, permisos y rotación, que esta práctica no implementa.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Preparar la jerarquía criptográfica de demostración en la base.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Tras estudiar control de acceso, introducimos una segunda forma de protección. Antes de cifrar, se necesitan objetos que custodien la clave de datos.

**Explicación detallada, instrucción por instrucción:**

1. La consulta a `sys.symmetric_keys` comprueba si ya existe la clave maestra de base con nombre de sistema `##MS_DatabaseMasterKey##`.
2. `CREATE MASTER KEY ENCRYPTION BY PASSWORD` crea la clave maestra de base, protegida con una contraseña local robusta. El texto `REEMPLAZAR_CON_FRASE_ROBUSTA_LOCAL` es SOLO un marcador: debe reemplazarse antes de ejecutar.
3. `sys.certificates` permite comprobar la existencia del certificado de laboratorio.
4. `CREATE CERTIFICATE ... WITH SUBJECT` crea un certificado de base de datos que podrá proteger la clave simétrica del paso 2.

**Qué debes observar después de ejecutarla:** Los objetos quedan disponibles sin mostrar su material secreto en resultados; puede inspeccionarse su presencia con vistas de catálogo.

**Cómo interpretar lo observado:** Un certificado no cifra automáticamente las columnas. Aquí solo preparamos la estructura necesaria.

**Precaución y error frecuente:** Ejecutar únicamente en entorno aislado. No compartir la contraseña, y no asumir que el laboratorio enseña una política completa de respaldo de claves.


**Qué vamos a lograr y por qué se hace ahora.** La clave maestra protege material criptográfico de la base; el certificado es un objeto de claves asimétricas. Una contraseña de demostración incluida como marcador NO es una credencial válida para usar sin cambio.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
USE S13_Seguridad_Lab;
GO
-- Sustituir SOLO EN LA ESTACIÓN DEL DOCENTE, nunca compartir el valor:
IF NOT EXISTS (SELECT 1 FROM sys.symmetric_keys WHERE name=N'##MS_DatabaseMasterKey##')
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'REEMPLAZAR_CON_FRASE_ROBUSTA_LOCAL';
IF NOT EXISTS (SELECT 1 FROM sys.certificates WHERE name=N'cert_s13_laboratorio')
    CREATE CERTIFICATE cert_s13_laboratorio WITH SUBJECT = N'Certificado educativo S13';
```



### Paso 2 — Generar clave simétrica vinculada al certificado

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

La base ya dispone de los objetos con los que protegeremos una clave de cifrado de datos. Ahora creamos esa clave concreta. La pregunta de este paso es ‘¿con qué clave simétrica se cifrará el valor?’ y no ‘¿qué usuario podrá leer la tabla?’. Estamos en una capa de seguridad distinta.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`IF NOT EXISTS` consulta `sys.symmetric_keys` para no repetir la creación de `clave_s13_datos`. `CREATE SYMMETRIC KEY` define el objeto. `WITH ALGORITHM = AES_256` especifica el algoritmo que se utilizará para proteger los valores; `ENCRYPTION BY CERTIFICATE cert_s13_laboratorio` indica cómo se protegerá la clave simétrica mientras esté almacenada. No estamos cifrando todavía ninguna fila ni mostrando bytes cifrados. Solo dejamos lista la herramienta necesaria para hacerlo en el próximo paso.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué proteger una clave mediante un certificado en vez de dejarla sin protección? Porque la gestión de la clave es parte esencial de la seguridad del dato. ¿Por qué no basta con haber creado el certificado en el paso 1? Porque el certificado y la clave simétrica tienen responsabilidades distintas: el primero protege la clave, mientras la segunda realiza el cifrado del texto del ejemplo.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Crear la clave simétrica de datos y protegerla con el certificado.

**¿Por qué ejecutamos esta consulta ahora y no otra?** La clave maestra y el certificado ya están listos; ahora definimos la clave que usará la operación de cifrado de datos.

**Explicación detallada, instrucción por instrucción:**

1. `IF NOT EXISTS` consulta `sys.symmetric_keys` para evitar duplicar la clave por nombre.
2. `CREATE SYMMETRIC KEY clave_s13_datos` crea el objeto criptográfico en la base.
3. `WITH ALGORITHM = AES_256` selecciona el algoritmo simétrico para cifrar los valores.
4. `ENCRYPTION BY CERTIFICATE cert_s13_laboratorio` protege la clave simétrica mediante el certificado creado anteriormente.

**Qué debes observar después de ejecutarla:** La definición termina correctamente; no retorna aún un dato cifrado.

**Cómo interpretar lo observado:** Es distinta la clave usada para cifrar el dato de los mecanismos empleados para proteger esa clave.

**Precaución y error frecuente:** Si falta el certificado del paso anterior, este CREATE no puede completarse.


**Qué vamos a lograr y por qué se hace ahora.** AES_256 es una clave simétrica; su protección se delega al certificado. No incluir secretos de negocio en este ejemplo.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
IF NOT EXISTS (SELECT 1 FROM sys.symmetric_keys WHERE name=N'clave_s13_datos')
 CREATE SYMMETRIC KEY clave_s13_datos
 WITH ALGORITHM = AES_256
 ENCRYPTION BY CERTIFICATE cert_s13_laboratorio;
```



### Paso 3 — Cifrar, descifrar y cerrar la clave

#### Explicación guiada para comprender antes de ejecutar

**1. ¿Qué problema estamos resolviendo y por qué esta consulta va primero?**

Hemos creado la clave maestra, el certificado y la clave simétrica. Es momento de responder la pregunta que da sentido a esa preparación: ¿podemos convertir un texto legible en bytes cifrados y recuperarlo correctamente usando la clave autorizada? Ejecutaremos un ciclo completo de apertura, uso, comprobación y cierre, sin guardar contraseñas ni datos personales.

**2. ¿Para qué sirve cada operación y qué demuestra al ejecutarla?**

`OPEN SYMMETRIC KEY ... DECRYPTION BY CERTIFICATE` hace disponible la clave simétrica en la sesión. Con `DECLARE @cifrado varbinary(8000)` reservamos espacio para bytes; `EncryptByKey(Key_GUID(...), N'Dato educativo')` usa el identificador de la clave para producir el valor cifrado. La siguiente consulta muestra dos columnas: `Cifrado` contiene datos binarios, mientras `DecryptByKey(@cifrado)` devuelve los bytes originales y `CONVERT(nvarchar(100),...)` los presenta como texto. Si todo está preparado correctamente, `Recuperado` dirá `Dato educativo`. `CLOSE SYMMETRIC KEY` cierra la clave que abrimos.

**3. ¿Por qué usar esta consulta y no otra?**

¿Por qué no presentar solo la columna cifrada? Porque una cadena de bytes no demuestra por sí sola que la configuración permita recuperar el dato original. La segunda columna valida el ciclo. ¿Por qué cerrar la clave? Porque su uso debe quedar acotado a la operación necesaria. Si la recuperación retorna `NULL`, revisamos primero que la clave esté abierta en la sesión correcta, que el certificado corresponda y que los pasos anteriores hayan terminado.

**4. Pregunta de razonamiento (responde en voz alta).** Antes de ejecutar, describe qué objeto, permiso o contexto cambia; después de ejecutar, explica si la salida es evidencia de una configuración, una capacidad efectiva o simplemente del estado de los datos. No son lo mismo.


**Ruta de razonamiento del estudiante:**


**Objetivo específico de esta consulta:** Cifrar una cadena, recuperarla y cerrar el acceso a la clave en la sesión.

**¿Por qué ejecutamos esta consulta ahora y no otra?** Una vez creada y protegida la clave simétrica, comprobamos el ciclo de uso completo: abrir → cifrar → descifrar → cerrar.

**Explicación detallada, instrucción por instrucción:**

1. `OPEN SYMMETRIC KEY ... DECRYPTION BY CERTIFICATE ...` abre la clave para utilizarla en esta sesión mediante su certificado protector.
2. `DECLARE @cifrado varbinary(8000)` reserva una variable de bytes y `EncryptByKey(Key_GUID(...), N'Dato educativo')` almacena la representación cifrada del texto; `Key_GUID` identifica la clave simétrica.
3. `SELECT @cifrado AS Cifrado` muestra bytes, no texto legible. `DecryptByKey(@cifrado)` recupera los bytes originales y `CONVERT(nvarchar(100), ...)` permite visualizarlos como Unicode.
4. `CLOSE SYMMETRIC KEY` cierra la clave abierta en esta sesión cuando ya no se requiere.

**Qué debes observar después de ejecutarla:** La columna `Recuperado` debe mostrar `Dato educativo`; `Cifrado` será binaria. Si el cifrado/descifrado no corresponde, se debe investigar antes de cerrar.

**Cómo interpretar lo observado:** El cifrado permite ocultar el valor almacenado mientras se dispone de gestión apropiada de claves; no reemplaza el control de acceso.

**Precaución y error frecuente:** Usar siempre datos ficticios y cerrar la clave tras el ensayo; no copiar contraseñas o material criptográfico al entregable.


**Qué vamos a lograr y por qué se hace ahora.** `EncryptByKey` produce bytes no legibles; `DecryptByKey` requiere la clave abierta. Cerrarla reduce la exposición dentro de la sesión.

**Acción exacta:** En SSMS, sitúate en una consulta con permisos de laboratorio y escribe y ejecuta el siguiente microbloque (si tiene `GO`, conserva su separación):

```sql
OPEN SYMMETRIC KEY clave_s13_datos
 DECRYPTION BY CERTIFICATE cert_s13_laboratorio;
DECLARE @cifrado varbinary(8000) = EncryptByKey(Key_GUID(N'clave_s13_datos'),N'Dato educativo');
SELECT @cifrado AS Cifrado,
 CONVERT(nvarchar(100),DecryptByKey(@cifrado)) AS Recuperado;
CLOSE SYMMETRIC KEY clave_s13_datos;
```



### Predicción, comprobación e interpretación

**Antes de ejecutar el último bloque, responde:** ¿qué resultado indica que la identidad o el permiso actúa como esperamos? Comprueba en SSMS. **Resultado esperado:** La columna Recuperado dice «Dato educativo» y la columna Cifrado contiene varbinary. Las claves no deben copiarse a producción.

**Si no coincide:** confirma la conexión, el contexto de base, el orden de ejemplos y el permiso de tu usuario preparador. Si aparece un error después de `EXECUTE AS`, restaura el contexto con `REVERT` antes de continuar; evita ejecutar operaciones administrativas en una sesión cuyo contexto no conoces.


### T08 — Tarea espejo de EJ08

**Situación y encargo:** Cifrar y descifrar el texto `Validación S13` usando la clave ya creada, y cerrar la clave al finalizar.

**Pistas sin resolver:** identifica la instrucción utilizada en el ejemplo, determina qué objeto y principal cambian y consulta una vista de catálogo o `HAS_PERMS_BY_NAME` para demostrar el resultado.

**Evidencia a entregar:** script SQL propio, captura o transcripción de resultado y dos frases que distingan el objetivo de la prueba y su efecto. **Criterio de éxito:** TextoVerificado = Validación S13.


## Cierre y comprobación final

La autenticación establece **quién** se conecta; la autorización determina **qué** puede hacer. Los roles simplifican la asignación, las pruebas de contexto corroboran los permisos efectivos y la consulta a catálogos documenta la configuración sin sustituir los mecanismos de auditoría de actividad. Claves y certificados añaden otro mecanismo de protección, pero requieren custodia, respaldo y procedimientos de recuperación diseñados por especialistas.

**Entrega posterior:** EJ01–EJ08 reproducidos; T01–T08 resueltas; tabla breve de evidencia por ID, resultado y conclusión. **Preguntas de salida:** ¿por qué `WITHOUT LOGIN` no permite conectarse?, ¿por qué `GRANT SELECT` a un rol no autoriza `INSERT`?, ¿por qué respaldar un certificado es diferente de respaldar los datos? **Puente:** estrategias de respaldo y restauración de bases de datos, incluyendo la recuperación del material criptográfico cuando exista dependencia.
