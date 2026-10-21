# Dockerización de la API .NET

## 1. Objetivo
Empaquetar la API del e-commerce (.NET 8) en una imagen Docker multi-stage, ejecutarla localmente y verificar que inicia y responde, inyectando su configuración mediante variables de entorno.

## 2. Dockerfile multi-stage
El repositorio incluye un `Dockerfile` en la raíz. Se revisó y se verificó su funcionamiento.

```dockerfile
# syntax=docker/dockerfile:1

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish Dsw2025Tpi.Api/Dsw2025Tpi.Api.csproj -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
EXPOSE 8080
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Dsw2025Tpi.Api.dll"]
```

| Etapa | Imagen base | Función |
|---|---|---|
| `build` | `dotnet/sdk:8.0` | Restaura paquetes, compila y publica la API en `/app/publish`. |
| `final` | `dotnet/aspnet:8.0` | Imagen liviana de ejecución: solo el runtime y los binarios publicados. |

El código fuente y el SDK no llegan a la imagen final, lo que reduce su tamaño y su superficie de ataque.

## 3. Archivo .dockerignore
Excluye del contexto de construcción los directorios `bin/` y `obj/`, archivos de usuario de Visual Studio (`*.user`, `*.suo`), `.DS_Store`, `.git`, `.gitignore`, `Dockerfile` y `.dockerignore`, para acelerar la construcción y no copiar archivos innecesarios.

## 4. Construcción de la imagen
```bash
docker build -t dsw2025tpi-api:local .
```
Resultado: imagen `dsw2025tpi-api:local` de 348 MB en disco (98,1 MB de contenido comprimido).

## 5. Ejecución local
```bash
docker run -d -p 8080:8080 --name mi-api-backend dsw2025tpi-api:local
```
- `-d`: segundo plano. `-p 8080:8080`: puerto de la PC : puerto del contenedor.
- El contenedor queda en estado `Up` y escucha en el puerto 8080.
- Con el entorno por defecto (`Production`), `http://localhost:8080/swagger` responde **404**: Swagger solo está habilitado en el entorno `Development`. No es un fallo de la API.

## 6. Inyección de variables de entorno
Los valores por defecto del `Dockerfile` se pueden reemplazar al ejecutar el contenedor con `-e`, sin modificar el código ni reconstruir la imagen.

| Variable | Valor por defecto | Función |
|---|---|---|
| `ASPNETCORE_ENVIRONMENT` | `Production` | Entorno de ejecución. |
| `ASPNETCORE_URLS` | `http://+:8080` | URL y puerto donde escucha la API. |
| `ConnectionStrings__Dsw2025TpiEntities` | (definida en `appsettings.json`) | Cadena de conexión a la base de datos. |

Pruebas realizadas:
```bash
# Entorno: Swagger pasa a estar disponible
docker run -d -p 8080:8080 -e ASPNETCORE_ENVIRONMENT=Development --name mi-api-backend dsw2025tpi-api:local
docker exec mi-api-backend printenv ASPNETCORE_ENVIRONMENT   # Development

# URL y puerto: la API escucha en el 9090
docker run -d -p 9090:9090 -e ASPNETCORE_URLS=http://+:9090 -e ASPNETCORE_ENVIRONMENT=Development --name mi-api-backend dsw2025tpi-api:local
```
En ambos casos Swagger respondió correctamente (`/swagger`).

## 7. Verificación de la conexión con la base de datos
La API lee la cadena de conexión con la clave `ConnectionStrings:Dsw2025TpiEntities` (`Program.cs`). En .NET, una variable de entorno con el nombre `ConnectionStrings__Dsw2025TpiEntities` (dos guiones bajos) reemplaza el valor de `appsettings.json`.

**Hallazgo 1: LocalDB.** Al invocar un endpoint que consulta datos (`GET` de productos) la API respondió `500`, y el log del contenedor (`docker logs`) indicó `LocalDB is not supported on this platform`. La cadena por defecto apunta a LocalDB, que existe solo en Windows; el contenedor ejecuta Linux.

**Hallazgo 2: la inyección funciona.** Se ejecutó el contenedor con una cadena de prueba apuntando a un servidor inexistente:
```bash
docker run -d -p 8080:8080 -e ASPNETCORE_ENVIRONMENT=Development \
  -e "ConnectionStrings__Dsw2025TpiEntities=Server=servidor-inexistente,1433;Database=Dsw2025Tpi;User Id=usuario;Password=clave;TrustServerCertificate=True" \
  --name mi-api-backend dsw2025tpi-api:local
```
El error cambió a `Name or service not known` para `servidor-inexistente,1433`, lo que confirma que la variable de entorno reemplazó a la cadena del archivo de configuración.

**Hallazgo 3: migraciones automáticas.** Al iniciar, `Program.cs` ejecuta `Database.Migrate()` sobre los dos contextos de datos (autenticación y dominio) y crea los roles y administradores iniciales. Esto ocurre sin condición de entorno y dentro de un bloque `try`, por lo que la API sigue iniciando aunque la base de datos no esté disponible.

## 8. Observaciones para el despliegue
- La cadena de conexión real se debe inyectar con `ConnectionStrings__Dsw2025TpiEntities`; no debe escribirse en el repositorio.
- Con `ASPNETCORE_ENVIRONMENT=Production`, Swagger no está disponible; será necesario decidir cómo exponerlo en el despliegue.
- Con una base de datos vacía accesible, las tablas se crean al iniciar la API mediante las migraciones.

## 9. Comandos útiles
| Comando | Función |
|---|---|
| `docker images` | Lista las imágenes locales. |
| `docker ps` / `docker ps -a` | Contenedores en ejecución / todos. |
| `docker logs <nombre>` | Muestra los logs del contenedor. |
| `docker exec <nombre> printenv <VAR>` | Consulta una variable dentro del contenedor. |
| `docker rm -f <nombre>` | Elimina un contenedor. |