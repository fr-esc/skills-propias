---
name: comment-to-sql
alias: sql-from-comments
description: Convierte archivos .sql con comentarios guía en sentencias SQL limpias y seguras in-place, conservando los comentarios originales y añadiendo un bloque resumen temporal al final.
version: 1.1.1
language: es
trigger: "implementa|implanta|genera el código de *.sql"
supported_extensions:
  - .sql
outputs:
  - in-place-sql-code
---

# Skill: `comment-to-sql`

## 1. Propósito y Cuándo Activarse
Esta skill se activa cuando el usuario solicita implementar, implantar o generar el código de un archivo `.sql` que contiene comentarios descriptivos (`-- ...` o `/* ... */`) indicando qué tablas, vistas, consultas o modificaciones debe realizar el script.

El objetivo es **rellenar el archivo `.sql` in-place**: mantener cada comentario original del usuario exactamente como ancla y escribir debajo de cada uno la sentencia SQL correspondiente, cerrando el archivo con un bloque de comentario multilínea (`/* Realizado: ... */`) que resuma lo implementado.

---

## 2. Reglas de Oro (Inquebrantables para el Modelo)

1. **Conservación de Comentarios Ancla:** NUNCA borres ni modifiques los comentarios originales (`-- ...`) del usuario. Escribe la sentencia SQL justo debajo de cada comentario.
2. **Cero Comentarios Extra:** No añadas comentarios explicativos línea por línea dentro de las tablas o consultas. La única excepción es el bloque resumen `/* Realizado: ... */` al final del archivo.
3. **Dialecto Estándar y Agnóstico:** Usa SQL estándar (ANSI / compatible con PostgreSQL y SQLite por defecto) salvo que un comentario especifique un motor concreto (`PostgreSQL`, `MySQL`, `SQLite`, `SQL Server`).
4. **Seguridad e Idempotencia:**
   - Usa palabras clave en MAYÚSCULAS (`SELECT`, `CREATE TABLE`, `WHERE`, `JOIN`) e identificadores de tablas/columnas en `snake_case` minúsculas.
   - En creación de estructuras (DDL), usa `IF NOT EXISTS` por defecto (`CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`).
   - Termina siempre cada sentencia con punto y coma (`;`).
   - En consultas o vistas, evita `SELECT *`; enumera las columnas explícitamente cuando se conozcan.
5. **Tablas y Vistas Externas (Dependencias):** Si un comentario hace referencia a una tabla existente (en un `FOREIGN KEY REFERENCES ...`, un `JOIN` o un `INSERT`), **asume que ya existe en otro archivo/migración y NO escribas su `CREATE TABLE`**. Anótalo como tabla externa asumida en el bloque `Realizado:`.
6. **Bloque Resumen Final Obligatorio:** Al final del archivo `.sql`, añade un bloque `/* Realizado: ... */` detallando brevemente qué sentencias se crearon, qué dialecto o tipos se asumieron y qué tablas externas se referenciaron.

---

## 3. System Prompt Directo (Para Modelos Locales <= 7B)

```text
You are a strict SQL code implementation agent inside an IDE.
Task: Read a .sql file containing user comments (-- ...) and write the corresponding SQL statements directly below each comment.

RULES:
1. KEEP ALL ORIGINAL USER COMMENTS (-- ...) intact. Write SQL right below them.
2. NO extra inline comments.
3. Use standard ANSI/PostgreSQL-compatible SQL unless another engine is specified in comments.
4. Formatting: Uppercase SQL keywords, snake_case identifiers, end every statement with ';', use IF NOT EXISTS for DDL.
5. External tables: If a comment references another table (via FOREIGN KEY or JOIN), DO NOT create that external table; just reference it.
6. FINAL SUMMARY: At the very bottom of the file, append a multiline comment block: /*\nRealizado:\n- [summary of changes in Spanish]\n*/
7. Output ONLY the complete valid .sql file content. No conversational text outside.
```

---

## 4. Ejemplos Few-Shot

### Ejemplo 1: Creación de Tabla e Índice con Clave Foránea Externa (DDL)
**Entrada:**
```sql
-- tabla pedidos con id autoincremental, usuario_id (foranea a tabla usuarios), total decimal y fecha creacion

-- indice para buscar pedidos rapido por usuario_id
```
**Salida:**
```sql
-- tabla pedidos con id autoincremental, usuario_id (foranea a tabla usuarios), total decimal y fecha creacion
CREATE TABLE IF NOT EXISTS pedidos (
    id SERIAL PRIMARY KEY,
    usuario_id INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE RESTRICT,
    total NUMERIC(10, 2) NOT NULL DEFAULT 0.00,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- indice para buscar pedidos rapido por usuario_id
CREATE INDEX IF NOT EXISTS idx_pedidos_usuario_id ON pedidos(usuario_id);

/*
Realizado:
- Crear tabla 'pedidos' con clave primaria serial, total numérico y marca de tiempo por defecto.
- Referenciar tabla externa 'usuarios(id)' mediante clave foránea sin recrear la tabla usuarios.
- Crear índice 'idx_pedidos_usuario_id' para optimizar búsquedas por usuario.
*/
```

### Ejemplo 2: Consulta Analítica con JOIN (DML)
**Entrada:**
```sql
-- obtener id, nombre y total gastado de los usuarios activos que tengan mas de 1 pedido, ordenado de mayor a menor gasto
```
**Salida:**
```sql
-- obtener id, nombre y total gastado de los usuarios activos que tengan mas de 1 pedido, ordenado de mayor a menor gasto
SELECT
    u.id,
    u.nombre,
    COUNT(p.id) AS total_pedidos,
    SUM(p.total) AS total_gastado
FROM usuarios u
INNER JOIN pedidos p ON p.usuario_id = u.id
WHERE u.activo = TRUE
GROUP BY u.id, u.nombre
HAVING COUNT(p.id) > 1
ORDER BY total_gastado DESC;

/*
Realizado:
- Implementar consulta SELECT con INNER JOIN entre las tablas existentes 'usuarios' y 'pedidos'.
- Filtrar por usuarios activos (WHERE), agrupar y filtrar aquellos con más de 1 pedido (HAVING COUNT > 1).
- Ordenar descendentemente por la suma total gastada.
*/
```

---

## Eval de aceptación

Prueba manual mínima. Se considera **APROBADA** solo si se cumplen los cuatro puntos.

**Entrada** (archivo `migracion.sql`)
```sql
-- Crear tabla pedidos con id, cliente_id, total y fecha
-- Crear indice sobre cliente_id
-- Total facturado por cliente
```

**Salida esperada**
- `CREATE TABLE IF NOT EXISTS pedidos (...)` con `FOREIGN KEY (cliente_id) REFERENCES clientes(id)` y **sin** crear la tabla `clientes`.
- `CREATE INDEX IF NOT EXISTS` sobre `cliente_id`.
- Consulta con columnas explícitas y `SUM(total)`, agrupando por cliente.
- Punto y coma al final de cada sentencia y palabras clave en MAYÚSCULAS.

**Prohibido en esta prueba**
- `SELECT *`.
- Crear la tabla `clientes` o cualquier tabla externa asumida.
- Añadir comentarios explicativos línea a línea.
