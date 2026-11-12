# Sistema Supermercado — Backend (Microservicios)

Backend de un sistema de supermercado con **arquitectura de microservicios** en .NET 8. Este repositorio contiene
el **API Gateway (Ocelot)** y el **`docker-compose.yml`** que levanta todo el sistema.

📘 **[Manual técnico](docs/Manual%20Tecnico.docx)**: arquitectura, cada microservicio, seguridad, base de datos,
Kafka, logs, configuración, despliegue y guías para desarrolladores.

| Servicio | Responsabilidad | Repositorio |
|---|---|---|
| **Gateway** | Punto de entrada único, rutas, validación de JWT y CORS | *(este repo)* |
| **Auth** | Usuarios, roles, menús, login y emisión de tokens JWT | [Sistema-Supermercado-Backend-Auth](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Auth) |
| **Admin** | Datos maestros, reportes, asistente de IA y logs del sistema | [Sistema-Supermercado-Backend-Admin](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Admin) |
| **Order** | Compras y ventas | [Sistema-Supermercado-Backend-Order](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Order) |
| **Inventory** | Stock, movimientos e historial de precios | [Sistema-Supermercado-Backend-Inventory](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Inventory) |

**Frontend (Angular):** [Sistema-Supermercardo-Frontend](https://github.com/AbrahamzzZ/Sistema-Supermercardo-Frontend).
Funciona con estos microservicios o con el [backend monolítico](https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend).

**Tecnologías:** .NET 8 · Ocelot · EF Core (Code First) · SQL Server · Apache Kafka · JWT RS256 + JWKS · Serilog + Seq · Docker Compose

## Arquitectura

```
   Cliente (front/Postman) ──► API Gateway (Ocelot) :8000 ── valida JWT
                                 │ /auth   │ /admin   │ /order   │ /inventory
                                 ▼         ▼          ▼          ▼
                               Auth      Admin      Order ──Kafka──► Inventory
                                 │         │          │   "movimientos-stock" │
                              DB_Auth   DB_Admin   DB_Order           DB_Inventory
```

Cada servicio tiene su propia base de datos. Se comunican por HTTP **a través del Gateway** y por **Kafka**
(Order publica los movimientos de stock e Inventory los consume). Los logs de todos van a **Seq**.

## Levantar todo con Docker

Requisitos: **Docker Desktop** (4 GB de RAM o más), **Git** y **Git Bash** (para OpenSSL).

**1. Clonar los 5 repositorios en la misma carpeta** (el compose usa rutas relativas como `../Sistema-Supermercado-Backend-Admin`):

```bash
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Gateway.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Auth.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Admin.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Order.git
git clone https://github.com/AbrahamzzZ/Sistema-Supermercado-Backend-Inventory.git
```

**2. Generar la clave para firmar los JWT** (solo la primera vez, en Git Bash):

```bash
cd Sistema-Supermercado-Backend-Gateway
mkdir -p secrets
openssl genrsa -out secrets/jwt_private_key.pem 2048
```

> ⚠️ `secrets/` está en `.gitignore`. Nunca subas la clave privada.

**3. (Opcional)** Copiar `.env.example` como `.env` para cambiar la contraseña de SQL Server (`SA_PASSWORD`) o los
días que Seq guarda los logs (`SEQ_RETENCION`, 30 días por defecto).

**4. Levantar:**

```bash
docker compose up -d --build
```

Cada servicio crea su base de datos, sus stored procedures y sus datos iniciales al arrancar. Si `dotnet restore`
falla por timeout (red lenta), construye las imágenes de una en una: `docker compose build auth-api`, etc.

**Usuario inicial:** `admin@supermercado.com` / `Admin123*` (rol Administrador). Cámbiale la clave después del
primer ingreso.

## URLs

| Servicio | URL |
|---|---|
| **API Gateway** | http://localhost:8000 |
| Swagger de Admin / Auth / Inventory / Order | http://localhost:8001/swagger · 8002 · 8003 · 8004 |
| **Seq** (logs de todos los servicios) | http://localhost:5341 |
| Kafka UI | http://localhost:8090 |
| SQL Server | `localhost,14330` · usuario `sa` |

El frontend y cualquier cliente deben usar **siempre el Gateway**. Rutas: `/auth/...`, `/admin/...`, `/order/...`,
`/inventory/...` (el login es `POST /auth/Usuario/login`).

## Comandos útiles

```bash
docker compose ps                          # estado
docker compose logs -f admin-api           # logs de un servicio
docker compose up -d --build admin-api     # reconstruir un servicio
docker compose down                        # detener (conserva los datos)
docker compose down -v                     # detener y BORRAR bases, Kafka y logs
```

Para ejecutar los servicios sin Docker, la configuración y los problemas comunes, ver el
[manual técnico](docs/Manual%20Tecnico.docx).
