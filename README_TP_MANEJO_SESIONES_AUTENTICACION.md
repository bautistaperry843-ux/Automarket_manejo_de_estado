# Trabajo Práctico — Manejo de Sesiones y Autenticación

## Descripción del proyecto

**AutoMarket** incorpora autenticación de usuarios mediante sesiones PHP y una base de datos MySQL. El sistema permite registrar cuentas, iniciar sesión, mantener el estado del usuario, cerrar sesión y limitar determinadas funciones según el rol.

Este trabajo corresponde al **TP8: Manejo de Estados de Sesión y Autenticación** y aplica controles básicos de seguridad según OWASP.

## Objetivos cumplidos

- Persistencia del estado de usuario con `$_SESSION`.
- Registro de cuentas en MySQL.
- Inicio de sesión con comparación segura de contraseñas.
- Generación y regeneración del identificador de sesión.
- Cierre completo de sesión y eliminación de cookies.
- Protección de páginas mediante un guardián reutilizable.
- Restricción de rutas administrativas según el rol `admin`.

## Tecnologías utilizadas

- PHP 8.x
- MySQL / MariaDB
- phpMyAdmin
- Apache / XAMPP
- HTML5
- CSS3
- JavaScript
- Git y GitHub

## Tabla de usuarios

La tabla `usuarios` utilizada por el proyecto contiene:

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `id` | INT AUTO_INCREMENT PRIMARY KEY | Identificador único del usuario |
| `nombre` | VARCHAR(100) | Nombre visible de la cuenta |
| `email` | VARCHAR(150) UNIQUE | Correo de autenticación |
| `password` | VARCHAR(255) | Hash seguro de la contraseña |
| `rol` | VARCHAR(50) | `usuario` o `admin` |
| `created_at` | TIMESTAMP | Fecha de creación |
| `updated_at` | TIMESTAMP | Última actualización |

El esquema se encuentra en `includes/schema.sql`.

```sql
CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    rol VARCHAR(50) DEFAULT 'usuario',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## Registro de usuarios

El módulo de registro se encuentra en `registro.php`.

### Funcionamiento

1. El usuario completa nombre, correo electrónico y contraseña.
2. Los datos recibidos se depuran con `trim()` y funciones de sanitización.
3. Se verifica que ningún campo obligatorio esté vacío.
4. Se consulta si el correo ya está registrado.
5. Si el correo está disponible, se genera el hash con:

```php
$password = password_hash($raw_password, PASSWORD_BCRYPT);
```

6. Se inserta el usuario mediante una consulta preparada.
7. El rol asignado por defecto es `usuario`.
8. Se muestra un mensaje de éxito o error y se conserva la experiencia clara para el usuario.

### Seguridad del registro

- No se almacena la contraseña original.
- Se utiliza `PASSWORD_BCRYPT`.
- El correo se compara antes de realizar la inserción.
- El campo `email` tiene restricción `UNIQUE` en la base de datos.
- Los datos mostrados se protegen con `htmlspecialchars()` cuando corresponde.
- La conexión y las consultas utilizan sentencias preparadas.

## Inicio de sesión

El módulo de autenticación se encuentra en `inicio de sesion.php`.

### Funcionamiento

1. Se inicia la sesión con `session_start()` si todavía no está activa.
2. Se obtiene el correo y la contraseña enviados por POST.
3. Se ejecuta una consulta preparada para buscar al usuario por correo.
4. La contraseña ingresada se compara con el hash almacenado:

```php
if (password_verify($password, $usuario['password'])) {
    // Credenciales válidas
}
```

5. Si coincide, se regenera el identificador de sesión:

```php
session_regenerate_id(true);
```

6. Se almacenan los datos necesarios:

```php
$_SESSION['usuario_id'] = $usuario['id'];
$_SESSION['usuario_nombre'] = $usuario['nombre'];
$_SESSION['rol'] = $usuario['rol'];
```

7. Un administrador se redirige al panel; un usuario normal se redirige al inicio.

### Protección ante fijación de sesión

`session_regenerate_id(true)` reemplaza el identificador anterior luego de una autenticación exitosa. De esta manera se reduce el riesgo de reutilización de una sesión fijada antes del inicio de sesión.

## Datos persistentes de la sesión

Durante la autenticación se conserva:

- `usuario_id`: identifica al usuario.
- `usuario_nombre`: se muestra en el encabezado y mensajes.
- `rol`: determina qué páginas administrativas puede acceder.

La sesión permite mantener el estado del usuario mientras navega por distintas páginas del sitio.

## Control de acceso

El archivo reutilizable `includes/auth_guard.php` contiene las funciones de protección:

- `verificarAutenticacion()`.
- `verificarAdmin()`.

### Protección básica

Si no existe `usuario_id`, se redirige al inicio de sesión:

```php
header('Location: ../inicio de sesion.php?error=acceso_denegado');
exit;
```

### Protección por roles

La función `verificarAdmin()` primero valida que exista una sesión y después confirma que el rol sea exactamente `admin`.

Las páginas administrativas usan:

```php
require_once __DIR__ . '/../includes/auth_guard.php';
verificarAdmin();
```

### Vistas protegidas

Se encuentran protegidas, al menos:

- `admin/index.php`
- `admin/crear.php`
- `admin/editar.php`
- `admin/eliminar.php`
- `vender auto.php`
- `favoritos.php`

Un usuario autenticado puede acceder a las funciones permitidas para su rol. Un usuario sin permisos recibe una pantalla de acceso denegado o es redirigido al inicio de sesión.

## Cierre de sesión

El módulo de cierre se encuentra en `logout.php`.

El proceso completo realiza:

1. Limpieza de todos los valores de `$_SESSION`:

```php
$_SESSION = [];
```

2. Eliminación de la cookie de sesión del navegador.
3. Destrucción de la sesión:

```php
session_destroy();
```

4. Redirección al inicio de sesión con el estado `status=logout`.

Esto evita que la información de autenticación siga disponible después de cerrar sesión.

## Cookies de sesión

PHP utiliza una cookie para identificar la sesión del navegador. Al cerrar sesión se obtiene los parámetros actuales y se establece una fecha de expiración pasada:

```php
setcookie(
    session_name(),
    '',
    time() - 42000,
    $params['path'],
    $params['domain'],
    $params['secure'],
    $params['httponly']
);
```

Para despliegues públicos se recomienda además configurar:

- `session.cookie_httponly = 1`.
- `session.cookie_samesite = Lax` o `Strict`.
- `session.cookie_secure = 1` al utilizar HTTPS.

## Ejecución local

1. Iniciar Apache y MySQL desde XAMPP.
2. Importar `includes/schema.sql` desde phpMyAdmin.
3. Copiar `.env.example` como `.env` si corresponde.
4. Abrir:

```text
http://localhost/Artaza/registro.php
```

5. Crear una cuenta de usuario.
6. Iniciar sesión desde:

```text
http://localhost/Artaza/inicio de sesion.php
```

7. Verificar que el encabezado muestre el nombre de la cuenta.
8. Intentar ingresar a una ruta administrativa con una cuenta normal.
9. Promover una cuenta a `admin` desde phpMyAdmin si se necesita probar el panel.
10. Cerrar sesión y comprobar que se pierde el acceso protegido.

### Crear una cuenta administradora

Desde phpMyAdmin ejecutar:

```sql
UPDATE usuarios
SET rol = 'admin'
WHERE email = 'admin@ejemplo.com';
```

Luego cerrar sesión y volver a iniciar sesión para actualizar los datos cargados en `$_SESSION`.

## Evidencias sugeridas

Agregar capturas en `docs/screenshots/`:

- Registro exitoso.
- Registro con correo duplicado.
- Hash almacenado en phpMyAdmin.
- Inicio de sesión correcto.
- Cambio de navegación para usuario autenticado.
- Intento de acceso denegado sin iniciar sesión.
- Usuario normal intentando ingresar al panel administrativo.
- Panel mostrado a una cuenta `admin`.
- Sesión cerrada y redirección al login.

## Flujo de demostración

1. Registrar una cuenta nueva.
2. Verificar la cuenta creada en phpMyAdmin.
3. Comprobar que la contraseña no está guardada en texto plano.
4. Iniciar sesión con credenciales correctas.
5. Observar el nombre y las opciones de sesión en el encabezado.
6. Abrir una página protegida.
7. Cambiar a una cuenta sin rol administrador y verificar el bloqueo.
8. Iniciar como administrador y abrir el panel.
9. Cerrar sesión.
10. Intentar volver al panel y comprobar el bloqueo.

## Mensajes de autenticación

El sistema presenta respuestas claras ante:

- Contraseña incorrecta.
- Correo no registrado.
- Correo ya registrado.
- Acceso denegado sin sesión.
- Acceso denegado por falta de permisos.
- Cierre de sesión.

## Seguridad aplicada

- Contraseñas almacenadas mediante `password_hash()` y verificadas con `password_verify()`.
- Identificador de sesión regenerado después del login.
- Sesiones destruidas al cerrar sesión.
- Cookies de sesión eliminadas al realizar logout.
- Rutas protegidas con una función centralizada.
- Restricción adicional por rol.
- Correo único en la base de datos.
- Consultas preparadas contra la base de datos.
- Datos mostrados con salida segura para HTML.

## Archivos principales

```text
registro.php                  Registro de usuarios
inicio de sesion.php          Inicio de sesión
logout.php                    Cierre de sesión
includes/auth_guard.php       Protección de rutas y roles
includes/header.php           Estado de sesión visible
admin/index.php               Panel protegido de administrador
admin/crear.php               Alta protegida
admin/editar.php              Edición protegida
admin/eliminar.php            Eliminación protegida
includes/schema.sql           Definición de usuarios y relaciones
```
