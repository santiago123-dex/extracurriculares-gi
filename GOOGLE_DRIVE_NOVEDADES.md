# Integración Google Drive — Novedades diarias

Cómo se conectó el Google Drive del colegio (carpeta de AppSheet donde se cargan las
novedades) con el backend, para que los profesores vean en su lista de asistencia qué
estudiantes salen temprano, se ausentan, etc.

Flujo: **Google Drive (`.xlsx` de AppSheet) → Sync cada 10 min → tabla `Novedad` →
endpoint del profesor → chip ámbar en la asistencia**.

---

## 1. Qué se hizo en Google Cloud Console

### 1.1 Proyecto
1. Ir a https://console.cloud.google.com → **Crear proyecto** (o usar uno existente).
   - Proyecto usado: `extracurriculares-gi`
   - ID del proyecto: `extracurriculares-gi`

### 1.2 Crear la Service Account
1. **IAM y administración → Cuentas de servicio → Crear cuenta de servicio**.
2. Nombre: `novedades-gi`, ID: `novedades-gi` (genera el email).
3. **No asignar roles** (la cuenta solo accede a Drive como lector de una carpeta
   compartida; los permisos de Drive se dan por compartir la carpeta, no por roles de GCP).
4. Cuenta creada:
   `novedades-gi@extracurriculares-gi.iam.gserviceaccount.com`

### 1.3 Generar la llave JSON
1. En la cuenta de servicio → pestaña **Claves → Agregar clave → Crear clave nueva**.
2. Tipo **JSON** → **Crear** → se descarga el archivo:
   `extracurriculares-gi-075620ddaa17.json`
3. Guardar ese JSON es crítico: es la credencial. **Nunca subirlo a git** (el
   `.gitignore` del repo solo exceptúa `.env`; si querés copiar el JSON al repo,
   agregalo también). En producción va en base64 como variable de entorno, ver §3.

### 1.4 Habilitar la "Google Drive API" en el proyecto
Para que el backend pueda llamar a `drive.googleapis.com` (listar la carpeta y
descargar el `.xlsx`) el proyecto **debe tener activa la Google Drive API**:

1. **APIs y servicios → Biblioteca** → buscar **Google Drive API** → **Habilitar**.

**En este proyecto ya está activa** (sin que hubiéramos ido a habilitarla): se
comprueba empíricamente porque el sync corre sin errores — los logs muestran
`[Novedades] Sync OK: 1 archivos, 19 novedades` repetidamente. Si la API estuviera
apagada, la primera llamada fallaría con `403 Access Not Configured`.

> Diagnóstico rápido: si el sync fallara con `403`, ir a *APIs y servicios → Biblioteca
> → Google Drive API → Habilitar* y reintentar. La autenticación de la service account
> (`oauth2.googleapis.com/token`) no depende de eso — es siempre libre.

---

## 2. Compartir la carpeta de Drive con la cuenta de servicio

1. En Google Drive, abrir la carpeta de novedades (la que AppSheet actualiza).
2. **Compartir → Agregar personas y grupos** → pegar el email de la service account:
   `novedades-gi@extracurriculares-gi.iam.gserviceaccount.com`
3. Rol: **Lector** (solo lectura; nunca Editor).
4. Copiar el **ID de la carpeta** (el UUID de la URL de Drive) para las variables de
   entorno:
   `GOOGLE_DRIVE_FOLDER_ID=1Ksnx4PCAQfnhs-TjYJUJx6Aa0qR0MvKu`

> Nota de riesgo: el alcance es mínimo — la cuenta solo puede VER esa carpeta y nada
> más del proyecto. Si se revoca el share, el sync deja de entrar (falla controlada).

---

## 3. Cómo quedó configurado el backend

### 3.1 Variables de entorno (local: `.env` / producción: Render)
| Variable | Valor |
|----------|-------|
| `GOOGLE_SERVICE_ACCOUNT_JSON` | el contenido del JSON `.json` convertido a **base64** en una sola línea |
| `GOOGLE_DRIVE_FOLDER_ID` | el UUID de la carpeta compartida |
| `NOVEDADES_SYNC_MINUTES` | `10` (cada cuántos minutos se sincroniza) |

`GOOGLE_SERVICE_ACCOUNT_JSON` va en base64 porque las variables de entorno no pueden
tener saltos de línea. Para generarlo desde el JSON descargado:

```bash
base64 -w0 extracurriculares-gi-075620ddaa17.json
# o con node:
node -e 'const fs=require("fs");console.log(Buffer.from(fs.readFileSync("extracurriculares-gi-075620ddaa17.json","utf8")).toString("base64"))'
```

### 3.2 En render.yaml / dashboard de Render
Las 3 variables se setean igual que `DATABASE_URL` o `JWT_SECRET`: en el dashboard
(**Environment**) o en `render.yaml` con `sync: false`. El backend **no arranca el
sync** si faltan `GOOGLE_SERVICE_ACCOUNT_JSON` o `GOOGLE_DRIVE_FOLDER_ID` (se ve el log
`[Novedades] Sync programado...`).

### 3.3 Código (referencia)
| Archivo | Qué hace |
|---------|----------|
| `backend/src/modules/novedades/googleDrive.service.ts` | Auth con la service account (JWT RS256) y descarga del `.xlsx` / Google Sheets |
| `backend/src/modules/novedades/novedades.parser.ts` | Normaliza headers del Excel, fechas `dd/mm/yyyy`, booleans, y expande una fila por estudiante |
| `backend/src/modules/novedades/novedades.service.ts` | Sync (reemplazo por archivo), consultas por estudiante y estado |
| `backend/src/modules/novedades/novedades.controller.ts` | Endpoints del profesor |
| `backend/src/server.ts` | Job periódico (primer run a los 5s, luego cada `NOVEDADES_SYNC_MINUTES`) |
| `backend/prisma/schema.prisma` | Modelo `Novedad` (verificado con `prisma db push` en Neon) |

---

## 4. Verificación

1. Logs del backend:
   - `[Novedades] Sync programado cada 10 minutos` → variables leídas OK.
   - `[Novedades] Sync OK: N archivos, N novedades` → la cuenta lee la carpeta.
   - `[Novedades] Sync con errores: <motivo>` → permisos, carpeta no compartida, o
     formato del Excel distinto.
2. Login de profesor (p. ej. `isait@gi.edu.co`) → iniciar la clase del día → el
   estudiante con novedad activa muestra el **chip ámbar** en la lista de asistencia.
3. "Activa" = novedad de **hoy** (zona horaria Bogotá). Las de otros días quedan
   históricas y no se muestran en el chip.

---

## 5. Seguridad / notas

- La service account solo lee la carpeta compartida (rol Lector). Sin permisos para
  editar, listar otras carpetas ni tocar nada del proyecto GCP.
- El JSON de la llave es un secreto: en `.env` (gitignoreado) y en Render como variable.
  **Nunca pegar el JSON crudo en un commit ni en logs.**
- El sync es **reemplazo por archivo** (delete + insert en transacción): la fuente de
  verdad es el Drive de AppSheet. Si se borra una fila en el Excel, desaparece también
  en la base.
- En la base local el schema se sincroniza con `prisma db push`; en producción ya se
  corrió contra Neon (tabla `Novedad`). No hay migraciones comiteadas (ver `DEPLOY.md`).