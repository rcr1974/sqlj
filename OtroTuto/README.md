# Guía completa de SQLJ, SQL y Java 8

Esta distribución divide la guía en cuatro entregas independientes en formato Markdown.

## Contenido

1. `01_fundamentos_sqlj.md`
   - Qué es SQLJ.
   - Arquitectura, traducción y ejecución.
   - Variables host, conexiones y contextos.
   - `SELECT INTO` e iteradores.

2. `02_operaciones_sql_principales.md`
   - `SELECT`, `INSERT`, `UPDATE` y `DELETE`.
   - `JOIN`, `UNION`, subconsultas y filtros.
   - Agrupaciones, agregaciones y valores `NULL`.
   - `COMMIT`, `ROLLBACK` y transacciones.

3. `03_ejemplos_java_completos.md`
   - Estructura de una aplicación SQLJ.
   - Diez ejemplos completos.
   - Contextos, iteradores, transacciones y recursos.
   - Gestión detallada de errores.

4. `04_comparativa_buenas_practicas_ejercicio.md`
   - SQLJ frente a JDBC.
   - Buenas prácticas.
   - Ejercicio final resuelto.
   - Mapa conceptual y lista de comprobación.

## Modelo común

```text
CLIENTE 1 ───── N PEDIDO 1 ───── N LINEA_PEDIDO N ───── 1 PRODUCTO
```

## Consideraciones

- Versión Java base: Java 8.
- Los archivos con cláusulas `#sql` deben procesarse con un traductor SQLJ.
- La configuración exacta del traductor, los perfiles SQL y el runtime depende de la implementación.
- Las extensiones específicas de Oracle, IBM Db2 u otro fabricante deben verificarse en su documentación.
- Los ejemplos no contienen credenciales reales ni dependen de una base de datos activa.
