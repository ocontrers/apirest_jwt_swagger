# API REST JWT — Colegio San Marcos

API REST para la gestión de alumnos de un colegio, con autenticación basada en **JWT**, control de acceso por **roles** y protección de endpoints mediante **API Key**. Construida con Node.js, Express 5 y Prisma sobre PostgreSQL.

## Características

- 🔐 Autenticación con JWT (registro, login, perfil, cambio de contraseña)
- 👥 Control de acceso por roles (`ADMIN`, `COORDINADOR`)
- 🔑 Protección adicional por API Key para operaciones de escritura sobre alumnos
- 🎓 CRUD completo de alumnos con filtrado por grado
- 🧱 Arquitectura en capas (rutas → controladores → servicios → repositorios)
- 🛡️ Manejo centralizado de errores
- 🔒 Contraseñas hasheadas con bcrypt

## Stack técnico

| Tecnología | Uso |
|---|---|
| [Express 5](https://expressjs.com/) | Framework HTTP |
| [Prisma](https://www.prisma.io/) + `@prisma/adapter-pg` | ORM sobre PostgreSQL |
| [PostgreSQL](https://www.postgresql.org/) | Base de datos |
| [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) | Generación y verificación de JWT |
| [bcryptjs](https://github.com/dcodeIO/bcrypt.js) | Hash de contraseñas |
| [dotenv](https://github.com/motdotla/dotenv) | Variables de entorno |

## Estructura del proyecto

```
.
├── index.js                     # Punto de entrada de la app
├── prisma.config.js             # Configuración de Prisma
├── prisma/
│   ├── schema.prisma             # Modelos de datos (Alumno, Usuario, Rol)
│   └── migrations/               # Historial de migraciones
└── src/
    ├── config/
    │   └── prisma.js              # Cliente de Prisma con adaptador pg
    ├── controllers/
    │   ├── alumno.controller.js
    │   └── auth.controller.js
    ├── errors/
    │   └── appError.js            # Clase de error personalizada
    ├── middlewares/
    │   ├── apiKey.js               # Verifica el header x-api-key
    │   ├── auth.js                 # Verifica el token JWT
    │   ├── errorHandler.js         # Manejo centralizado de errores
    │   └── requireRole.js          # Restringe acceso por rol
    ├── repositories/
    │   ├── alumno.repository.js    # Acceso a datos de alumnos
    │   └── usuario.repository.js   # Acceso a datos de usuarios
    ├── routes/
    │   ├── alumno.routes.js
    │   └── auth.routes.js
    ├── services/
    │   ├── alumno.service.js       # Lógica de negocio de alumnos
    │   └── auth.service.js         # Lógica de negocio de autenticación
    └── utils/
        ├── password.js             # Hash y comparación de contraseñas
        └── token.js                 # Generación y verificación de JWT
```

## Requisitos previos

- Node.js 18 o superior
- Una base de datos PostgreSQL accesible

## Instalación

1. Clona el repositorio:

   ```bash
   git clone https://github.com/ocontrers/apirest_jwt_swagger.git
   cd apirest_jwt_swagger
   ```

2. Instala las dependencias:

   ```bash
   npm install
   ```

3. Crea un archivo `.env` en la raíz del proyecto con las siguientes variables:

   ```env
   # Puerto en el que corre el servidor (opcional, por defecto 3000)
   PORT=3000

   # Cadena de conexión a PostgreSQL
   DATABASE_URL="postgresql://usuario:password@localhost:5432/colegio_san_marcos"

   # Secreto usado para firmar y verificar los JWT
   JWT_SECRET="una_clave_larga_y_secreta"

   # Clave estática requerida para crear, editar o eliminar alumnos
   API_KEY="una_api_key_segura"
   ```

4. Ejecuta las migraciones de Prisma para crear las tablas:

   ```bash
   npx prisma migrate deploy
   ```

   (En desarrollo también puedes usar `npx prisma migrate dev`.)

5. Inicia el servidor en modo desarrollo (con recarga automática):

   ```bash
   npm run dev
   ```

   El servidor quedará disponible en `http://localhost:3000`.

## Modelo de datos

**Alumno**

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | Int | Autoincremental |
| `nombre` | String | Nombre del alumno |
| `apellido` | String | Apellido del alumno |
| `grado` | String | Grado que cursa |
| `seccion` | String | Sección asignada |

**Usuario**

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | Int | Autoincremental |
| `nombre` | String | Nombre del usuario |
| `email` | String | Único, usado para login |
| `passwordHash` | String | Contraseña hasheada con bcrypt |
| `rol` | `ADMIN` \| `COORDINADOR` | Rol del usuario (por defecto `COORDINADOR`) |

## Endpoints

### Autenticación — `/api/auth`

| Método | Ruta | Auth requerida | Descripción |
|---|---|---|---|
| `POST` | `/registro` | No | Crea una nueva cuenta de usuario |
| `POST` | `/login` | No | Inicia sesión y devuelve un token JWT |
| `GET` | `/perfil` | JWT | Devuelve los datos del usuario autenticado |
| `PATCH` | `/usuarios/:id/password` | No* | Cambia la contraseña de un usuario |
| `GET` | `/usuarios` | JWT + rol `ADMIN` | Lista todos los usuarios registrados |

\* Esta ruta valida la contraseña actual del usuario, pero actualmente no exige un token JWT. Si vas a exponerla en producción, se recomienda añadir `requireAuth` y verificar que el `id` del token coincida con el `id` de la URL.

**Ejemplo — Registro**

```bash
curl -X POST http://localhost:3000/api/auth/registro \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Ana Pérez", "email": "ana@colegio.com", "password": "contrasena123"}'
```

**Ejemplo — Login**

```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "ana@colegio.com", "password": "contrasena123"}'
```

Respuesta:

```json
{
  "usuario": { "id": 1, "nombre": "Ana Pérez", "email": "ana@colegio.com" },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

El token debe enviarse en las peticiones protegidas dentro del header:

```
Authorization: Bearer <token>
```

### Alumnos — `/api/alumnos`

| Método | Ruta | Auth requerida | Descripción |
|---|---|---|---|
| `GET` | `/` | No | Lista todos los alumnos (admite `?grado=` para filtrar) |
| `GET` | `/:id` | No | Obtiene un alumno por su ID |
| `POST` | `/` | API Key | Crea un nuevo alumno |
| `PATCH` | `/:id` | API Key | Actualiza campos de un alumno existente |
| `DELETE` | `/:id` | API Key | Elimina un alumno |

Las rutas protegidas por API Key requieren el header:

```
x-api-key: <valor definido en API_KEY>
```

**Ejemplo — Crear alumno**

```bash
curl -X POST http://localhost:3000/api/alumnos \
  -H "Content-Type: application/json" \
  -H "x-api-key: una_api_key_segura" \
  -d '{"nombre": "Luis", "apellido": "Gómez", "grado": "5to", "seccion": "A"}'
```

## Roles y permisos

- **COORDINADOR** (rol por defecto al registrarse): acceso a su propio perfil.
- **ADMIN**: además de lo anterior, puede listar todos los usuarios registrados (`GET /api/auth/usuarios`).

Los roles adicionales sobre alumnos no están restringidos por rol, sino por la API Key compartida.

## Manejo de errores

Todas las rutas delegan los errores a un middleware centralizado (`errorHandler.js`), que responde con un JSON consistente:

```json
{ "error": "Mensaje descriptivo del error" }
```

Códigos de estado usados:

| Código | Cuándo se usa |
|---|---|
| `400` | Datos inválidos o campos faltantes |
| `401` | Credenciales incorrectas o token ausente/ inválido |
| `403` | Rol sin permisos suficientes |
| `404` | Recurso no encontrado |
| `409` | Conflicto (email o alumno duplicado) |
| `500` | Error interno no controlado |
