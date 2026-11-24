# Sistema Restaurante Hong Kong (Laravel 13 + MySQL)

Sistema web para el restaurante: 20 mesas, pedidos en salón, para llevar y delivery, monitor de cocina con cronómetros, cancelación en los primeros 7 minutos, cobro con división de cuenta, caja con arqueo y reportes.

- **Reglas del negocio:** [`../docs/bd/REGLAS_Y_FLUJOS.md`](../docs/bd/REGLAS_Y_FLUJOS.md)
- **Diagrama de la base de datos:** [`../docs/bd/diagrama_bd.png`](../docs/bd/diagrama_bd.png)
- **Script MySQL:** [`../docs/bd/hong_kong.sql`](../docs/bd/hong_kong.sql)

## Instalar en Laragon

1. Descomprime `HONG_KONG_Sistema_Completo_Estructura_Laravel.zip` y copia la carpeta `HONG_KONG_RESTAURANTE` en `C:\laragon\www\`.
2. Abre Laragon y pulsa **Start All** (Apache + MySQL).
3. Pulsa **Terminal** y ejecuta una línea a la vez:

```
cd C:\laragon\www\HONG_KONG_RESTAURANTE\sistema_hong_kong
composer install
copy .env.example .env
php artisan key:generate
mysql -u root -e "CREATE DATABASE IF NOT EXISTS hong_kong CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"
php artisan migrate --seed
php artisan serve
```

4. Abre **http://127.0.0.1:8000** en el navegador.

Para usarlo desde celulares o tablets de la misma red WiFi, inicia el servidor con
`php artisan serve --host=0.0.0.0` y entra desde cada equipo con la IP de la PC, por ejemplo `http://192.168.1.20:8000`.

> **Fuera de horario:** el restaurante atiende de 06:00 a 19:00, y fuera de ese horario no se aceptan pedidos.
> Para probar de noche, entra como administrador → **Carta y horarios** → desmarca *Validar horario*.

## Primera cuenta de administrador

El sistema **no trae usuarios ni contraseñas de demostración**. Después de `php artisan migrate --seed`, crea la única
cuenta inicial del administrador (la contraseña se pide por pantalla y se guarda con el hash seguro de Laravel):

```
php artisan hongkong:crear-admin
```

Opciones: `--usuario=administrador --nombre="Nombre Apellido" --email=correo@dominio.com`. Si registras un correo, el
administrador podrá recuperar su contraseña por correo. El comando se niega a crear una segunda cuenta de administrador.

*Alternativa:* define `ADMIN_USUARIO`, `ADMIN_NOMBRE`, `ADMIN_EMAIL` y `ADMIN_PASSWORD` en `.env` antes de `migrate --seed`
y **borra `ADMIN_PASSWORD` del archivo** cuando termine.

Si el administrador olvida su contraseña y no tiene correo: `php artisan hongkong:cambiar-password administrador`.

## Trabajadores (Mesero, Cocina y Caja)

Solo el administrador los gestiona: **Configuración › Usuarios y roles**. Puede crear, editar, cambiar el rol, restablecer la
contraseña (botón *Generar* para una clave segura), activar/desactivar y eliminar. No existen contraseñas predeterminadas.

- No se puede eliminar la propia cuenta ni la última cuenta de administrador.
- *Eliminar* un trabajador impide su acceso pero **conserva sus pedidos, pagos y arqueos** en el historial.
- Cada alta, edición y baja queda en la tabla `auditorias`.

## Recuperar contraseña

En el inicio de sesión, «¿Olvidó su contraseña?» envía un enlace de un solo uso (vence en 30 minutos) al correo de la cuenta.
Para que el correo llegue de verdad, configura `MAIL_MAILER=smtp` y los datos SMTP en `.env`. Con `MAIL_MAILER=log` (valor
por defecto) el enlace solo se escribe en `storage/logs/laravel.log`. Las cuentas sin correo piden el cambio al administrador.

## Cancelación de pedidos

Cada pedido tiene su propio botón **Cancelar** (y cada plato su opción *Cancelar plato*). Siempre se pide confirmación y un motivo.

- Solo se cancela **ese** pedido (o ese plato); los demás pedidos y mesas no se tocan.
- Se registra quién canceló, la fecha y hora y el motivo (`pedidos.anulado_por`, `anulado_en`, `motivo_anulacion`; igual en `detalle_pedidos`).
- El pedido sigue en el historial con el estado **CANCELADO** y todos sus platos.
- No se puede cancelar dos veces, ni un pedido con pagos registrados, ni un plato ya listo, entregado o pagado.
- **Mesero:** solo sus propios pedidos, dentro de los primeros minutos configurados (7 por defecto) y sin platos listos o entregados.
- **Administrador:** cualquier pedido o plato abierto, en cualquier momento. **Cocina y Caja:** no pueden cancelar (el servidor responde 403).
- Si se cancelan todos los platos de un pedido sin pagos, el pedido queda cancelado y la mesa libre.

## Pedidos por modalidad

- **Salón:** mesa, comensales (para dividir la cuenta), comensal por plato y mapa «Estado de mesas».
- **Para llevar / Delivery:** sin mesa, sin comensales y sin mapa de mesas; al cambiar de modalidad se limpian los datos que ya no aplican.
  Nombre del cliente obligatorio; en **Delivery** además teléfono / celular y dirección (tabla `envios`). El **costo de envío** es editable
  (S/ 5.00 por defecto en delivery, S/ 0.00 para llevar) y se muestra aparte de los platos.
- No se guarda un pedido sin platos ni con S/ 0.00 en platos. «Indicaciones para cocina» (por plato) y «Observación general» (del pedido) son campos distintos.
- Al cambiar el turno, el pedido en curso se actualiza con los precios vigentes y se quitan los platos que ya no están en la carta.
- El botón de guardar se bloquea mientras se envía (sin duplicados por doble clic).

## Caja, estados de pago y arqueo

- Estado de pago guardado en `pedidos.estado_pago`: **Pendiente**, **Pago parcial** (con saldo restante) y **Pagado** (solo cuando el servidor deja el saldo en S/ 0.00).
  La lista de caja filtra por estado de pago y por modalidad.
- «Recibido» y «Vuelto» aparecen solo con **Efectivo**; Yape, Plin y Tarjeta cobran el monto exacto. «Por consumo» solo existe en salón con más de un comensal.
- La mesa se libera sola cuando el pedido queda pagado al 100 % o se cancela.
- Arqueo y reportes **excluyen pedidos cancelados** y calculan el efectivo esperado = fondo + efectivo recibido − vuelto entregado.
- Si la caja está cerrada, el aviso con el botón «Abrir caja» aparece dentro del cobro y no borra lo que ya elegiste.
- **Ticket de venta** (serie `T001-…`, no es comprobante electrónico): en delivery muestra cliente, teléfono y dirección; detalla el envío aparte;
  lista cada pago con su método (y «Recibido»/«Vuelto» en efectivo); el saldo solo aparece si quedó en pago parcial. Pie: «¡Gracias por su preferencia!».

## Confirmación automática de pagos por pasarela

Endpoint: `POST /api/pasarela/confirmacion` (público, protegido por firma). Define `PASARELA_SECRETO` en `.env` (sin él responde 503).
La pasarela debe enviar la cabecera `X-Firma: sha256=<HMAC-SHA256 del cuerpo con PASARELA_SECRETO>` y un JSON:

```
{"proveedor":"culqi","transaccion_id":"chr_123","estado":"aprobado","monto":45.00,"moneda":"PEN","pedido":"00015","metodo":"Tarjeta"}
```

- Solo registra pagos **aprobados**, en soles, con monto ≤ saldo del pedido abierto. Un pago parcial deja el pedido en «Pago parcial».
- **Idempotencia:** la misma `transaccion_id` nunca se registra dos veces (índice único + bloqueo del pedido).
- El cajero ve una alerta («Pedido #00015 pagado – S/ 45.00»), con sonido, y la caja y la lista se refrescan solas. Si un cobro no puede aplicarse
  (pedido cancelado, monto mayor al saldo…) queda en la auditoría y se muestra una alerta roja para resolverlo a mano.
- Si la caja estaba cerrada, el pago entra a la caja que se abra después.
- Cada pasarela real (Culqi, Izipay, Mercado Pago) usa su propio formato y firma: al contratar la cuenta hay que escribir un pequeño adaptador que
  traduzca su aviso a este formato. **Eso queda pendiente; aquí está preparado el flujo y validado con pruebas.**
- Para ver la alerta sin pasarela (solo con `APP_ENV=local`): `php artisan hongkong:simular-pago 00015 45`.

## Pruebas automáticas

```
php artisan test
```

`tests/Feature/SistemaHongKongTest.php` comprueba: acceso del administrador, ausencia de cuentas de prueba y de voz, gestión de
usuarios, permisos en el servidor, cancelación individual y recuperación de contraseña.

## Seguridad para uso real

- En producción usa `APP_ENV=production` y `APP_DEBUG=false` en `.env`, y sirve el sitio con HTTPS.
- Las contraseñas se guardan con hash (bcrypt); se rechazan las de demostración y las más comunes; 5 intentos fallidos bloquean la cuenta 5 minutos.
- Las páginas de `public/modulos` son HTML estático: **todos los datos y acciones pasan por la API**, que valida sesión y rol en el servidor
  en cada petición. Cambiar la URL a mano no da acceso a nada administrativo.

## Cómo está organizado

| Carpeta | Contenido |
|---|---|
| `database/migrations` | 27 migraciones en orden de niveles (las 9 de nivel 1 son las del equipo) |
| `database/seeders/DatabaseSeeder.php` | 20 mesas, carta, tarifas, taper y configuración (sin usuarios) |
| `app/Models` | 24 modelos Eloquent con sus relaciones |
| `app/Http/Controllers/Api` | API: sesión y recuperación, carta, mesas, usuarios, comandas, caja y reportes |
| `app/Console/Commands` | `hongkong:crear-admin` y `hongkong:cambiar-password` |
| `app/Services/Restaurante.php` | Reglas comunes: horario, 7 minutos, armado de la comanda y caja |
| `app/Http/Middleware/Autenticar.php` | Valida la sesión y el rol en cada ruta |
| `routes/api.php` | Todas las rutas `/api/...` con el rol que puede usarlas |
| `public/modulos` | Pantallas de mesero, cocina, caja y administrador |
| `public/js/datos.js` | Conexión de las pantallas con la API (y avisos cada 3 s) |

## Volver a empezar con datos limpios

```
php artisan migrate --seed
```

Borra todos los pedidos, pagos y cierres de caja, y vuelve a cargar los datos iniciales.


## Introducción de bienvenida

La pantalla inicial muestra el logo de Hong Kong, una animación de entrada y una bienvenida en español mediante la voz del navegador. Si el navegador bloquea la voz automática, pulsa **Escuchar bienvenida**. El botón **Entrar al aplicativo** permite saltar la introducción.
