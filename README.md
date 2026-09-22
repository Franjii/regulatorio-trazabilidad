# Trazabilidad de Estudios

Primera versión funcional del módulo de Regulatorio de Grupo Sur. Usa la misma base técnica definida para Compliance: **Go + HTMX + SQLite**. La interfaz usa HTML renderizado por Go, navegación progresivamente mejorada con HTMX y JavaScript mínimo; si HTMX no carga, los enlaces y formularios siguen funcionando de manera tradicional.

## Alcance incluido

- Dashboard con indicadores y Top 5 de estudios más avanzados.
- Administración de Estudios con `study_id` UUID estable.
- Factibilidad con estados automáticos y transición a Start-Up.
- Historial de Factibilidad con sus cuatro fechas, incluso después de avanzar a Start-Up.
- Start-Up dividido en PRI, ANMAT, CCIS y activación del centro.
- Hasta tres rondas PRI, sin obligar a completar rondas innecesarias.
- Días hábiles de lunes a viernes y días corridos, calculados por el backend.
- Tiempo abierto para hitos pendientes.
- Historial append-only por usuario, campo y grupo de cambios.
- Contactos por estudio y notificación única al cargar Luz Verde.
- Tema claro/oscuro persistido en el navegador.
- Acceso preparado para integrarse con `login.gruposur.ar` mediante encabezado confiable.

La carga del primer paciente screeneado exige que exista Luz Verde. Con ambas fechas, Start-Up queda completado automáticamente.

## Estructura

```text
cmd/server/                 arranque HTTP y worker de notificaciones
configs/                    variables de entorno de ejemplo
data/                       base SQLite local (no se versiona)
docs/                       decisiones funcionales y modelo
internal/auth/              identidad recibida del portal
internal/audit/             historial append-only
internal/database/          conexión y migraciones
internal/feasibility/       lógica de Factibilidad
internal/notifications/     cola y envío de correo
internal/startup/           lógica de Start-Up y tiempos
internal/studies/           estudio central y study_id
internal/web/               rutas y controladores
migrations/                 esquema SQLite versionado
ui/templates/               vistas HTML
ui/static/                  CSS y JavaScript
```

## Puesta en marcha local

Requisitos: Go 1.23 o posterior y WSL/Ubuntu, igual que el proyecto de Compliance.

```bash
cp configs/example.env .env
set -a
source .env
set +a
go mod tidy
go run ./cmd/server
```

Abrir `http://localhost:8082`. Este puerto se usa para no interferir con Compliance, que permanece en `8081`.

Para ver la interfaz con un estudio demostrativo, establecer antes del primer arranque:

```bash
export APP_SEED_DEMO=true
```

El seed solo se ejecuta cuando la base no contiene estudios.

## Integración con login.gruposur.ar

La aplicación no administra roles internos. Espera que el portal o reverse proxy valide el acceso y envíe:

```text
X-User-Email: usuario@gruposur.ar
X-User-Name: Nombre Apellido
```

`APP_TRUSTED_AUTH_HEADER` permite cambiar el nombre del encabezado de email. En producción, la aplicación no debe quedar expuesta directamente: solo el proxy confiable puede inyectar estos encabezados. `APP_DEV_USER_EMAIL` y `APP_DEV_USER_NAME` son únicamente para ejecución local.

## Correo de Luz Verde

Al guardar Luz Verde por primera vez se crea una notificación única `green_light_contract_request`. Los destinatarios se copian desde los contactos activos del estudio marcados para recibir el aviso. Esto evita cambios posteriores de destinatarios en el registro histórico del envío.

Variables opcionales para envío SMTP:

```text
SMTP_HOST
SMTP_PORT
SMTP_USERNAME
SMTP_PASSWORD
SMTP_FROM
APP_BASE_URL
```

Si SMTP no está configurado, la notificación queda pendiente y la interfaz lo informa. En la integración final se puede reemplazar el adaptador SMTP por Gmail API o por el mecanismo que utilice el portal, sin cambiar la regla de negocio ni las tablas.

## Convención de tiempos

- Fechas regulatorias: `YYYY-MM-DD` en SQLite.
- Auditoría: ISO 8601 con zona horaria.
- Zona por defecto: `America/Argentina/Buenos_Aires`.
- Días hábiles: lunes a viernes, sin feriados.
- Intervalos: se excluye el día inicial y se incluye el día final.
- No se guarda texto del tipo “35 días hábiles”; se calcula siempre desde las fechas.

## Pruebas

```bash
go test ./...
```

Antes de subir el repositorio a GitHub:

```bash
go mod tidy
go test ./...
```

No se incluye una base real ni datos del Excel de referencia.
