# Publicación de la imagen en Docker Hub

## 1. Objetivo
Guardar la imagen Docker de la API en un registro público para que el pipeline de GitHub Actions pueda subirla y Azure App Service pueda descargarla.

```
GitHub Actions  →  push  →  Docker Hub  →  pull  →  Azure App Service
```

## 2. Repositorio en Docker Hub
| Dato | Valor |
|---|---|
| Usuario | `santicornejoutn` |
| Repositorio | `ics-backend` |
| Nombre completo de la imagen | `santicornejoutn/ics-backend` |
| Visibilidad | Pública |

Se eligió visibilidad pública porque Azure App Service se configura con *Image type: public* y descarga la imagen sin credenciales.

## 3. Token de acceso
Para que GitHub Actions pueda subir imágenes sin usar la contraseña de la cuenta, se generó un **Personal Access Token** (Docker Hub → Account settings → Personal access tokens).

| Parámetro | Valor |
|---|---|
| Descripción | `github-actions-ics` |
| Permisos | Read, Write, Delete |
| Expiración | Posterior a la fecha de entrega del TP |

El token se muestra una sola vez al generarlo y puede revocarse en cualquier momento. No se guarda en el repositorio.

## 4. Verificación manual
Antes de automatizar, se comprobó que el usuario y el token funcionan:

```bash
docker login -u santicornejoutn
docker tag dsw2025tpi-api:local santicornejoutn/ics-backend:manual-test
docker push santicornejoutn/ics-backend:manual-test
```

Resultado: `Login Succeeded` y la imagen subida con la etiqueta `manual-test`. El identificador de contenido (`sha256:0ccd5cee844f...`) coincide con el ID de la imagen construida localmente.

También se verificó la descarga desde el registro, que es lo que hará Azure:

```bash
docker rmi santicornejoutn/ics-backend:manual-test
docker run -d -p 8080:8080 -e ASPNETCORE_ENVIRONMENT=Development --name prueba-hub santicornejoutn/ics-backend:manual-test
```

Docker descargó la imagen desde Docker Hub y la API respondió en `/swagger`.

## 5. Secretos en GitHub Actions
En el repositorio: Settings → Secrets and variables → Actions → Repository secrets.

| Secreto | Contenido |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Token de acceso con permisos de lectura, escritura y borrado |

Los valores no se guardan en el código ni en los archivos del workflow: el pipeline los referencia como `${{ secrets.NOMBRE }}`, y GitHub los oculta en los logs.

## 6. Etiquetado de imágenes
El pipeline publicará dos etiquetas por cada ejecución:

| Etiqueta | Uso |
|---|---|
| `github.sha` (hash del commit) | Trazabilidad exacta: indica qué commit generó cada imagen. |
| `latest` | La versión más reciente construida; es la que descarga Azure. |

## 7. Observaciones y mejoras pendientes
- La clave `Jwt` está escrita en `appsettings.json` y, por lo tanto, viaja dentro de la imagen pública. Lo adecuado sería inyectarla por variable de entorno (`Jwt__key`), igual que la cadena de conexión.
- La etiqueta `latest` se sobrescribe en cada ejecución. Para identificar versiones se usa el hash del commit.
- El token tiene fecha de vencimiento: si vence, el pipeline falla en el inicio de sesión y hay que generar uno nuevo y actualizar el secreto.
- La etiqueta `manual-test` es solo una prueba y puede eliminarse del repositorio.