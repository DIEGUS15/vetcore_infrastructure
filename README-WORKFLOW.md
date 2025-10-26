# VetCore - Guía de Trabajo con Microservicios

Esta guía explica cómo trabajar con la arquitectura de microservicios de VetCore tanto para desarrollo individual como para integración completa.

## Estructura del Proyecto

Cada microservicio tiene su propio repositorio (o puede tenerlo) y su propio `docker-compose.yml`:

```
vetcore-auth/              → Repo individual con docker-compose.yml
vetcore-patients/          → Repo individual con docker-compose.yml
vetcore-apigateway/        → Repo individual con docker-compose.yml
vetcore-frontend/          → Repo individual con docker-compose.yml
vetcore-infrastructure/    → Repo de orquestación (este)
```

## Configuración Inicial

### 1. Crear cuenta en Docker Hub (opcional pero recomendado)

1. Ve a https://hub.docker.com/signup
2. Crea tu cuenta (ejemplo: usuario `juanperez`)
3. Haz login desde terminal:
   ```bash
   docker login
   # Username: juanperez
   # Password: tu-password
   ```

### 2. Actualizar nombres de imágenes

En `docker-compose.yml`, reemplaza `vetcore/` con tu usuario de Docker Hub:

```yaml
# Cambiar esto:
image: vetcore/auth-service:latest

# Por esto (con tu usuario):
image: juanperez/vetcore-auth:latest
```

## Flujos de Trabajo

### Opción 1: Desarrollo Individual de un Servicio

Cuando trabajas SOLO en un microservicio (sin necesidad de otros servicios):

#### Sin Docker (más rápido):
```bash
cd vetcore_auth_msvc
npm install
npm run dev
```

#### Con Docker (para probar el contenedor):
```bash
cd vetcore_auth_msvc
docker-compose up
```

Esto levantará:
- El servicio Auth
- Su base de datos MySQL

**Ventaja:** Desarrollas y pruebas rápidamente sin levantar todo el sistema.

---

### Opción 2: Integración Completa (Todos los Servicios)

Cuando necesitas probar el flujo completo (frontend → gateway → auth → patients):

#### Paso 1: Construir las imágenes

Primero debes construir las imágenes de cada servicio:

```bash
# 1. Construir Auth Service
cd vetcore_auth_msvc
docker build -t juanperez/vetcore-auth:latest .

# 2. Construir Patients Service
cd ../vetcore_patients_msvc
docker build -t juanperez/vetcore-patients:latest .

# 3. Construir API Gateway
cd ../vetcore_apigateway_msvc
docker build -t juanperez/vetcore-apigateway:latest .
```

**Nota:** Reemplaza `juanperez` con tu usuario de Docker Hub (o cualquier nombre local si no usas Docker Hub).

#### Paso 2: Levantar todos los servicios

```bash
cd vetcore_infrastructure
docker-compose up -d
```

Esto levantará:
- Bases de datos (MySQL para Auth y Patients)
- Auth Service
- Patients Service
- API Gateway

**Ventaja:** Pruebas de integración completas, ambiente similar a producción.

---

### Opción 3: Publicar Imágenes (Para compartir o producción)

Si quieres que otros puedan usar tus imágenes sin tener el código fuente:

#### Paso 1: Construir imágenes (si aún no lo hiciste)
```bash
cd vetcore_auth_msvc
docker build -t juanperez/vetcore-auth:latest .
```

#### Paso 2: Subir a Docker Hub
```bash
docker push juanperez/vetcore-auth:latest
```

#### Paso 3: Repetir para cada servicio
```bash
cd ../vetcore_patients_msvc
docker build -t juanperez/vetcore-patients:latest .
docker push juanperez/vetcore-patients:latest

cd ../vetcore_apigateway_msvc
docker build -t juanperez/vetcore-apigateway:latest .
docker push juanperez/vetcore-apigateway:latest
```

Ahora cualquiera puede hacer:
```bash
cd vetcore_infrastructure
docker-compose pull  # Descarga las imágenes
docker-compose up    # Levanta todo
```

---

## Flujo de Trabajo Recomendado

### Para Desarrollo Diario:

1. **Desarrollo rápido sin Docker:**
   ```bash
   cd vetcore_auth_msvc
   npm run dev
   ```

2. **Probar con Docker (solo tu servicio):**
   ```bash
   cd vetcore_auth_msvc
   docker-compose up
   ```

3. **Cuando haces cambios y quieres probar integración:**
   ```bash
   # Reconstruir la imagen del servicio modificado
   cd vetcore_auth_msvc
   docker build -t juanperez/vetcore-auth:latest .

   # Reiniciar solo ese servicio en infraestructura
   cd ../vetcore_infrastructure
   docker-compose up -d auth-service
   ```

### Para Pruebas de Integración:

```bash
# 1. Construir todas las imágenes (solo cuando hay cambios)
cd vetcore_auth_msvc
docker build -t juanperez/vetcore-auth:latest .
cd ../vetcore_patients_msvc
docker build -t juanperez/vetcore-patients:latest .
cd ../vetcore_apigateway_msvc
docker build -t juanperez/vetcore-apigateway:latest .

# 2. Levantar todo
cd ../vetcore_infrastructure
docker-compose up -d

# 3. Ver logs
docker-compose logs -f

# 4. Detener todo
docker-compose down
```

---

## Comandos Útiles

### Ver imágenes construidas:
```bash
docker images | grep vetcore
```

### Ver contenedores corriendo:
```bash
docker ps
```

### Ver logs de un servicio:
```bash
cd vetcore_infrastructure
docker-compose logs -f auth-service
```

### Reconstruir y reiniciar un servicio específico:
```bash
# Si modificaste Auth Service
cd vetcore_auth_msvc
docker build -t juanperez/vetcore-auth:latest .

cd ../vetcore_infrastructure
docker-compose up -d auth-service
```

### Limpiar todo:
```bash
# Detener servicios
docker-compose down

# Eliminar imágenes viejas
docker image prune -a

# Eliminar volúmenes (⚠️ borra datos de BD)
docker-compose down -v
```

---

## Diferencias entre docker-compose.yml

### En cada servicio (`vetcore_auth_msvc/docker-compose.yml`):
- **Propósito:** Desarrollo individual del servicio
- **Contiene:** Solo ese servicio + sus dependencias directas (ej: su BD)
- **Network:** Network privada del servicio
- **Puertos:** Expone puertos del servicio

### En infraestructura (`vetcore_infrastructure/docker-compose.yml`):
- **Propósito:** Orquestación completa del sistema
- **Contiene:** TODOS los servicios
- **Network:** Network compartida (`vetcore-network`)
- **Usa:** Referencias a imágenes (`image:`) en lugar de `build:`

---

## Notas Importantes

1. **JWT_SECRET:** Debe ser el mismo en todos los servicios que validan tokens (Auth, Patients, API Gateway).

2. **Nombres de imágenes:** Si no usas Docker Hub, puedes usar nombres locales:
   ```yaml
   image: vetcore-auth:latest  # Solo funciona localmente
   ```

3. **Hot reload:** Para desarrollo con hot-reload, descomenta las líneas de `volumes` en los `docker-compose.yml` de cada servicio:
   ```yaml
   volumes:
     - ./src:/app/src
     - /app/node_modules
   ```

4. **Versiones:** Puedes usar tags específicos en lugar de `latest`:
   ```bash
   docker build -t juanperez/vetcore-auth:v1.0.0 .
   docker build -t juanperez/vetcore-auth:v1.0.1 .
   ```

---

## Troubleshooting

### Error: "image not found"
Construye la imagen primero:
```bash
cd vetcore_auth_msvc
docker build -t juanperez/vetcore-auth:latest .
```

### Los servicios no se comunican
Verifica que estén en la misma red:
```bash
docker network inspect vetcore_vetcore-network
```

### Puerto ya en uso
Cambia el puerto en `docker-compose.yml`:
```yaml
ports:
  - "3002:3000"  # Mapea puerto 3002 del host al 3000 del container
```

### Cambios no se reflejan
Reconstruye la imagen:
```bash
docker build -t juanperez/vetcore-auth:latest .
docker-compose up -d auth-service --force-recreate
```
