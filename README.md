(verificar archivo de robustez.md, para la base de datos robusta con procedimiento almacenado)
# Inventory_database
### Proyecto integrador, Programación Full Stack, para la materia Base de datos III

**Intengrantes:** Gamarra Uziel,  y Parejas Lucas

Creamos una base de datos relacional con un sistema de inventarios anidados, tomando de referencia un famosísimo videojuego llamado Minecraft, haciendo uso de las JSONB para guardar dentro de la base de datos, un inventario que contiene otros inventarios

Aquí nuestro progreso

# Creación de la base y las tablas de datos

SQL
```
---1. creamos la base de datos
postgres=# create database minecraft_inventory
postgres-# ;

```

SQL
```
--- aquí para verificar que exista 
postgres=# \l
                                                            List of databases
        Name         |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges   
---------------------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 minecraft_inventory | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | 
 postgres            | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | 
 soundwave_db        | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | 
 template0           | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | =c/postgres          +
                     |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1           | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | =c/postgres          +
                     |          |          |                 |             |             |            |           | postgres=CTc/postgres
 tienda_videojuegos  | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | 
 turismo_reservas    | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | 
 veterinaria         | postgres | UTF8     | libc            | es_AR.UTF-8 | es_AR.UTF-8 |            |           | 
(8 rows)
-- Como verán, aparecen bastantes bases de datos, pero la importante es la primera
```

SQL
```
postgres=# \c minecraft_invnetory
falló la conexión al servidor en el socket «/var/run/postgresql/.s.PGSQL.5432»: FATAL:  database "minecraft_invnetory" does not exist                                       
Previous connection kept                                                                                                                                                    
---aquí nos equivocamos y se nos cruzó el dedo, suele pasar
```

SQL
```
postgres=# \c minecraft_inventory                                                                                                                                          --escribimos este comando ara conectarse
You are now connected to database "minecraft_inventory" as user "postgres".
minecraft_inventory=# --y ahora estamos dentro dentro de la base de datos
```

SQL
```
minecraft_inventory=# CREATE TABLE jugadores (
    uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(), --el UserUniqueID, que si no se le asigna, se pone por defecto, un valor al azar
    username VARCHAR(16) NOT NULL UNIQUE,            --el nombre de usuario, de límite 16, no debe ser nulo, y debe ser único
    rango VARCHAR(20) DEFAULT 'usuario');            --el rango en el servidor, (es decir, el rol), de limite 20 caracteres, por defecto 'usuario'
minecraft_inventory=# --tabla de jugadores y usuarios, creada
```

SQL
```
minecraft_inventory=# CREATE TABLE inventarios (
    id SERIAL PRIMARY KEY,                           --creamos un ID, que aumenta automáticamente, y lo dejamos de clave primaria para acceder a la tabla
    jugador_uuid UUID REFERENCES jugadores(uuid) ON DELETE CASCADE, --creamos el UserUniqueID del jugador, que referencia a la tabla jugadores, en el uuid, y activamos el eliminar con cascada para facilitar la limpieza de errores de dedo, jeje
    tipo_inventario VARCHAR(20) NOT NULL,            --aquí ponemos los tipos de inventario: 'PRINCPIAL' (cada usuario tiene 1 'Principal'), 'SHULKER_BOX'(un usuario puede tener múltiples SHULKERs, y cada SHULKER pertenecer a muchos usuarios) y 'ENDER_CHEST' (Cada ususario puede tener un ENDER_CHEST)
    ultima_actualizacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP); --para que sepamos cuando, fué la última vez que agregó un ítem
minecraft_inventory=# --tabla de inventarios creada, siguiente
minecraft_inventory=# --ahora la tabla de ítems, e inventarios anidados
```

SQL
```
minecraft_inventory=# CREATE TABLE slots_items ( id BIGSERIAL PRIMARY KEY, inventario_id INT REFERENCES inventarios(id) ON DELETE CASCADE, slot_n INT NOT NULL, item_id, VACCHAR(50) NOT NULL, cantidad INT DEFAULT 1,
minecraft_inventory(# -- Si el item está dentro de la shulker (que es un inventario que contiene otro inventario) usamos JSONB
minecraft_inventory(# shulker_contenido JSONB,
minecraft_inventory(# durabilidad_egistro INT4RANGE, CONSTRAINT unco_slot_por_inventario UNIQUE(inventario_id, slot_n));
ERROR:  syntax error at or near ","
LÍNEA 1: ...d) ON DELETE CASCADE, slot_n INT NOT NULL, item_id, VACCHAR(...
---aquí tuvimos otro error de dedo, sólo habíamos observado el ., pero luego, nos dimos cuenta de VACCHAR, el sospechoso, oculto en la sintaxis
```

SQL
```
minecraft_inventory=# CREATE TABLE slots_items (
    id BIGSERIAL PRIMARY KEY,                                       -- Debido a que Serial tiene límites de 32 bits, usamos BigSerial, para 64 bits, y manejar números enormes
    inventario_id INT REFERENCES inventarios(id) ON DELETE CASCADE, -- Creamos la referencia al id del inventario, con eliminación por cascada
    slot_n INT NOT NULL,                                            -- colocamos el número del slot en el inventario, no puede ser nulo
    item_id VARCHAR(50) NOT NULL,                                   -- colocamos el nombre del item (ejemplo: minecraft:dirt), ponemos de límite 50, es bastante generoso
    cantidad INT DEFAULT 1,                                         -- aquí estaría la cantidad de cada item (como si fuera el stock), por defecto es 1
-- Si el item está dentro de la shulker (que es un inventario que contiene otro inventario) usamos JSONB
    shulker_contenido JSONB,
    durabilidad_egistro INT4RANGE, CONSTRAINT unco_slot_por_inventario UNIQUE(inventario_id, slot_n));
```
# Inserción de datos

SQL
```
minecraft_inventory=# INSERT INTO inventarios (jugador_uuid, tipo_inventario) SELECT uuid. 'PRINCIPAL' FROM jugadores;
ERROR:  syntax error at or near "'PRINCIPAL'"
LÍNEA 1: ...rios (jugador_uuid, tipo_inventario) SELECT uuid. 'PRINCIPAL...
```
Aqui se muestra que a veces, los puntos y las comas, son muy fundamentales de diferenciar, y que...esos errores de dedo, pueden cansar a cualquiera, hasta que al fun funciona

SQL
```
minecraft_inventory=# INSERT INTO inventarios (jugador_uuid, tipo_inventario) SELECT uuid, 'PRINCIPAL' FROM jugadores;
INSERT 0 3
minecraft_inventory=# INSERT INTO inventarios (jugador_uuid, tipo_inventario) SELECT uuid, 'ENDER_CHEST' FROM jugadores;
INSERT 0 3
--¡SÍ!
```

# Inserción de datos con lógica

SQL
```
minecraft_inventory=# INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad, shulker_contenido, dureabilidad_registro) SELECT (g % 6) + 1 AS inventario_id, --distribuye entre los 6 inventarios
minecraft_inventory-# (g % 36) AS slot_n, ---distribuye entre los slots del 0 al 35
minecraft_inventory-# CASE WHEN g % 10 = 0 THEN 'minecraft:blue_shulker_box'
minecraft_inventory-# WHEN g % 10 = 1 then 'minecraft:diamond_pickaxe'
minecraft_inventory-# ELSE 'minecraft:cobblestone'
minecraft_inventory-# END AS item_id, CASE WHEN g % 10 = 0 THEN 1
minecraft_inventory-# ELSE (g % 64) + 1
minecraft_inventory-# END AS cantidad, --Si es una Shulker (g % 10 == 0), le metemos un JSON con sus items internos
minecraft_inventory-# CASE WHEN g % 10 = 0 THEN jsonb_build_array(json_build_object('slot', 0, 'id',
minecraft_inventory(# 'minecraft:netherite_ingot, 'count', (g + 4) + 1),
minecraft_inventory'# json_build_object('slot', 1, 'id', 'minecraft:elytra', jsonb_build_object('slot', 2, 'id', (ARRAY['minecraft:emerald'. 'minecraft:totem_of_undying'])[g%2 + 1], 'count', g%5))
minecraft_inventory'# ELSE NULL END AS shulker_contenido, ---Rango de durabilidad para ñas herramientas
minecraft_inventory'# CASE WHEN g % 10 = 1 THEN int4range(0, (g % 1561)) --tomada durabilidad del pico de diamante del juego
minecraft_inventory'# ELSE NULL
minecraft_inventory'# END AS durabilidad_registro FROM generate_series(1, 1000000) AS g;
minecraft_inventory'# 
minecraft_inventory'# ;
minecraft_inventory'# 
---la peor pesadilla de un programador de SQL, que el ; no te diga qué pasó, que no te diga ni error, ni que lo hicistebien, todo por una "'"
```

Intentamos nuevamente

SQL
``` 
minecraft_inventory=# -- 1. Insertar algunos jugadores de prueba
INSERT INTO jugadores (username, rango) VALUES 
('Uziel_gamma', 'OP'), ('Donato_Craft', 'VIP'), ('Faty_Miner', 'Usuario'); --aquí cometimos el error (porque no sabíamos que era una "'", mal puesta, de hacer todo de nuevo, lo cuál, fué bastante ingenuo de nuestra parte 😿

-- 2. Crearles sus inventarios
INSERT INTO inventarios (jugador_uuid, tipo_inventario) 
SELECT uuid, 'PRINCIPAL' FROM jugadores;
INSERT INTO inventarios (jugador_uuid, tipo_inventario) 
SELECT uuid, 'ENDER_CHEST' FROM jugadores;

-- 3. Inyección Masiva de 1.000.000 de slots usando matemáticas para simular Shulkers
INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad, shulker_contenido, durabilidad_registro)
SELECT 
    (g % 6) + 1 AS inventario_id, -- Distribuye entre los 6 inventarios creados
    (g % 36) AS slot_n,           -- Slots del 0 al 35
    CASE 
        WHEN g % 10 = 0 THEN 'minecraft:blue_shulker_box'
        WHEN g % 10 = 1 THEN 'minecraft:diamond_pickaxe'
        ELSE 'minecraft:cobblestone'
    END AS item_id,
    CASE 
        WHEN g % 10 = 0 THEN 1
        ELSE (g % 64) + 1
    END AS cantidad,
    -- Si es una Shulker (g % 10 == 0), le metemos un JSON con sus items internos
    CASE 
        WHEN g % 10 = 0 THEN 
            jsonb_build_array(
                jsonb_build_object('slot', 0, 'id', 'minecraft:netherite_ingot', 'count', (g % 4) + 1),
                jsonb_build_object('slot', 1, 'id', 'minecraft:elytra', 'count', 1),
                jsonb_build_object('slot', 2, 'id', (ARRAY['minecraft:emerald', 'minecraft:totem_of_undying'])[g%2 + 1], 'count', g%5)
            )
        ELSE NULL
    END AS shulker_contenido,
    -- Rango de durabilidad para las herramientas
    CASE 
        WHEN g % 10 = 1 THEN int4range(0, (g % 1561))
        ELSE NULL
    END AS durabilidad_registro
FROM generate_series(1, 1000000) AS g;
ERROR:  duplicate key value violates unique constraint "jugadores_username_key"
DETALLE:  Key (username)=(Uziel_gamma) already exists.
INSERT 0 3
INSERT 0 3
ERROR:  relation "slots_item" does not exist
LÍNEA 1: INSERT INTO slots_item (inventario_id, slot_n, item_id, cant...
                     ^

```
Esta vez, intentamos primero limpiar, sino, no íbamos a avanzar 🫤

SQL
```
minecraft_inventory=# -- 1. Limpieza total por si quedaron restos de tablas previas
DROP TABLE IF EXISTS slots_item CASCADE; --eliminar la tabla si existe, en cascada, para borrar referencias
DROP TABLE IF EXISTS inventarios CASCADE;
DROP TABLE IF EXISTS jugadores CASCADE;

--Creamos TODO de nuevo
-- 2. Creación de la Tabla Jugadores
CREATE TABLE jugadores (
    uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(16) NOT NULL UNIQUE,
    rango VARCHAR(20) DEFAULT 'usuario'
);

-- 3. Creación de la Tabla Inventarios
CREATE TABLE inventarios (
    id SERIAL PRIMARY KEY,
    jugador_uuid UUID REFERENCES jugadores(uuid) ON DELETE CASCADE,
    tipo_inventario VARCHAR(20) NOT NULL, -- 'PRINCIPAL', 'ENDER_CHEST'
    ultima_actualizacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. Creación de la Tabla Slots (La que te faltaba)
CREATE TABLE slots_item (
    id BIGSERIAL PRIMARY KEY,
    inventario_id INT REFERENCES inventarios(id) ON DELETE CASCADE,
    slot_n INT NOT NULL, 
    item_id VARCHAR(50) NOT NULL, 
    cantidad INT DEFAULT 1,
    shulker_contenido JSONB, 
    durabilidad_registro INT4RANGE,
    CONSTRAINT unco_slot_por_inventario UNIQUE(inventario_id, slot_n)
);

-- 5. Inserción Limpia de Jugadores
INSERT INTO jugadores (username, rango) VALUES 
('Uziel_gamma', 'OP'), 
('Donato_Craft', 'VIP'), 
('Faty_Miner', 'Usuario');

-- 6. Creación de Inventarios (Exactamente 2 por jugador: 6 en total)
INSERT INTO inventarios (jugador_uuid, tipo_inventario) 
SELECT uuid, 'PRINCIPAL' FROM jugadores;

INSERT INTO inventarios (jugador_uuid, tipo_inventario) 
FROM generate_series(1, 1000000) AS g;(0, (g % 1561))ARRAY['minecraft:emerald', 'minecraft:totem_of_undying'])[g%2 + 1], 'count', g%5)
NOTICE:  table "slots_item" does not exist, skipping
DROP TABLE
NOTICE:  drop cascades to constraint slots_items_inventario_id_fkey on table slots_items
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
ERROR:  relation "unco_slot_por_inventario" already exists
INSERT 0 3
INSERT 0 3
INSERT 0 3
ERROR:  relation "slots_item" does not exist
LÍNEA 1: INSERT INTO slots_item (inventario_id, slot_n, item_id, cant...
                     ^

```
También hubo un error de estándares, por el nombre de una fila (la regla de nunca llamar las tablas con plural)

SQL
```
minecraft_inventory=# \dt
            List of relations
 Schema |    Name     | Type  |  Owner   
--------+-------------+-------+----------
 public | inventarios | table | postgres
 public | jugadores   | table | postgres
 public | slots_items | table | postgres
(3 rows)
--al principio tuvimos eso, (slots_items vs slots_item), ahora que encontramos el error, podíamos arreglarlo
minecraft_inventory=# -- 1. Limpieza total por si quedaron restos de tablas previas
DROP TABLE IF EXISTS slots_iInsertar algunos jugadores de prueba
INSERT INTO jugadores (usernLimpieza total por si quedaron restos de tablas previas
DROP TABLE IF EXISTS slots_iInsertar algunos jugadores de prueba
INSERT INTO jugadores (usernLimpieza total por si quedaron restos de tablas previas
DROP TABLE IF EXISTS slots_iInsertar algunos jugadores de prueba
INSERT INTO jugadores (usernLimpieza total por si quedaron restos de tablas previas
DROP TABLE IF EXISTS s\dt
                      -- 1. Limpieza total por si quedaron restos de tablas previas
DROP TABLE IF EXISTS slots_iInsertar algunos jugadores de prueba
INSERT INTO jugadores -- 1. Limpieza total por si quedaron restos de tablas previas
DROP TABLE IF EXISTS s\dt
                      ;
minecraft_inventory=# -- 1. Limpieza total de absolventemte todo (singular y plural)
DROP TABLE IF EXISTS slots_item CASCADE;
DROP TABLE IF EXISTS slots_items CASCADE; -- <--- Borramos al fantasma
DROP TABLE IF EXISTS inventarios CASCADE;
DROP TABLE IF EXISTS jugadores CASCADE;

-- 2. Estructura Limpia
CREATE TABLE jugadores (
    uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(16) NOT NULL UNIQUE,
    rango VARCHAR(20) DEFAULT 'usuario'
);

CREATE TABLE inventarios (
    id SERIAL PRIMARY KEY,
    jugador_uuid UUID REFERENCES jugadores(uuid) ON DELETE CASCADE,
    tipo_inventario VARCHAR(20) NOT NULL,
    ultima_actualizacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE slots_item (
    id BIGSERIAL PRIMARY KEY,
    inventario_id INT REFERENCES inventarios(id) ON DELETE CASCADE,
    slot_n INT NOT NULL, 
    item_id VARCHAR(50) NOT NULL, 
    cantidad INT DEFAULT 1,
    shulker_contenido JSONB, 
    durabilidad_registro INT4RANGE,
    CONSTRAINT unco_slot_por_inventario UNIQUE(inventario_id, slot_n)
);

-- 3. Carga de Datos
INSERT INTO jugadores (username, rango) VALUES 
('Uziel_gamma', 'OP'), ('Donato_Craft', 'VIP'), ('Faty_Miner', 'Usuario');

INSERT INTO inventarios (jugador_uuid, tipo_inventario) 
SELECT uuid, 'PRINCIPAL' FROM jugadores;

INSERT INTO inventarios (jugador_uuid, tipo_inventario) 
SELECT uuid, 'ENDER_CHEST' FROM jugadores;

-- 4. El Millón de registros definitivo
INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad, shulker_contenido, durabilidad_registro)
FROM generate_series(1, 1000000) AS g;(0, (g % 1561))ARRAY['minecraft:emerald', 'minecraft:totem_of_undying'])[g%2 + 1], 'count', g%5)
NOTICE:  table "slots_item" does not exist, skipping
DROP TABLE
DROP TABLE
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
INSERT 0 3
INSERT 0 3
INSERT 0 3
ERROR:  duplicate key value violates unique constraint "unco_slot_por_inventario"
DETALLE:  Key (inventario_id, slot_n)=(2, 1) already exists.
--aquí otro problema, el % generaba demasiadas colisiones
-- pero ya teníamos la estructura completa 
minecraft_inventory=# \d
                 List of relations
 Schema |        Name        |   Type   |  Owner   
--------+--------------------+----------+----------
 public | inventarios        | table    | postgres
 public | inventarios_id_seq | sequence | postgres
 public | jugadores          | table    | postgres
 public | slots_item         | table    | postgres
 public | slots_item_id_seq  | sequence | postgres
(5 rows)

```

Intentemos nuevamente

SQL
```
minecraft_inventory=# -- 1. Limpieza rápida de los datos actuales
TRUNCATE TABLE slots_item CASCADE;
DELETE FROM inventarios;

-- 2. Creamos 30.000 inventarios para los jugadores (así entra el millón de slots sin repetirse)
INSERT INTO inventarios (jugador_uuid, tipo_inventario)
SELECT 
    uuid, 
    'COFRE_VIRTUAL_' || i AS tipo_inventario
FROM jugadores
CROSS JOIN generate_series(1, 10000) AS i; -- 3 jugadores * 10.000 = 30.000 inventarios

-- 3. Ahora sí, el Millón de registros definitivo sin colisiones
INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad, shulker_contenido, durabilidad_registro)
SELECT 
    (g % 27777) + 1 AS inventario_id, -- Distribuye en el rango de los inventarios creados
    (g % 36) AS slot_n,               -- Slots del 0 al 35 (Sin repetir por inventario)
    CASE 
        WHEN g % 10 = 0 THEN 'minecraft:blue_shulker_box'
        WHEN g % 10 = 1 THEN 'minecraft:diamond_pickaxe'
        ELSE 'minecraft:cobblestone'
    END AS item_id,
    CASE 
        WHEN g % 10 = 0 THEN 1
        ELSE (g % 64) + 1
    END AS cantidad,
    CASE 
        WHEN g % 10 = 0 THEN 
            jsonb_build_array(
                jsonb_build_object('slot', 0, 'id', 'minecraft:netherite_ingot', 'count', (g % 4) + 1),
                jsonb_build_object('slot', 1, 'id', 'minecraft:elytra', 'count', 1),
                jsonb_build_object('slot', 2, 'id', (ARRAY['minecraft:emerald', 'minecraft:totem_of_undying'])[g%2 + 1], 'count', g%5)
            )
        ELSE NULL
    END AS shulker_contenido,
    CASE 
        WHEN g % 10 = 1 THEN int4range(0, (g % 1561))
        ELSE NULL
    END AS durabilidad_registro
FROM generate_series(1, 1000000) AS g;

TRUNCATE TABLE
DELETE 30000
INSERT 0 1000000
--- al fin pudimos meter los 1M de registros, ahora podemos seguir avanzando
```

# Creación de índices
SQL
```
minecraft_inventory=# -- Ejecutá esto para activar la magia de indexación sobre el millón de filas
CREATE INDEX idx_slots_n ON slots_item USING btree (slot_n);
CREATE INDEX idx_item_id_hash ON slots_item USING hash (item_id);
CREATE INDEX idx_shulker_contenido_gin ON slots_item USING gin (shulker_contenido);
CREATE INDEX idx_durabilidad_gist ON slots_item USING gist (durabilidad_registro);
CREATE INDEX
CREATE INDEX
CREATE INDEX
CREATE INDEX
--los índices se crearon correctamente, ahora podemos borrarlos para el test de velocidad
minecraft_inventory=# -- Borramos el índice temporalmente para la prueba
DROP INDEX IF EXISTS idx_shulker_contenido_gin;
--MODO SIN INDICES
-- Pedimos el plan detallado en formato JSON
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON)
SELECT s.slot_n, s.item_id 
FROM slots_item s
WHERE s.shulker_contenido @> '[{"id": "minecraft:netherite_ingot"}]';
DROP INDEX
minecraft_inventory=# -- Recreamos el índice GIN
CREATE INDEX idx_shulker_contenido_gin ON slots_item USING gin (shulker_contenido);
--MODO CON INDICES
-- Volvemos a pedir el plan estructurado
EXPLAIN (ANALYZE, COSTS, VERBOSE, BUFFERS, FORMAT JSON)
SELECT s.slot_n, s.item_id 
FROM slots_item s
WHERE s.shulker_contenido @> '[{"id": "minecraft:netherite_ingot"}]';
CREATE INDEX

```
# Los archivos de las pruebas están en
Json
# Los archivos de las imagenes de la base de datos (el DER) están en
Diagrams
