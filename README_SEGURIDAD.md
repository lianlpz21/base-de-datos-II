# PETSALUD - Seguridad con Spring Security y JWT

## Requisitos
- Java 17
- MySQL
- Maven (o Maven Wrapper)

## Configuracion local
1. Crear una base de datos MySQL llamada `petsalud_db` o ajustar `spring.datasource.url`.
2. Configurar usuario y contrasena de MySQL en `application.properties`.
3. En produccion, definir la variable de entorno `JWT_SECRET` con una clave Base64 segura. El valor por defecto del proyecto es solo para la actividad local.
4. Ejecutar la aplicacion con `./mvnw spring-boot:run` (Linux/macOS) o `mvnw.cmd spring-boot:run` (Windows).

## Flujo de prueba
### 1. Registrar un usuario
`POST /api/usuarios/registro`

```json
{
  "email": "cliente@petsalud.cl",
  "password": "ClaveSegura123!",
  "nombre": "Cliente",
  "apellido": "Demo",
  "telefono": "123456789",
  "direccion": "Santiago"
}
```

El registro publico asigna siempre el rol `CLIENTE` y la contrasena se almacena con BCrypt.

### 2. Iniciar sesion
`POST /api/auth/login`

```json
{
  "email": "cliente@petsalud.cl",
  "password": "ClaveSegura123!"
}
```

La respuesta contiene un token JWT:

```json
{
  "token": "<jwt>",
  "tipo": "Bearer"
}
```

### 3. Consumir un endpoint protegido
Agregar el header:

`Authorization: Bearer <jwt>`

Por ejemplo: `GET /api/productos`.

## Prueba de roles
Para probar los tres roles de la pauta, registrar tres usuarios y cambiar dos roles directamente en la base de datos (simulando la asignacion administrativa de roles):

```sql
UPDATE usuarios SET rol = 'EMPLEADO' WHERE email = 'empleado@petsalud.cl';
UPDATE usuarios SET rol = 'GERENTE' WHERE email = 'gerente@petsalud.cl';
```

Luego iniciar sesion nuevamente con cada usuario para obtener un JWT.

Ejemplos de autorizacion implementada:
- `CLIENTE`: puede usar endpoints autenticados generales, pero no administrar productos ni usuarios.
- `EMPLEADO`: puede crear/actualizar/eliminar productos, consultar stock bajo, actualizar estados de citas y crear expedientes.
- `GERENTE`: posee los permisos anteriores y ademas puede listar/eliminar usuarios y eliminar expedientes.

Resultados esperados:
- Sin JWT en un endpoint protegido: HTTP 401.
- JWT valido pero rol insuficiente: HTTP 403.
- JWT valido y rol autorizado: la solicitud llega al controlador.

## Componentes de seguridad
- `SecurityConfig`: API stateless, endpoints publicos, BCrypt, filtro JWT y manejo 401/403.
- `JwtUtil`: genera y valida JWT firmados con HS256, con `sub`, `role`, fecha de emision y expiracion.
- `JwtAuthenticationFilter`: extrae `Authorization: Bearer`, valida el token y establece la autenticacion en `SecurityContextHolder`.
- `UsuarioDetailsService`: carga el usuario desde MySQL por email.
- `AuthController`: autentica credenciales mediante `AuthenticationManager` y genera el JWT.
- `@PreAuthorize`: restringe operaciones segun `CLIENTE`, `EMPLEADO` y `GERENTE`.
