# Entrega 4. SQLJ frente a JDBC, buenas prácticas y ejercicio final

## 1. Comparativa

| Criterio | SQLJ | JDBC |
|---|---|---|
| Tipo de SQL | Principalmente estático | Estático o dinámico |
| Representación | Cláusulas `#sql` | Cadenas de texto |
| Preparación | Traductor SQLJ | API JDBC en ejecución |
| Parámetros | Variables host `:` | Marcadores `?` y `setXxx()` |
| Una fila | `SELECT INTO` | `ResultSet` |
| Varias filas | Iteradores | `ResultSet` |
| Validación anticipada | Posible durante traducción | Generalmente en preparación/ejecución |
| SQL dinámico | Limitado | Adecuado |
| Código ceremonial | Menor | Mayor |
| Dependencias | Traductor, runtime y driver | Driver JDBC |
| Caso típico | SQL estable en plataforma consolidada | Acceso general o SQL dinámico |

## 2. Misma consulta con SQLJ

```java
#sql iterator PedidoResumenIterator(
    int idPedido,
    java.sql.Date fecha,
    java.math.BigDecimal importe,
    String estado
);
```

```java
public List<PedidoResumen> buscarPedidosSQLJ(
        DefaultContext contexto,
        int idCliente,
        BigDecimal importeMinimo) throws SQLException {

    List<PedidoResumen> resultado = new ArrayList<PedidoResumen>();
    PedidoResumenIterator iterador = null;

    try {
        #sql [contexto] iterador = {
            SELECT id_pedido AS idPedido,
                   fecha AS fecha,
                   importe AS importe,
                   estado AS estado
            FROM pedido
            WHERE id_cliente = :idCliente
              AND importe >= :importeMinimo
            ORDER BY fecha, id_pedido
        };

        while (iterador.next()) {
            resultado.add(new PedidoResumen(
                iterador.idPedido(), iterador.fecha(),
                iterador.importe(), iterador.estado()));
        }
        return resultado;
    } finally {
        if (iterador != null) iterador.close();
    }
}
```

## 3. Misma consulta con JDBC

```java
public List<PedidoResumen> buscarPedidosJDBC(
        Connection connection,
        int idCliente,
        BigDecimal importeMinimo) throws SQLException {

    String sql =
        "SELECT id_pedido, fecha, importe, estado " +
        "FROM pedido " +
        "WHERE id_cliente = ? AND importe >= ? " +
        "ORDER BY fecha, id_pedido";

    List<PedidoResumen> resultado = new ArrayList<PedidoResumen>();

    try (PreparedStatement statement = connection.prepareStatement(sql)) {
        statement.setInt(1, idCliente);
        statement.setBigDecimal(2, importeMinimo);

        try (ResultSet rs = statement.executeQuery()) {
            while (rs.next()) {
                resultado.add(new PedidoResumen(
                    rs.getInt("id_pedido"),
                    rs.getDate("fecha"),
                    rs.getBigDecimal("importe"),
                    rs.getString("estado")));
            }
        }
    }
    return resultado;
}
```

## 4. Criterio de elección

### SQLJ

- Aplicación existente en SQLJ.
- Consultas estables.
- Traductor y runtime soportados.
- Equipo con experiencia.
- Valor real de la validación anticipada.

### JDBC

- Aplicación nueva sin requisito SQLJ.
- SQL dinámico.
- Filtros, proyecciones u ordenaciones variables.
- Necesidad de metadatos o control fino.
- Integración sencilla con herramientas Java actuales.

Una aplicación puede conservar SQLJ para operaciones estáticas y utilizar JDBC en módulos dinámicos.

## 5. Buenas prácticas

### Parámetros

Usar variables host o marcadores preparados. No concatenar datos del usuario.

### Tipos

- Importes: `BigDecimal`.
- Fechas: `java.sql.Date` o `java.sql.Timestamp` según la columna.
- Columnas anulables: tipos de referencia.

### Consultas

- Evitar `SELECT *`.
- Seleccionar solo las columnas necesarias.
- Aplicar filtros selectivos.
- Usar `ORDER BY` solo cuando el orden sea relevante.
- Revisar índices en filtros y relaciones.

### JOIN

- Usar alias claros.
- Diferenciar `ON` y `WHERE`.
- Tratar `NULL` en uniones externas.
- Verificar cardinalidad para evitar duplicados inesperados.

### UNION

- `UNION` cuando deban eliminarse duplicados.
- `UNION ALL` cuando deban conservarse o los conjuntos sean disjuntos.

### Modificaciones

- Comprobar el número de filas afectadas.
- No ejecutar `UPDATE` o `DELETE` sin filtro salvo decisión explícita.
- Mantener restricciones de integridad en la base de datos.

### Transacciones

```text
Unidad de negocio completa
├── todas las operaciones correctas → COMMIT
└── cualquier error → ROLLBACK
```

- Evitar commits parciales.
- Mantener transacciones breves.
- No devolver una conexión incierta al pool.
- Restaurar `autoCommit` y otras propiedades modificadas.

### Errores

- Analizar `SQLState` y código del fabricante.
- No depender del texto del mensaje.
- Conservar errores de rollback/cierre mediante `addSuppressed()`.
- No registrar credenciales ni datos personales completos.

## 6. Ejercicio integral

### Enunciado

Implementar el alta de un pedido con varias líneas:

1. Validar cliente y productos.
2. Impedir productos duplicados.
3. Recuperar precios.
4. Insertar cabecera.
5. Insertar líneas.
6. Calcular y actualizar el total.
7. Confirmar todo con `COMMIT`.
8. Deshacer todo con `ROLLBACK` ante cualquier fallo.
9. Consultar después el pedido con cliente y productos.

### Flujo

```text
Validar entrada
      │
      ▼
Comprobar cliente
      │
      ▼
INSERT PEDIDO
      │
      ▼
Por cada línea
├── comprobar producto
├── recuperar precio
├── calcular subtotal
└── INSERT LINEA
      │
      ▼
UPDATE total
      │
      ▼
COMMIT / ROLLBACK
```

### Solución principal

```java
public BigDecimal crearPedido(DatosPedido datos) throws SQLException {
    validar(datos);
    boolean autoCommitOriginal = connection.getAutoCommit();
    SQLException errorPrincipal = null;

    try {
        connection.setAutoCommit(false);
        comprobarCliente(datos.getIdCliente());

        int idPedido = datos.getIdPedido();
        int idCliente = datos.getIdCliente();
        java.sql.Date fecha = datos.getFecha();
        String estado = datos.getEstado();
        BigDecimal total = BigDecimal.ZERO;

        #sql [contexto] {
            INSERT INTO pedido
                (id_pedido, id_cliente, fecha, importe, estado)
            VALUES
                (:idPedido, :idCliente, :fecha, :total, :estado)
        };

        for (Linea linea : datos.getLineas()) {
            int idProducto = linea.getIdProducto();
            int cantidad = linea.getCantidad();
            BigDecimal precio = buscarPrecio(idProducto);

            total = total.add(
                precio.multiply(BigDecimal.valueOf(cantidad)));

            #sql [contexto] {
                INSERT INTO linea_pedido
                    (id_pedido, id_producto, cantidad)
                VALUES
                    (:idPedido, :idProducto, :cantidad)
            };
        }

        #sql [contexto] {
            UPDATE pedido
            SET importe = :total
            WHERE id_pedido = :idPedido
        };

        connection.commit();
        return total;

    } catch (SQLException e) {
        errorPrincipal = e;
        try {
            connection.rollback();
        } catch (SQLException rollbackError) {
            e.addSuppressed(rollbackError);
        }
        throw e;
    } finally {
        try {
            connection.setAutoCommit(autoCommitOriginal);
        } catch (SQLException restoreError) {
            if (errorPrincipal != null) {
                errorPrincipal.addSuppressed(restoreError);
            } else {
                throw restoreError;
            }
        }
    }
}
```

### Consulta posterior

```java
#sql iterator DetallePedidoIterator(
    int idPedido,
    String cliente,
    java.sql.Date fecha,
    String estado,
    BigDecimal importeTotal,
    int idProducto,
    String producto,
    BigDecimal precio,
    int cantidad,
    BigDecimal subtotal
);
```

```java
#sql [contexto] iterador = {
    SELECT p.id_pedido AS idPedido,
           c.nombre AS cliente,
           p.fecha AS fecha,
           p.estado AS estado,
           p.importe AS importeTotal,
           pr.id_producto AS idProducto,
           pr.nombre AS producto,
           pr.precio AS precio,
           lp.cantidad AS cantidad,
           lp.cantidad * pr.precio AS subtotal
    FROM pedido p
    INNER JOIN cliente c ON c.id_cliente = p.id_cliente
    INNER JOIN linea_pedido lp ON lp.id_pedido = p.id_pedido
    INNER JOIN producto pr ON pr.id_producto = lp.id_producto
    WHERE p.id_pedido = :idPedido
    ORDER BY pr.nombre
};
```

### Errores frecuentes

- `COMMIT` dentro del bucle.
- Importes calculados con `double`.
- Cliente o producto inexistente.
- Producto duplicado.
- Cantidad cero o negativa.
- Error de clave primaria o ajena.
- No comprobar filas afectadas.
- Ocultar el error de rollback.
- Reutilizar una conexión de estado incierto.

### Variantes

- Descuentos por categoría.
- Impuestos.
- Comprobación y reserva de stock.
- Histórico de estados.
- Precio aplicado almacenado en la línea.
- Control optimista.
- Cancelación transaccional.
- Reintentos idempotentes.
- Versión equivalente con JDBC.

## 7. Mapa conceptual

```text
SQLJ
├── Consultas
│   ├── SELECT INTO
│   ├── Iteradores
│   ├── JOIN
│   ├── Subconsultas
│   └── UNION
├── Modificación
│   ├── INSERT
│   ├── UPDATE
│   └── DELETE
├── Procesamiento
│   ├── GROUP BY
│   ├── HAVING
│   └── Agregaciones
├── Transacciones
│   ├── COMMIT
│   └── ROLLBACK
└── Integración Java
    ├── Variables host
    ├── Contextos
    ├── ExecutionContext
    ├── Iteradores
    └── SQLException
```

## 8. Comprobación final

- [ ] Explicar SQLJ y SQL estático.
- [ ] Configurar traducción, runtime y contexto.
- [ ] Usar variables host.
- [ ] Recuperar una y varias filas.
- [ ] Ejecutar todas las operaciones DML principales.
- [ ] Construir JOIN, UNION y subconsultas.
- [ ] Agrupar y agregar datos.
- [ ] Tratar `NULL`.
- [ ] Delimitar transacciones.
- [ ] Gestionar errores y recursos.
- [ ] Comparar objetivamente SQLJ y JDBC.
- [ ] Identificar dependencias de Oracle, Db2 u otro fabricante.
