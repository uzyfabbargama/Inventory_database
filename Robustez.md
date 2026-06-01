#Aquí mosytramos lo que agregamos para crear una base de datos robusta

### registros de auditoría
SQL
```
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    usuario TEXT DEFAULT CURRENT_USER,
    estado_sql TEXT,
    mensaje_error TEXT,
    contexto TEXT
);
```
Se nos pide un procedimiento que gestione una tarea compleja. 
Vamos a crear uno para "Agregar un ítem a un inventario" use:
  COMMIT, ROLLBACK y %TYPE para asegurar robuste
SQL
```
CREATE OR REPLACE PROCEDURE agregar_item_seguro(
    p_jugador_username jugadores.username%TYPE,
    p_tipo_inv inventarios.tipo_inventario%TYPE,
    p_item_id slots_item.item_id%TYPE,
    p_cantidad slots_item.cantidad%TYPE,
    p_slot slots_item.slot_n%TYPE
)
AS $$
DECLARE
    v_jugador_uuid jugadores.uuid%TYPE;
    v_inventario_id inventarios.id%TYPE;
BEGIN
    -- Buscamos el UUID del jugador
    SELECT uuid INTO v_jugador_uuid FROM jugadores WHERE username = p_jugador_username;
    
    -- Buscamos o creamos el inventario
    SELECT id INTO v_inventario_id FROM inventarios 
    WHERE jugador_uuid = v_jugador_uuid AND tipo_inventario = p_tipo_inv;

    -- Intento de inserción
    INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad)
    VALUES (v_inventario_id, p_slot, p_item_id, p_cantidad);

    COMMIT; -- Confirmamos la operación [cite: 15]
    
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK; -- Si falla (ej. slot ocupado), deshacemos todo [cite: 15]
        -- Registro en la tabla de auditoría (Requisito C.20)
        INSERT INTO audit_logs (estado_sql, mensaje_error)
        VALUES (SQLSTATE, SQLERRM);
        RAISE NOTICE 'Error registrado en audit_logs';
END;
$$ LANGUAGE plpgsql;
```
Aquí nos dió otro error:
NOTICE:  type reference jugadores.username%TYPE converted to character varying
NOTICE:  type reference inventarios.tipo_inventario%TYPE converted to character varying
ERROR:  relation "slots_item" does not exist
para resolverlo, debemos:
¡No se asusten! Lo que ven no es un error fatal, sino un NOTICE (un aviso)
de PostgreSQL.

El motor nos está avisando que leyó correctamente el comando %TYPE y lo 
"tradujo" al tipo de dato real (como character variable) para poder 
ejecutarlo. 
Sin embargo, hay un detalle técnico con el uso de transacciones (COMMIT/ROLLBACK) dentro de un bloque con EXCEPTION que sí nos va a dar problemas al ejecutarlo.  

Para cumplir con la Robustez de Tipos y la Gestión de Transacciones que pide 
el proyecto, debemos ajustar dos cosas:  
1. El conflicto del COMMIT y EXCEPTION

En PostgreSQL, no se puede usar COMMIT o ROLLBACK dentro de un bloque BEGIN ... EXCEPTION de un procedimiento si este se invoca dentro de otra 
transacción atómica. Además, el requisito B.16 pide el uso de un SAVEPOINT
El código corregido (arreglando ese pequeñísimo bug)
SQL
```
CREATE OR REPLACE PROCEDURE agregar_item_seguro(
    p_jugador_username jugadores.username%TYPE,
    p_tipo_inv inventarios.tipo_inventario%TYPE,
    p_item_id slots_item.item_id%TYPE,
    p_cantidad slots_item.cantidad%TYPE,
    p_slot slots_item.slot_n%TYPE
)
AS $$
DECLARE
    v_jugador_uuid jugadores.uuid%TYPE;
    v_inventario_id inventarios.id%TYPE;
    v_state TEXT;
    v_msg TEXT;
BEGIN
    -- 1. Buscar el UUID del jugador
    SELECT uuid INTO v_jugador_uuid FROM jugadores WHERE username = p_jugador_username;
    
    -- 2. Buscar el inventario
    SELECT id INTO v_inventario_id FROM inventarios 
    WHERE jugador_uuid = v_jugador_uuid AND tipo_inventario = p_tipo_inv;

    -- 3. Lógica con SAVEPOINT para manejo de errores parciales (Requisito B.16)
    BEGIN
        INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad)
        VALUES (v_inventario_id, p_slot, p_item_id, p_cantidad);
    EXCEPTION
        WHEN OTHERS THEN
            -- Captura forense de datos (Requisito C.20)
            GET STACKED DIAGNOSTICS 
                v_state = RETURNED_SQLSTATE,
                v_msg = MESSAGE_TEXT;
            
            INSERT INTO audit_logs (estado_sql, mensaje_error, contexto)
            VALUES (v_state, v_msg, 'Error al insertar item en slot ' || p_slot);
            
            RAISE NOTICE 'Fallo parcial detectado. Error guardado en audit_logs.';
    END;

    -- El COMMIT final asegura la Atomicidad (Requisito B.15)
    COMMIT; 
END;
$$ LANGUAGE plpgsql;
```
minecraft_inventory$# $$;
por ese comando, nos quedó el $$ abierto, y lo cerramos con un $$;, lo cuál nos dió error, que arreglamos con
SQL
```
CREATE OR REPLACE PROCEDURE agregar_item_seguro(
    p_jugador_username VARCHAR,
    p_tipo_inv VARCHAR,
    p_item_id VARCHAR,
    p_cantidad INT,
    p_slot INT
)
AS $$
DECLARE
    v_jugador_uuid UUID;
    v_inventario_id INT;
    v_state TEXT;
    v_msg TEXT;
BEGIN
    -- Abstracción y Lógica Procedural
    SELECT uuid INTO v_jugador_uuid FROM jugadores WHERE username = p_jugador_username;
    
    SELECT id INTO v_inventario_id FROM inventarios 
    WHERE jugador_uuid = v_jugador_uuid AND tipo_inventario = p_tipo_inv;

    BEGIN
        -- Gestión de Transacciones y Robustez
        INSERT INTO slots_items (inventario_id, slot_n, item_id, cantidad)
        VALUES (v_inventario_id, p_slot, p_item_id, p_cantidad);
    EXCEPTION
        WHEN OTHERS THEN
            -- Capa de Auditoría y Forense de Datos
            GET STACKED DIAGNOSTICS 
                v_state = RETURNED_SQLSTATE,
                v_msg = MESSAGE_TEXT;
            
            INSERT INTO audit_logs (estado_sql, mensaje_error, contexto)
            VALUES (v_state, v_msg, 'Error en slot: ' || p_slot);
            
            RAISE NOTICE 'Error capturado en audit_logs';
    END;
    COMMIT; -- Atomicidad
END;
$$ LANGUAGE plpgsql;
CREATE PROCEDURE
minecraft_inventory=# 

```
¡Al fin funcionó!, procedimiento almacenado, creado  con éxito

### LLAMAMOS A LA FUNCIÓN.
SQL
```
CALL agregar_item_seguro('Uziel_gamma', 'PRINCIPAL', 'minecraft:diamond', 64, 1);
minecraft_inventory=# CALL agregar_item_seguro('Uziel_gamma', 'PRINCIPAL', 'minecraft:diamond', 64, 1);
CALL
minecraft_inventory=# 
```
# Creación de nuevas funciones
Contador de items por usuarios:

SQL
```
CREATE OR REPLACE FUNCTION total_items_jugador(p_username VARCHAR)
RETURNS BIGINT AS $$
DECLARE
    v_total BIGINT;
BEGIN
    SELECT SUM(s.cantidad) INTO v_total
    FROM slots_items s
    JOIN inventarios i ON s.inventario_id = i.id
    JOIN jugadores j ON i.jugador_uuid = j.uuid
    WHERE j.username = p_username;

    RETURN COALESCE(v_total, 0);
END;
$$ LANGUAGE plpgsql STABLE;
CREATE FUNCTION
minecraft_inventory=#
```
Automatizador de Triggers
SQL
```
-- Función que ejecutará el trigger
CREATE OR REPLACE FUNCTION actualizar_fecha_inventario()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE inventarios 
    SET ultima_actualizacion = CURRENT_TIMESTAMP 
    WHERE id = NEW.inventario_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Creación del Trigger
CREATE TRIGGER trg_sync_fecha_inventario
AFTER INSERT OR UPDATE ON slots_items
FOR EACH ROW
EXECUTE FUNCTION actualizar_fecha_inventario();
CREATE FUNCTION
CREATE TRIGGER
minecraft_inventory=#
```
Blindaje Final:
SQL
```

minecraft_inventory=# -- Blindaje del procedimiento que ya creaste
ALTER PROCEDURE agregar_item_seguro(VARCHAR, VARCHAR, VARCHAR, INT, INT) 
SECURITY DEFINER 
SET search_path = public, pg_temp;

-- Blindaje de la nueva función
ALTER FUNCTION total_items_jugador(VARCHAR) 
SECURITY DEFINER 
SET search_path = public, pg_temp;
ALTER PROCEDURE
ALTER FUNCTION
minecraft_inventory=# 
```
Perfecto, ahora es más segura, hemos modificado la función para agregar ítems
Trabajo en grupo de Uziel Gamarra y Parejas Lucas
