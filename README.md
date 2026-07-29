# -4H-Concesionario-CampusCar-Raymond-Zabala

# Documentación Técnica del Modelo de Datos
## Sistema de Gestión de Concesionario de Vehículos & Servicio Técnico

Este repositorio contiene la especificación y diseño físico/lógico de la base de datos relacional para el sistema de control comercial y operativo del concesionario. El modelo cubre el ciclo de vida completo: catálogo de vehículos, proceso de venta, gestión de clientes, vendedores y servicios de mantenimiento postventa.

---

## 1. Justificación del Diseño y Arquitectura

El diseño de la base de datos se estructuró aplicando los principios fundamentales de la **Tercera Forma Normal (3FN)** para garantizar la integridad de los datos, eliminar la redundancia y optimizar la ejecución de transacciones complejas.

### 1.1. Estructura de Entidades y Descomposición
* **Desacoplamiento Transaccional (Ventas y Vehículos):** Un vehículo solo puede ser vendido en una transacción específica, pero una transacción comercial (factura/orden) puede incluir la venta de uno o más vehículos. Para soportar esto de manera limpia y flexible, se diseñó la entidad asociativa `VentaVehiculo`. Esto resuelve la relación M:N (Muchos a Muchos) entre `Ventas` y `Vehiculos`.
* **Histórico e Inmutabilidad de Precios:** El precio de lista de un vehículo (`Vehiculos.Precio`) varía con el tiempo o la negociación. Al concretar una venta, la tabla asociativa guarda `PrecioFinalUnidad`. Esto asegura que auditorías históricas no alteren el monto real facturado si el catálogo cambia de precio posteriormente.
* **Gestión Independiente de Mantenimiento (Postventa):** La entidad `Mantenimiento` vincula directamente un vehículo (vía `VIN`) con las intervenciones técnicas prestadas. Se definió la relación con `Clientes` como opcional (clave foránea que permite valor nulo) para contemplar mantenimientos realizados a vehículos del inventario propio (previos a la venta) o servicios prestados a clientes registrados.
* **Identificación Unívoca del Vehículo (VIN):** En lugar de crear un ID autonumérico artificial para los vehículos, se adoptó el **VIN (Vehicle Identification Number)** como Clave Primaria Natural. El VIN es un estándar internacional de 17 caracteres alfanuméricos que garantiza unicidad física en la industria automotriz.

### 1.2. Normalización y Control de Redundancia
Todas las tablas cumplen rigurosamente con 3FN:
1. **1FN:** Todos los atributos contienen valores atómicos indivisibles (sin listas repetitivas de servicios o vehículos dentro de una columna).
2. **2FN:** Todos los atributos que no forman parte de la clave primaria dependen funcionalmente de la clave completa. En `VentaVehiculo`, el `PrecioFinalUnidad` depende de la combinación exacta de `(VentaID, VIN)`.
3. **3FN:** No existen dependencias transitivas entre atributos no clave. Atributos como datos personales del cliente o vendedor se mantienen strictly en sus respectivas entidades maestras.

---

## 2. Diccionario de Datos, Claves y Restricciones

### 2.1. Tabla: Clientes
| Campo | Tipo de Dato | Restricciones | Descripción / Regla de Negocio |
| :--- | :--- | :--- | :--- |
| `Cliente_ID` | INT (AUTO_INCREMENT) | **PK**, NOT NULL | Identificador único interno del cliente. |
| `Nombre` | VARCHAR(120) | NOT NULL | Nombre completo o Razón Social del cliente. |
| `Telefono` | VARCHAR(20) | NULL | Número telefónico principal de contacto. |
| `Email` | VARCHAR(100) | **UNIQUE**, NOT NULL | Correo electrónico único para notificaciones y facturación. |
| `Direccion` | VARCHAR(200) | NULL | Dirección física de residencia o fiscal. |

### 2.2. Tabla: Vendedores
| Campo | Tipo de Dato | Restricciones | Descripción / Regla de Negocio |
| :--- | :--- | :--- | :--- |
| `Vendedor_ID` | INT (AUTO_INCREMENT) | **PK**, NOT NULL | Identificador único del vendedor. |
| `Nombre` | VARCHAR(120) | NOT NULL | Nombre completo del empleado comercial. |
| `NumeroEmpleado` | VARCHAR(20) | **UNIQUE**, NOT NULL | Código institucional de empleado (p. ej. VEN-001). |
| `FechaContratacion` | DATE | NOT NULL | Fecha de ingreso a la compañía. Debe ser `<= CURRENT_DATE`. |

### 2.3. Tabla: Vehiculos
| Campo | Tipo de Dato | Restricciones | Descripción / Regla de Negocio |
| :--- | :--- | :--- | :--- |
| `VIN` | VARCHAR(17) | **PK**, NOT NULL | Número de Identificación Vehicular estandarizado. |
| `Marca` | VARCHAR(50) | NOT NULL | Fabricante del vehículo (p. ej. Toyota, Ford). |
| `Modelo` | VARCHAR(50) | NOT NULL | Línea o modelo del vehículo (p. ej. Corolla, Mustang). |
| `Anio` | INT | NOT NULL | Año de fabricación. Restricción: `Anio >= 1900`. |
| `Precio` | DECIMAL(12,2) | NOT NULL | Precio de catálogo. Restricción: `Precio > 0`. |
| `Color` | VARCHAR(30) | NULL | Color exterior del vehículo. |
| `TipoCombustible` | VARCHAR(30) | NOT NULL | Gasolina, Diésel, Híbrido, Eléctrico. |
| `TipoTransmision` | VARCHAR(30) | NOT NULL | Manual, Automática, CVT. |
| `EstadoVehiculo` | VARCHAR(10) | NOT NULL | CHECK: `EstadoVehiculo IN ('nuevo', 'usado')`. |
| `Disponible` | BOOLEAN | NOT NULL | 1 = Disponible para venta; 0 = Vendido / Reservado. |

### 2.4. Tabla: Ventas
| Campo | Tipo de Dato | Restricciones | Descripción / Regla de Negocio |
| :--- | :--- | :--- | :--- |
| `Venta_ID` | INT (AUTO_INCREMENT) | **PK**, NOT NULL | Identificador de la transacción de venta. |
| `ClienteID` | INT | **FK**, NOT NULL | Referencia a `Clientes(Cliente_ID)`. |
| `VendedorID` | INT | **FK**, NOT NULL | Referencia a `Vendedores(Vendedor_ID)`. |
| `FechaVenta` | DATE | NOT NULL | Fecha del contrato/venta. Default: `CURRENT_DATE`. |
| `Total` | DECIMAL(12,2) | NOT NULL | Monto total negociado. Restricción: `Total >= 0`. |
| `MetodoPago` | VARCHAR(50) | NOT NULL | Contado, Crédito, Transferencia, Leasing. |

### 2.5. Tabla: VentaVehiculo (Tabla Asociativa M:N)
| Campo | Tipo de Dato | Restricciones | Descripción / Regla de Negocio |
| :--- | :--- | :--- | :--- |
| `VentaID` | INT | **PK**, **FK**, NOT NULL | Clave Compuesta. Referencia a `Ventas(Venta_ID)`. |
| `VIN` | VARCHAR(17) | **PK**, **FK**, NOT NULL | Clave Compuesta. Referencia a `Vehiculos(VIN)`. |
| `PrecioFinalUnidad` | DECIMAL(12,2) | NOT NULL | Precio acordado específico de esta unidad en la transacción. |

### 2.6. Tabla: Mantenimiento
| Campo | Tipo de Dato | Restricciones | Descripción / Regla de Negocio |
| :--- | :--- | :--- | :--- |
| `Mantenimiento_ID` | INT (AUTO_INCREMENT) | **PK**, NOT NULL | Identificador de la orden de servicio. |
| `VIN` | VARCHAR(17) | **FK**, NOT NULL | Referencia a `Vehiculos(VIN)`. Obligatorio. |
| `ClienteID` | INT | **FK**, NULL | Referencia opcional a `Clientes(Cliente_ID)`. |
| `TipoServicio` | VARCHAR(20) | NOT NULL | CHECK: `TipoServicio IN ('preventivo', 'correctivo')`. |
| `Costo` | DECIMAL(10,2) | NOT NULL | Costo cobrado por el servicio. Restricción: `Costo >= 0`. |
| `FechaServicio` | DATE | NOT NULL | Fecha de ejecución del mantenimiento. |

---

## 3. Especificación de Relaciones UML y Cardinalidad

1. **Clientes — Ventas (Asociación 1 : N)**
   * **Multiplicidad:** `Clientes (1) ──── (0..*) Ventas`
   * **Semántica:** Un cliente registrado puede realizar cero o múltiples compras en el concesionario a lo largo del tiempo. Cada registro de venta debe pertenecer obligatoriamente a un único cliente.
   * **Acción Referencial (FK):** `ON DELETE RESTRICT` / `ON UPDATE CASCADE`. Previene la eliminación accidental de un cliente con historial de compras activo.

2. **Vendedores — Ventas (Asociación 1 : N)**
   * **Multiplicidad:** `Vendedores (1) ──── (0..*) Ventas`
   * **Semántica:** Un vendedor atiende cero o muchas ventas. Toda transacción de venta finalizada debe estar acreditada a exactamente un vendedor responsable.
   * **Acción Referencial (FK):** `ON DELETE RESTRICT`. Los registros contables y de comisiones impiden borrar un vendedor con facturas asociadas.

3. **Ventas — Vehiculos (Asociación N : M vía VentaVehiculo)**
   * **Multiplicidad:** `Ventas (1..*) ──── (1..*) Vehiculos`
   * **Estructura UML:** Clase de Asociación / Tabla Intermedia `VentaVehiculo`.
   * **Semántica:** Una transacción comercial incluye uno o más vehículos. Un vehículo físicamente puede registrarse en una venta. La tabla intermedia almacena el atributo derivado o propio de la relación: `PrecioFinalUnidad`.
   * **Acción Referencial (FK):** `ON DELETE CASCADE` en la tabla asociativa para mantener coherencia si se anula la cabecera de la venta.

4. **Vehiculos — Mantenimiento (Asociación 1 : N)**
   * **Multiplicidad:** `Vehiculos (1) ──── (0..*) Mantenimiento`
   * **Semántica:** Un vehículo identificado por su VIN puede ingresar múltiples veces al taller para servicio técnico (preventivo o correctivo). Cada orden de mantenimiento corresponde rigurosamente a un único vehículo.
   * **Acción Referencial (FK):** `ON DELETE CASCADE` / `RESTRICT`. Se preserva el historial de servicios.

5. **Clientes — Mantenimiento (Asociación Opcional 0..1 : N)**
   * **Multiplicidad:** `Clientes (0..1) ──── (0..*) Mantenimiento`
   * **Semántica:** Permite asociar la orden de servicio al cliente propietario del vehículo. Es opcional (`0..1`) para poder registrar órdenes sobre vehículos del stock propio del concesionario que aún no han sido comercializados.

---

## 4. Reglas de Integridad Avanzadas y Disparadores (Triggers)

### Control de Disponibilidad del Vehículo
Para evitar la doble venta de una misma unidad física, el sistema debe implementar una validación estricta de disponibilidad mediante triggers o lógica transaccional:

* **Trigger `INSERT` en `VentaVehiculo`:** Verifica que `Vehiculos.Disponible = TRUE` antes de permitir la inserción. Si está disponible, inserta la relación y ejecuta inmediatamente un `UPDATE Vehiculos SET Disponible = FALSE WHERE VIN = NEW.VIN`.
* **Trigger `DELETE` / Anulación en `VentaVehiculo`:** Restablece la disponibilidad del vehículo marcando `Disponible = TRUE`.
* **Cálculo Automático del Total en `Ventas`:** El atributo `Ventas.Total` puede actualizarse dinámicamente mediante la suma de los `PrecioFinalUnidad` de la tabla asociativa `VentaVehiculo`.
