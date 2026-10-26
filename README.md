# Práctica 4 · Alquiler de DVD

**PostgreSQL · Consultas SQL, vistas, restricciones y disparadores**  
**Autor:** Diego Vegas Acosta  
**Universidad:** Universidad de La Laguna  
**Herramientas:** PostgreSQL, pgAdmin y PL/pgSQL

---

## Contenido

1. [Restauración de la base de datos](#1-restauración-de-la-base-de-datos)
2. [Identificación de objetos](#2-identificación-de-objetos)
3. [Tablas principales y relaciones](#3-tablas-principales-y-relaciones)
4. [Consultas SQL](#4-consultas-sql)
5. [Creación de vistas](#5-creación-de-vistas)
6. [Restricciones CHECK](#6-restricciones-check)
7. [Disparador `last_updated`](#7-disparador-last_updated)
8. [Registro de inserciones](#8-registro-de-inserciones)
9. [Registro de eliminaciones](#9-registro-de-eliminaciones)
10. [Secuencias](#10-secuencias)

## Organización del repositorio

```text
Blockbuster_README/
├── README.md
├── imagenes/
│   ├── fig_01.png ... fig_14.png
└── sql/
    ├── 01_consultas.sql
    ├── 02_vistas.sql
    ├── 03_restricciones.sql
    ├── 04_triggers.sql
    └── 05_inspeccion.sql
```

> **Nota sobre la entrega:** es necesario añadir al repositorio una exportación **SQL comprimida** de la base de datos final. El material original no contiene una captura de la restauración ni de la consulta del apartado 7. Las imágenes proceden de las capturas del trabajo y su legibilidad está limitada por la resolución original.

## 1. Restauración de la base de datos

La práctica parte de la copia de seguridad `alquilerdvd.tar`. La restauración se realiza desde pgAdmin, creando una base de datos y utilizando **Restore** con formato **Tar**. Después se comprueba que las tablas estén disponibles en el esquema `public`.

No se dispone de una captura del proceso de restauración en el documento original.

## 2. Identificación de objetos

Para conocer la estructura de la base de datos se consultaron las tablas, vistas y secuencias del esquema `public` mediante `information_schema`.

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE';

SELECT table_name
FROM information_schema.views
WHERE table_schema = 'public';

SELECT sequence_name
FROM information_schema.sequences
WHERE sequence_schema = 'public';
```

En las capturas se observan tablas como `actor`, `store`, `address`, `category`, `city`, `country`, `customer` y `film_actor`. La consulta inicial de vistas no devolvió filas y la de secuencias mostró, entre otras, `customer_customer_id_seq` y `actor_actor_id_seq`.

![Listado de tablas](imagenes/fig_01.png)
*Figura 1. Identificación de tablas.*

![Listado inicial de vistas](imagenes/fig_02.png)
*Figura 2. Consulta de vistas del esquema public.*

![Listado de secuencias](imagenes/fig_03.png)
*Figura 3. Secuencias asociadas a las tablas.*

## 3. Tablas principales y relaciones

| Tabla | Función |
|---|---|
| `film` | Datos de las películas: título, descripción, duración, precio y clasificación. |
| `actor`, `film_actor` | Actores y relación entre actores y películas. |
| `category`, `film_category` | Categorías y su asociación con películas. |
| `inventory` | Copias de películas disponibles por tienda. |
| `rental` | Operaciones de alquiler y devolución. |
| `payment` | Pagos realizados. |
| `customer` | Clientes registrados. |
| `staff`, `store` | Empleados y establecimientos. |
| `address`, `city`, `country` | Información geográfica. |
| `language` | Idiomas asociados a las películas. |

Las claves primarias identifican cada registro y las claves foráneas conectan las tablas. Las tablas intermedias `film_actor` y `film_category` permiten representar relaciones entre películas, actores y categorías.

## 4. Consultas SQL

Se realizaron cuatro consultas utilizando `JOIN`, `SUM`, `GROUP BY`, `ORDER BY` y `STRING_AGG`. El código completo está en [`sql/01_consultas.sql`](sql/01_consultas.sql).

### 4.1. Ventas totales por categoría

Se suman los importes de `payment` a través de las relaciones entre películas, inventario y alquileres, agrupando por categoría y ordenando descendentemente.

```sql
SELECT c.name AS categoria, SUM(p.amount) AS ventas_totales
FROM category c
JOIN film_category fc ON c.category_id = fc.category_id
JOIN inventory i ON fc.film_id = i.film_id
JOIN rental r ON i.inventory_id = r.inventory_id
JOIN payment p ON r.rental_id = p.rental_id
GROUP BY c.category_id, c.name
ORDER BY ventas_totales DESC;
```

En la captura, **Sports** aparece en primer lugar con **4892.19**, seguida de **Sci-Fi** (4336.01) y **Animation** (4245.31).

![Ventas por categoría](imagenes/fig_04.png)
*Figura 4. Consulta de ventas por categoría y resultado obtenido.*

### 4.2. Ventas totales por tienda

Se relacionan tiendas, direcciones, ciudades, países, personal y pagos. La ubicación se presenta concatenando ciudad y país, junto con el encargado y las ventas totales.

```sql
SELECT s.store_id,
       ci.city || ', ' || co.country AS ubicacion,
       m.first_name || ' ' || m.last_name AS encargado,
       SUM(p.amount) AS ventas_totales
FROM store s
JOIN address ad ON s.address_id = ad.address_id
JOIN city ci ON ad.city_id = ci.city_id
JOIN country co ON ci.country_id = co.country_id
JOIN staff m ON s.manager_staff_id = m.staff_id
JOIN staff e ON e.store_id = s.store_id
JOIN payment p ON p.staff_id = e.staff_id
GROUP BY s.store_id, ci.city, co.country,
         m.first_name, m.last_name
ORDER BY ventas_totales DESC;
```

La evidencia recoge dos tiendas, con ventas totales de **31059.92** y **30252.12**.

![Ventas por tienda y listado de películas](imagenes/fig_05.png)
*Figura 5. Resultados de ventas por tienda y consulta de películas.*

### 4.3. Películas con sus actores

La consulta muestra identificador, título, descripción, categoría, precio de alquiler, duración, clasificación y nombres de los actores concatenados mediante `STRING_AGG`.

### 4.4. Actores con categorías y películas

La consulta agrupa los registros por actor y concatena las categorías y títulos de las películas utilizando `:` como separador.

![Consulta de actores](imagenes/fig_06.png)
*Figura 6. Resultado de la consulta de actores y películas.*

## 5. Creación de vistas

Las cuatro consultas anteriores se guardaron como vistas con prefijo `view_`, para reutilizar sus resultados sin escribir nuevamente las uniones y agrupaciones. El código completo está en [`sql/02_vistas.sql`](sql/02_vistas.sql).

| Vista | Contenido |
|---|---|
| `view_ventas_categoria` | Ventas agregadas por categoría. |
| `view_ventas_tienda` | Ventas por establecimiento y encargado. |
| `view_peliculas_actores` | Películas con información de actores. |
| `view_ventas_peliculas` | Actores, categorías y películas, según el nombre usado en la captura. |

Las siguientes capturas muestran mensajes de ejecución correcta `CREATE VIEW`.

![Creación de vista por categoría](imagenes/fig_07.png)
*Figura 7. Creación de la vista de ventas por categoría.*

![Creación de vista por tienda](imagenes/fig_08.png)
*Figura 8. Creación de la vista de ventas por tienda.*

![Creación de vista de películas y actores](imagenes/fig_09.png)
*Figura 9. Creación de la vista de películas y actores.*

![Creación de vista de actores y categorías](imagenes/fig_10.png)
*Figura 10. Creación de la vista de actores, categorías y películas.*

## 6. Restricciones CHECK

Se añadieron reglas para impedir datos inconsistentes, como duraciones no positivas, importes negativos y fechas de devolución anteriores al alquiler. El código está en [`sql/03_restricciones.sql`](sql/03_restricciones.sql).

```sql
ALTER TABLE film ADD CONSTRAINT check_film_length CHECK (length > 0);
ALTER TABLE film ADD CONSTRAINT check_rental_duration CHECK (rental_duration > 0);
ALTER TABLE film ADD CONSTRAINT check_rental_rate CHECK (rental_rate >= 0);
ALTER TABLE film ADD CONSTRAINT check_replacement_cost CHECK (replacement_cost >= 0);
ALTER TABLE payment ADD CONSTRAINT check_payment_amount CHECK (amount >= 0);
ALTER TABLE rental ADD CONSTRAINT check_return_date CHECK (return_date >= rental_date);
```

Una fecha `return_date` nula sigue siendo válida cuando el alquiler todavía no se ha devuelto. El dominio `year` ya contempla una validación para los años de estreno.

![Aplicación de restricciones](imagenes/fig_11.png)
*Figura 11. Ejecución de las restricciones CHECK.*

## 7. Disparador `last_updated`

La sentencia:

```sql
last_updated BEFORE UPDATE ON customer
FOR EACH ROW EXECUTE PROCEDURE last_updated()
```

Define un disparador que actúa **antes** de actualizar cada fila de `customer`. La función asigna `CURRENT_TIMESTAMP` al campo `NEW.last_update` y devuelve `NEW`, actualizando automáticamente la fecha de modificación. La tabla `actor` utiliza una solución similar.

Para localizar otros disparadores equivalentes:

```sql
SELECT event_object_table AS tabla,
       trigger_name AS disparador,
       action_timing AS momento,
       event_manipulation AS operacion,
       action_statement AS funcion
FROM information_schema.triggers
WHERE trigger_name = 'last_updated'
ORDER BY event_object_table;
```

Esta consulta también está recogida en [`sql/05_inspeccion.sql`](sql/05_inspeccion.sql). El documento original no incluye una captura de su resultado.

## 8. Registro de inserciones

Se creó la tabla `film_insert_log` con los campos `id`, `film_id` y `fecha`. Una función PL/pgSQL registra `NEW.film_id` junto con `CURRENT_TIMESTAMP` y se ejecuta mediante un trigger `AFTER INSERT ON film FOR EACH ROW`.

```sql
CREATE OR REPLACE FUNCTION registrar_film_insert()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO film_insert_log (film_id, fecha)
    VALUES (NEW.film_id, CURRENT_TIMESTAMP);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

La captura muestra un registro correspondiente a la película **1001**, con fecha **26/10/2026**. La definición completa de tabla, función y trigger se encuentra en [`sql/04_triggers.sql`](sql/04_triggers.sql).

![Registro de inserción](imagenes/fig_12.png)
*Figura 12. Registro automático de una película insertada.*

## 9. Registro de eliminaciones

Se creó `film_delete_log` para guardar el identificador de cada película eliminada y la fecha de eliminación. En este caso se utiliza `OLD.film_id`, ya que se necesita el valor de la fila anterior a su borrado.

```sql
CREATE OR REPLACE FUNCTION registrar_film_delete()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO film_delete_log (film_id, fecha)
    VALUES (OLD.film_id, CURRENT_TIMESTAMP);
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

El disparador se ejecuta `AFTER DELETE ON film FOR EACH ROW`. La captura muestra el registro de eliminación de la película **1001**.

![Registro de eliminación](imagenes/fig_13.png)
*Figura 13. Registro automático de la eliminación de una película.*

## 10. Secuencias

Las secuencias generan valores numéricos automáticamente, normalmente para identificadores. Por ejemplo, `film_film_id_seq` permite asignar valores a `film_id` mediante `nextval()`.

```sql
SELECT sequencename, start_value, increment_by, last_value
FROM pg_sequences
WHERE schemaname = 'public'
ORDER BY sequencename;
```

Son relevantes porque evitan calcular manualmente los identificadores y permiten gestionar inserciones concurrentes. **No garantizan una numeración sin saltos**; la unicidad de los registros se garantiza mediante las restricciones `PRIMARY KEY`.

![Secuencias de PostgreSQL](imagenes/fig_14.png)
*Figura 14. Secuencias configuradas en la base de datos.*

---

## Reproducción y entrega

1. Restaurar `alquilerdvd.tar` en PostgreSQL mediante pgAdmin.
2. Ejecutar las consultas de inspección y las consultas SQL.
3. Crear las vistas y las restricciones propuestas, comprobando que no existan previamente.
4. Crear los disparadores y probarlos con una película nueva sin dependencias.
5. Exportar la base de datos final como SQL, comprimirla y añadirla al repositorio.

> Las capturas documentan la ejecución realizada originalmente. No se han vuelto a ejecutar los scripts sobre la instancia de PostgreSQL durante la preparación de este README.
