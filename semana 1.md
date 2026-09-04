# GUÍA DEL ESTUDIANTE

## Programación Avanzada de Base de Datos - Sesión 01

**Tema:** Implementación de base de datos con SQL Server  
**Práctica:** Arquitectura cliente/servidor, DDL, DML, creación y modificación de tablas, constraints, validación con `INSERT`, separación y adjunción de bases de datos.  
**Caso integrador:** Base de datos `AlquilerCoches`.  
**Duración sugerida:** 180 minutos para la ruta guiada completa, más el tiempo de entrega del trabajo práctico.  
**Modalidad:** laboratorio local en Windows con SQL Server y SQL Server Management Studio (SSMS).

---

## 1. Propósito de la sesión

Al finalizar esta sesión podrás **crear una base de datos relacional en SQL Server, construir y modificar tablas, aplicar restricciones de integridad, insertar y modificar registros, comprobar errores de validación y realizar una práctica controlada de separar y volver a adjuntar una base de datos**.

La meta no es memorizar sentencias. Debes comprender la relación causa-efecto entre lo que escribes, lo que SQL Server ejecuta y el estado final de la base de datos.

## 2. Resultado observable

Al terminar deberás poder mostrar una base de datos funcional en SQL Server donde:

- existan las tablas del caso `AlquilerCoches`;
- las claves y relaciones impidan datos incoherentes;
- el DNI de un cliente no pueda repetirse;
- la fecha inicial de una reserva tenga un valor predeterminado con `GETDATE()`;
- el precio de alquiler no acepte valores negativos;
- puedas agregar una nueva columna con `ALTER TABLE`;
- puedas insertar, actualizar y eliminar registros de prueba;
- puedas explicar qué diferencia existe entre DDL y DML;
- puedas separar y volver a adjuntar una base de datos de laboratorio siguiendo un procedimiento seguro.

## 3. Conocimientos previos

Antes de empezar conviene reconocer:

- qué es una base de datos relacional;
- qué representa una tabla, una fila y una columna;
- diferencia entre dato numérico, texto y fecha;
- noción de identificador único;
- relación básica entre una tabla principal y una tabla dependiente.

No necesitas dominar procedimientos almacenados ni administración avanzada.

## 4. Herramientas recomendadas para esta práctica

| Herramienta | Recomendación para clase | Para qué la usaremos |
|---|---|---|
| SQL Server | **SQL Server 2025 Standard Developer**, actualizado con el último CU disponible. A fecha 03/09/2026: **CU8 - 17.0.4075.5** | Motor de base de datos local para desarrollo y laboratorio |
| SQL Server Management Studio | **SSMS 22.9.2** | Conectarnos al motor, ejecutar T-SQL y revisar objetos/resultados |
| Windows | Windows 10/11 compatible con SSMS 22 | Entorno de laboratorio |

**Por qué Standard Developer:** es gratuita para desarrollo y pruebas, no para producción, y ofrece un entorno completo para las prácticas del curso.

### Recursos oficiales

- Descargas SQL Server: https://www.microsoft.com/es-es/sql-server/sql-server-downloads
- Instalación SQL Server 2025: https://learn.microsoft.com/es-es/sql/database-engine/install-windows/install-sql-server-from-the-installation-wizard-setup?view=sql-server-ver17
- Actualizaciones SQL Server: https://learn.microsoft.com/es-es/troubleshoot/sql/releases/download-and-install-latest-updates
- Instalación de SSMS: https://learn.microsoft.com/en-us/ssms/install/install
- Notas de versión SSMS 22: https://learn.microsoft.com/en-us/ssms/release-notes-22

> **Importante:** SSMS 22 se instala mediante un pequeño ejecutable que abre **Visual Studio Installer**. Esto no significa que debas instalar Visual Studio completo. SSMS es un producto separado.

## 5. Mapa de la sesión

```text
ARQUITECTURA CLIENTE/SERVIDOR
          ↓
SQL: DDL vs DML
          ↓
CREATE DATABASE / CREATE TABLE
          ↓
ALTER TABLE / DROP TABLE
          ↓
INTEGRIDAD Y CONSTRAINTS
          ↓
INSERT + VALIDACIONES
          ↓
PK / FK / DEFAULT / CHECK / UNIQUE
          ↓
CASCADE EN LABORATORIO
          ↓
DETACH / ATTACH
          ↓
CASO INTEGRADOR: AlquilerCoches
```

---

# PARTE I. Preparación del entorno

## 6. Instalar SQL Server 2025 para laboratorio

Si el motor ya está instalado y puedes conectarte, pasa a la sección 8.

### 6.1. Descarga

1. Abre la página oficial de descargas de SQL Server.
2. Ubica **SQL Server 2025 Developer**.
3. Para este curso, elige **Standard Developer**.
4. Descarga el instalador oficial.

### 6.2. Instalación recomendada

1. Ejecuta el instalador con permisos normales de usuario; acepta elevación de administrador solo cuando Windows la solicite.
2. En el asistente de SQL Server, inicia una **nueva instalación independiente**.
3. Selecciona la edición gratuita de desarrollo correspondiente.
4. Acepta los términos de licencia.
5. Mantén seleccionado el **Database Engine Services**. No agregues componentes que la sesión no requiere.
6. Para una laptop de laboratorio, utiliza una **instancia predeterminada** si no tienes otra instancia instalada.
7. En la configuración del motor, agrega tu usuario de Windows como administrador de SQL Server.
8. Finaliza la instalación.
9. Aplica las actualizaciones disponibles para SQL Server. A fecha de verificación de esta guía, SQL Server 2025 CU8 corresponde a `17.0.4075.5`.

### ¿Qué estamos instalando realmente?

El **motor de SQL Server** es el servicio que guarda, procesa y protege las bases de datos. No es lo mismo que SSMS. Puedes tener el motor instalado y todavía no tener una interfaz gráfica para administrarlo.

## 7. Instalar SSMS 22

1. Abre la página oficial de instalación de SSMS.
2. Descarga el instalador de **SQL Server Management Studio 22**.
3. El archivo descargado es un *bootstrapper* que abre **Visual Studio Installer**.
4. En Visual Studio Installer, confirma que el producto seleccionado sea **SQL Server Management Studio**.
5. Selecciona **Instalar**.
6. Espera a que finalice y abre SSMS.

### Error frecuente: "se abrió Visual Studio Installer"

Eso es normal en SSMS 22. El instalador de SSMS utiliza Visual Studio Installer como mecanismo de instalación, pero **no necesitas instalar Visual Studio Community/Professional/Enterprise** para esta sesión.

## 8. Primera conexión

1. Abre SSMS.
2. En **Server type**, usa `Database Engine`.
3. Si instalaste una instancia predeterminada local, prueba como servidor:

```text
localhost
```

También puede funcionar:

```text
.
```

4. Usa **Windows Authentication**.
5. Conéctate.

### Si instalaste una instancia con nombre

La forma habitual es:

```text
localhost\NOMBRE_INSTANCIA
```

Por ejemplo, si al finalizar la instalación SQL Server muestra una instancia llamada `MSSQLSERVER02`, puedes probar:

```text
localhost\MSSQLSERVER02
```

El nombre exacto depende de cómo se instaló SQL Server en cada equipo. No copies el nombre de otro compañero sin comprobar primero tu propia instancia.

> **Caso observado en el equipo docente:** SQL Server 2025 Standard Developer quedó instalado inicialmente como RTM `17.0.1000.7` y SSMS como `22.9.2`. La instalación RTM es funcional; para el laboratorio se recomienda aplicar posteriormente las actualizaciones oficiales disponibles. A la fecha de verificación de esta guía, SQL Server 2025 CU8 corresponde a `17.0.4075.5`.

## 9. Validación inicial

Abre **New Query / Nueva consulta** y ejecuta:

```sql
SELECT
    @@SERVERNAME AS Servidor,
    SERVERPROPERTY('ProductVersion') AS VersionProducto,
    SERVERPROPERTY('Edition') AS Edicion;
GO
```

### ¿Qué debes observar?

- `Servidor`: nombre de la instancia a la que te conectaste.
- `VersionProducto`: versión instalada.
- `Edicion`: edición del motor.

Si trabajas con SQL Server 2025 CU8, la versión esperada es `17.0.4075.5`. Una compilación posterior también es válida si corresponde a una actualización oficial posterior.

---

# PARTE II. Comprender cliente y servidor

## 10. Arquitectura cliente/servidor aplicada a SQL Server

En esta sesión debes distinguir dos roles:

- **Servidor:** el motor de SQL Server recibe solicitudes, ejecuta operaciones sobre la base de datos y devuelve resultados.
- **Cliente:** SSMS se conecta al servidor, envía instrucciones T-SQL y muestra los resultados.

```text
SSMS (CLIENTE)
     |
     |  sentencia SQL
     v
SQL SERVER (SERVIDOR)
     |
     |  procesa datos / metadatos
     v
BASE DE DATOS
     |
     |  resultado o error
     v
SSMS (CLIENTE)
```

### Pregunta de comprobación

Si SSMS está abierto pero el servicio de SQL Server está detenido, ¿puedes ejecutar una consulta?  
**Respuesta esperada:** no, porque el cliente existe, pero no puede comunicarse con el servidor.

---

# PARTE III. Fundamentos de SQL antes de escribir sentencias

## 11. ¿Qué es SQL y qué problema resuelve?

SQL es el lenguaje que utilizaremos para comunicarnos con el motor de base de datos. En esta sesión no se busca memorizar instrucciones aisladas; se busca comprender **qué queremos cambiar** y **qué instrucción corresponde usar**.

Piensa en este flujo:

```text
ESTUDIANTE / APLICACIÓN
        |
        | escribe una instrucción SQL
        v
SQL SERVER
        |
        | interpreta y ejecuta
        v
BASE DE DATOS
        |
        | devuelve resultado o error
        v
SSMS / APLICACIÓN CLIENTE
```

Cuando escribimos una sentencia, SQL Server puede actuar sobre dos grandes aspectos trabajados en esta sesión:

1. la **estructura** de la base de datos;
2. los **datos** almacenados dentro de esa estructura.

Esa diferencia nos lleva a DDL y DML.

---

## 12. Antes de DDL y DML: base de datos, tabla, campo y registro

Antes de clasificar instrucciones, debemos diferenciar los elementos sobre los que trabajan.

### Base de datos

Es el contenedor lógico que agrupa tablas y otros objetos relacionados.

Ejemplo de la sesión:

```text
AlquilerCoches
```

### Tabla

Organiza información sobre una entidad o concepto.

Ejemplos del caso:

```text
CLIENTE
AGENCIA
RESERVA
COCHE
GARAJE
DETALLE_RES
```

### Campo o columna

Describe una característica que almacenará la tabla.

Ejemplo en `CLIENTE`:

```text
IdCliente
NomCliente
DNICli
DirCli
TelCli
```

### Registro o fila

Representa una ocurrencia concreta almacenada en la tabla.

Ejemplo conceptual:

| IdCliente | NomCliente | DNICli |
|---:|---|---|
| 1 | Cliente Uno | 70000001 |

### Relación visual

```text
BASE DE DATOS: AlquilerCoches
        |
        +-- TABLA: CLIENTE
                |
                +-- CAMPOS: IdCliente, NomCliente, DNICli, ...
                |
                +-- REGISTRO 1
                +-- REGISTRO 2
                +-- REGISTRO 3
```

Esta distinción es importante porque **DDL cambia principalmente la estructura**, mientras que **DML trabaja con los registros**.

---

# PARTE IV. DDL y DML: estructura frente a datos

## 13. DDL - Data Definition Language

DDL significa **Data Definition Language** o **Lenguaje de Definición de Datos**.

Su propósito en esta sesión es **crear, modificar o eliminar estructuras de la base de datos**.

Las instrucciones principales que trabajaremos son:

| Instrucción | Qué hace | Ejemplo de la sesión |
|---|---|---|
| `CREATE` | crea un objeto | crear una base o tabla |
| `ALTER` | modifica la estructura de un objeto existente | agregar o quitar una columna |
| `DROP` | elimina un objeto | eliminar una tabla de prueba |

### Ejemplo mínimo: CREATE TABLE

```sql
CREATE TABLE ClienteDemo (
    IdCliente INT,
    Nombre VARCHAR(80)
);
GO
```

### ¿Qué ocurrió?

SQL Server creó una **estructura** llamada `ClienteDemo`. Todavía no hemos almacenado clientes.

```text
CREATE TABLE
     |
     v
CREA ESTRUCTURA
     |
     v
DDL
```

### Ejemplo mínimo: ALTER TABLE

```sql
ALTER TABLE ClienteDemo
ADD DNI CHAR(8) NULL;
GO
```

### ¿Qué ocurrió?

No insertamos ni modificamos una fila. Cambiamos la **definición de la tabla** agregando una columna.

Por eso `ALTER TABLE` también es DDL.

### Ejemplo mínimo: DROP TABLE

```sql
DROP TABLE ClienteDemo;
GO
```

### ¿Qué ocurrió?

La tabla dejó de existir como objeto. Por eso `DROP TABLE` debe utilizarse únicamente sobre objetos de laboratorio cuya eliminación sea intencional.

> **Idea clave:** DDL responde a la pregunta: **¿cómo está construida la base de datos?**

---

## 14. DML - Data Manipulation Language

DML significa **Data Manipulation Language** o **Lenguaje de Manipulación de Datos**.

En el tratamiento de esta sesión se utiliza para **consultar, insertar, actualizar y eliminar registros**.

| Instrucción | Qué hace | Efecto principal |
|---|---|---|
| `SELECT` | consulta registros | lee datos |
| `INSERT` | agrega registros | crea filas |
| `UPDATE` | modifica registros existentes | cambia valores de filas |
| `DELETE` | elimina registros | quita filas |

### Ejemplo mínimo: INSERT

Supongamos que existe una tabla `ClienteDemo`.

```sql
INSERT INTO ClienteDemo (IdCliente, Nombre)
VALUES (1, 'Ana');
GO
```

Ahora sí estamos almacenando una fila.

```text
INSERT
   |
   v
TRABAJA CON REGISTROS
   |
   v
DML
```

### Ejemplo mínimo: SELECT

```sql
SELECT *
FROM ClienteDemo;
GO
```

Consulta los registros existentes.

### Ejemplo mínimo: UPDATE

```sql
UPDATE ClienteDemo
SET Nombre = 'Ana Torres'
WHERE IdCliente = 1;
GO
```

Modifica un dato almacenado; no cambia la estructura de la tabla.

### Ejemplo mínimo: DELETE

```sql
DELETE FROM ClienteDemo
WHERE IdCliente = 1;
GO
```

Elimina una fila que cumple la condición.

> **Idea clave:** DML responde a la pregunta: **¿qué hacemos con los datos que están dentro de las tablas?**

---

## 15. Diferencia fundamental entre DDL y DML

La comparación que debes poder explicar con tus propias palabras es:

```text
BASE DE DATOS
|
+-- ESTRUCTURA --------------> DDL
|      |
|      +-- CREATE
|      +-- ALTER
|      +-- DROP
|
+-- DATOS / REGISTROS -------> DML
       |
       +-- INSERT
       +-- SELECT
       +-- UPDATE
       +-- DELETE
```

Una analogía sencilla:

- **DDL** construye o modifica el recipiente.
- **DML** trabaja con lo que colocamos dentro del recipiente.

### Clasificación guiada

#### Caso 1

```sql
CREATE DATABASE AlquilerCoches;
```

**Respuesta:** DDL, porque crea una estructura.

#### Caso 2

```sql
INSERT INTO CLIENTE (IdCliente, NomCliente, DNICli)
VALUES (1, 'Cliente Uno', '70000001');
```

**Respuesta:** DML, porque agrega un registro.

#### Caso 3

```sql
ALTER TABLE CLIENTE
ADD FechaNac DATE NULL;
```

**Respuesta:** DDL, porque modifica la estructura de la tabla.

#### Caso 4

```sql
UPDATE CLIENTE
SET DirCli = 'Nueva direccion'
WHERE IdCliente = 3;
```

**Respuesta:** DML, porque modifica un valor de una fila.

### Comprobación rápida

Antes de continuar, responde:

1. Si agregas una columna, ¿DDL o DML?
2. Si insertas un cliente, ¿DDL o DML?
3. Si eliminas una tabla, ¿DDL o DML?
4. Si eliminas una fila de una tabla, ¿DDL o DML?
5. ¿Por qué `DROP TABLE` y `DELETE FROM` no significan lo mismo?

**Respuestas:** 1) DDL, 2) DML, 3) DDL, 4) DML, 5) `DROP TABLE` elimina el objeto; `DELETE FROM` elimina filas del objeto que continúa existiendo.

---

# PARTE V. Integridad de datos y constraints antes de probar INSERT

## 16. ¿Qué significa integridad de datos?

La integridad de los datos se refiere a mantener los datos **consistentes y exactos**. No basta con crear columnas: debemos definir qué valores son válidos y cómo hará SQL Server para impedir datos que violen las reglas del modelo.

Ejemplo:

```text
Regla del negocio:
"El DNI de un cliente no se puede repetir"
        |
        v
La base de datos debe impedir duplicados
        |
        v
Constraint UNIQUE
```

Un **constraint** es una restricción declarada en la base de datos para controlar qué datos puede aceptar una tabla o cómo deben relacionarse sus filas.

---

## 17. Los tres niveles de integridad trabajados en la sesión

### 17.1. Integridad de entidad

Busca que cada fila pueda identificarse de forma única.

En la sesión se relaciona principalmente con:

- `PRIMARY KEY`;
- `UNIQUE`;
- `IDENTITY` como propiedad para generar valores autonuméricos cuando corresponde.

Ejemplo conceptual:

```text
CLIENTE
IdCliente = 1  ---> identifica una fila
IdCliente = 2  ---> identifica otra fila
```

No deberían existir dos filas con la misma clave primaria.

### 17.2. Integridad de dominio

Controla qué valores son aceptables para un atributo.

En la sesión aparecen mecanismos como:

- `CHECK`;
- `DEFAULT`;
- `NOT NULL`;
- claves foráneas dentro del tratamiento presentado por la sesión.

Ejemplo del trabajo práctico:

```text
PrecioAlq >= 0
```

Si se intenta guardar `-50`, la base debe rechazarlo.

### 17.3. Integridad referencial

Mantiene coherentes las relaciones entre tablas.

Ejemplo:

```text
CLIENTE (tabla padre)
   |
   | IdCliente
   v
RESERVA (tabla hija)
```

Una reserva no debería apuntar a un cliente inexistente cuando existe una `FOREIGN KEY` que protege la relación.

---

## 18. PRIMARY KEY - identificar cada fila

Una `PRIMARY KEY` define el identificador principal de una tabla.

Ejemplo:

```sql
CREATE TABLE CLIENTE_DEMO (
    IdCliente INT PRIMARY KEY,
    Nombre VARCHAR(80)
);
GO
```

### ¿Qué problema evita?

Evita que dos filas tengan el mismo identificador principal.

Caso válido:

```sql
INSERT INTO CLIENTE_DEMO VALUES (1, 'Ana');
INSERT INTO CLIENTE_DEMO VALUES (2, 'Luis');
GO
```

Caso que debe fallar:

```sql
INSERT INTO CLIENTE_DEMO VALUES (1, 'Otro cliente');
GO
```

### Qué debes interpretar

El error no significa que SQL Server esté fallando. Significa que **la regla funcionó y protegió la entidad**.

---

## 19. UNIQUE - impedir valores repetidos

`UNIQUE` sirve cuando un valor debe ser único aunque no sea la clave primaria.

En `AlquilerCoches` la regla es:

> El DNI de un cliente no se puede repetir.

Por eso podemos declarar:

```sql
CONSTRAINT UQ_CLIENTE_DNI UNIQUE (DNICli)
```

### Diferencia práctica frente a PRIMARY KEY

En esta sesión basta con recordar:

- `PRIMARY KEY`: identificador principal de la fila;
- `UNIQUE`: impide duplicar otro valor que también debe ser exclusivo.

En el modelo, `IdCliente` identifica al cliente y `DNICli` debe mantenerse sin duplicados.

---

## 20. DEFAULT - valor automático cuando no se proporciona uno

`DEFAULT` permite que SQL Server coloque un valor predeterminado cuando el `INSERT` no proporciona ese campo.

Ejemplo de la sesión:

```sql
ALTER TABLE Miembros
ADD Departamento CHAR(2) NULL;
GO

ALTER TABLE Miembros
ADD CONSTRAINT DepartInicial
DEFAULT 'LI' FOR Departamento;
GO
```

### Causa y efecto

```text
INSERT no envía Departamento
        |
        v
DEFAULT está definido
        |
        v
SQL Server usa 'LI'
```

En el caso `AlquilerCoches` la regla equivalente es que `FechaInicio` use `GETDATE()` por defecto.

---

## 21. CHECK - validar una condición

`CHECK` controla que un valor cumpla una condición.

Ejemplo conceptual presentado en la sesión: un sueldo base inferior al mínimo definido debe ser rechazado.

En el trabajo práctico utilizaremos:

```sql
CHECK (PrecioAlq >= 0)
```

### Caso correcto

```text
PrecioAlq = 150.00
150.00 >= 0  -> verdadero -> se acepta
```

### Caso incorrecto

```text
PrecioAlq = -50.00
-50.00 >= 0  -> falso -> se rechaza
```

La idea importante no es memorizar la palabra `CHECK`; es comprender que la base verifica una **condición lógica** antes de aceptar el dato.

---

## 22. FOREIGN KEY - mantener la relación entre tablas

Una `FOREIGN KEY` conecta un campo de una tabla hija con una clave de una tabla padre.

Ejemplo del modelo:

```text
CLIENTE
IdCliente (PK)
    |
    +----------------+
                     |
                     v
RESERVA
IdCliente (FK)
```

Si intentas crear una reserva para un `IdCliente` inexistente, la base debe rechazarla.

### Orden de trabajo que debes recordar

```text
1. Crear tabla padre
2. Crear tabla hija con FOREIGN KEY
3. Insertar primero el registro padre
4. Insertar después el registro hijo
```

---

## 23. IDENTITY - autonumeración utilizada en la sesión

`IDENTITY(inicio, incremento)` permite que SQL Server genere automáticamente valores numéricos.

Ejemplo:

```sql
IdAgencia INT IDENTITY(1,1) NOT NULL
```

Significa:

- iniciar en `1`;
- incrementar de `1` en `1`.

Por eso, al insertar una agencia normalmente no escribiremos manualmente `IdAgencia`.

---

## 24. ON UPDATE CASCADE y ON DELETE CASCADE

Estas opciones pertenecen a una relación con clave foránea y permiten propagar cambios desde la tabla padre hacia la tabla hija.

### ON UPDATE CASCADE

```text
Padre cambia clave
      |
      v
Hijos relacionados actualizan la FK
```

### ON DELETE CASCADE

```text
Padre es eliminado
      |
      v
Hijos relacionados también son eliminados
```

En esta clase se probará únicamente en una **base de laboratorio aislada**, porque `ON DELETE CASCADE` puede eliminar múltiples filas automáticamente.

---

## 25. Mapa conceptual antes de comenzar los ejemplos

```text
SQL
|
+-- DDL: estructura
|     +-- CREATE
|     +-- ALTER
|     +-- DROP
|
+-- DML: datos
      +-- SELECT
      +-- INSERT
      +-- UPDATE
      +-- DELETE

ESTRUCTURA + DATOS
        |
        v
INTEGRIDAD
|
+-- ENTIDAD ------> PRIMARY KEY / UNIQUE / IDENTITY
+-- DOMINIO ------> CHECK / DEFAULT / NOT NULL
+-- REFERENCIAL --> FOREIGN KEY / CASCADE
```

Cuando este mapa tenga sentido para ti, los scripts siguientes dejarán de parecer una lista de comandos y empezarán a verse como decisiones sobre la estructura y los datos.

---

# PARTE VI. Ejemplos guiados progresivos


## Ejemplo 1. Crear una base de datos y comprobarla

> **Antes de escribir código:** recuerda que `CREATE DATABASE` es DDL porque crea estructura. Aún no estamos insertando datos.


### 1. ¿Qué aprenderemos con este ejemplo?

Crear una base de datos usando `CREATE DATABASE` y validar que el servidor la registró.

### 2. Problema o situación

Necesitamos un contenedor lógico donde crear tablas y almacenar registros de una biblioteca de laboratorio.

### 3. Concepto que necesitamos

`CREATE DATABASE` es una instrucción DDL. SQL Server crea la base de datos y sus archivos físicos en las rutas predeterminadas de la instancia, salvo que especifiquemos rutas válidas explícitas.

### 4. Antes de comenzar

Debes estar conectado a la instancia local desde SSMS.

### 5. Estado inicial

No debe existir una base llamada `Biblioteca_S01`.

### 6. Paso 1 - Cambiar al contexto `master`

**Qué hacemos:** trabajamos desde la base del sistema usada habitualmente como contexto para crear otra base.

```sql
USE master;
GO
```

### 7. Paso 2 - Crear la base

```sql
CREATE DATABASE Biblioteca_S01;
GO
```

**Por qué lo hacemos:** queremos que SQL Server cree la base usando sus ubicaciones de archivos predeterminadas. Esto evita depender de una ruta como `E:\DATA`, que puede no existir en tu equipo.

### 8. Paso 3 - Validar

```sql
SELECT name
FROM sys.databases
WHERE name = N'Biblioteca_S01';
GO
```

**Resultado esperado:** una fila con `Biblioteca_S01`.

### 9. Modificación con los estudiantes

Cambia el nombre a `Biblioteca_Apellido` y predice qué cambiará en el resultado de `sys.databases`.

### 10. Error frecuente

**Error:** `Database 'Biblioteca_S01' already exists.`  
**Causa:** ya ejecutaste el script antes.  
**Corrección:** usa la base existente o crea otra con un nombre diferente. No elimines bases que no hayas creado para esta práctica.

### 11. Idea clave

Crear una base no significa crear todavía sus tablas. Primero existe el contenedor; luego definimos su estructura interna.

---

## Ejemplo 2. Crear tablas con tipos de datos e IDENTITY

### 1. ¿Qué aprenderemos?

Crear tablas usando `INT`, `VARCHAR`, `CHAR`, `DATETIME` e `IDENTITY`.

### 2. Estado inicial

La base `Biblioteca_S01` existe y está vacía.

### 3. Paso 1 - Cambiar al contexto correcto

```sql
USE Biblioteca_S01;
GO
```

### 4. Paso 2 - Crear una tabla similar al ejemplo de la sesión

```sql
CREATE TABLE Juvenil (
    NumeroSocio       INT NOT NULL,
    NumMiembroAdulto  INT NOT NULL,
    FechaNac          DATETIME NOT NULL
);
GO
```

### ¿Qué significa cada columna?

| Elemento | Significado |
|---|---|
| `NumeroSocio INT` | Identificador numérico del socio juvenil |
| `NOT NULL` | La columna no admite ausencia de valor |
| `FechaNac DATETIME` | Almacena fecha y hora |

### 5. Paso 3 - Crear una tabla con autonumeración

```sql
CREATE TABLE Miembros (
    NroMiembro  INT IDENTITY(1,1) NOT NULL,
    Apellidos   VARCHAR(20) NOT NULL,
    Nombres     VARCHAR(20) NOT NULL,
    Iniciales   CHAR(1) NULL
);
GO
```

### ¿Qué hace `IDENTITY(1,1)`?

- primer `1`: valor inicial;
- segundo `1`: incremento;
- el motor genera automáticamente el número para cada fila nueva.

### 6. Validación

```sql
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'dbo'
ORDER BY TABLE_NAME;
GO
```

### 7. Predicción

Antes de ejecutar un `INSERT` en `Miembros`, responde: ¿deberías escribir manualmente el valor de `NroMiembro`?  
**Respuesta:** no en el caso normal, porque es `IDENTITY`.

---

## Ejemplo 3. ALTER TABLE y DROP TABLE con verificación

### Objetivo

Modificar una tabla existente y distinguir una modificación reversible de una eliminación destructiva.

### Paso 1 - Crear una tabla de prueba

```sql
USE Biblioteca_S01;
GO

CREATE TABLE Socio (
    IdSocio    INT NOT NULL,
    Nombres    VARCHAR(20) NOT NULL,
    Apellidos  VARCHAR(30) NOT NULL
);
GO
```

### Paso 2 - Verificar que existe

```sql
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME = 'Socio';
GO
```

### Paso 3 - Agregar una columna

```sql
ALTER TABLE Socio
ADD Edad TINYINT NULL;
GO
```

### Paso 4 - Verificar la estructura

```sql
EXEC sp_help 'Socio';
GO
```

### Paso 5 - Retirar la columna añadida

```sql
ALTER TABLE Socio
DROP COLUMN Edad;
GO
```

### Paso 6 - Eliminar la tabla de prueba

> Ejecuta este paso solo sobre `Socio`, que fue creada para este ejercicio.

```sql
DROP TABLE Socio;
GO
```

### Paso 7 - Comprobar que ya no existe

```sql
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME = 'Socio';
GO
```

**Resultado esperado:** cero filas.

### Error frecuente

Intentar eliminar una tabla referenciada por una clave foránea puede fallar. SQL Server protege la integridad estructural.

---

## Ejemplo 4. DEFAULT, CHECK y UNIQUE

> **Antes de escribir código:** aquí veremos la diferencia entre definir una regla y comprobarla. Primero creamos el constraint; después ejecutamos DML para observar si la regla acepta o rechaza los datos.


### Objetivo

Crear reglas que el motor aplique automáticamente al insertar o modificar datos.

### Parte A - DEFAULT

Agregamos un departamento y un valor por defecto.

```sql
USE Biblioteca_S01;
GO

ALTER TABLE Miembros
ADD Departamento CHAR(2) NULL;
GO

ALTER TABLE Miembros
ADD CONSTRAINT DF_Miembros_Departamento
DEFAULT 'LI' FOR Departamento;
GO
```

Insertamos sin escribir `Departamento`:

```sql
INSERT INTO Miembros (Apellidos, Nombres, Iniciales)
VALUES ('Carrasco', 'Joel', 'J');
GO

SELECT * FROM Miembros;
GO
```

**Resultado esperado:** `Departamento` toma `LI`.

### Parte B - CHECK

```sql
ALTER TABLE Miembros
ADD Telefono VARCHAR(13) NULL;
GO

ALTER TABLE Miembros
ADD CONSTRAINT CK_Miembros_Telefono
CHECK (Telefono IS NULL OR Telefono LIKE '[0-9][0-9][0-9]-[0-9][0-9][0-9]-[0-9][0-9][0-9]');
GO
```

Caso válido:

```sql
UPDATE Miembros
SET Telefono = '999-555-111'
WHERE NroMiembro = 1;
GO
```

Caso inválido controlado:

```sql
UPDATE Miembros
SET Telefono = 'ABC'
WHERE NroMiembro = 1;
GO
```

**Qué debe ocurrir:** el segundo `UPDATE` debe ser rechazado por el `CHECK`.

### Parte C - UNIQUE

```sql
CREATE TABLE Editorial (
    IdEditorial INT IDENTITY(1,1) NOT NULL,
    NomEdit     VARCHAR(20) NOT NULL
);
GO

ALTER TABLE Editorial
ADD CONSTRAINT UQ_Editorial_NomEdit UNIQUE (NomEdit);
GO
```

Inserciones válidas:

```sql
INSERT INTO Editorial (NomEdit) VALUES ('AMAZONAS');
INSERT INTO Editorial (NomEdit) VALUES ('ALGODATA');
GO
```

Error controlado:

```sql
INSERT INTO Editorial (NomEdit) VALUES ('AMAZONAS');
GO
```

**Resultado esperado:** SQL Server rechaza el duplicado.

### Idea clave

La aplicación puede validar datos, pero los constraints hacen que **la propia base de datos** proteja sus reglas.

---

## Ejemplo 5. Integridad referencial y acciones en cascada

> **Antes de escribir código:** una clave foránea no solo conecta tablas; también puede definir qué debe pasar cuando cambia o desaparece la fila padre. Por eso primero leeremos la relación y recién después ejecutaremos `UPDATE` y `DELETE`.


### Objetivo

Observar cómo una clave foránea relaciona una tabla padre con una tabla hija y cómo `ON UPDATE CASCADE` / `ON DELETE CASCADE` propagan cambios.

> Este ejemplo se realiza en una base aislada de laboratorio porque `ON DELETE CASCADE` puede eliminar registros hijos automáticamente.

### Paso 1 - Crear la base aislada

```sql
USE master;
GO
CREATE DATABASE Prueba_2_Cascada;
GO
USE Prueba_2_Cascada;
GO
```

### Paso 2 - Crear la tabla padre

```sql
CREATE TABLE CLIENTE2 (
    CLICOD CHAR(2) PRIMARY KEY,
    CLINOM VARCHAR(20) NOT NULL
);
GO
```

### Paso 3 - Crear la tabla hija

```sql
CREATE TABLE PEDIDO2 (
    PEDCOD   CHAR(3) PRIMARY KEY,
    PEDFECHA DATETIME DEFAULT GETDATE(),
    CLICOD   CHAR(2) NOT NULL,
    CONSTRAINT FK_PEDIDO2_CLIENTE2
        FOREIGN KEY (CLICOD)
        REFERENCES CLIENTE2(CLICOD)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
GO
```

### Paso 4 - Insertar clientes y pedidos

```sql
INSERT INTO CLIENTE2 VALUES ('01','JOEL CARRASCO');
INSERT INTO CLIENTE2 VALUES ('02','CESAR QUISPE');
INSERT INTO CLIENTE2 VALUES ('03','RAUL CHUCO');
INSERT INTO CLIENTE2 VALUES ('04','CESAR GUERRA');
INSERT INTO CLIENTE2 VALUES ('05','GUSTAVO CORONEL');
GO

INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P01','01');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P02','01');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P03','02');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P04','02');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P05','02');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P06','03');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P07','03');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P08','03');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P09','04');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P10','04');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P11','05');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P12','02');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P13','01');
INSERT INTO PEDIDO2 (PEDCOD, CLICOD) VALUES ('P14','01');
GO
```

### Paso 5 - Confirmar el estado inicial

```sql
SELECT * FROM CLIENTE2 ORDER BY CLICOD;
SELECT * FROM PEDIDO2 ORDER BY PEDCOD;
GO
```

El cliente `01` debe tener cuatro pedidos: `P01`, `P02`, `P13` y `P14`.

### Paso 6 - Antes de ejecutar, predice

¿Qué ocurrirá con esos cuatro pedidos si cambiamos `CLICOD` de `01` a `29`?

### Paso 7 - Actualizar la clave padre

```sql
UPDATE CLIENTE2
SET CLICOD = '29'
WHERE CLICOD = '01';
GO

SELECT * FROM PEDIDO2 WHERE CLICOD = '29';
GO
```

**Resultado esperado:** los cuatro pedidos cambian automáticamente a `29` por `ON UPDATE CASCADE`.

### Paso 8 - Antes de ejecutar, predice

¿Qué ocurrirá con los pedidos del cliente `29` si eliminamos ese cliente?

### Paso 9 - Eliminar el padre en el laboratorio

```sql
DELETE FROM CLIENTE2
WHERE CLICOD = '29';
GO

SELECT * FROM PEDIDO2
WHERE PEDCOD IN ('P01','P02','P13','P14');
GO
```

**Resultado esperado:** no quedan filas para esos pedidos porque `ON DELETE CASCADE` propagó el borrado.

### Interpretación

La cascada no es un "borrado mágico". Es una regla declarada en la clave foránea. Si existe, SQL Server aplica la acción automáticamente; si no existe, normalmente impediría eliminar el padre mientras existan referencias hijas.

---

# PARTE VII. Separar y adjuntar una base de datos

## 13. Qué significa separar una base de datos

Separar (*detach*) quita la base de una instancia de SQL Server, pero deja sus archivos físicos en el sistema de archivos. Después puede volver a adjuntarse (*attach*).

**No uses bases de producción ni archivos de origen desconocido.** Esta práctica debe hacerse solo con una base creada para laboratorio.

## 14. Ejemplo 6 - Separar y volver a adjuntar una base de laboratorio

Usaremos `Biblioteca_S01`.

### Paso 1 - Identificar los archivos físicos antes de separar

```sql
USE Biblioteca_S01;
GO

SELECT type_desc, name, physical_name
FROM sys.database_files;
GO
```

**Qué debes hacer:** copia las rutas del archivo de datos `.mdf` y del log `.ldf`.

### Paso 2 - Cambiar a `master`

```sql
USE master;
GO
```

### Paso 3 - Cerrar ventanas que estén usando la base

Si existe una consulta abierta con `USE Biblioteca_S01`, cambia a `master` o cierra esa ventana. La base debe quedar sin conexiones que bloqueen el detach.

### Paso 4 - Separar

```sql
EXEC sys.sp_detach_db
    @dbname = N'Biblioteca_S01',
    @skipchecks = N'true';
GO
```

### Validación

```sql
SELECT name
FROM sys.databases
WHERE name = N'Biblioteca_S01';
GO
```

**Resultado esperado:** cero filas. Los archivos siguen existiendo en disco.

### Paso 5 - Adjuntar nuevamente

Reemplaza las rutas por las que obtuviste en el Paso 1:

```sql
CREATE DATABASE Biblioteca_S01
ON
    (FILENAME = 'C:\RUTA_REAL\Biblioteca_S01.mdf'),
    (FILENAME = 'C:\RUTA_REAL\Biblioteca_S01_log.ldf')
FOR ATTACH;
GO
```

### Paso 6 - Comprobar

```sql
SELECT name, state_desc
FROM sys.databases
WHERE name = N'Biblioteca_S01';
GO
```

**Resultado esperado:** `ONLINE`.

### Si falla el attach

Comprueba:

- que la ruta sea exacta;
- que el servicio de SQL Server tenga acceso a la carpeta;
- que los archivos no estén siendo usados por otro proceso;
- que todos los archivos necesarios estén disponibles;
- que la base proceda de una fuente confiable.

---

# PARTE VIII. Caso integrador - AlquilerCoches

## 15. Modelo que vamos a implementar

El caso contiene estas entidades y relaciones:

```text
CLIENTE 1 ----- N RESERVA N ----- 1 AGENCIA
                    |
                    | 1
                    |
                    N
                DETALLE_RES
                 /        \
                N          N
               /            \
          1 COCHE            
              |
              N
              |
              1
            GARAJE
```

### Tablas del modelo

- `CLIENTE`
- `AGENCIA`
- `RESERVA`
- `DETALLE_RES`
- `COCHE`
- `GARAJE`

### Reglas de negocio de la sesión

1. El código de Agencia es autonumérico.
2. El DNI del cliente no se puede repetir.
3. La fecha de inicio de reserva usa `GETDATE()` por defecto.
4. El precio de alquiler no puede ser negativo.
5. Deben probarse los constraints con registros de prueba.
6. Se agrega `FechaNac` a `CLIENTE`.
7. Se ingresan cuatro clientes.
8. Se cambia la dirección del tercer cliente.
9. Se elimina el último cliente ingresado.

## 16. Ejemplo 7 - Implementación completa de `AlquilerCoches`

### 16.1. Crear la base

```sql
USE master;
GO

CREATE DATABASE AlquilerCoches;
GO

USE AlquilerCoches;
GO
```

### 16.2. Crear tablas maestras

```sql
CREATE TABLE CLIENTE (
    IdCliente   INT NOT NULL,
    NomCliente  VARCHAR(80) NOT NULL,
    DNICli      CHAR(8) NOT NULL,
    DirCli      VARCHAR(90) NULL,
    TelCli      VARCHAR(20) NULL,
    CONSTRAINT PK_CLIENTE PRIMARY KEY (IdCliente),
    CONSTRAINT UQ_CLIENTE_DNI UNIQUE (DNICli)
);
GO

CREATE TABLE AGENCIA (
    IdAgencia  INT IDENTITY(1,1) NOT NULL,
    DirecAg    VARCHAR(100) NOT NULL,
    TelfAgenc  VARCHAR(13) NULL,
    CONSTRAINT PK_AGENCIA PRIMARY KEY (IdAgencia)
);
GO

CREATE TABLE GARAJE (
    IdGaraje    INT NOT NULL,
    DirGar      VARCHAR(80) NOT NULL,
    TefGar      VARCHAR(13) NULL,
    Encargado   VARCHAR(90) NULL,
    CONSTRAINT PK_GARAJE PRIMARY KEY (IdGaraje)
);
GO
```

### 16.3. Crear COCHE

```sql
CREATE TABLE COCHE (
    IdCoche    INT NOT NULL,
    NroPlaca   VARCHAR(20) NOT NULL,
    Marca      VARCHAR(50) NULL,
    Modelo     VARCHAR(50) NULL,
    Descrip    VARCHAR(20) NULL,
    Color      VARCHAR(20) NULL,
    IdGaraje   INT NOT NULL,
    CONSTRAINT PK_COCHE PRIMARY KEY (IdCoche),
    CONSTRAINT FK_COCHE_GARAJE
        FOREIGN KEY (IdGaraje) REFERENCES GARAJE(IdGaraje)
);
GO
```

### 16.4. Crear RESERVA

```sql
CREATE TABLE RESERVA (
    NumReserva    INT NOT NULL,
    FechaInicio   DATETIME NOT NULL
        CONSTRAINT DF_RESERVA_FechaInicio DEFAULT GETDATE(),
    IdCliente     INT NOT NULL,
    FechaTermino  DATETIME NULL,
    IdAgencia     INT NOT NULL,
    CONSTRAINT PK_RESERVA PRIMARY KEY (NumReserva),
    CONSTRAINT FK_RESERVA_CLIENTE
        FOREIGN KEY (IdCliente) REFERENCES CLIENTE(IdCliente),
    CONSTRAINT FK_RESERVA_AGENCIA
        FOREIGN KEY (IdAgencia) REFERENCES AGENCIA(IdAgencia)
);
GO
```

### 16.5. Crear DETALLE_RES

```sql
CREATE TABLE DETALLE_RES (
    NumReserva  INT NOT NULL,
    IdCoche     INT NOT NULL,
    PrecioAlq   MONEY NOT NULL,
    [Desc]      MONEY NULL,
    CONSTRAINT PK_DETALLE_RES PRIMARY KEY (NumReserva, IdCoche),
    CONSTRAINT FK_DETALLE_RES_RESERVA
        FOREIGN KEY (NumReserva) REFERENCES RESERVA(NumReserva),
    CONSTRAINT FK_DETALLE_RES_COCHE
        FOREIGN KEY (IdCoche) REFERENCES COCHE(IdCoche),
    CONSTRAINT CK_DETALLE_RES_PrecioAlq CHECK (PrecioAlq >= 0)
);
GO
```

### 16.6. Validar la estructura creada

```sql
SELECT TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_NAME;
GO
```

Debes observar las seis tablas.

### 16.7. Agregar `FechaNac` a CLIENTE

```sql
ALTER TABLE CLIENTE
ADD FechaNac DATE NULL;
GO
```

### 16.8. Insertar datos maestros

```sql
INSERT INTO AGENCIA (DirecAg, TelfAgenc)
VALUES ('Av. Central 100', '014445566');
GO

INSERT INTO GARAJE (IdGaraje, DirGar, TefGar, Encargado)
VALUES (1, 'Av. Garajes 200', '015557777', 'Ana Torres');
GO

INSERT INTO COCHE (IdCoche, NroPlaca, Marca, Modelo, Descrip, Color, IdGaraje)
VALUES (101, 'ABC-123', 'Toyota', 'Yaris', 'Sedan', 'Azul', 1);
GO
```

### 16.9. Ingresar cuatro clientes

```sql
INSERT INTO CLIENTE
    (IdCliente, NomCliente, DNICli, DirCli, TelCli, FechaNac)
VALUES
    (1, 'Cliente Uno',    '70000001', 'Direccion 1', '999111111', '2000-01-10'),
    (2, 'Cliente Dos',    '70000002', 'Direccion 2', '999222222', '1999-04-20'),
    (3, 'Cliente Tres',   '70000003', 'Direccion 3', '999333333', '2001-08-15'),
    (4, 'Cliente Cuatro', '70000004', 'Direccion 4', '999444444', '1998-12-05');
GO
```

### 16.10. Probar UNIQUE con el DNI

Antes de ejecutar, predice: ¿qué ocurrirá si intentamos ingresar otro cliente con DNI `70000001`?

```sql
INSERT INTO CLIENTE
    (IdCliente, NomCliente, DNICli, DirCli)
VALUES
    (5, 'Cliente Duplicado', '70000001', 'Otra direccion');
GO
```

**Resultado esperado:** error por `UQ_CLIENTE_DNI`.

### 16.11. Cambiar la dirección del tercer cliente

```sql
UPDATE CLIENTE
SET DirCli = 'Nueva direccion del tercer cliente'
WHERE IdCliente = 3;
GO

SELECT IdCliente, NomCliente, DirCli
FROM CLIENTE
WHERE IdCliente = 3;
GO
```

### 16.12. Eliminar el último cliente ingresado

En este caso, el último cliente válido es `IdCliente = 4`.

```sql
DELETE FROM CLIENTE
WHERE IdCliente = 4;
GO
```

Validación:

```sql
SELECT * FROM CLIENTE ORDER BY IdCliente;
GO
```

### 16.13. Probar DEFAULT de la reserva

```sql
INSERT INTO RESERVA
    (NumReserva, IdCliente, FechaTermino, IdAgencia)
VALUES
    (1001, 1, DATEADD(DAY, 3, GETDATE()), 1);
GO

SELECT NumReserva, FechaInicio, FechaTermino
FROM RESERVA
WHERE NumReserva = 1001;
GO
```

**Resultado esperado:** `FechaInicio` se completa automáticamente.

### 16.14. Probar CHECK del precio

Caso válido:

```sql
INSERT INTO DETALLE_RES (NumReserva, IdCoche, PrecioAlq, [Desc])
VALUES (1001, 101, 150.00, 10.00);
GO
```

Caso inválido controlado:

```sql
INSERT INTO DETALLE_RES (NumReserva, IdCoche, PrecioAlq, [Desc])
VALUES (1001, 101, -50.00, 0.00);
GO
```

El segundo caso debe ser rechazado por `CK_DETALLE_RES_PrecioAlq`. Si el primer registro ya existe, el segundo también puede chocar antes con la clave primaria compuesta. Para probar exclusivamente el `CHECK`, usa otro `IdCoche` válido creado para laboratorio.

---

# PARTE IX. Actividades espejo

## 17. Actividad espejo 1 - Clasificar sentencias

Clasifica sin ejecutar:

```sql
ALTER TABLE CLIENTE ADD Correo VARCHAR(100) NULL;
UPDATE CLIENTE SET DirCli = 'Lima' WHERE IdCliente = 1;
CREATE TABLE PRUEBA (Id INT);
DELETE FROM CLIENTE WHERE IdCliente = 2;
```

Indica cuáles son DDL y cuáles DML y explica por qué.

## 18. Actividad espejo 2 - Diseñar una regla

Necesitas impedir que un valor de descuento sea menor que cero. Escribe solo la restricción `CHECK` que aplicarías.

## 19. Actividad espejo 3 - Predecir un error

Explica qué ocurrirá si intentas crear una reserva con un `IdCliente` que no existe, suponiendo que la clave foránea está activa.

## 20. Actividad espejo 4 - Interpretar una cascada

En una relación padre-hijo con `ON DELETE CASCADE`, responde:

1. ¿qué tabla inicia el borrado?;
2. ¿qué tabla recibe el efecto?;
3. ¿por qué esta regla requiere cuidado en sistemas reales?

---

# PARTE X. Tabla de errores frecuentes

| Situación | Causa probable | Cómo corregir |
|---|---|---|
| No conecta a `localhost` | servicio/instancia incorrecta | confirma nombre de instancia y que SQL Server esté iniciado |
| `Database already exists` | script ejecutado antes | reutiliza la base o usa otro nombre |
| `Invalid object name` | contexto de base incorrecto o tabla no creada | revisa `USE ...` y la existencia de la tabla |
| Error por `UNIQUE` | valor duplicado | cambia el dato o revisa si el registro ya existe |
| Error por `CHECK` | dato fuera de regla | corrige el valor para que cumpla la condición |
| Error por `FOREIGN KEY` | padre inexistente | crea primero el registro padre válido |
| No puede hacer detach | conexiones activas | cambia las consultas a `master` y cierra conexiones a la base |
| Attach falla por acceso | ruta o permisos | confirma ruta y permisos del servicio de SQL Server |
| Ruta `E:\DATA` no existe | ruta del ejemplo no existe en tu PC | usa ubicación predeterminada o una carpeta real autorizada |
| `INT(11)` genera error | esa forma no corresponde al tipo `INT` de SQL Server | usa `INT` |

---

# PARTE XI. Puntos de control

## 21. Punto de control A - Entorno

- [ ] Puedo abrir SSMS.
- [ ] Puedo conectarme al Database Engine.
- [ ] Ejecuté la consulta de versión.
- [ ] Sé distinguir SSMS del motor de SQL Server.

## 22. Punto de control B - DDL/DML

- [ ] Puedo explicar `CREATE`, `ALTER` y `DROP`.
- [ ] Puedo explicar `INSERT`, `UPDATE`, `DELETE` y `SELECT`.
- [ ] Sé qué instrucciones cambian estructura y cuáles trabajan con registros.

## 23. Punto de control C - Integridad

- [ ] Puedo explicar `PRIMARY KEY`.
- [ ] Puedo explicar `FOREIGN KEY`.
- [ ] Puedo demostrar `DEFAULT`.
- [ ] Puedo demostrar `CHECK`.
- [ ] Puedo provocar de forma controlada un error por `UNIQUE`.

## 24. Punto de control D - Caso integrador

- [ ] Creé `AlquilerCoches`.
- [ ] Creé las seis tablas.
- [ ] Implementé DNI único.
- [ ] Implementé fecha por defecto.
- [ ] Implementé precio no negativo.
- [ ] Agregué `FechaNac`.
- [ ] Ingresé cuatro clientes.
- [ ] Actualicé al tercer cliente.
- [ ] Eliminé al cuarto cliente.

---

# PARTE XII. Evidencia y entrega

## 25. Evidencia mínima

Guarda todos los scripts que utilizaste en un único archivo:

```text
Tarea-S01-Nombre-ApellidoPaterno.sql
```

El archivo debe contener, en orden:

1. `CREATE DATABASE`;
2. creación de tablas;
3. constraints;
4. `ALTER TABLE` para `FechaNac`;
5. inserciones;
6. pruebas de constraints;
7. actualización del tercer cliente;
8. eliminación del cuarto cliente;
9. consultas de validación.

## 26. Checklist final

- [ ] El script es legible y está ordenado.
- [ ] No depende de rutas que no existan en otra PC, salvo la sección explícita de attach.
- [ ] Las tablas se crean en orden compatible con las claves foráneas.
- [ ] Los datos padres existen antes de insertar datos hijos.
- [ ] Probé al menos un caso correcto y un caso rechazado por constraint.
- [ ] Puedo explicar por qué ocurrió cada error de validación.
- [ ] No ejecuté pruebas destructivas fuera de bases de laboratorio.

## 27. Mapa final de la sesión

```text
CLIENTE SSMS
   ↓ consulta
SERVIDOR SQL SERVER
   ↓
DDL crea/modifica estructura
   ↓
CONSTRAINTS protegen integridad
   ↓
DML inserta/actualiza/elimina datos
   ↓
VALIDACIÓN confirma reglas
   ↓
DETACH/ATTACH gestiona archivos de laboratorio
   ↓
ALQUILERCOCHES integra todo
```

## 28. Cierre

En esta sesión aprendiste a pasar de una idea de arquitectura cliente/servidor a una implementación concreta en SQL Server. Creaste bases y tablas, modificaste estructuras, aplicaste restricciones, validaste registros y observaste cómo las relaciones pueden controlar o propagar cambios.

Repasa especialmente:

- diferencia entre estructura y datos;
- orden correcto para crear tablas relacionadas;
- propósito de cada constraint;
- lectura de mensajes de error;
- precauciones con `DROP`, `DELETE CASCADE` y detach/attach.

La siguiente sesión podrá apoyarse en esta base porque ya tienes el entorno y los fundamentos de integridad preparados.

**Blog:** https://lideratecacademy.com/  
**Canal YouTube:** https://www.youtube.com/@LideratecAcademy
