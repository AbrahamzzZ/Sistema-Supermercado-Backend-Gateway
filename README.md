# Sistema Supermercado — Backend (Microservicios)

Backend de un sistema de supermercado construido con **arquitectura de microservicios** en .NET 8.
Este repositorio contiene el **API Gateway (Ocelot)** y la **orquestación con Docker Compose** de todo el sistema.

| Microservicio | Responsabilidad | Repositorio |
|---|---|---|
| **Gateway** | Punto de entrada único, enrutamiento y validación de JWT | *(este repo)* |
| **Auth** | Usuarios, roles, menús, login y emisión de tokens JWT | [Sistema-Supermercado-Backend-Auth](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Auth) |
| **Admin** | Datos maestros: productos, categorías, clientes, proveedores, sucursales, ofertas, reportes | [Sistema-Supermercado-Backend-Admin](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Admin) |
| **Order** | Compras, ventas y logs | [Sistema-Supermercado-Backend-Order](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Order) |
| **Inventory** | Stock de productos y movimientos de inventario | [Sistema-Supermercado-Backend-Inventory](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Inventory) |

**Frontend (Angular):** [Sistema-Supermercardo-Frontend](https://github.com/AbrahamzzZ/Sistema-Supermercardo-Frontend).
Funciona con estos microservicios o con la versión monolítica del backend
([Sistema-Supermercado-Backend](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend)); ver [Frontend](#frontend).

**Tecnologías:** .NET 8 · Ocelot · Entity Framework Core (Code First) · SQL Server · Apache Kafka · JWT RS256 + JWKS · FluentValidation · Docker Compose

---

## Arquitectura

```
                          ┌──────────────────────────┐
   Cliente (front/Postman)│   API Gateway (Ocelot)   │  :8000
  ───────────────────────►│   valida JWT (RS256)     │
                          └──┬──────┬──────┬──────┬──┘
                /auth/*      │      │      │      │   /inventory/*
            ┌────────────────┘      │      │      └────────────────┐
            ▼                /admin/*      /order/*                ▼
     ┌─────────────┐      ┌─────────────┐ ┌─────────────┐   ┌─────────────┐
     │    Auth     │      │    Admin    │ │    Order    │   │  Inventory  │
     │ firma JWT   │      │             │ │             │   │             │
     │ publica JWKS│      │             │ │             │   │             │
     └──────┬──────┘      └──────┬──────┘ └──┬───────┬──┘   └──────┬──────┘
            │                    │   HTTP vía│Gateway│ Kafka       │
            │                    │◄──────────┘       │ "movimientos-stock"
            ▼                    ▼                   ▼             ▼
        DB_Auth              DB_Admin            DB_Order     DB_Inventory
                     (un mismo contenedor SQL Server, una base por servicio)
```

- **Base de datos por servicio:** cada microservicio tiene su propia base y es el único que la toca.
- **Comunicación síncrona (HTTP):** Order consulta a Admin y Auth **a través del Gateway**, reenviando el token del usuario.
- **Comunicación asíncrona (Kafka):** al registrar una compra o venta, Order publica un evento en el tópico
  `movimientos-stock`; Inventory lo consume y actualiza el stock (compra = `ENTRADA`, venta = `SALIDA`).
- **Autenticación:** ver [Autenticación (JWT RS256 + JWKS)](#autenticación-jwt-rs256--jwks).

---

## Requisitos previos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (con al menos **4 GB de RAM** asignados; SQL Server necesita 2 GB)
- [Git](https://git-scm.com/)
- **OpenSSL** para generar la clave de firma JWT (viene incluido con Git for Windows: usa *Git Bash*)
- Opcional, para desarrollo sin Docker: [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) y SQL Server / LocalDB
- Opcional: [Ollama](https://ollama.com/) con el modelo `qwen3:8b` para el reporte con IA de Admin

---

## Levantar todo con Docker

### 1. Clonar los 5 repositorios en la misma carpeta

El `docker-compose.yml` construye cada servicio usando rutas relativas (`../Sistema-Supermercado-Backend-Admin`, etc.),
así que **los repos deben quedar uno al lado del otro**:

```bash
mkdir "Microservicios Sistema Supermercado" && cd "Microservicios Sistema Supermercado"

git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Gateway.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Auth.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Admin.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Order.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Inventory.git
```

```
Microservicios Sistema Supermercado/
├─ Sistema-Supermercado-Backend-Admin
├─ Sistema-Supermercado-Backend-Auth
├─ Sistema-Supermercado-Backend-Gateway   ← docker-compose.yml
├─ Sistema-Supermercado-Backend-Inventory
└─ Sistema-Supermercado-Backend-Order
```

### 2. Generar la clave privada para firmar los JWT (solo la primera vez)

Desde la carpeta del Gateway, en **Git Bash**:

```bash
cd Sistema-Supermercado-Backend-Gateway
mkdir -p secrets
openssl genrsa -out secrets/jwt_private_key.pem 2048
```

> ⚠️ La carpeta `secrets/` está en `.gitignore`. **Nunca subas la clave privada al repositorio.**

### 3. (Opcional) Configurar variables

```bash
cp .env.example .env
```

| Variable | Descripción | Valor por defecto |
|---|---|---|
| `SA_PASSWORD` | Contraseña del usuario `sa` de SQL Server (8+ caracteres, mayúsculas, minúsculas, números y símbolos) | `Supermercado_2026!` |

### 4. Levantar los contenedores

```bash
docker compose up -d --build
```

El orden de arranque lo controla el propio compose:

1. `sqlserver` y `kafka` arrancan y esperan a estar *healthy*.
2. `kafka-init` crea el tópico `movimientos-stock` y termina.
3. Los microservicios arrancan; cada uno **aplica sus migraciones** de EF Core automáticamente al iniciar:
   tablas, **stored procedures** y **datos iniciales** (roles, menús, usuario administrador, negocio, categorías y sucursales).
4. El `gateway` arranca al final.

### Usuario inicial

| Correo | Clave | Rol |
|---|---|---|
| `admin@supermercado.com` | `Admin123*` | Administrador |

> ⚠️ Cambia esta clave después del primer inicio de sesión. Se guarda con un hash antiguo (SHA-256) y Auth la
> convierte automáticamente a **BCrypt** en el primer login.

Ver el estado y los logs:

```bash
docker compose ps
docker compose logs -f auth-api          # o admin-api, order-api, inventory-api, gateway
```

### 5. URLs disponibles

| Servicio | URL |
|---|---|
| **API Gateway** | http://localhost:8000 |
| Admin (Swagger) | http://localhost:8001/swagger |
| Auth (Swagger) | http://localhost:8002/swagger |
| Inventory (Swagger) | http://localhost:8003/swagger |
| Order (Swagger) | http://localhost:8004/swagger |
| Kafka UI | http://localhost:8090 |
| **Seq (logs de todos los servicios)** | http://localhost:5341 |
| SQL Server | `localhost,14330` · usuario `sa` · contraseña `SA_PASSWORD` |
| Kafka (desde el host) | `localhost:29092` |

> Los puertos 8001–8004 están expuestos solo para depurar con Swagger. Un cliente real debe usar **siempre el Gateway**.

### Detener / reiniciar

```bash
docker compose down          # detiene y elimina contenedores (los datos se conservan)
docker compose down -v       # además BORRA los volúmenes (bases de datos y Kafka)
docker compose up -d --build admin-api   # reconstruir solo un servicio tras cambiar su código
```

---

## Rutas del Gateway

| Ruta en el Gateway | Se redirige a | Métodos | Requiere token |
|---|---|---|---|
| `POST /auth/Usuario/login` | Auth `/api/Usuario/login` | POST | No |
| `/auth/{...}` | Auth `/api/{...}` | GET, POST, PUT, DELETE | Sí |
| `/admin/{...}` | Admin `/api/{...}` | GET, POST, PUT, DELETE | Sí |
| `/order/{...}` | Order `/api/{...}` | GET, POST | Sí |
| `/inventory/{...}` | Inventory `/api/{...}` | GET, POST | Sí |

La configuración está en `ApiGateway/ocelot.json` (local) y `ApiGateway/ocelot.Docker.json` (Docker).
El Gateway carga automáticamente `ocelot.{ASPNETCORE_ENVIRONMENT}.json` si existe; si no, usa `ocelot.json`.

### Ejemplo de uso

```bash
# 1. Login (devuelve el token)
curl -X POST http://localhost:8000/auth/Usuario/login \
  -H "Content-Type: application/json" \
  -d '{ "correo_Electronico": "admin@correo.com", "clave": "tu-clave" }'

# 2. Usar el token en cualquier otra ruta
curl http://localhost:8000/admin/Producto \
  -H "Authorization: Bearer <token>"
```

### CORS

El frontend debe apuntar **solo al Gateway** (`http://localhost:8000`). Por eso CORS se configura **únicamente
aquí** (política `Frontend` en `Program.cs`). Los microservicios no tienen CORS: el navegador nunca los llama
directamente y entre servidores (Gateway → servicios, Order → Inventory) CORS no aplica.

Sin configuración se acepta cualquier origen (desarrollo). En producción limita los orígenes en `appsettings.json`:

```json
"Cors": { "OrigenesPermitidos": [ "https://mi-frontend.com" ] }
```

### Frontend

El [frontend](https://github.com/AbrahamzzZ/Sistema-Supermercardo-Frontend) usa una URL por módulo, con el
prefijo de cada microservicio en el Gateway:

| Módulo del front | URL (microservicios) | Recursos |
|---|---|---|
| `admin` | `http://localhost:8000/admin/` | Categoria, Cliente, Negocio, Oferta, Producto, Proveedor, Sucursal, Transportista |
| `auth` | `http://localhost:8000/auth/` | Usuario (login: `auth/Usuario/login`), Menu, Rol |
| `order` | `http://localhost:8000/order/` | Compra, Venta, Log |
| `inventory` | `http://localhost:8000/inventory/` | ProductoStock, MovimientoStock |

Con `npm start` el front usa estos microservicios. Con `npm start -- --configuration monolito` usa el monolito
(`http://localhost:5299/api/`). Los detalles están en el README del frontend.

---

## Autenticación (JWT RS256 + JWKS)

Los tokens se firman con **RS256 (clave asimétrica)**:

- **Auth** es el **único** que tiene la clave **privada** y el único que puede emitir tokens.
- Auth publica la clave **pública** en:
  - `GET /.well-known/openid-configuration`
  - `GET /.well-known/jwks.json`
- **Gateway, Admin, Order e Inventory** solo **validan** tokens: al arrancar descargan la clave pública desde
  `Jwt:Authority` (la URL de Auth) y la guardan en caché. No tienen ningún secreto.
- La validación ocurre **en el Gateway y también en cada servicio** (defensa en profundidad).

```
Auth ──firma con clave privada──► token (header: alg=RS256, kid=...)
Gateway/Admin/Order/Inventory ──GET /.well-known/jwks.json──► validan con la clave pública
```

**Origen de la clave privada en Auth** (en este orden):

1. `Jwt:PrivateKeyPath` → ruta a un archivo `.pem` (en Docker: `/run/secrets/jwt_private_key`)
2. `Jwt:PrivateKey` → contenido PEM directamente (útil con `dotnet user-secrets`)
3. Ninguno → Auth genera una clave temporal y lo avisa en el log. **Solo para pruebas**: los tokens dejan de ser
   válidos cada vez que Auth se reinicia.

**Rotar la clave:** genera un nuevo `.pem`, reemplaza `secrets/jwt_private_key.pem` y reinicia Auth
(`docker compose restart auth-api`). El `kid` cambia solo y los demás servicios descargan la nueva clave pública
al recibir un token con un `kid` desconocido. Los tokens firmados con la clave anterior dejan de ser válidos.

**Contraseñas:** se guardan con **BCrypt** (work factor 11) y se verifican en C#, no en SQL. Las claves con el
hash antiguo (SHA-256) siguen funcionando y se convierten a BCrypt en el siguiente login exitoso.

---

## Auditoría y borrado lógico

Las tablas de datos maestros (Admin), `USUARIO` (Auth) y `PRODUCTO_STOCK` (Inventory) tienen las mismas columnas:

| Columna | Quién la llena |
|---|---|
| `ID_USUARIO_CREACION` / `ID_USUARIO_MODIFICACION` | El stored procedure, con el usuario **del token JWT** (nunca del body) |
| `FECHA_CREACION` / `FECHA_MODIFICACION` | SQL Server con `SYSUTCDATETIME()` (**todas las fechas en UTC**) |

- Cada servicio obtiene el usuario con `ICurrentUserService`, que lee el claim `NameIdentifier` del token.
- **Borrado lógico:** `DELETE` no borra la fila; marca `ELIMINADO = 1` y guarda quién y cuándo. Los listados no
  muestran registros eliminados. Así no se pierden datos que otros microservicios referencian (una venta antigua
  sigue apuntando a su cliente).
- **Compras y ventas** (Order) no llevan columnas de auditoría porque no se editan: `ID_USUARIO` (tomado del token)
  y `FECHA_COMPRA` / `FECHA_VENTA` ya indican quién y cuándo.
- **Kafka:** como Inventory procesa mensajes sin token, Order incluye `IdUsuario` en cada mensaje.

## Historial de precios

Inventory guarda en `HISTORIAL_PRECIO` cada cambio de precio de compra o de venta que llega con una compra. Solo
se insertan registros, nunca se modifican. Así queda un respaldo de todos los precios que tuvo un producto.

- `GET /inventory/ProductoStock/{idProducto}`: stock y precio actuales
- `GET /inventory/ProductoStock/{idProducto}/historial-precios`: historial, del más reciente al más antiguo

**Stock:** Order consulta el stock en Inventory **antes** de registrar una venta y la rechaza si no alcanza.
Si aun así llega un movimiento sin stock (por ejemplo, dos ventas simultáneas), Inventory lo registra como
`Warning` y descarta el mensaje en vez de reintentarlo indefinidamente.

---

## Logs (Serilog + Seq)

Los 5 servicios escriben **logs estructurados** con Serilog en la consola y en **Seq**, un servidor central de logs
con interfaz web: http://localhost:5341.

| Nivel | Cuándo | Ejemplo |
|---|---|---|
| `Information` | Todo salió bien | "Venta 00012 registrada", peticiones 2xx |
| `Warning` | Se incumplió una regla de negocio | stock insuficiente, código duplicado, login fallido, peticiones 4xx |
| `Error` | Fallo técnico | excepción no controlada, base de datos o Kafka caídos, peticiones 5xx |

**Correlation Id:** el Gateway genera un `X-Correlation-Id` por petición, o usa el que envíe el cliente. Ese id viaja
a cada microservicio por HTTP y a Inventory en los headers de Kafka. Todos los logs lo llevan, así que en Seq
puedes seguir una venta completa (Gateway → Order → Admin/Inventory → Kafka → Inventory) con un solo filtro:

```
CorrelationId = 'el-id-que-devolvio-la-respuesta'
```

Cada log incluye además `Servicio`, `IdUsuario`, método, ruta, código de estado y duración. Ante un error no
controlado, la API responde `500` con el mensaje `Código de seguimiento: {correlationId}` para buscarlo en Seq.

> **¿Por qué no una tabla LOG en cada base de datos?** Si la base de datos falla, no puedes registrar el error de que
> la base falló. Además, cada log restaría rendimiento a la base de negocio y tendrías que consultar 4 bases
> para seguir un problema. La tabla `LOG` de Order se mantiene por ahora para no romper el front (ver [Pendientes](#pendientes)).

---

## Desarrollo local (sin Docker)

1. **Infraestructura:** puedes levantar solo SQL Server y Kafka con Docker y correr las APIs desde Visual Studio:

   ```bash
   docker compose up -d sqlserver kafka kafka-init kafka-ui seq
   ```

   En ese caso usa `localhost,14330` en la cadena de conexión y `localhost:29092` como `Kafka:BootstrapServers`.
   También puedes usar tu SQL Server / LocalDB local, por ejemplo:

   ```
   Server=(localdb)\MSSQLLocalDB;Database=DB_Admin;Integrated Security=True;TrustServerCertificate=True;
   ```

2. **Clave JWT de Auth** con user-secrets (para que no cambie en cada reinicio):

   ```powershell
   cd Sistema-Supermercado-Backend-Auth\Auth
   dotnet user-secrets init
   dotnet user-secrets set "Jwt:PrivateKeyPath" "C:\ruta\a\Sistema-Supermercado-Backend-Gateway\secrets\jwt_private_key.pem"
   ```

3. **Orden de arranque:** primero **Auth** (los demás le piden la clave pública), luego Admin, Inventory, Order y por último el Gateway.

| Servicio | URL local (perfil `https`) |
|---|---|
| Gateway | https://localhost:7134 |
| Auth | https://localhost:7186 |
| Admin | https://localhost:7047 |
| Order | https://localhost:7118 |
| Inventory | https://localhost:7091 |

---

## Configuración por variables de entorno

Todos los valores de `appsettings.json` se pueden sobrescribir con variables de entorno usando `__` como separador
(así lo hace el `docker-compose.yml`):

| Variable | Usada por | Ejemplo (Docker) |
|---|---|---|
| `ConnectionStrings__CadenaSQL` | Auth, Admin, Order, Inventory | `Server=sqlserver,1433;Database=DB_Admin;...` |
| `Jwt__Issuer` / `Jwt__Audience` | Todos | `SupermercadoAPI` / `SupermercadoClient` |
| `Jwt__PrivateKeyPath` | Auth | `/run/secrets/jwt_private_key` |
| `Jwt__Authority` | Gateway, Admin, Order, Inventory | `http://auth-api:8080` |
| `Jwt__RequireHttpsMetadata` | Gateway, Admin, Order, Inventory | `false` (solo dentro de Docker) |
| `Services__GatewayUrl` | Order, Inventory | `http://gateway:8080` |
| `Kafka__BootstrapServers` | Order, Inventory | `kafka:9092` |
| `Kafka__GroupId` / `Kafka__Enabled` | Inventory | `inventory-service` / `true` |
| `Ollama__BaseUrl` | Admin | `http://host.docker.internal:11434` |
| `Seq__ServerUrl` | Todos | `http://seq:80` |

---

## Problemas comunes

| Síntoma | Causa / solución |
|---|---|
| Un servicio se reinicia varias veces al arrancar | SQL Server tarda en estar listo. Los servicios reintentan la conexión; espera 1–2 minutos. |
| `401 Unauthorized` en todas las rutas justo después de levantar | Los servicios aún no descargaron la clave pública de Auth. Espera unos segundos y reintenta. |
| `401` con un token que antes funcionaba | El token expiró, Auth se reinició con una clave temporal (sin `.pem`), o es un token antiguo HS256. Vuelve a hacer login. |
| `auth-api` no arranca: *No se encontró la clave privada JWT* | Falta `secrets/jwt_private_key.pem`. Ver el [paso 2](#2-generar-la-clave-privada-para-firmar-los-jwt-solo-la-primera-vez). |
| Los reportes de `/admin/Negocio/...` responden *"No se encontraron registros"* | Todavía no hay compras o ventas en `DB_Order`. |
| Los reportes responden *"No se pudieron obtener los datos de otro servicio"* | Admin no pudo consultar a Order (o a Inventory/Auth) por el Gateway. Revisa `docker compose logs order-api` y busca el `CorrelationId` en Seq. |
| Venta rechazada con *"No se pudo verificar el stock"* | Inventory no responde. Revisa `docker compose logs inventory-api`. |
| El stock no se actualiza tras una compra/venta | Busca el `CorrelationId` de la petición en Seq (http://localhost:5341) y revisa el tópico `movimientos-stock` en Kafka UI (http://localhost:8090). |
| El asistente de IA responde *"El servicio de IA no está disponible"* | Ollama no está corriendo en tu máquina o falta el modelo: `ollama pull qwen3:8b`. |

---

## Pendientes

- [x] **Reportes de Admin** (`/admin/Negocio/...`): Order calcula los totales por ID (`/order/Estadistica/...`) y
      Admin completa los nombres con sus propias tablas. Ver *Reportes del negocio* en el README de Admin.
- [ ] **Tabla `LOG` de Order:** decidir si se elimina (los logs ya están en Seq) o si se mantiene para la pantalla del front.
- [ ] Índice único para `PRODUCTO.CODIGO` y `USUARIO.CORREO_ELECTRONICO` (con borrado lógico conviene un índice filtrado `WHERE ELIMINADO = 0`).
