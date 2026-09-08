# Entrega 1. Fundamentos, arquitectura e iteradores SQLJ

## 1. Modelo de datos

```text
CLIENTE
├── id_cliente
├── nombre
├── email
└── ciudad

PEDIDO
├── id_pedido
├── id_cliente
├── fecha
├── importe
└── estado

PRODUCTO
├── id_producto
├── nombre
├── precio
└── categoria

LINEA_PEDIDO
├── id_pedido
├── id_producto
└── cantidad
```

```text
┌──────────────────┐
│     CLIENTE      │
├──────────────────┤
│ PK id_cliente    │
└────────┬─────────┘
         │ 1:N
┌────────▼─────────┐
│      PEDIDO      │
├──────────────────┤
│ PK id_pedido     │
│ FK id_cliente    │
└────────┬─────────┘
         │ 1:N
┌────────▼─────────┐       ┌──────────────────┐
│  LINEA_PEDIDO    │ N:1   │     PRODUCTO     │
├──────────────────┤──────►├──────────────────┤
│ PK id_pedido     │       │ PK id_producto   │
│ PK id_producto   │       │    precio        │
│    cantidad      │       └──────────────────┘
└──────────────────┘
```

## 2. Qué es SQLJ

SQLJ permite incluir sentencias SQL estáticas dentro de código Java. Las cláusulas comienzan con `#sql` y se procesan antes de compilar el programa.

```java
int idCliente = 10;
String nombreCliente = null;

#sql {
    SELECT nombre
    INTO :nombreCliente
    FROM cliente
    WHERE id_cliente = :idCliente
};
```

SQLJ no sustituye a SQL. Define cómo integrar SQL estático en Java.

### SQL estático

La estructura se conoce durante la traducción. Los valores pueden cambiar, pero no la tabla, las columnas o la forma de la consulta.

```text
Puede cambiar: idCliente = 10, 20, 30...
No cambia: SELECT nombre FROM cliente WHERE id_cliente = ?
```

Una variable host representa un valor, no sintaxis SQL.

```java
// Correcto
WHERE ciudad = :ciudad

// Incorrecto conceptualmente
FROM :nombreTabla
```

## 3. Diferencias esenciales

### SQL convencional

```sql
SELECT nombre
FROM cliente
WHERE id_cliente = 10;
```

### SQLJ

```java
#sql {
    SELECT nombre
    INTO :nombre
    FROM cliente
    WHERE id_cliente = :idCliente
};
```

### JDBC

```java
String sql = "SELECT nombre FROM cliente WHERE id_cliente = ?";
try (PreparedStatement ps = connection.prepareStatement(sql)) {
    ps.setInt(1, idCliente);
    try (ResultSet rs = ps.executeQuery()) {
        if (rs.next()) {
            nombre = rs.getString("nombre");
        }
    }
}
```

### SQL dinámico

Cuando cambian columnas, filtros, tablas u ordenación durante la ejecución, JDBC suele resultar más apropiado.

### Procedimientos almacenados

La lógica se ejecuta principalmente en la base de datos. SQLJ puede invocarlos, pero las convenciones de llamada y tipos pueden depender del fabricante.

### JPA e Hibernate

JPA/Hibernate realizan mapeo objeto-relacional. SQLJ trabaja con SQL explícito y no proporciona seguimiento de entidades, carga diferida o caché ORM.

## 4. Ventajas y limitaciones

### Ventajas

- SQL visible y explícito.
- Variables host claramente identificadas.
- Menor código ceremonial que JDBC.
- Iteradores tipados.
- Posible comprobación anticipada de sintaxis, esquema y tipos.

### Limitaciones

- Requiere traductor y runtime SQLJ.
- Añade una fase al proceso de construcción.
- Menor presencia en proyectos modernos.
- Menor flexibilidad para SQL dinámico.
- La portabilidad depende del traductor, runtime, dialecto y tipos.

## 5. Traducción, compilación y ejecución

```text
Archivo .sqlj
Java + #sql
     │
     ▼
Traductor SQLJ
     ├── Procesa cláusulas
     ├── Comprueba variables host
     ├── Genera iteradores
     └── Genera código/perfiles
     │
     ▼
Código Java traducido
     │
     ▼
Compilador javac
     │
     ▼
Clases Java + runtime SQLJ
     │
     ▼
Driver / infraestructura del fabricante
     │
     ▼
Base de datos
```

Los perfiles, la personalización y el `bind` dependen de la implementación. Oracle y Db2 no deben considerarse idénticos.

## 6. Anatomía de una cláusula

```java
#sql {
    sentencia SQL
};
```

```text
#sql    {    sentencia SQL    }    ;
 │      │          │          │    └─ fin
 │      │          │          └──── cierre
 │      │          └─────────────── SQL y variables host
 │      └────────────────────────── apertura
 └───────────────────────────────── marcador SQLJ
```

### Variables de entrada y salida

```java
String nombreCliente = null;
int idCliente = 10;

#sql {
    SELECT nombre
    INTO :nombreCliente
    FROM cliente
    WHERE id_cliente = :idCliente
};
```

```text
idCliente ──► SQL ──► nombre ──► nombreCliente
 entrada                         salida
```

## 7. Contextos y conexiones

```text
Cláusula SQLJ
      │
      ▼
Contexto de conexión
      │
      ▼
Conexión JDBC
      │
      ▼
Base de datos
```

### Contexto predeterminado

```java
Connection connection = DriverManager.getConnection(url, user, password);
connection.setAutoCommit(false);

DefaultContext context = new DefaultContext(connection);
DefaultContext.setDefaultContext(context);
```

### Contexto explícito

```java
#sql context ContextoAplicacion;

#sql [contexto] {
    UPDATE pedido
    SET estado = :nuevoEstado
    WHERE id_pedido = :idPedido
};
```

El contexto de traducción y el de ejecución pueden apuntar a entornos diferentes, pero los esquemas deben ser compatibles.

## 8. Recuperación de una fila

```java
String nombreCliente = null;
int idCliente = 10;

#sql [contexto] {
    SELECT nombre
    INTO :nombreCliente
    FROM cliente
    WHERE id_cliente = :idCliente
};
```

```text
¿Cuántas filas?
├── 0: condición de no encontrado o SQLException
├── 1: asignación correcta
└── más de 1: error de cardinalidad
```

Para columnas anulables deben preferirse tipos de referencia:

```text
INTEGER  → Integer
VARCHAR  → String
DECIMAL  → BigDecimal
DATE     → java.sql.Date
```

## 9. Recuperación de varias filas

### Iterador nombrado

```java
#sql iterator PedidoIterator(
    int idPedido,
    java.sql.Date fecha,
    java.math.BigDecimal importe,
    String estado
);
```

```java
PedidoIterator pedidos = null;
try {
    #sql [contexto] pedidos = {
        SELECT id_pedido AS idPedido,
               fecha AS fecha,
               importe AS importe,
               estado AS estado
        FROM pedido
        WHERE id_cliente = :idCliente
        ORDER BY fecha, id_pedido
    };

    while (pedidos.next()) {
        System.out.println(pedidos.idPedido());
    }
} finally {
    if (pedidos != null) {
        pedidos.close();
    }
}
```

### Iterador posicional

```java
#sql iterator PedidoPosicional(
    int,
    java.sql.Date,
    java.math.BigDecimal,
    String
);
```

El orden y los tipos deben coincidir exactamente con las columnas seleccionadas.

## 10. Lista de comprobación

- [ ] Comprender SQL estático.
- [ ] Distinguir SQLJ, JDBC, SQL dinámico y ORM.
- [ ] Entender el proceso de traducción.
- [ ] Utilizar variables host.
- [ ] Configurar un contexto.
- [ ] Recuperar una fila con `SELECT INTO`.
- [ ] Recorrer varias filas mediante iteradores.
- [ ] Tratar correctamente `NULL`.
- [ ] Cerrar iteradores y conexiones según su propiedad.
- [ ] Identificar dependencias del fabricante.
