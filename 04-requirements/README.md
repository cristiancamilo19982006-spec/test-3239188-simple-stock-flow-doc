# 04 · Requisitos del Sistema — Simple Stock Flow

## 1. Historias de Usuario (Funcionales)

### HU-01: Gestión y Consulta de Catálogo de Productos
- **Como:** Administrador del sistema.
- **Quiero:** Registrar, consultar y actualizar productos con nombre, precio, stock, categoría e imagen opcional.
- **Para:** Mantener actualizada la oferta comercial del negocio.
- **Criterios de Aceptación:**
  - El producto exige nombre obligatorio no vacío recortado, precio estrictamente mayor a 0 y categoría obligatoria existente (`FK-1`) [data-model §2.1, §2.2, §5].
  - La imagen es opcional; si no se provee se almacena como `NULL` (`image_key IS NULL`), nunca cadena vacía [data-model §1, §2.2, §3].
  - La eliminación de un producto no debe borrar la fila físicamente, sino registrar la fecha de baja lógica (`deleted_at`), protegiendo la integridad de ventas previas (`FK-3`) [data-model §2.2, §3, §5, ADR-003].
  - Debe permitir búsqueda paginada filtrada por categoría, texto y estado activo (Patrón Q1) [data-model §6.1].

### HU-02: Registro Inmutable de Venta
- **Como:** Vendedor autenticado (`seller`).
- **Quiero:** Registrar una transacción de venta indicando productos y cantidades.
- **Para:** Formalizar el hecho comercial y actualizar existencias en una sola operación atómica.
- **Criterios de Aceptación:**
  - La venta debe registrar el instante exacto en UTC (`sold_at`) y el operador responsable (`sold_by`) [data-model §3, §8].
  - La venta exige al menos una línea para confirmarse [data-model §2.3].
  - No se permite repetir un mismo producto en varias líneas de la misma venta (`IX_sale_item_sale_id_product_id`) [data-model §2.3, §4].
  - El registro de la línea y el descuento de inventario ocurren juntos; si la cantidad supera el disponible o el stock final resulta negativo, la operación falla (`ck_product_stock_non_negative`) [data-model §2.2, §2.3, §4].
  - La línea de venta congela una copia del nombre del producto, precio unitario y categoría en el momento de la venta [data-model §1, §2.4, §3].
  - Una venta registrada es estrictamente inmutable: no se edita ni se borra [data-model §1, §2.3, §7.1].

### HU-03: Consulta de Reporte Consolidado de Ventas
- **Como:** Administrador del negocio.
- **Quiero:** Consultar un reporte consolidado de ventas sobre un rango de fechas.
- **Para:** Conocer unidades vendidas e importes totales agrupados por producto y categoría.
- **Criterios de Aceptación:**
  - El rango de fechas exige fecha de fin mayor o igual a fecha de inicio [data-model §1].
  - La agregación se calcula en el motor agrupando por `product_id`, `product_name` y `category_name` congelados (Patrón Q9) [data-model §1, §6.1, §11.1].
  - Si un producto fue recategorizado históricamente, el reporte muestra filas separadas respetando las etiquetas congeladas, sin recalcular períodos cerrados [data-model §11.1, ADR-004].
  - El reporte no desglosa por vendedor ni expone datos de clientes (no existen perfiles de comprador) [data-model §1, §7, DP-02].

### HU-04: Autenticación de Operadores
- **Como:** Operador interno (`admin` o `seller`).
- **Quiero:** Iniciar sesión con mi nombre de usuario y contraseña.
- **Para:** Acceder a las operaciones autorizadas según mi rol.
- **Criterios de Aceptación:**
  - El nombre de usuario se normaliza a minúsculas y sin espacios antes de verificar (`IX_user_username`) [data-model §2.5, §4].
  - La contraseña nunca se valida ni almacena en texto plano; se coteja contra `password_hash` mediante un puerto criptográfico [data-model §1, §2.5, §3, D-09].
  - El rol asignado pertenece al conjunto cerrado `'admin'` o `'seller'` [data-model §1, §2.5].

---

## 2. Requisitos No Funcionales (RNF)

- **RNF-01 (Integridad y Concurrencia Optimista):** El sistema previene sobreventas concurrentes mediante la columna de sistema `xmin` como testigo de versión y la restricción `ck_product_stock_non_negative` en el motor de base de datos [data-model §2.2, §3, D-04, ADR-002].
- **RNF-02 (Inmutabilidad Contable):** Toda venta registrada es permanente e inalterable; no existen puertos de edición ni borrado físico (`FK-2` existe para modelar composición de ciclo de vida, no para borrados operativos) [data-model §1, §2.3, §7.1].
- **RNF-03 (Seguridad y Privacidad):** El secreto `password_hash` jamás se expone en respuestas de API, logs ni índices. Los nombres de usuario son datos personales y su acceso se restringe a interfaces autorizadas [data-model §7].
- **RNF-04 (Rendimiento en Reportes):** El índice compuesto único `IX_sale_item_sale_id_product_id` con `INCLUDE (quantity, unit_price)` permite resolver el reporte (Q9) directamente sobre el índice sin acceder a la tabla base [data-model §6.2, T-13].
- **RNF-05 (Precisión Monetaria):** Precisión decimal fija (`numeric(18,2)`) con estrategia de redondeo `MidpointRounding.AwayFromZero` en el objeto `Money` idéntica a la definición de columna [data-model §2.2, §3].
- **RNF-06 (Restricción Monomoneda):** *(Supuesto basado en D-05)* El sistema opera con una única moneda implícita; no existen tipos de cambio ni columnas de divisa [data-model §1, §3, D-05].