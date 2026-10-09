# 02 · Dominio — Simple Stock Flow

## 1. Entidades del Dominio y Agregados

### Agregado `Product` (Catálogo)
- **Raíz de Agregado:** `Product`.
- **Identidad:** Clave primaria `id` de tipo UUID.
- **Atributos:**
  - `name`: Nombre descriptivo obligatorio no vacío.
  - `price`: Objeto de valor `Money` con precisión decimal fija a 2 decimales (`numeric(18,2)`), no negativo y con política de redondeo simétrica [data-model §1, §2.2, §3].
  - `stock`: Cantidad entera disponible para venta en inventario [data-model §2.2, §3].
  - `category_id`: Referencia obligatoria a una categoría válida (`FK-1`) [data-model §2.1, §2.2, §5].
  - `image_key`: Clave opaca opcional para almacenamiento externo de binarios; admite valor nulo [data-model §1, §2.2, §3, D-08].
- **Manejo de Ciclo de Vida:** Baja lógica mediante marca temporal `deleted_at`; no admite eliminación física para preservar consistencia relacional con ventas pasadas (`FK-3`) [data-model §2.2, §3, §5, ADR-003].

### Agregado `Sale` (Ventas)
- **Raíz de Agregado:** `Sale`.
- **Identidad:** Clave primaria `id` de tipo UUID.
- **Atributos:**
  - `sold_at`: Marca temporal obligatoria del instante de venta en tiempo universal coordinado (UTC) [data-model §3, §8].
  - `sold_by`: Identificador del operador interno autenticado que efectuó la transacción [data-model §3, §7].
- **Composición Interna:**
  - `SaleItem`: Entidad subordinada (composición 1:N). Solo existe dentro del ciclo de vida de `Sale` (`FK-2` en cascada a nivel relacional) [data-model §2.3, §2.4, §5].
  - Contiene datos transaccionales congelados: `product_id`, `product_name`, `quantity`, `unit_price` y `category_name` [data-model §1, §2.4, §3, ADR-004].

### Agregado `User` (Identidad y Operación)
- **Raíz de Agregado:** `User`.
- **Identidad:** Clave primaria `id` de tipo UUID.
- **Atributos:**
  - `username`: Identificador de inicio de sesión normalizado en minúsculas y sin espacios (`IX_user_username`) [data-model §2.5, §4].
  - `password_hash`: Secreto criptográfico protegido para validación de acceso [data-model §1, §2.5, §3, D-09].
  - `role`: Rol funcional restringido al conjunto cerrado `'admin'` o `'seller'` [data-model §1, §2.5].

### Entidad de Referencia `Category`
- **Naturaleza:** Entidad de solo lectura sembrada inicialmente en base de datos con 5 registros predefinidos [data-model §2.1, §9.1]. No tiene ciclo de vida modificable dentro del dominio.

---

## 2. Reglas e Invariantes del Dominio

- **R-01 (Stock No Negativo):** El inventario de un producto nunca puede quedar en números negativos (`stock >= 0`), garantizado en persistencia mediante la restricción de verificación `ck_product_stock_non_negative` [data-model §2.2, §4].
- **R-02 (Precio Estrictamente Positivo):** El precio de venta establecido para un producto en catálogo debe ser superior a cero (`price > 0`) [data-model §2.2, §4].
- **R-03 (Unicidad de Ítem por Venta):** No se permite duplicar un mismo `product_id` en múltiples líneas dentro de una misma transacción de venta (`IX_sale_item_sale_id_product_id`) [data-model §2.3, §4].
- **R-04 (Venta con Contenido Obligatorio):** Una transacción de venta debe incluir al menos una línea de detalle (`SaleItem`) para considerarse válida y confirmada [data-model §2.3].
- **R-05 (Congelación Histórica de Precios y Nombres):** En el momento del registro de la venta, los valores actuales de `name`, `price` y `category_name` se copian y fijan dentro del `SaleItem`. Cambios futuros en el catálogo no afectan transacciones pasadas [data-model §1, §2.4, §3, ADR-004].
- **R-06 (Inmutabilidad de la Venta):** Las ventas confirmadas son transacciones históricas inmutables; no permiten modificaciones posteriores ni eliminaciones lógicas/físicas [data-model §1, §2.3, §7.1].
- **R-07 (Baja Lógica de Producto):** Los productos con o sin ventas previas no se borran físicamente del catálogo, sino que se inhabilitan asignando la fecha de baja a `deleted_at` [data-model §2.2, §3, §5, ADR-003].

---

## 3. Eventos de Dominio

- **`SaleRegistered`:** Se emite cuando una venta es confirmada con éxito, provocando la reducción atómica del inventario de los productos asociados [data-model §2.3, §6.1 Q6].
- **`ProductStockAdjusted`:** Se emite ante operaciones de reposición o retiro explícito de existencias en el catálogo [data-model §2.2].
- **`ProductDeactivated`:** Se emite cuando un producto es dado de baja lógica mediante el marcado de `deleted_at` [data-model §2.2, ADR-003].

---

## 4. Lenguaje Ubicuo (Glosario)

- **Producto:** Bien comercializable registrado con nombre, precio, stock, categoría asignada y referencia opcional a imagen binaria [data-model §2.2].
- **Venta:** Hecho transaccional consumado e inmutable registrado por un operador interno en una estampa temporal UTC [data-model §2.3, §8].
- **Línea de Venta (`SaleItem`):** Componente individual de una venta que congela la cantidad, el importe y los descriptores del producto en dicho instante [data-model §2.4].
- **Operador:** Usuario interno autenticado (`admin` o `seller`) facultado para realizar operaciones en la plataforma; el sistema no modela perfiles de compradores externos [data-model §1, §7].