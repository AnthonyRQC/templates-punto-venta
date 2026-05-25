# Esquema de Base de Datos - Sistema Minimarket

Para soportar todas las funcionalidades modeladas en las interfaces (Directorio de Productos, Kardex de Inventario, Punto de Venta con Tickets, Facturas OCR y Órdenes de Compra con Recepción de Mercancías), la base de datos está estructurada de forma relacional.

A continuación se detalla el diseño de la base de datos con sus tablas, columnas y relaciones.

---

### 1. Tabla: `departamentos` (Categorización de Productos)
*Almacena los departamentos y subdepartamentos (ej. Lácteos > Yogurts) mediante una estructura jerárquica.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`nombre`**: Texto, Obligatorio (ej. "Lácteos", "Yogurts").
- **`parent_id`**: Entero, **Llave Foránea** que enlaza consigo misma a `departamentos(id)`. Si es nulo (`NULL`), es una categoría padre.
- **`descripcion`**: Texto, Opcional.

### 2. Tabla: `proveedores`
*Directorio de empresas de distribución (formales e informales).*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`nombre`**: Texto, Obligatorio (ej. "PIL Andina S.A.", "Coca-Cola", "Don Juan - Quesos Artesanales").
- **`nit`**: Texto, Único, Obligatorio (NIT oficial de Bolivia, o un código interno autogenerado `INF-XXXX` si es un proveedor informal).
- **`contacto`**: Texto (Nombre de la persona o agente de ventas).
- **`telefono`**: Texto.
- **`email`**: Texto.
- **`direccion`**: Texto, Opcional.
- **`es_informal`**: Booleano (Si es un proveedor local/artesanal que no emite NIT ni facturas oficiales).
- **`activo`**: Booleano (por defecto `True`, para compras futuras).

### 3. Tabla: `productos`
*El catálogo central de mercancías (Inventario Global).*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`codigo_barras`**: Texto, Único, Opcional (Código de barras universal EAN-13/UPC. Puede ser nulo si el producto no tiene código de barra físico y se genera un SKU interno).
- **`nombre_legible`**: Texto, Obligatorio (Nombre comercial y limpio para la tienda, ej. "Coca-Cola 2 Litros").
- **`departamento_id`**: Entero, **Llave Foránea** que enlaza con `departamentos(id)`.
- **`proveedor_defecto_id`**: Entero, **Llave Foránea** que enlaza con `proveedores(id)` (Distribuidor sugerido por defecto).
- **`precio_costo`**: Decimal/Moneda (Último precio de compra neto por unidad).
- **`precio_venta`**: Decimal/Moneda (Precio base al público general).
- **`existencia`**: Decimal (Stock disponible en tienda. Aumenta con recepciones conciliadas y disminuye con ventas).
- **`stock_minimo`**: Decimal (Límite crítico para disparar alertas en "Compras sugeridas").
- **`stock_maximo`**: Decimal (Nivel tope recomendado de almacenamiento).
- **`usa_inventario`**: Booleano (Si descuenta stock físicamente o si es un servicio).
- **`fecha_creacion`**: Fecha y Hora.

### 3.5 Tabla: `proveedor_productos` (Tabla de Mapeo/Equivalencias)
*Vincula los códigos y descripciones caóticas del proveedor con nuestro catálogo limpio.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`proveedor_id`**: Entero, **Llave Foránea** que enlaza con `proveedores(id)`.
- **`codigo_proveedor`**: Texto, Obligatorio (Código que el proveedor le asigna a su producto en su factura).
- **`nombre_original_proveedor`**: Texto, Obligatorio (Nombre oficial/codificado en la factura, ej. "COCA COLA 2L NR RET").
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)` (Mapeo a nuestro catálogo limpio).
- **`unidades_por_paquete`**: Entero (Cantidad de unidades individuales por paquete/caja de compra. Por defecto 1).
- *Restricción Única:* `UniqueConstraint(proveedor_id, codigo_proveedor)` para asegurar que un código de proveedor apunte a un solo producto maestro.

### 4. Tabla: `promociones` (Escalas de Precios / Por Mayor)
*Reglas de descuento y precios por volumen (al por mayor).*
- **`id`**: Entero, Llave Primaria.
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)`.
- **`descripcion`**: Texto (ej. "Precio por Docena", "3x2 en refrescos").
- **`precio_promocional`**: Decimal (Precio especial si se cumple la cantidad).
- **`cantidad_requerida`**: Decimal (ej. `12` unidades para activar el precio por mayor).
- **`fecha_inicio`**: Fecha y Hora de activación (nulo si es precio de por mayor fijo).
- **`fecha_fin`**: Fecha y Hora de vencimiento (nulo si es fijo).
- **`activa`**: Booleano.

### 5. Tabla: `usuarios`
*Control de accesos y auditoría de acciones.*
- **`id`**: Entero, Llave Primaria.
- **`username`**: Texto, Único (Nombre de usuario).
- **`nombre_completo`**: Texto.
- **`rol`**: Texto (ej. "Administrador", "Cajero").
- **`password_hash`**: Texto (Contraseña cifrada).
- **`activo`**: Booleano.

### 6. Tabla: `movimientos_inventario` (El Kardex Valorado)
*Bitácora detallada e inmutable de variaciones físicas de stock y su valoración.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)`.
- **`usuario_id`**: Entero, **Llave Foránea** que enlaza con `usuarios(id)` (autor del movimiento).
- **`fecha`**: Fecha y Hora exacta del registro.
- **`tipo`**: Texto (Enum: `'Entrada'`, `'Salida'`, `'Ajuste'`).
- **`concepto`**: Texto (ej. "Venta Mostrador", "Merma por caducidad", "Ingreso por Compra").
- **`cantidad`**: Decimal (Cantidad alterada, siempre positivo).
- **`costo_unitario`**: Decimal (Costo del producto en ese momento, guardando un histórico).
- **`saldo`**: Decimal (Existencia resultante después de aplicar el movimiento).
- **`observaciones`**: Texto (Notas añadidas en mermas o ajustes manuales).

### 7. Tabla: `ventas` (Cabecera del Ticket)
*Resumen de transacciones del Punto de Venta.*
- **`id`**: Entero, Llave Primaria.
- **`folio`**: Texto, Único (ej. "T-4021").
- **`fecha`**: Fecha y Hora de cobro.
- **`usuario_id`**: Entero, **Llave Foránea** que enlaza con `usuarios(id)` (cajero que realizó el cobro).
- **`total`**: Decimal (Monto final liquidado).
- **`metodo_pago`**: Texto (ej. "Efectivo", "Tarjeta", "Transferencia").
- **`pago_con`**: Decimal (Efectivo entregado por cliente).
- **`cambio`**: Decimal (Monto retornado al cliente).

### 8. Tabla: `ventas_detalles` (Líneas del Ticket)
*Artículos incluidos en cada venta.*
- **`id`**: Entero, Llave Primaria.
- **`venta_id`**: Entero, **Llave Foránea** que enlaza con `ventas(id)` con eliminación en cascada.
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)`.
- **`cantidad`**: Decimal (Cantidad de unidades vendidas).
- **`precio_unitario`**: Decimal (Precio base en catálogo en ese momento).
- **`descuento_aplicado`**: Decimal (Monto descontado si se aplicó un precio por mayor o promo).
- **`subtotal`**: Decimal (Calculado: `(cantidad * precio_unitario) - descuento_aplicado`).

### 9. Tabla: `lista_compras` (Apuntes y Sugerencias de Pedido)
*Funciona como la lista de compras o cuaderno de apuntes del minimarket. Registra sugerencias automáticas por bajo stock y notas manuales de productos a pedir.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)`.
- **`proveedor_id`**: Entero, **Llave Foránea** que enlaza con `proveedores(id)` (Proveedor sugerido para realizar el pedido).
- **`cantidad_sugerida`**: Decimal (Cantidad estimada a pedir para reponer stock o según solicitud).
- **`motivo`**: Texto (Enum: `'Stock Bajo'`, `'Manual'`).
- **`notas`**: Texto (ej. "Pedir sabor frutilla preferentemente", "Cliente Don Pedro encargó").
- **`fecha_registro`**: Fecha y Hora.
- **`estado`**: Texto (Enum: `'Pendiente'`, `'Comprado'`, `'Descartado'`).

### 10. Tabla: `recepciones_compra` (Cabecera de Recepción)
*Registro de ingresos de mercancía validados físicamente. Aquí se consolida la factura real acordada al momento.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`proveedor_id`**: Entero, **Llave Foránea** que enlaza con `proveedores(id)`.
- **`usuario_id`**: Entero, **Llave Foránea** que enlaza con `usuarios(id)` (quien recibe e ingresa al sistema).
- **`folio_factura`**: Texto (Número de factura física/PDF/computarizada ingresado, ej. "FAC-9982").
- **`fecha_recepcion`**: Fecha y Hora.
- **`total_factura`**: Decimal (Suma total real cobrada por el proveedor).

### 11. Tabla: `recepciones_compra_detalles` (Líneas de Recepción)
*Detalle contable e inventario del ingreso de mercancías. Soporta compra por paquete y almacenamiento por unidad.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`recepcion_id`**: Entero, **Llave Foránea** que enlaza con `recepciones_compra(id)`.
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)`.
- **`cantidad_paquetes`**: Decimal (Cantidad de paquetes comprados según factura, ej. 5 packs).
- **`unidades_por_paquete`**: Entero (Multiplicador de conversión registrado en `proveedor_productos`, ej. 6 unidades. Por defecto es 1).
- **`cantidad_unidades_total`**: Decimal (Stock real ingresado. Calculado: `cantidad_paquetes * unidades_por_paquete`, ej. 30 unidades).
- **`costo_paquete_real`**: Decimal (Costo del paquete cobrado por el proveedor, ej. 60 Bs).
- **`costo_unitario_real`**: Decimal (Costo real por unidad de venta. Calculado: `costo_paquete_real / unidades_por_paquete`, o en base al subtotal neto con ICE si aplica. Este valor actualiza el costo de catálogo y el Kardex).

### 12. Tabla: `documentos_digitalizados` (Historial Inmutable OCR)
*Almacenamiento permanente y de solo lectura de las facturas digitalizadas y procesadas por la API OCR.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`xml_pdf_path`**: Texto (Ruta física del archivo PDF/XML o imagen guardado de forma permanente).
- **`fecha_subida`**: Fecha y Hora.
- **`proveedor_nit_detectado`**: Texto (NIT detectado por el motor OCR, si aplica).
- **`nro_factura_detectado`**: Texto (Número de factura detectado, si aplica).
- **`total_detectado`**: Decimal.
- **`datos_extraidos_raw`**: JSON/JSONB (Almacena el JSON crudo retornado por `pdf_processor.py` exactamente como se extrajo.
  *Estructura del JSON:*
  ```json
  {
    "archivo": "factura_embol_9912.pdf",
    "datos_cabecera": {
      "Proveedor": "EMBOTELLADORAS BOLIVIANAS UNIDAS S.A. EMBOL",
      "NIT": "1020473022",
      "Nro_Factura": "991244",
      "Fecha": "24/05/2026",
      "Hora": "14:35"
    },
    "productos_extraidos": [
      {
        "CÓDIGO PRODUCTO": "0101-030750",
        "DESCRIPCIÓN": "Singani Etiqueta Azul 6x750ml",
        "CANTIDAD": 6.0,
        "PRECIO UNITARIO": 24.45,
        "DESCUENTO": 0.0,
        "ICE %": 6.38145,
        "ICE ESP.": 20.925,
        "SUBTOTAL": 174.00645
      }
    ]
  }
  ```
)
- **`estado_conciliacion`**: Texto (Enum: `'Pendiente'`, `'Conciliado'`, `'Ignorado'`).
- **`recepcion_id`**: Entero, **Llave Foránea** que enlaza con `recepciones_compra(id)` (Se llena al momento de conciliar y guardar los productos físicamente en stock).

### 13. Tabla: `sincronizacion_catalogo` (Cola de Sincronización de Catálogo)
*Bitácora intermedia que encola las diferencias detectadas durante la recepción de mercancías (nuevos precios, nuevos nombres o productos inexistentes en catálogo), permitiendo que el administrador revise y decida su aplicación definitiva en otra sesión.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`recepcion_id`**: Entero, **Llave Foránea** que enlaza con `recepciones_compra(id)` (Origen del cambio).
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)` (Nulo si el producto es nuevo).
- **`codigo_proveedor`**: Texto, Obligatorio (Código asignado por el proveedor).
- **`nombre_factura`**: Texto, Obligatorio (Nombre tal como vino en la factura/OCR).
- **`costo_unitario_factura`**: Decimal (Costo real unitario calculado en la conciliación).
- **`nombre_actual_catalogo`**: Texto, Opcional (Nombre registrado en catálogo).
- **`costo_unitario_actual_catalogo`**: Decimal, Opcional (Costo registrado en catálogo).
- **`tipo_cambio`**: Texto (Enum: `'Precio'`, `'Nombre'`, `'Ambos'`, `'Nuevo'`).
- **`estado`**: Texto (Enum: `'Pendiente'`, `'Sincronizado'`, `'Descartado'`).
- **`fecha_deteccion`**: Fecha y Hora.

---

## 📝 Apuntes y Fórmulas del Negocio

Para mantener la integridad de los costos y el inventario cuando se presentan casos complejos en las facturas (como Impuestos ICE o compras por paquetes), el sistema aplica las siguientes fórmulas en la **Capa de Conciliación** antes de guardar en la base de datos:

### 1. Fórmulas para Costos Reales con ICE (Impuesto al Consumo Específico)
El "Precio Unitario" que muestra la factura a menudo no refleja el dinero real que pagas, debido a los impuestos (ICE, ICE ESP.) y descuentos que alteran el subtotal. Para no complicar la base de datos con columnas de cada impuesto, el sistema deduce el **Costo Real** matemáticamente basándose en lo que realmente pagaste (`SUBTOTAL`):

*   **Fórmula Principal:**
    `Costo Real = SUBTOTAL / CANTIDAD FACTURADA`
*   **Ejemplo (Singani Etiqueta Azul):** 
    Si la factura indica 6 botellas con Unitario de 24.45 Bs, pero al sumar el ICE el Subtotal es 174.00 Bs:
    `Costo Real = 174.00 Bs / 6 = 29.00 Bs por botella.` *(Este es el valor exacto que se guarda en el catálogo y Kardex).*

### 2. Fórmulas para Conversión de Paquetes a Unidades
Cuando se compran productos por paquetes (ej. Six-packs, Cajas de 12) pero se venden por unidad, la cantidad facturada debe convertirse a unidades individuales para el inventario. Se utiliza la variable `unidades_por_paquete` de la tabla `proveedor_productos`:

*   **Fórmulas de Conversión:**
    `Stock a Ingresar = CANTIDAD FACTURADA * unidades_por_paquete`
    `Costo Real Unitario = SUBTOTAL / (CANTIDAD FACTURADA * unidades_por_paquete)`
*   **Ejemplo (Six-Pack Coca Cola):**
    Si compras 5 six-packs (subtotal 300 Bs) y tu `unidades_por_paquete` configurado es 6:
    `Stock a Ingresar = 5 * 6 = 30 unidades sueltas.`
    `Costo Real Unitario = 300 Bs / (5 * 6) = 10 Bs por unidad suelta.`

### 3. Flujo de Conciliación en Pantalla (Integración OCR + Inventario)
Para procesar compras dinámicas y conciliar la factura del proveedor con el inventario real:
1. **Subida/OCR:** Se digitaliza la factura del proveedor $\rightarrow$ Se genera un registro en `documentos_digitalizados` con estado `'Pendiente'` y el JSON extraído (`datos_extraidos_raw`).
2. **Interfaz de Conciliación:** El usuario abre la factura en el sistema. Se visualiza a la izquierda el detalle extraído por el OCR y a la derecha el catálogo para mapear cada ítem a un `producto_id` de la tienda.
3. **Ajuste y Resolución:** El usuario puede modificar cantidades reales (si el proveedor no entregó todo), verificar el `costo_unitario_real` recalculado con impuestos/descuentos, y asociar productos nuevos al catálogo maestro si es necesario (creando su equivalencia en `proveedor_productos`).
4. **Confirmación de Ingreso:** Al guardar la conciliación:
   - Se crea el registro oficial en `recepciones_compra` y `recepciones_compra_detalles`.
   - Se actualiza la `existencia` en `productos` (`existencia = existencia + cantidad_unidades_total`).
   - Se genera una entrada en `movimientos_inventario` (Kardex) para cada artículo con su costo unitario real.
   - El documento digitalizado se marca como `'Conciliado'`.
   - Los ítems de la `lista_compras` de ese proveedor que coincidan con la factura se marcan como `'Comprados'`.

### 14. Tabla: `auditorias_inventario` (Sesiones de Control Físico)
*Almacena la cabecera de las auditorías de inventario de fin de mes o parciales.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`fecha_inicio`**: Fecha y Hora.
- **`fecha_fin`**: Fecha y Hora (nulo si la sesión está en borrador).
- **`usuario_id`**: Entero, **Llave Foránea** que enlaza con `usuarios(id)` (auditor que realiza el control).
- **`departamento_id`**: Entero, **Llave Foránea** que enlaza con `departamentos(id)` (nulo si se audita toda la tienda).
- **`estado`**: Texto (Enum: `'Borrador'`, `'Procesada'`, `'Cancelada'`).
- **`total_diferencias_faltante`**: Decimal (Valor total en Bs. de productos faltantes).
- **`total_diferencias_sobrante`**: Decimal (Valor total en Bs. de productos sobrantes).
- **`valor_neto_ajuste`**: Decimal (Valor neto final del ajuste contable).
- **`notas`**: Texto (ej. "Auditoría Mensual - Mayo 2026").

### 15. Tabla: `auditorias_inventario_detalles` (Detalle de Toma Física)
*Artículos contados físicamente en la auditoría con sus respectivas discrepancias.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`auditoria_id`**: Entero, **Llave Foránea** que enlaza con `auditorias_inventario(id)` con eliminación en cascada.
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)`.
- **`stock_sistema`**: Decimal (Existencia teórica en el sistema al momento de iniciar la auditoría).
- **`stock_fisico`**: Decimal (Existencia física real contada por el usuario).
- **`diferencia`**: Decimal (Calculado: `stock_fisico - stock_sistema`).
- **`costo_unitario`**: Decimal (Costo unitario del producto para valorizar la diferencia).
- **`valor_diferencia`**: Decimal (Calculado: `diferencia * costo_unitario`).

### 16. Tabla: `historial_actualizaciones_productos` (Auditoría de Cambios en Catálogo)
*Bitácora inmutable (estilo "simple history" o tabla de auditoría) para rastrear cuándo se modificó o decidió no modificar (descartar) un costo, nombre o precio.*
- **`id`**: Entero, Llave Primaria (Autoincremental).
- **`producto_id`**: Entero, **Llave Foránea** que enlaza con `productos(id)` (Nulo si el producto es totalmente nuevo).
- **`codigo_producto`**: Texto (Código de barra universal o SKU).
- **`nombre_producto`**: Texto (Nombre comercial legible registrado).
- **`tipo_origen`**: Texto (ej. `'Factura OCR (FAC-9912)'`, `'Ajuste Manual'`, `'Sincronización Catálogo'`).
- **`campo_modificado`**: Texto (Enum: `'Costo Compra'`, `'Precio Venta'`, `'Nombre Producto'`, `'Nuevo Producto'`).
- **`valor_anterior`**: Texto (ej. `'Bs. 45.00'`, `'Leche Pil Entera'`).
- **`valor_nuevo`**: Texto (ej. `'Bs. 50.00'`).
- **`estado_aplicacion`**: Texto (Enum: `'Aplicado'`, `'Descartado'`).
- **`usuario_id`**: Entero, **Llave Foránea** que enlaza con `usuarios(id)`.
- **`fecha`**: Fecha y Hora exacta del registro de auditoría.

---

## 📝 Apuntes y Fórmulas del Negocio (Ajuste de Inventario)

### 4. Flujo de Auditoría e Inventario Físico (Fin de Mes)
1. **Inicio de Auditoría:** El administrador crea una sesión de auditoría (`auditorias_inventario`) con estado `'Borrador'`, seleccionando un departamento o la tienda completa. El sistema captura la foto del stock actual (`stock_sistema`) para cada producto.
2. **Conteo Físico:** El personal de la tienda cuenta físicamente los productos en los estantes y registra la cantidad real (`stock_fisico`) en el sistema.
3. **Guardar Borrador:** Se permite guardar la auditoría a medias para continuar al día siguiente sin alterar las existencias.
4. **Procesar y Ajustar:** Al finalizar la auditoría, el administrador revisa el informe de discrepancias. Al presionar "Procesar":
   - El estado de la auditoría cambia a `'Procesada'`.
   - Se actualiza la `existencia` en `productos` a la cantidad física real (`existencia = stock_fisico`).
   - Se genera un movimiento de tipo `'Ajuste'` en `movimientos_inventario` (Kardex) para cada producto que haya tenido diferencia, detallando el concepto "Ajuste por Auditoría Física" y registrando la diferencia como entrada (si sobra) o salida (si falta).

### 5. Flujo de Auditoría de Cambios de Catálogo (Historial de Actualizaciones)
1. **Detección de Cambios:** En la conciliación OCR, si los costos/nombres difieren de la base de datos, se envían a la cola de sincronización.
2. **Aplicar / Descartar:** El administrador puede aplicar o descartar los cambios en lote o individualmente.
3. **Registro de Auditoría:** Cada decisión genera un registro en `historial_actualizaciones_productos`. Si se descarta, el sistema almacena el cambio propuesto y su estado `'Descartado'`, lo que proporciona una bitácora transparente de por qué no se modificó un precio o descripción (ej. para mantener precios competitivos).

