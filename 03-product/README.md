# 03 · Producto — Simple Stock Flow

## 1. Problema que Resuelve
El sistema resuelve la gestión ágil y consistente de ventas directas de mostrador y control de inventario [data-model §1, §6.1]:
- **Prevención de sobreventas:** Evita vender mercancía sin existencia física real mediante la restricción a nivel de base de datos (`ck_product_stock_non_negative`) y el control de concurrencia optimista (`xmin`) [data-model §2.2, §3, D-04, ADR-002].
- **Protección de la historia contable:** Impide que los reportes de ventas pasadas cambien si un producto cambia de precio o de nombre en el catálogo, congelando esos valores en la línea de venta al momento de la transacción (`sale_item`) [data-model §1, §2.4, §3, ADR-004].
- **Cero complejidad innecesaria:** Elimina carritos de compra, sesiones temporales, pasarelas de pago, perfiles de clientes y multiplicidad de monedas, funcionando bajo una arquitectura monomoneda estricta (D-05) con ventas inmediatas e inmutables [data-model §1, §3, §7].

## 2. Visión del Producto
Ofrecer una herramienta ligera, transaccional y sin fricción para el control de catálogo y ventas directas en mostrador *(supuesto de negocio apoyado en las 5 categorías semilla: General, Herramientas, Electricidad, Fontanería, Pinturas)* [data-model §9.1]. El valor central es la fiabilidad de las existencias y la auditabilidad histórica de las ventas sin sobrecargar la infraestructura [data-model §2.3, §7.1, §8].

## 3. Usuarios del Producto
El sistema está restringido exclusivamente a operadores internos autenticados; **no contempla clientes finales ni perfiles de compradores** [data-model §1, §7]:
- **Vendedor (`seller`):** Consulta el catálogo y las existencias disponibles (Q1) y efectúa registros atómicos de ventas de mostrador (Q6) [data-model §1, §2.5, §6.1].
- **Administrador (`admin`):** Administra el catálogo de productos (creación, actualización de precios, vinculación de imágenes externas mediante `image_key` y bajas lógicas) y consulta los reportes agregados consolidados sobre rangos de fechas (Q9) [data-model §1, §2.2, §6.1, §11 H-3].
- *(Supuesto)* El administrador inicial se configura por variables de entorno en el despliegue; la asignación del rol `admin` no está disponible como autoservicio en la API [data-model §9.2, §11 H-3].