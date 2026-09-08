# Entrega 2. Operaciones SQL principales mediante SQLJ

## 1. SELECT de una fila

```sql
SELECT nombre, ciudad
FROM cliente
WHERE id_cliente = 1;
```

```java
int idCliente = 1;
String nombre = null;
String ciudad = null;

#sql [contexto] {
    SELECT nombre, ciudad
    INTO :nombre, :ciudad
    FROM cliente
    WHERE id_cliente = :idCliente
};
```

Resultado esperado: `nombre = "Ana"`, `ciudad = "Bilbao"`.

Errores habituales: cero filas, varias filas, orden incorrecto de salidas y tipos incompatibles.

## 2. SELECT de varias filas

```java
#sql iterator ClienteIterator(
    int idCliente,
    String nombre,
    String email,
    String ciudad
);
```

```java
ClienteIterator clientes = null;
try {
    #sql [contexto] clientes = {
        SELECT id_cliente AS idCliente,
               nombre AS nombre,
               email AS email,
               ciudad AS ciudad
        FROM cliente
        WHERE ciudad = :ciudadBuscada
        ORDER BY nombre
    };

    while (clientes.next()) {
        System.out.println(clientes.nombre());
    }
} finally {
    if (clientes != null) clientes.close();
}
```

## 3. INSERT

```sql
INSERT INTO cliente (id_cliente, nombre, email, ciudad)
VALUES (5, 'Elena', 'elena@example.com', 'Vitoria');
```

```java
#sql [contexto] {
    INSERT INTO cliente (id_cliente, nombre, email, ciudad)
    VALUES (:idCliente, :nombre, :email, :ciudad)
};
```

Recomendación: indicar siempre las columnas y no ejecutar `COMMIT` dentro de un método reutilizable si la inserción puede formar parte de una operación mayor.

## 4. UPDATE

```java
#sql [contexto] {
    UPDATE pedido
    SET estado = :nuevoEstado
    WHERE id_pedido = :idPedido
      AND estado = :estadoAnterior
};
```

La condición sobre el estado anterior reduce el riesgo de sobrescribir un cambio concurrente.

## 5. DELETE

```java
#sql [contexto] {
    DELETE FROM linea_pedido
    WHERE id_pedido = :idPedido
};

#sql [contexto] {
    DELETE FROM pedido
    WHERE id_pedido = :idPedido
};
```

Ambas sentencias deben formar parte de la misma transacción. No debe asumirse la existencia de `ON DELETE CASCADE`.

## 6. INNER JOIN

```text
CLIENTE                         PEDIDO
┌────────────┬─────────┐       ┌───────────┬────────────┐
│ id_cliente │ nombre  │       │ id_pedido │ id_cliente │
├────────────┼─────────┤       ├───────────┼────────────┤
│ 1          │ Ana     │◄─────►│ 101       │ 1          │
│ 2          │ Luis    │◄─────►│ 104       │ 2          │
│ 3          │ Marta   │◄─────►│ 103       │ 3          │
│ 4          │ Pablo   │       │           │            │
└────────────┴─────────┘       └───────────┴────────────┘
```

```sql
SELECT c.id_cliente, c.nombre, p.id_pedido, p.fecha, p.importe, p.estado
FROM cliente c
INNER JOIN pedido p ON p.id_cliente = c.id_cliente;
```

Solo aparecen clientes con pedidos.

## 7. LEFT JOIN

```sql
SELECT c.id_cliente, c.nombre, p.id_pedido, p.fecha, p.importe, p.estado
FROM cliente c
LEFT JOIN pedido p ON p.id_cliente = c.id_cliente;
```

Los clientes sin pedido aparecen con columnas de `PEDIDO` a `NULL`. En el iterador, `idPedido` debe ser `Integer`, no `int`.

### ON frente a WHERE

```sql
LEFT JOIN pedido p
  ON p.id_cliente = c.id_cliente
 AND p.estado = 'PENDIENTE'
```

Conserva todos los clientes. Si el filtro se coloca en `WHERE`, los clientes sin pedidos pendientes desaparecen.

## 8. RIGHT JOIN

```sql
SELECT c.nombre, p.id_pedido, p.estado
FROM cliente c
RIGHT JOIN pedido p ON p.id_cliente = c.id_cliente;
```

El soporte debe comprobarse en el dialecto. Para mejorar legibilidad y portabilidad puede intercambiarse el orden de las tablas y utilizar `LEFT JOIN`.

## 9. JOIN de cuatro tablas

```sql
SELECT p.id_pedido,
       c.nombre AS cliente,
       pr.nombre AS producto,
       lp.cantidad,
       pr.precio,
       lp.cantidad * pr.precio AS subtotal
FROM cliente c
INNER JOIN pedido p ON p.id_cliente = c.id_cliente
INNER JOIN linea_pedido lp ON lp.id_pedido = p.id_pedido
INNER JOIN producto pr ON pr.id_producto = lp.id_producto;
```

```text
CLIENTE → PEDIDO → LINEA_PEDIDO → PRODUCTO
```

## 10. UNION y UNION ALL

```text
RESULTADO A        RESULTADO B
Bilbao             Madrid
Madrid             Sevilla

UNION     → Bilbao, Madrid, Sevilla
UNION ALL → Bilbao, Madrid, Madrid, Sevilla
```

```sql
SELECT ciudad FROM cliente WHERE id_cliente <= 2
UNION
SELECT ciudad FROM cliente WHERE id_cliente >= 2;
```

Requisitos:

- Mismo número de columnas.
- Tipos compatibles por posición.
- `UNION` elimina duplicados.
- `UNION ALL` conserva duplicados.

## 11. Subconsultas

```sql
SELECT id_pedido, fecha, importe, estado
FROM pedido
WHERE importe > (
    SELECT AVG(importe)
    FROM pedido
);
```

La subconsulta escalar debe devolver una sola fila.

## 12. EXISTS y NOT EXISTS

```sql
SELECT c.id_cliente, c.nombre
FROM cliente c
WHERE EXISTS (
    SELECT 1
    FROM pedido p
    WHERE p.id_cliente = c.id_cliente
);
```

```sql
SELECT c.id_cliente, c.nombre
FROM cliente c
WHERE NOT EXISTS (
    SELECT 1
    FROM pedido p
    WHERE p.id_cliente = c.id_cliente
);
```

`NOT EXISTS` suele ser preferible a `NOT IN` cuando la subconsulta puede producir `NULL`.

## 13. IN y NOT IN

```java
String ciudad1 = "Bilbao";
String ciudad2 = "Madrid";

#sql [contexto] clientes = {
    SELECT id_cliente AS idCliente,
           nombre AS nombre,
           email AS email,
           ciudad AS ciudad
    FROM cliente
    WHERE ciudad IN (:ciudad1, :ciudad2)
};
```

Una variable host no sustituye una lista completa de longitud variable.

## 14. GROUP BY y HAVING

```sql
SELECT c.id_cliente,
       c.nombre,
       COUNT(p.id_pedido) AS numero_pedidos,
       SUM(p.importe) AS importe_total,
       AVG(p.importe) AS importe_medio
FROM cliente c
INNER JOIN pedido p ON p.id_cliente = c.id_cliente
GROUP BY c.id_cliente, c.nombre
HAVING COUNT(p.id_pedido) >= 2;
```

```text
WHERE  → filtra filas antes de agrupar
HAVING → filtra grupos después de agrupar
```

## 15. Agregaciones

```java
long numeroPedidos = 0;
BigDecimal total = null;
BigDecimal media = null;
BigDecimal minimo = null;
BigDecimal maximo = null;

#sql [contexto] {
    SELECT COUNT(*), SUM(importe), AVG(importe), MIN(importe), MAX(importe)
    INTO :numeroPedidos, :total, :media, :minimo, :maximo
    FROM pedido
};
```

`COUNT(*)` cuenta filas. `COUNT(importe)` excluye valores nulos. `SUM`, `AVG`, `MIN` y `MAX` pueden devolver `NULL` si no hay valores.

## 16. Valores NULL

```sql
SELECT id_pedido
FROM pedido
WHERE importe IS NULL;
```

```sql
SELECT COALESCE(SUM(importe), 0)
FROM pedido;
```

No debe equipararse automáticamente `NULL` con cero. La decisión depende del significado funcional.

## 17. ORDER BY

```sql
SELECT id_pedido, fecha, importe, estado
FROM pedido
ORDER BY importe DESC, id_pedido ASC;
```

No se puede parametrizar `ASC` o `DESC` como si fuera un valor. Para pocas variantes, deben utilizarse ramas SQLJ estáticas.

## 18. COMMIT, ROLLBACK y transacciones

```text
Inicio
  │
  ▼
INSERT PEDIDO
  │
  ▼
INSERT LINEAS
  │
  ▼
¿Todo correcto?
├── Sí → COMMIT
└── No → ROLLBACK
```

```java
try {
    connection.setAutoCommit(false);
    // Operaciones SQLJ.
    connection.commit();
} catch (SQLException e) {
    try {
        connection.rollback();
    } catch (SQLException rollbackError) {
        e.addSuppressed(rollbackError);
    }
    throw e;
}
```

## 19. Prevención de modificaciones masivas

```java
ExecutionContext ejecucion = new ExecutionContext();

#sql [contexto, ejecucion] {
    UPDATE pedido
    SET estado = :nuevoEstado
    WHERE id_pedido = :idPedido
};

if (ejecucion.getUpdateCount() != 1) {
    throw new SQLException("Número inesperado de filas afectadas");
}
```

## 20. Lista de comprobación

- [ ] SELECT de una y varias filas.
- [ ] INSERT, UPDATE y DELETE seguros.
- [ ] JOIN internos y externos.
- [ ] UNION y UNION ALL.
- [ ] Subconsultas y EXISTS.
- [ ] GROUP BY y HAVING.
- [ ] Agregaciones y NULL.
- [ ] ORDER BY.
- [ ] COMMIT y ROLLBACK.
- [ ] Verificación de filas afectadas.
