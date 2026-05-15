# El error de los ID fantasma
Intentamos agregar los 1M o aunque sea 100K, pero sólo nos tomaba re pocos,

```
minecraft_inventory=# INSERT INTO inventarios (jugador_uuid, tipo_inventario)
SELECT uuid, 'COFRE_VIRTUAL_' || i 
FROM jugadores CROSS JOIN generate_series(1, 10000) AS i;
INSERT 0 30000
--aquí sólo generó 30000, de 10000, lo cuál era muy extraño
minecraft_inventory=# INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad, shulker_contenido, durabilidad_registro)
SELECT 
    ((g - 1) / 36) + 1, 
    ((g - 1) % 36),
    CASE WHEN g % 10 = 0 THEN 'minecraft:blue_shulker_box' ELSE 'minecraft:cobblestone' END,
    1,
    CASE WHEN g % 10 = 0 THEN jsonb_build_array(jsonb_build_object('id', 'minecraft:netherite_ingot')) ELSE NULL END,
    NULL
FROM generate_series(1, 100000) AS g;
ERROR:  insert or update on table "slots_item" violates foreign key constraint "slots_item_inventario_id_fkey"
DETALLE:  Key (inventario_id)=(1) is not present in table "inventarios".
minecraft_inventory=# INSERT INTO inventarios (jugador_uuid, tipo_inventario)
SELECT uuid, 'COFRE_VIRTUAL_' || i 
FROM jugadores CROSS JOIN generate_series(1, 10000) AS i;
INSERT 0 30000
minecraft_inventory=# INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad, shulker_contenido)
SELECT 
    ((g - 1) / 36) + 1, 
    ((g - 1) % 36),
    CASE WHEN g % 10 = 0 THEN 'minecraft:blue_shulker_box' ELSE 'minecraft:cobblestone' END,
    1,
    CASE WHEN g % 10 = 0 THEN jsonb_build_array(jsonb_build_object('id', 'minecraft:netherite_ingot')) ELSE NULL END
FROM generate_series(1, 30000) AS g;
ERROR:  insert or update on table "slots_item" violates foreign key constraint "slots_item_inventario_id_fkey"
DETALLE:  Key (inventario_id)=(1) is not present in table "inventarios".
-- aquí tuvimos el mismo problema con que los slots_item, no se conectaban a los IDs y ejecutamos estos dos comandos
```

### Los comandos que nos salvaron

SQL
```
minecraft_inventory=# -- 1. Encuentra el nombre de la secuencia (normalmente 'nombre_tabla_nombre_columna_seq')
SELECT pg_get_serial_sequence('inventarios', 'id');
  pg_get_serial_sequence   
---------------------------
 public.inventarios_id_seq
(1 row)

minecraft_inventory=# -- 2. Reinicia la secuencia al valor máximo + 1
SELECT setval(                                     
    pg_get_serial_sequence('inventarios', 'id'),
    COALESCE(MAX(id), 1)
) FROM inventarios;
 setval 
--------
 120006 --aquí vimos el problema, los IDs estaban muy lejos, y SERIAL no se reiniciaba a 1
(1 row)
```

Empezamos la limpieza

SQL
```
minecraft_inventory=# -- 1. Limpiamos cualquier intento fallido en slots
TRUNCATE TABLE slots_item CASCADE;

-- 2. Insertamos items usando los IDs que REALMENTE existen (los que terminan en 120006)
INSERT INTO slots_item (inventario_id, slot_n, item_id, cantidad, shulker_contenido)
SELECT 
    i.id, 
    0, -- Todos al slot 0 para simplificar
    'minecraft:blue_shulker_box', 
    1,
    jsonb_build_array(jsonb_build_object('id', 'minecraft:netherite_ingot', 'count', 1))
FROM (SELECT id FROM inventarios LIMIT 30000) AS i;
TRUNCATE TABLE
INSERT 0 30000
```

El momento de la verdad

SQL
```
SELECT count(*) FROM slots_item;
minecraft_inventory=# SELECT count(*) FROM slots_item;
 count 
-------
 30000
(1 row)

minecraft_inventory=#
```
SIIII
