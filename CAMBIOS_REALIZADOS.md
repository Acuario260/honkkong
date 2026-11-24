# Cambios realizados — Hong Kong Restaurant · Cevichería

## Inicio de sesión
- Eliminada la tabla «Cuentas de prueba» y todos los usuarios/contraseñas de demostración (vista, seeder, README, instrucciones y volcado SQL).
- Eliminada la voz de la pantalla de bienvenida (síntesis de voz y botón «Escuchar bienvenida»). No existe reconocimiento ni acceso por voz.
- Se mantienen los 4 roles con su selector. Nuevo fondo gastronómico (`img/fondo-gastronomia.svg`), logotipo original sin deformar con marco dorado,
  diseño adaptable a computadora, tablet y celular (`css/login.css`).
- «¿Olvidó su contraseña?» ahora envía un enlace real por correo (pantalla `/restablecer`).

## Seguridad
- Sin usuarios por defecto. Cuenta inicial: `php artisan hongkong:crear-admin` (contraseña por pantalla, hash de Laravel). Solo una cuenta inicial.
- `php artisan hongkong:cambiar-password <usuario>` para recuperar el acceso del administrador desde el servidor.
- Se rechazan contraseñas de demostración/comunes e iguales al usuario. Recuperación: token de un solo uso, 30 min, límite de intentos, respuesta
  idéntica exista o no el correo, cierra todas las sesiones al completarse.
- Límite de peticiones en inicio de sesión y recuperación. Cuentas eliminadas/desactivadas no entran ni conservan sesión.
- Permisos validados en el servidor en cada ruta (middleware `rol`): usuarios, configuración y productos = solo administrador.

## Usuarios (solo administrador)
- Crear Mesero/Cocina/Caja, editar, cambiar rol, restablecer contraseña (con generador), activar/desactivar, eliminar y listar.
- No se elimina la propia cuenta ni la última de administrador; no se puede desactivar/degradar la última. Un trabajador no se asciende a administrador.
- Eliminar conserva el historial (pedidos, pagos, arqueos) mediante baja lógica (`users.eliminado_en`). Altas/ediciones/bajas quedan en `auditorias`.

## Cancelación individual
- Botón **Cancelar** por pedido y **Cancelar plato** por renglón, con confirmación y motivo obligatorios.
- Solo se cancela ese pedido/plato; otras mesas y pedidos no se tocan. La mesa se libera solo si no tiene otro pedido abierto.
- Se registra quién, cuándo (`pedidos.anulado_en`, nueva) y por qué. Estado visible: **CANCELADO**. El historial y los platos se conservan.
- Bloqueo contra doble cancelación (transacción con bloqueo de fila, respuesta 409).
- Mesero: solo sus pedidos y dentro del plazo configurado; administrador: cualquiera; Cocina y Caja: 403.
- Si se cancela el último plato activo de un pedido sin pagos, ese pedido queda cancelado.

## Base de datos
- Nueva migración `2026_11_23_000001_add_cancelacion_y_eliminacion` (2 columnas; no borra datos, rellena `anulado_en` de pedidos ya cancelados).
- `docs/bd/hong_kong.sql` actualizado (sin usuarios, con las columnas nuevas y la migración registrada).

## Archivos nuevos
`app/Console/Commands/{CrearAdmin,CambiarPassword}.php`, `app/Notifications/RestablecerContrasena.php`, `app/Support/CuentaInicial.php`,
`config/hongkong.php`, `resources/views/auth/{login,restablecer}.blade.php` (la vista antigua `administrador/inde.blade.php` se renombró),
`public/js/restablecer.js`, `public/css/login.css`, `public/img/fondo-gastronomia.svg`, `tests/Feature/SistemaHongKongTest.php`.

## Qué NO se pudo comprobar en este entorno
No había PHP, Composer ni navegador disponibles, así que **no se ejecutó Laravel ni se hicieron capturas**. Se verificó: sintaxis de todo el JavaScript
(`node --check`), balance de llaves/paréntesis en los PHP, estructura HTML y que los IDs usados por los scripts existan. Las pruebas de
`tests/Feature/SistemaHongKongTest.php` están escritas pero sin ejecutar: corre `php artisan test` al instalar.

## Pendiente / a tener en cuenta
- La carpeta `prototipo/` (maqueta estática original con datos de ejemplo propios) no forma parte del sistema y no se modificó; no la publiques en producción.
- Revisión de textos: la interfaz ya estaba en español; no se encontraron textos visibles en inglés salvo el nombre de marca «Restaurant» del logotipo.


---
# Segunda entrega: pedidos por modalidad, caja, pasarela, ticket y cocina

Migración nueva: `2026_11_24_000001_add_estado_pago_pasarela_y_ticket` (`pedidos.estado_pago`; `pagos.origen/pasarela/transaccion_id` con índice único;
`pagos.arqueo_id` y `user_id` opcionales; serie de comprobantes B001 → T001). Ejecuta `php artisan migrate`.

- **Pedidos** (`pedidos.html/js`, `ComandaController`): campos y columnas condicionales por modalidad, limpieza de residuos al cambiar de tipo,
  cliente / teléfono / dirección, costo de envío editable, validación de platos > S/ 0.00, indicaciones por plato separadas de la observación,
  recálculo al cambiar turno/plato, anti doble clic. `mesa_id` es opcional fuera de salón.
- **Caja** (`caja.html/js`, `CajaController`, `ReporteController`, `Restaurante`): estados Pendiente / Pago parcial / Pagado persistentes y filtros,
  efectivo con recibido/vuelto solo en efectivo, «Por consumo» oculto cuando no aplica, mesa liberada al pagar o cancelar, arqueo y reportes sin cancelados
  y con el vuelto restado, aviso de caja cerrada sin perder lo elegido.
- **Pasarela:** `Pasarela.php` + `POST /api/pasarela/confirmacion` (firma HMAC, idempotencia, validación de estado/monto/moneda), alerta visible con sonido en caja.
- **Ticket:** «TICKET DE VENTA», serie `T001`, sin «Atendido por» ni «Documento interno…», cabecera de delivery, envío aparte, pagos detallados, saldo solo si es parcial.
- **Cocina / errores:** indicaciones de cada plato resaltadas, etiqueta de modalidad sin mesa fuera de salón; avisos claros de sesión vencida, sin conexión y error de servidor (`app.js`).
- **Tests nuevos:** `tests/Feature/CajaPedidosTest.php`.

Tampoco se ejecutó esta entrega (sin PHP en el entorno): se verificó sintaxis JS y balance de PHP. Corre `php artisan test`.
