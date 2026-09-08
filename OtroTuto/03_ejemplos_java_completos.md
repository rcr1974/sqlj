# Entrega 3. Ejemplos Java completos, transacciones y errores

## 1. Estructura sugerida

```text
src/ejemplo/sqlj/
├── Modelo.java
├── Iteradores.sqlj
├── RepositorioSQLJ.sqlj
└── AplicacionSQLJ.java
```

## 2. Iteradores

```java
package ejemplo.sqlj;

import java.math.BigDecimal;
import java.sql.Date;

#sql public iterator ClienteIterator(
    int idCliente,
    String nombre,
    String email,
    String ciudad
);

#sql public iterator ClientePedidoIterator(
    int idCliente,
    String nombreCliente,
    Integer idPedido,
    Date fecha,
    BigDecimal importe,
    String estado
);

#sql public iterator DetalleProductoIterator(
    int idPedido,
    String cliente,
    int idProducto,
    String producto,
    int cantidad,
    BigDecimal precioUnitario,
    BigDecimal subtotal
);

#sql public iterator ResumenClienteIterator(
    int idCliente,
    String nombreCliente,
    long numeroPedidos,
    BigDecimal importeTotal,
    BigDecimal importeMedio
);
```

## 3. Repositorio base

```java
package ejemplo.sqlj;

import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.Date;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

import sqlj.runtime.ExecutionContext;
import sqlj.runtime.ref.DefaultContext;

public final class RepositorioSQLJ {

    private final Connection connection;
    private final DefaultContext contexto;

    public RepositorioSQLJ(Connection connection, DefaultContext contexto) {
        if (connection == null || contexto == null) {
            throw new IllegalArgumentException("Conexión y contexto obligatorios");
        }
        this.connection = connection;
        this.contexto = contexto;
    }
}
```

## 4. Ejemplo 1: consultar cliente

```java
public Cliente buscarCliente(int idCliente) throws SQLException {
    if (idCliente <= 0) {
        throw new IllegalArgumentException("Identificador no válido");
    }

    String nombre = null;
    String email = null;
    String ciudad = null;

    #sql [contexto] {
        SELECT nombre, email, ciudad
        INTO :nombre, :email, :ciudad
        FROM cliente
        WHERE id_cliente = :idCliente
    };

    return new Cliente(idCliente, nombre, email, ciudad);
}
```

## 5. Ejemplo 2: insertar cliente

```java
public void insertarCliente(Cliente cliente) throws SQLException {
    int idCliente = cliente.getIdCliente();
    String nombre = cliente.getNombre();
    String email = cliente.getEmail();
    String ciudad = cliente.getCiudad();

    ExecutionContext ejecucion = new ExecutionContext();

    #sql [contexto, ejecucion] {
        INSERT INTO cliente (id_cliente, nombre, email, ciudad)
        VALUES (:idCliente, :nombre, :email, :ciudad)
    };

    verificarFilas(ejecucion, 1, "Insertar cliente");
}
```

## 6. Ejemplo 3: actualizar estado

```java
public void actualizarEstadoPedido(
        int idPedido,
        String estadoEsperado,
        String nuevoEstado) throws SQLException {

    ExecutionContext ejecucion = new ExecutionContext();

    #sql [contexto, ejecucion] {
        UPDATE pedido
        SET estado = :nuevoEstado
        WHERE id_pedido = :idPedido
          AND estado = :estadoEsperado
    };

    verificarFilas(ejecucion, 1, "Actualizar estado");
}
```

## 7. Ejemplo 4: eliminar pedido

```java
public void eliminarPedido(final int idPedido) throws SQLException {
    ejecutarEnTransaccion(new TrabajoTransaccional() {
        @Override
        public void ejecutar() throws SQLException {
            #sql [contexto] {
                DELETE FROM linea_pedido
                WHERE id_pedido = :idPedido
            };

            ExecutionContext ejecucion = new ExecutionContext();
            #sql [contexto, ejecucion] {
                DELETE FROM pedido
                WHERE id_pedido = :idPedido
            };

            verificarFilas(ejecucion, 1, "Eliminar pedido");
        }
    });
}
```

## 8. Ejemplo 5: INNER JOIN

```java
public List<ClientePedido> buscarClientesConPedidos() throws SQLException {
    List<ClientePedido> resultado = new ArrayList<ClientePedido>();
    ClientePedidoIterator iterador = null;

    try {
        #sql [contexto] iterador = {
            SELECT c.id_cliente AS idCliente,
                   c.nombre AS nombreCliente,
                   p.id_pedido AS idPedido,
                   p.fecha AS fecha,
                   p.importe AS importe,
                   p.estado AS estado
            FROM cliente c
            INNER JOIN pedido p ON p.id_cliente = c.id_cliente
            ORDER BY c.id_cliente, p.id_pedido
        };

        while (iterador.next()) {
            resultado.add(new ClientePedido(
                iterador.idCliente(), iterador.nombreCliente(),
                iterador.idPedido(), iterador.fecha(),
                iterador.importe(), iterador.estado()));
        }
        return resultado;
    } finally {
        if (iterador != null) iterador.close();
    }
}
```

## 9. Ejemplo 6: LEFT JOIN

```java
public List<ClientePedido> buscarTodosConPedidos() throws SQLException {
    List<ClientePedido> resultado = new ArrayList<ClientePedido>();
    ClientePedidoIterator iterador = null;

    try {
        #sql [contexto] iterador = {
            SELECT c.id_cliente AS idCliente,
                   c.nombre AS nombreCliente,
                   p.id_pedido AS idPedido,
                   p.fecha AS fecha,
                   p.importe AS importe,
                   p.estado AS estado
            FROM cliente c
            LEFT JOIN pedido p ON p.id_cliente = c.id_cliente
            ORDER BY c.id_cliente, p.id_pedido
        };

        while (iterador.next()) {
            resultado.add(new ClientePedido(
                iterador.idCliente(), iterador.nombreCliente(),
                iterador.idPedido(), iterador.fecha(),
                iterador.importe(), iterador.estado()));
        }
        return resultado;
    } finally {
        if (iterador != null) iterador.close();
    }
}
```

## 10. Ejemplo 7: cuatro tablas

```java
public List<DetalleProducto> buscarDetalle(int idPedido) throws SQLException {
    List<DetalleProducto> resultado = new ArrayList<DetalleProducto>();
    DetalleProductoIterator iterador = null;

    try {
        #sql [contexto] iterador = {
            SELECT p.id_pedido AS idPedido,
                   c.nombre AS cliente,
                   pr.id_producto AS idProducto,
                   pr.nombre AS producto,
                   lp.cantidad AS cantidad,
                   pr.precio AS precioUnitario,
                   lp.cantidad * pr.precio AS subtotal
            FROM cliente c
            INNER JOIN pedido p ON p.id_cliente = c.id_cliente
            INNER JOIN linea_pedido lp ON lp.id_pedido = p.id_pedido
            INNER JOIN producto pr ON pr.id_producto = lp.id_producto
            WHERE p.id_pedido = :idPedido
            ORDER BY pr.nombre
        };

        while (iterador.next()) {
            resultado.add(new DetalleProducto(
                iterador.idPedido(), iterador.cliente(),
                iterador.idProducto(), iterador.producto(),
                iterador.cantidad(), iterador.precioUnitario(),
                iterador.subtotal()));
        }
        return resultado;
    } finally {
        if (iterador != null) iterador.close();
    }
}
```

## 11. Ejemplo 8: UNION

```java
#sql public iterator CiudadIterator(String ciudad);

public List<String> combinarCiudades(String ciudad1, String ciudad2)
        throws SQLException {

    List<String> resultado = new ArrayList<String>();
    CiudadIterator iterador = null;

    try {
        #sql [contexto] iterador = {
            SELECT ciudad AS ciudad FROM cliente
            UNION
            SELECT :ciudad1 AS ciudad
            FROM cliente
            WHERE id_cliente = (SELECT MIN(id_cliente) FROM cliente)
            UNION
            SELECT :ciudad2 AS ciudad
            FROM cliente
            WHERE id_cliente = (SELECT MIN(id_cliente) FROM cliente)
            ORDER BY ciudad
        };

        while (iterador.next()) resultado.add(iterador.ciudad());
        return resultado;
    } finally {
        if (iterador != null) iterador.close();
    }
}
```

La forma de seleccionar una constante sin `FROM` depende del fabricante. El ejemplo reutiliza una tabla del modelo para evitar `DUAL` u otra tabla especial.

## 12. Ejemplo 9: informe agrupado

```java
public List<ResumenCliente> obtenerInforme() throws SQLException {
    List<ResumenCliente> resultado = new ArrayList<ResumenCliente>();
    ResumenClienteIterator iterador = null;

    try {
        #sql [contexto] iterador = {
            SELECT c.id_cliente AS idCliente,
                   c.nombre AS nombreCliente,
                   COUNT(p.id_pedido) AS numeroPedidos,
                   COALESCE(SUM(p.importe), 0) AS importeTotal,
                   COALESCE(AVG(p.importe), 0) AS importeMedio
            FROM cliente c
            LEFT JOIN pedido p ON p.id_cliente = c.id_cliente
            GROUP BY c.id_cliente, c.nombre
            ORDER BY c.id_cliente
        };

        while (iterador.next()) {
            resultado.add(new ResumenCliente(
                iterador.idCliente(), iterador.nombreCliente(),
                iterador.numeroPedidos(), iterador.importeTotal(),
                iterador.importeMedio()));
        }
        return resultado;
    } finally {
        if (iterador != null) iterador.close();
    }
}
```

## 13. Ejemplo 10: crear pedido completo

```java
public void crearPedidoCompleto(
        final Pedido pedido,
        final List<LineaPedido> lineas) throws SQLException {

    ejecutarEnTransaccion(new TrabajoTransaccional() {
        @Override
        public void ejecutar() throws SQLException {
            int idPedido = pedido.getIdPedido();
            int idCliente = pedido.getIdCliente();
            Date fecha = pedido.getFecha();
            String estado = pedido.getEstado();
            BigDecimal importeTotal = BigDecimal.ZERO;

            #sql [contexto] {
                INSERT INTO pedido
                    (id_pedido, id_cliente, fecha, importe, estado)
                VALUES
                    (:idPedido, :idCliente, :fecha, :importeTotal, :estado)
            };

            for (LineaPedido linea : lineas) {
                int idProducto = linea.getIdProducto();
                int cantidad = linea.getCantidad();
                BigDecimal precio = buscarPrecio(idProducto);

                importeTotal = importeTotal.add(
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
                SET importe = :importeTotal
                WHERE id_pedido = :idPedido
            };
        }
    });
}

private BigDecimal buscarPrecio(int idProducto) throws SQLException {
    BigDecimal precio = null;
    #sql [contexto] {
        SELECT precio
        INTO :precio
        FROM producto
        WHERE id_producto = :idProducto
    };
    if (precio == null || precio.signum() < 0) {
        throw new SQLException("Precio no válido para el producto");
    }
    return precio;
}
```

## 14. Infraestructura transaccional

```java
@FunctionalInterface
private interface TrabajoTransaccional {
    void ejecutar() throws SQLException;
}

private void ejecutarEnTransaccion(TrabajoTransaccional trabajo)
        throws SQLException {

    boolean autoCommitOriginal = connection.getAutoCommit();
    SQLException errorPrincipal = null;

    try {
        connection.setAutoCommit(false);
        trabajo.ejecutar();
        connection.commit();
    } catch (SQLException e) {
        errorPrincipal = e;
        try {
            connection.rollback();
        } catch (SQLException rollbackError) {
            e.addSuppressed(rollbackError);
        }
        throw e;
    } catch (RuntimeException e) {
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

Si la conexión ya pertenece a una transacción externa, el método no debería apropiarse de ella. Debe existir una política explícita de transacciones.

## 15. Gestión de errores

```java
private static void registrar(SQLException exception) {
    SQLException actual = exception;
    while (actual != null) {
        System.err.println(
            "SQLState=" + actual.getSQLState() +
            ", código=" + actual.getErrorCode() +
            ", mensaje=" + actual.getMessage());

        for (Throwable suprimida : actual.getSuppressed()) {
            System.err.println("Error secundario: " + suprimida.getMessage());
        }
        actual = actual.getNextException();
    }
}
```

Clases de SQLState habituales:

- `08`: conexión.
- `22`: datos o conversión.
- `23`: integridad.
- `40`: rollback o conflicto transaccional.

El significado exacto debe verificarse en la implementación.

## 16. Riesgos críticos

- Error de `COMMIT`: el resultado puede ser incierto.
- Error de `ROLLBACK`: conservar como excepción suprimida e invalidar la conexión.
- Timeout: reintentar solo si la operación completa es segura e idempotente.
- Iterador no cerrado: riesgo de agotar cursores.
- Conexión de contexto distinta de la conexión transaccional: control incorrecto.

## 17. Lista de comprobación

- [ ] Estructurar fuentes `.sqlj` y `.java`.
- [ ] Crear iteradores reutilizables.
- [ ] Convertir filas en objetos Java.
- [ ] Comprobar filas afectadas.
- [ ] Implementar diez casos completos.
- [ ] Centralizar transacciones.
- [ ] Conservar errores secundarios.
- [ ] Cerrar iteradores y conexiones según su propiedad.
