# Documentación de la Base de Datos: LTSolution

## 1. Introducción

- Propósito: gestionar integralmente operaciones de una empresa de transporte de maquinaria pesada, abarcando flota, viajes, cotizaciones, facturación/pagos, empleados, incidencias, documentos y notificaciones.
- Alcance: tablas operativas, catálogos generales (TiposMaestros), auditable (historial de estados), y datos geográficos (Ubigeo).

## 2. Convenciones y Campos Comunes

- Identificadores primarios: `...ID` enteros (PK clustered) por tabla.
- Auditoría: `FechaCreacion`, `FechaModificacion`, `CreadoPor`, `ModificadoPor` presentes en la mayoría de tablas; `CreadoPor` y `ModificadoPor` referencian `Usuarios`.
- Monedas y estados: se gestionan mediante `TiposMaestros` con clasificación por `Seccion`.
- Fechas: uso de `date` para fechas y `datetime` para instantes; valores por defecto con `getdate()` en creación.
- Integridad: claves foráneas (FK) agregadas con `ALTER TABLE ... ADD FOREIGN KEY`; validaciones con `CHECK` en montos, fechas y formatos.

## 3. Catálogo TiposMaestros

- Tabla: `dbo.TiposMaestros (TipoID PK, Nombre, Codigo, Seccion, Descripcion, Activo, Orden, FechaCreacion, FechaModificacion, CreadoPor, ModificadoPor)`.
- Secciones registradas (ejemplos):
  - Identificación persona: `TIPODNI` (DNI, CE, Pasaporte)
  - Documento persona: `TIPO_DOCUMENTO` (DNI, Licencia, SCTR, Pasaporte, CEX)
  - Documento vehículo: `TIPO_DOCUMENTO_VEHICULO` (Tarjeta Propiedad, Revisión Técnica, SOAT, Póliza, Permiso Operación)
  - Estados: `ESTADO_EMPLEADO`, `ESTADO_VEHICULO`, `ESTADO_VIAJE`, `ESTADO_FACTURA`, `ESTADO_SERVICIO`, `ESTADO_INCIDENTE`, `ESTADO_COTIZACION`, `ESTADO_USUARIO`, `ESTADO_TRABAJO`
  - Monedas: `MONEDA` (PEN, USD, EUR)
  - Notificaciones: `TIPO_NOTIFICACION`, `PRIORIDAD_NOTIFICACION`
  - Pagos: `METODO_PAGO`, `PERIODO_PAGO`
  - Viaje: `TIPO_PARADA`, `TIPO_GASTO`
  - Mantenimiento: `MOTIVO_SERVICIO`, `TIPO_SERVICIO`, `TIPO_TRABAJO`, `UNIDAD_MEDIDA`
  - Incidentes: `TIPO_INCIDENTE`, `GRAVEDAD_INCIDENTE`
- Uso en el esquema (referencias típicas):
  - `Empleados.TipoDocumentoID` → `TIPODNI`
  - `Empleados.EstadoID` → `ESTADO_EMPLEADO`
  - `Empleados.MonedaSalarioID` → `MONEDA`
  - `Usuarios.EstadoUsuarioID` → `ESTADO_USUARIO`
  - `Facturas.MonedaID` → `MONEDA`; `Facturas.EstadoID` → `ESTADO_FACTURA`
  - `Pagos.MonedaPagoID` → `MONEDA`; `Pagos.MetodoPagoID` → `METODO_PAGO`
  - `Cotizaciones.MonedaID` → `MONEDA`; `Cotizaciones.EstadoID` → `ESTADO_COTIZACION`
  - `DetallesCargaViaje.MonedaValorID` → `MONEDA`
  - `CotizacionDetalles.MonedaPrecioID` → `MONEDA`; `CotizacionDetalles.TipoVehiculoID` → `TiposVehiculo`
  - `Flota.EstadoID` → `ESTADO_VEHICULO`; `Flota.TipoID` → `TiposVehiculo`
  - `Viajes.EstadoID` → `ESTADO_VIAJE`
  - `Incidentes.EstadoID` → `ESTADO_INCIDENTE`; `Incidentes.GravedadID` → `GRAVEDAD_INCIDENTE`; `Incidentes.TipoIncidenteID` → `TIPO_INCIDENTE`; `Incidentes.MonedaCostoID` → `MONEDA`
  - `GastosViaje.TipoGastoID` → `TIPO_GASTO`; `GastosViaje.MonedaGastoID` → `MONEDA`
  - `Notificaciones.TipoNotificacionID` → `TIPO_NOTIFICACION`; `Notificaciones.PrioridadID` → `PRIORIDAD_NOTIFICACION`
  - `ProveedoresTalleres.TipoProveedorID` → `TIPOPROVEEDOR`; `ProveedoresTalleres.EstadoID` → `ESTADO_PROVEEDOR`
  - `MantenimientoReparaciones.MotivoID` → `MOTIVO_SERVICIO`; `MantenimientoReparaciones.TipoServicioID` → `TIPO_SERVICIO`; `MantenimientoReparaciones.EstadoID` → `ESTADO_SERVICIO`; `MantenimientoReparaciones.MonedaCostoID` → `MONEDA`
  - `DocumentosEmpleado.TipoDocumentoID` → `TIPO_DOCUMENTO`; `DocumentosVehiculo.TipoDocumentoVehiculoID` → `TIPO_DOCUMENTO_VEHICULO`

## 4. Diccionario de Datos (por tabla)

### 4.1 Empleados

- Propósito: información del personal, estado y datos de contratación.
- Clave: `EmpleadoID` (PK).
- Campos clave: `Nombres`, `Apellidos`, `TipoDocumentoID` (FK `TIPODNI`), `NumeroDocumentoIdentidad`, `Email`, `Telefono`, `FechaContratacion`, `EstadoID` (FK `ESTADO_EMPLEADO`), `MonedaSalarioID` (FK `MONEDA`), `Salario`.
- Auditoría: `FechaCreacion` (DEFAULT `getdate()`), `FechaModificacion`, `CreadoPor` (FK `Usuarios`), `ModificadoPor` (FK `Usuarios`).
- Constraints: `CHK_Empleados_Email` (`Email LIKE '%@%.%'`), `CHK_Empleados_Salario` (`Salario >= 0`).
- Consultas típicas: búsqueda por `NumeroDocumentoIdentidad`, listado por `EstadoID`, aniversarios de contratación.

### 4.2 Usuarios

- Propósito: autenticación/autorización de acceso.
- Clave: `UsuarioID` (PK).
- Campos clave: `EmpleadoID` (FK), `NombreUsuario`, `Email`, `ContrasenaHash`, `RolID` (FK `Roles`), `EstadoUsuarioID` (FK `ESTADO_USUARIO`).
- Seguridad: `IntentosLogin` (DEFAULT 0), `FechaBloqueo`, `UltimoAcceso`, `TokenRecuperacion`, `FechaExpiracionToken`, `RequiereCambioPassword` (DEFAULT 0).
- Auditoría: `FechaCreacion` (DEFAULT), `FechaModificacion`, `CreadoPor`/`ModificadoPor` (FK a `Usuarios`).
- Constraints: `CHK_Usuarios_Email` (`Email LIKE '%@%.%'`).
- Consultas: usuarios activos, bloqueados, últimos accesos.

### 4.3 Roles

- Propósito: catálogo de roles.
- Clave: `RolID` (PK). Campos: `NombreRol`, `Descripcion`.
- Uso: `Usuarios.RolID`.

### 4.4 Clientes

- Propósito: datos comerciales de clientes.
- Clave: `ClienteID` (PK).
- Campos: `NombreCliente`, `RUC`, `Direccion`, `PersonaContacto`, `EmailContacto`, `TelefonoContacto`.
- Sugerido: `UNIQUE (RUC)` y `UNIQUE (EmailContacto)`.

### 4.5 TiposVehiculo

- Propósito: catálogo de tipos de vehículos.
- Clave: `TipoID` (PK). Campos: `NombreTipo`, `Descripcion`.
- Uso: `Flota.TipoID`, `CotizacionDetalles.TipoVehiculoID`.

### 4.6 Flota

- Propósito: vehículos de la empresa.
- Clave: `VehiculoID` (PK).
- Campos: `TipoID` (FK `TiposVehiculo`), `Placa`, `Marca`, `Modelo`, `Ano`, `Ejes`, `Capacidad`, `EstadoID` (FK `ESTADO_VEHICULO`), `KilometrajeActual`, `FechaUltimaRevision`, `ProximoMantenimiento`.
- Auditoría: `FechaCreacion` (DEFAULT), `FechaModificacion`, `CreadoPor`, `ModificadoPor`.
- Constraints: `CHK_Flota_CapacidadCarga` (`Capacidad > 0`), `CHK_Flota_Placa` (`LEN(Placa) >= 6`).
- Sugerido: `UNIQUE (Placa)`.

### 4.7 Cotizaciones

- Propósito: propuestas comerciales y condiciones.
- Clave: `CotizacionID` (PK).
- Campos: `ClienteID` (FK), `FechaCotizacion`, `Origen`, `Destino`, `DescripcionGeneral`, `EstadoID` (FK `ESTADO_COTIZACION`), `MontoTotal`, `MonedaID` (FK `MONEDA`), `FechaVencimiento`.
- Auditoría: `FechaCreacion` (DEFAULT), `FechaModificacion`, `CreadoPor`, `ModificadoPor`.
- Sugerido: `CHECK (FechaCotizacion <= FechaVencimiento)`.

### 4.8 CotizacionDetalles

- Propósito: items/servicios dentro de una cotización.
- Clave: `DetalleID` (PK).
- Campos: `CotizacionID` (FK), `TipoVehiculoID` (FK `TiposVehiculo`), `CantidadVehiculos`, `DescripcionCarga`, dimensiones/peso, `PrecioUnitario`, `MonedaPrecioID` (FK `MONEDA`), `Subtotal`.

### 4.9 Viajes

- Propósito: ejecución de transporte.
- Clave: `ViajeID` (PK).
- Campos: `CotizacionID` (FK), `ConductorID` (FK `Empleados`), `FechaInicio`, `FechaFin`, `KilometrajeInicial/Final`, `EstadoID` (FK `ESTADO_VIAJE`), pesos/dimensiones totales, origen/destino (`UbigeoOrigenID`/`UbigeoDestinoID`), direcciones.
- Auditoría: `FechaCreacion` (DEFAULT), `FechaModificacion`, `CreadoPor`, `ModificadoPor`.
- Constraints: `CHK_Viajes_Fechas` (`FechaInicio <= FechaFin OR FechaFin IS NULL`).

### 4.10 VehiculosViaje

- Propósito: asignación de vehículos a viajes.
- Clave: `VehiculoViajeID` (PK).
- Campos: `ViajeID` (FK), `VehiculoID` (FK).

### 4.11 ViajePermisos

- Propósito: permisos asociados al viaje.
- Clave: `ViajePermisoID` (PK).
- Campos: `ViajeID` (FK), `FechaEmision`, `FechaVencimiento`, `URLDocumento`.

### 4.12 DetallesCargaViaje

- Propósito: detalle de mercadería transportada.
- Clave: `DetalleCargaID` (PK).
- Campos: `ViajeID` (FK), descripción, dimensiones/peso, `CantidadPiezas` (DEFAULT 1), `ValorDeclarado` (CHECK `>= 0`), `MonedaValorID` (FK `MONEDA`), `RequiereCuidadoEspecial` (DEFAULT 0), `Observaciones`, `FechaCreacion` (DEFAULT), `CreadoPor`.
- Constraints: `CHK_DetallesCarga_Peso`, `CHK_DetallesCarga_Valor`.

### 4.13 ParadasDescanso

- Propósito: registro de paradas durante el viaje.
- Clave: `ParadaID` (PK).
- Campos: `ViajeID` (FK), `Ubicacion`, `Latitud/Longitud`, `Notas`, `FechaCreacion` (DEFAULT), `CreadoPor`.

### 4.14 GuiasRemision

- Propósito: guías asociadas a viajes.
- Clave: `GuiaID` (PK).
- Campos: `ViajeID` (FK), `NumeroGuia`, `FechaEmision`, `URLDocumento`.
- Sugerido: `UNIQUE (NumeroGuia)`.

### 4.15 Incidentes

- Propósito: eventos/accidentes durante el viaje.
- Clave: `IncidenteID` (PK).
- Campos: `ViajeID` (FK), `TipoIncidenteID` (FK), `Descripcion`, `FechaHora`, `Ubicacion`, `Latitud/Longitud`, `GravedadID` (FK), `EstadoID` (FK), `CostoEstimado`, `MonedaCostoID` (FK), `FechaResolucion`, `ResueltoPor` (FK `Usuarios`), `FechaCreacion` (DEFAULT), `CreadoPor`.
- Constraints: ninguna adicional a nivel de costo; recomendaciones en §7.

### 4.16 HistorialEstados

- Propósito: auditoría de cambios de estado.
- Clave: `HistorialID` (PK).
- Campos: `TipoEntidad` (texto), `EntidadID`, `EstadoAnterior`, `EstadoNuevo`, `FechaCambio` (DEFAULT), `CambiadoPor` (FK `Usuarios`), `Motivo`.
- Nota: `TipoEntidad` textual facilita flexibilidad; usar catálogo opcional.

### 4.17 MantenimientoReparaciones

- Propósito: servicios y reparaciones de vehículos.
- Clave: `RegistroID` (PK).
- Campos: `VehiculoID` (FK), `ProveedorID` (FK), `FechaServicio`, `MotivoID` (FK), `TipoServicioID` (FK), `Descripcion`, `KilometrajeServicio`, `Costo` (CHECK `>= 0`), `MonedaCostoID` (FK), `RealizadoPor`, `EstadoID` (FK `ESTADO_SERVICIO`), `ProximoServicio`, `Observaciones`, `FechaCreacion` (DEFAULT), `CreadoPor`.
- Constraints: `CHK_Mantenimiento_Costo`.

### 4.18 ProveedoresTalleres

- Propósito: talleres y proveedores.
- Clave: `ProveedorID` (PK).
- Campos: `NombreTaller`, `RUC`, `Direccion`, ubicación administrativa, contactos, `Especialidad`, `TipoProveedorID` (FK), `CalificacionServicio`, `EstadoID` (FK), `HorarioAtencion`, `ServicioEmergencia` (DEFAULT 0), `Observaciones`, `FechaCreacion` (DEFAULT), `FechaModificacion`, `CreadoPor`, `ModificadoPor`.
- Sugerido: `UNIQUE (RUC)`.

### 4.19 DocumentosEmpleado

- Propósito: documentos del personal.
- Clave: `DocumentoID` (PK).
- Campos: `EmpleadoID` (FK), `TipoDocumentoID` (FK `TIPO_DOCUMENTO`), `NumeroDocumento`, `FechaEmision`, `FechaVencimiento`, `URLDocumento`.
- Sugerido: `CHECK (FechaEmision <= FechaVencimiento)`, `UNIQUE (EmpleadoID, TipoDocumentoID, NumeroDocumento)`.

### 4.20 DocumentosVehiculo

- Propósito: documentos del vehículo.
- Clave: `DocumentoVehiculoID` (PK).
- Campos: `VehiculoID` (FK), `TipoDocumentoVehiculoID` (FK), `NumeroDocumento`, `FechaEmision`, `FechaVencimiento`, `URLDocumento`.
- Sugerido: `CHECK (FechaEmision <= FechaVencimiento)`, `UNIQUE (VehiculoID, TipoDocumentoVehiculoID, NumeroDocumento)`.

### 4.21 Facturas

- Propósito: facturación a clientes.
- Clave: `FacturaID` (PK).
- Campos: `ClienteID` (FK), `NumeroFactura`, `FechaEmision`, `FechaVencimiento`, `MontoTotal` (CHECK `>= 0`), `MonedaID` (FK), `EstadoID` (FK), `FechaCreacion` (DEFAULT), `FechaModificacion`, `CreadoPor`, `ModificadoPor`.
- Constraints: `CHK_Facturas_MontoTotal`.
- Sugerido: `UNIQUE (NumeroFactura)`.

### 4.22 FacturaDetalles

- Propósito: vinculación de viajes facturados.
- Clave: `FacturaDetalleID` (PK).
- Campos: `FacturaID` (FK), `ViajeID` (FK).

### 4.23 Pagos

- Propósito: pagos de facturas.
- Clave: `PagoID` (PK).
- Campos: `FacturaID` (FK), `FechaPago`, `MontoPagado` (CHECK `>= 0`), `MonedaPagoID` (FK), `TipoCambio`, `MetodoPagoID` (FK), `NumeroReferencia`, `FechaCreacion` (DEFAULT), `CreadoPor`.
- Constraints: `CHK_Pagos_Monto`.
- Nota: considerar tabla histórica de tipos de cambio.

### 4.24 PagosEmpleados

- Propósito: liquidaciones del personal.
- Clave: `PagoEmpleadoID` (PK).
- Campos: `EmpleadoID` (FK), período (`FechaInicioPeriodo`, `FechaFinPeriodo`), `FechaPago`, `SalarioBase`, `Bonificaciones` (DEFAULT 0), `Descuentos` (DEFAULT 0), `MontoTotal` (CHECK `>= 0`), `MonedaPagoID` (FK), `Observaciones`.
- Sugerido: `CHECK (FechaInicioPeriodo <= FechaFinPeriodo)`.

### 4.25 GastosViaje

- Propósito: gastos operativos por viaje.
- Clave: `GastoID` (PK).
- Campos: `ViajeID` (FK), `TipoGastoID` (FK), `Monto` (CHECK `>= 0`), `MonedaGastoID` (FK), `FechaGasto`, `Descripcion`, `URLRecibo`.
- Constraints: `CHK_GastosViaje_Monto`.

### 4.26 Notificaciones

- Propósito: alertas y comunicaciones internas.
- Clave: `NotificacionID` (PK).
- Campos: `EmpleadoID` (FK), `Titulo`, `Mensaje`, `TipoNotificacionID` (FK), `PrioridadID` (FK), `Leida` (DEFAULT 0), `FechaCreacion` (DEFAULT), `FechaLectura`, `URLReferencia`.

### 4.27 Historial/Estados y Auditoría

- `HistorialEstados` documentada en §4.16.

### 4.28 Ubigeo

- Propósito: catálogo geográfico (departamento/provincia/distrito y coordenadas).
- Clave: `UbigeoID` (PK).
- Campos: `UbigeoInei`, `DepartamentoInei`, `Departamento`, `ProvinciaInei`, `Provincia`, `Distrito`, `Region`, `Superficie`, `Altitud`, `Latitud`, `Longitud`, `Frontera`.
- Uso: `Viajes.UbigeoOrigenID`, `Viajes.UbigeoDestinoID`.

### 4.29 Configuraciones

- Propósito: parámetros del sistema.
- Clave: `ConfiguracionID` (PK).
- Campos: `Clave`, `Valor`, `Descripcion`, `TipoDato` (DEFAULT 'String'), `Categoria` (DEFAULT 'General'), `ModificadoPor` (FK), `FechaModificacion` (DEFAULT).
- Claves iniciales (ejemplos): `MONEDA_DEFAULT`, `TIPO_CAMBIO_USD_PEN`, `TIPO_CAMBIO_EUR_PEN`, `DIAS_VENCIMIENTO_COTIZACION`, `NOTIFICAR_MANTENIMIENTO_DIAS`, `NOTIFICAR_VENCIMIENTO_DOCUMENTOS`, `EMPRESA_*`, `BACKUP_AUTOMATICO`, `HORAS_MAXIMAS_CONDUCCION`.

### 4.30 AsistenciasPermisos

- Propósito: permisos/asistencias de empleados.
- Clave: `RegistroID` (PK).
- Campos: `EmpleadoID` (FK), `FechaSolicitud` (DEFAULT), período (`FechaInicio`, `FechaFin`), `TipoRegistroID` (FK `TIPO_PERMISO`), `MotivoDetallado`, `DocumentoSoporte`, `EstadoID` (FK `ESTADO_PERMISO`), aprobación (`AprobadoPor` FK, `FechaAprobacion`, `ComentarioAprobacion`), `CreadoPor`, `Observaciones`.

## 5. Claves Foráneas (panorama)

- Relaciones principales confirmadas vía `ALTER TABLE ... FOREIGN KEY` entre: `Viajes`↔`Cotizaciones/Empleados/Ubigeo/TiposMaestros/Usuarios`, `Facturas`↔`Clientes/TiposMaestros/Usuarios`, `Pagos`↔`Facturas/TiposMaestros/Usuarios`, `Flota`↔`TiposVehiculo/TiposMaestros/Usuarios`, `Incidentes`↔`Viajes/TiposMaestros/Usuarios`, `MantenimientoReparaciones`↔`Flota/TiposMaestros/ProveedoresTalleres/Usuarios`, `Notificaciones`↔`Empleados/TiposMaestros`, `GastosViaje`↔`Viajes/TiposMaestros`, `Documentos*`↔`Empleados/Flota/TiposMaestros`, `Usuarios`↔`Empleados/Roles/TiposMaestros/Usuarios`, `CotizacionDetalles`↔`Cotizaciones/TiposVehiculo/TiposMaestros`.

## 6. Defaults y Constraints (existentes)

- Defaults (`getdate()` en creación): `Cotizaciones`, `DetallesCargaViaje`, `Empleados`, `Facturas`, `Flota`, `HistorialEstados`, `Incidentes`, `MantenimientoReparaciones`, `Notificaciones`, `Pagos`, `ParadasDescanso`, `ProveedoresTalleres`, `TiposMaestros`, `Usuarios`, `Viajes`.
- Otros defaults: `CantidadPiezas=1`, `RequiereCuidadoEspecial=0`, `Leida=0`, `IntentosLogin=0`, `RequiereCambioPassword=0`, `ServicioEmergencia=0`; `Configuraciones.TipoDato='String'`, `Configuraciones.Categoria='General'`.
- Checks:
  - `CHK_DetallesCarga_Peso` (`PesoMercaderia >= 0`)
  - `CHK_DetallesCarga_Valor` (`ValorDeclarado >= 0`)
  - `CHK_Empleados_Email` (`Email LIKE '%@%.%'`)
  - `CHK_Empleados_Salario` (`Salario >= 0`)
  - `CHK_Facturas_MontoTotal` (`MontoTotal >= 0`)
  - `CHK_Flota_CapacidadCarga` (`Capacidad > 0`)
  - `CHK_Flota_Placa` (`LEN(Placa) >= 6`)
  - `CHK_GastosViaje_Monto` (`Monto >= 0`)
  - `CHK_Mantenimiento_Costo` (`Costo >= 0`)
  - `CHK_Pagos_Monto` (`MontoPagado >= 0`)
  - `CHK_PagosEmpleados_Monto` (`MontoTotal >= 0`)
  - `CHK_Usuarios_Email` (`Email LIKE '%@%.%'`)
  - `CHK_Viajes_Fechas` (`FechaInicio <= FechaFin OR FechaFin IS NULL`)

## 7. Índices (estado y recomendaciones)

- Estado actual: sin índices no-cluster explícitos (aparte de PKs clustered).
- Recomendaciones:
  - `Viajes`: `IX_Viajes_Estado_FechaInicio (EstadoID, FechaInicio)`, `IX_Viajes_Conductor (ConductorID)`, `IX_Viajes_Cotizacion (CotizacionID)`.
  - `Facturas`: `IX_Facturas_Cliente_Estado (ClienteID, EstadoID)`, `IX_Facturas_FechaVencimiento (FechaVencimiento)`; `UNIQUE (NumeroFactura)`.
  - `Pagos`: `IX_Pagos_Factura_Fecha (FacturaID, FechaPago)`.
  - `GastosViaje`: `IX_Gastos_Viaje_Tipo (ViajeID, TipoGastoID)`.
  - `DocumentosVehiculo`: `IX_DocVehiculo_Vehiculo_Tipo_Vence (VehiculoID, TipoDocumentoVehiculoID, FechaVencimiento)`; `UNIQUE (VehiculoID, TipoDocumentoVehiculoID, NumeroDocumento)`.
  - `DocumentosEmpleado`: `IX_DocEmpleado_Empleado_Tipo_Vence`; `UNIQUE (EmpleadoID, TipoDocumentoID, NumeroDocumento)`.
  - `Notificaciones`: `IX_Notificaciones_Empleado_Leida_Tipo (EmpleadoID, Leida, TipoNotificacionID)`.
  - `Usuarios`: `UNIQUE (Email)`, `UNIQUE (NombreUsuario)`.
  - `Clientes`: `UNIQUE (RUC)`.
  - `Flota`: `UNIQUE (Placa)`.
  - `GuiasRemision`: `UNIQUE (NumeroGuia)`.
  - `Configuraciones`: `UNIQUE (Clave)`.

## 8. Validación por Sección (con TiposMaestros)

- Modelo elegido: usar `TiposMaestros.Seccion` para categorizar; mantenerlo así.
- Auditorías recomendadas (patrón):
  - Empleados (TipoDocumentoID):
    ```sql
    SELECT e.EmpleadoID, e.TipoDocumentoID, tm.Seccion
    FROM dbo.Empleados e
    JOIN dbo.TiposMaestros tm ON tm.TipoID = e.TipoDocumentoID
    WHERE tm.Seccion <> 'TIPODNI';
    ```
  - Facturas (Moneda y Estado):
    ```sql
    SELECT f.FacturaID, tmMon.Seccion AS SeccionMoneda, tmEst.Seccion AS SeccionEstado
    FROM dbo.Facturas f
    JOIN dbo.TiposMaestros tmMon ON tmMon.TipoID = f.MonedaID
    JOIN dbo.TiposMaestros tmEst ON tmEst.TipoID = f.EstadoID
    WHERE tmMon.Seccion <> 'MONEDA' OR tmEst.Seccion <> 'ESTADO_FACTURA';
    ```
  - Repetir según las referencias listadas en §3.
- Alternativa (no aplicada): FKs compuestos `(TipoID, Seccion)` con columna de sección “sombra” por tabla.

## 9. Consultas Operativas Útiles

- Documentos por vencer (vehículos):
  ```sql
  SELECT v.VehiculoID, dv.TipoDocumentoVehiculoID, dv.NumeroDocumento, dv.FechaVencimiento
  FROM dbo.DocumentosVehiculo dv
  JOIN dbo.Flota v ON v.VehiculoID = dv.VehiculoID
  WHERE dv.FechaVencimiento IS NOT NULL
    AND dv.FechaVencimiento <= DATEADD(DAY, CAST((SELECT Valor FROM dbo.Configuraciones WHERE Clave='NOTIFICAR_VENCIMIENTO_DOCUMENTOS') AS int), GETDATE());
  ```
- Próximos mantenimientos:
  ```sql
  SELECT f.VehiculoID, f.ProximoMantenimiento
  FROM dbo.Flota f
  WHERE f.ProximoMantenimiento IS NOT NULL
    AND f.ProximoMantenimiento <= DATEADD(DAY, CAST((SELECT Valor FROM dbo.Configuraciones WHERE Clave='NOTIFICAR_MANTENIMIENTO_DIAS') AS int), GETDATE());
  ```
- Facturación pendiente y vencida:
  ```sql
  SELECT f.FacturaID, c.NombreCliente, f.MontoTotal, f.FechaVencimiento, tm.Nombre AS Estado
  FROM dbo.Facturas f
  JOIN dbo.Clientes c ON c.ClienteID = f.ClienteID
  JOIN dbo.TiposMaestros tm ON tm.TipoID = f.EstadoID
  WHERE tm.Seccion='ESTADO_FACTURA' AND tm.Codigo IN ('NPA','VEN');
  ```
- Viajes por estado y periodo:
  ```sql
  SELECT v.ViajeID, tm.Nombre AS Estado, v.FechaInicio, v.ConductorID
  FROM dbo.Viajes v
  JOIN dbo.TiposMaestros tm ON tm.TipoID = v.EstadoID
  WHERE tm.Seccion='ESTADO_VIAJE' AND v.FechaInicio BETWEEN @Inicio AND @Fin;
  ```

## 10. Flujos de Negocio

- Comercial: `Clientes` → `Cotizaciones` → `Viajes` → `GuiasRemision` → `Facturas` → `Pagos`.
- Operación: `Viajes` ↔ `VehiculosViaje` ↔ `Flota`; carga en `DetallesCargaViaje`; paradas en `ParadasDescanso`; documentos en `DocumentosVehiculo`/`DocumentosEmpleado`.
- Soporte: `MantenimientoReparaciones` con `ProveedoresTalleres`; incidencias en `Incidentes`.
- Gobierno de datos: `HistorialEstados` y `Notificaciones`; parámetros en `Configuraciones`.

## 11. Mantenimiento y Retención

- Retención: considerar archivado/partición por fecha en `Notificaciones`, `HistorialEstados`, `Incidentes`, `Pagos` y `GastosViaje`.
- Búsqueda: índices de texto completo opcionales en campos `nvarchar(max)` si se requiere.
- Tipo de cambio: crear una tabla `TipoCambioHistorico` si se usan conversiones frecuentes.

## 12. Seguridad y Cumplimiento

- Usuarios: reforzar con `UNIQUE (Email)`, `UNIQUE (NombreUsuario)`, opcional `PasswordSalt`, `PasswordUpdatedAt`, `EmailVerifiedAt`.
- Auditoría: garantizar que `CreadoPor`/`ModificadoPor` se establecen desde la capa de aplicación.
- Accesos: vistas de administración para cuentas bloqueadas o con intentos altos.

## 13. Calidad de Datos (sugerencias no disruptivas)

- Unicidad natural: `Clientes(RUC)`, `Flota(Placa)`, `Facturas(NumeroFactura)`, `GuiasRemision(NumeroGuia)`, `Usuarios(Email, NombreUsuario)`, `Configuraciones(Clave)`.
- Validaciones de fechas: `Documentos* (FechaEmision <= FechaVencimiento)`, `Cotizaciones (FechaCotizacion <= FechaVencimiento)`, `PagosEmpleados (Inicio <= Fin)`.
- Auditorías periódicas de sección (ver §8).

## 14. Glosario rápido

- `Seccion`: categoría dentro de `TiposMaestros` que clasifica el significado de cada `TipoID`.
- `EstadoID`: referencia al estado correspondiente según el contexto (viaje, factura, empleado, etc.).
- `Moneda*ID`: referencias a `MONEDA` para precios, costos y pagos.
- `Ubigeo`: catálogo geográfico oficial usado para origen/destino.

---

Documento generado para apoyar la comprensión, mantenimiento y evolución de LogisticaDB. Mantener este archivo sincronizado ante cambios del esquema.
