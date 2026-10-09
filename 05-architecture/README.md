# 05 · Arquitectura del Sistema — Simple Stock Flow

## 1. Estilo Arquitectónico
El sistema implementa una **Arquitectura Hexagonal (Puertos y Adaptadores)** orientada al dominio (DDD) [data-model §0, §2]:
- **Núcleo de Dominio:** Encapsula las entidades, agregados y objetos de valor con sus invariantes de negocio, sin depender de librerías externas ni motores de bases de datos [data-model §0, §2].
- **Capa de Aplicación:** Coordina los casos de uso a través de puertos de entrada y despacha efectos secundarios mediante puertos de salida [data-model §1, §6.1].
- **Adaptadores Outbound (Secundarios):**
  - **Persistencia:** Implementado con Entity Framework Core sobre PostgreSQL 16 (base `simple_stock_flow`, esquema `sales`) [data-model preámbulo, §0, §3.1].
  - **Almacenamiento de Binarios:** Adaptador externo de almacenamiento de archivos; el dominio y la base de datos solo conocen una clave opaca (`image_key`), nunca el binario ni rutas físicas directas [data-model §1, §3, D-08].
  - **Criptografía/Seguridad:** Puerto de hash para la creación y verificación segura de credenciales de usuario [data-model §1, §2.5, D-09].

## 2. Límites de Agregados y Raíces de Agregado
A partir del modelo de datos de 5 tablas se identifican 3 raíces de agregado y 1 entidad de referencia [data-model §2]:

1. **Agregado `Product` (Catálogo):**
   - **Raíz:** `Product` [data-model §2.2].
   - **Entidades internas:** Ninguna [data-model §2].
   - **Objetos de valor:** `Money` (valor monetario positivo con redondeo simétrico a 2 decimales `MidpointRounding.AwayFromZero`, monomoneda estricto sin divisa) [data-model §1, §2.2, D-05].
   - **Límites e Invariantes:** Controla existencias (`stock >= 0`). La concurrencia se resuelve por concurrencia optimista apoyada en la columna de sistema `xmin` [data-model §3, D-04, ADR-002]. El borrado se gestiona exclusivamente por baja lógica mediante la propiedad sombra `deleted_at` [data-model §2.2, §3, ADR-003].

2. **Agregado `Sale` (Ventas):**
   - **Raíz:** `Sale` [data-model §2.3].
   - **Entidades internas:** `SaleItem` (composición pura 1:N; no posee ciclo de vida independiente fuera de su venta) [data-model §2.4, §5 FK-2].
   - **Objetos de valor:** `Quantity` (entero estrictamente positivo) [data-model §1, §2.4].
   - **Límites e Invariantes:** Inmutable tras ser registrada [data-model §1, §2.3]. Al registrar la venta se congela una copia del nombre del producto, precio unitario y categoría para garantizar la auditabilidad histórica [data-model §1, §2.4, §3, ADR-004].

3. **Agregado `User` (Identidad y Operación):**
   - **Raíz:** `User` [data-model §2.5].
   - **Atributos de valor:** `Role` (restringido a `'admin'` o `'seller'`) [data-model §1, §2.5].
   - **Límites:** Modela operadores internos del comercio. El modelo no incluye entidades para clientes o compradores [data-model §1, §7].

4. **Entidad de Referencia `Category`:**
   - **Naturaleza:** Entidad estática de solo lectura [data-model §2.1].
   - **Límites:** No es raíz de agregado ni dispone de puertos para mutación en runtime; está sembrada en base de datos con 5 identificadores UUID fijos [data-model §2.1, §9.1].

## 3. Puertos de la Arquitectura Hexagonal

### Puertos de Entrada (Casos de Uso)
- `IProductCatalogUseCase`: Creación de productos, ajuste de precios, movimiento de existencias y baja lógica [data-model §2.2, §6.1 Q1-Q3].
- `IRegisterSaleUseCase`: Registro atómico de ventas y reducción de existencias en catálogo [data-model §2.3, §6.1 Q6].
- `ISalesReportQuery`: Consulta optimizada de agregación de ventas por producto y categoría congelada en rangos de fechas [data-model §1, §6.1 Q9, §11.1].
- `IAuthenticationUseCase`: Autenticación de operadores buscando por nombre de usuario normalizado [data-model §2.5, §6.1 Q10].
- *(Supuesto)* `IUserProvisioningUseCase`: Creación de vendedores por parte de administradores [data-model §9.2, §11 H-3].

### Puertos de Salida (Adaptadores de Infraestructura)
- `IProductRepository`: Lecturas unitarias, búsquedas paginadas (Q1) y lecturas por lotes para contención de stock (Q3) [data-model §6.1].
- `ISaleRepository`: Persistencia de la raíz `Sale` junto a su composición `SaleItem` [data-model §2.3, §6.1 Q6-Q7].
- `IReadOnlyCategoryRepository`: Consulta del catálogo base de categorías sembradas (Q4, Q5) [data-model §2.1, §6.1].
- `IUserRepository`: Consulta de operadores por username exacto (Q10) [data-model §2.5, §6.1].
- `ISalesReportPort`: Puerto de lectura directa a base de datos que resuelve la agregación del reporte sin hidratar agregados de dominio [data-model §1, §6.1 Q9].
- `IImageStoragePort`: Subida y baja física de imágenes externas referenciadas por `image_key` [data-model §1, §7.1, D-08].
- `IPasswordHasher`: Generación unidireccional y verificación de credenciales [data-model §1, §2.5, D-09].

## 4. Matriz de Invariantes: Dominio vs Motor

| Invariante / Regla | Capa de Dominio | Motor de Base de Datos | Cita Modelo |
|---|---|---|---|
| Stock no negativo | `Product.Withdraw` / `Restock` | Restricción `ck_product_stock_non_negative` | data-model §2.2, §4 |
| Retiro mayor a existencias | `Product.Withdraw` lanza excepción | No expresable en SQL estático | data-model §2.2 |
| Precio estrictamente mayor a 0 | Objeto `Money` / `Product.ChangePrice` | Pendiente `CHECK` en motor (T-20) | data-model §2.2, §4 |
| Inmutabilidad de la venta | `Sale` no expone métodos de edición | Ausencia de puertos/rutas de actualización | data-model §1, §2.3, §7.1 |
| Composición Venta - Líneas | `Sale.AddItem` vincula renglones | `FK_sale_item_sale_sale_id ON DELETE CASCADE` | data-model §2.4, §5 FK-2 |
| No huerfanar ventas de producto | Dominio aplica baja lógica (`deleted_at`) | `FK_sale_item_product_product_id ON DELETE RESTRICT` | data-model §4, §5 FK-3 |
| Sin productos duplicados en venta | `Sale.AddItem` rechaza duplicado | Índice único `(sale_id, product_id)` en `sale_item` | data-model §2.3, §4 |
| Unicidad de nombre de usuario | `User.NormalizeUsername` en minúsculas | Índice único `IX_user_username` | data-model §2.5, §4 |
| Roles permitidos (`admin`, `seller`) | Validación en `Roles.IsValid` | Pendiente restricción `CHECK` (T-20) | data-model §2.5, §4 |
| Datos históricos congelados | `Sale.AddItem` copia valores vivos | Columnas `product_name` y `category_name` en `sale_item` | data-model §1, §2.4, §3 |