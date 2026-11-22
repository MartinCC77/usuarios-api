# usuarios-api

Microservicio de **usuarios y login** desarrollado con Spring Boot. Lo usa la app móvil
[Kaiju Inventario](https://github.com/MartinCC77/kaiju-inventario) (DSY1105, Duoc UC, CITT San Bernardo, grupo 8).

Proyecto académico con datos ficticios.

## Integrantes

| Integrante

| Martin Cortes 
| Christian Montecinos 
| Gabriel Fuentes 

**Planificación del equipo (Trello):** https://trello.com/invite/b/6b03691635d3801f38de7de7/ATTI3a1f8e6c111b2879966f20b47fb3c95b45CA7F2F/kaiju-inventario-g8

## Tecnología

- Java 17 (se ejecuta en el servidor con Java 21)
- Spring Boot 4.1.2: Web MVC, Data JPA, Validation y Security (solo para el cifrado BCrypt)
- MySQL 8

## Endpoints

**URL base en producción (AWS EC2):** `http://52.45.85.64:8080`

| Método | Endpoint | Descripción | Respuestas |
|---|---|---|---|
| POST | `/api/auth/login` | Inicia sesión con correo y contraseña | 200 con el usuario, 401 credenciales incorrectas, 400 datos vacíos |
| GET | `/api/usuarios` | Lista todos los usuarios | 200 |
| GET | `/api/usuarios/{id}` | Obtiene un usuario | 200, 404 |
| POST | `/api/usuarios` | Crea un usuario | 201, 400, 409 correo repetido |
| PUT | `/api/usuarios/{id}` | Modifica un usuario (contraseña y rol opcionales) | 200, 400, 404, 409 |
| DELETE | `/api/usuarios/{id}` | Elimina un usuario | 204, 404 |

Ejemplo de login:

```
POST /api/auth/login
{"correo": "admin@kaiju.cl", "contrasena": "kaiju123"}

200 OK
{"id": 1, "nombre": "Administrador Kaiju", "correo": "admin@kaiju.cl", "rol": "ADMIN"}
```

Ejemplo de error:

```
401 Unauthorized
{"mensaje": "Correo o contraseña incorrectos"}
```

Reglas de negocio:

- El correo es único y se guarda en minúsculas.
- La contraseña se guarda cifrada con BCrypt y **nunca** se devuelve en las respuestas.
- El rol puede ser `ADMIN` o `USUARIO` (por defecto `USUARIO`).
- Si el correo no existe o la contraseña es incorrecta, el login responde lo mismo, para no revelar qué correos están registrados.

## Estructura

```
src/main/java/cl/duoc/usuarios/
├── config/        SecurityConfig (BCrypt y reglas de acceso)
├── controller/    AuthController y UsuarioController
├── dto/           Datos de entrada y de salida
├── exception/     Errores controlados y respuestas JSON
├── model/         Entidad Usuario
├── repository/    UsuarioRepository (Spring Data JPA)
└── service/       AuthService y UsuarioService
```

## Pasos para ejecutar en local

### 1. Base de datos

Con MySQL 8 instalado, crear la base y un usuario propio (cambiar la clave):

```sql
CREATE DATABASE usuarios_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'usuarios_app'@'localhost' IDENTIFIED BY 'CLAVE_QUE_ELIJAS';
GRANT ALL PRIVILEGES ON usuarios_db.* TO 'usuarios_app'@'localhost';
```

Las tablas las crea Spring automáticamente al arrancar.

### 2. Variables de entorno

La conexión se configura por variables de entorno, para no escribir claves en el código:

| Variable | Valor por defecto | Descripción |
|---|---|---|
| `DB_PASSWORD` | (sin valor, obligatoria) | Clave del usuario de la base de datos |
| `DB_USER` | `usuarios_app` | Usuario de la base de datos |
| `DB_HOST` | `localhost` | Servidor de la base de datos |
| `DB_PORT` | `3306` | Puerto de la base de datos |
| `DB_NAME` | `usuarios_db` | Nombre de la base de datos |

### 3. Arrancar

En Windows (PowerShell):

```
$env:DB_PASSWORD = 'CLAVE_QUE_ELIJAS'
.\mvnw.cmd spring-boot:run
```

En Linux:

```
export DB_PASSWORD='CLAVE_QUE_ELIJAS'
./mvnw spring-boot:run
```

El servicio queda en `http://localhost:8080`. Para crear el primer usuario:

```
curl -X POST http://localhost:8080/api/usuarios -H "Content-Type: application/json" \
  -d '{"nombre":"Administrador Kaiju","correo":"admin@kaiju.cl","contrasena":"kaiju123","rol":"ADMIN"}'
```

## Despliegue en AWS

El servicio corre en una instancia **EC2 con Ubuntu**, con MySQL instalado en la misma máquina y una IP elástica.

1. Compilar: `./mvnw -DskipTests package` (genera `target/usuarios-api-0.0.1-SNAPSHOT.jar`).
2. Guardar la clave de la base de datos en un archivo protegido (`/etc/usuarios-api.env`, permisos `600`) con la línea `DB_PASSWORD=...`.
3. Registrar el servicio en `systemd`, que arranca el JAR automáticamente, lee ese archivo y lo reinicia si falla.
4. En el grupo de seguridad solo se abren los puertos **22** (administración) y **8080** (API). El puerto de MySQL (3306) **no** se expone a internet.

## Limitaciones conocidas

- No hay autenticación con token (JWT): los endpoints no exigen sesión.
- La comunicación es por `http`; para producción debería usarse `https`.
- No hay restricción por rol en las operaciones sobre usuarios.
