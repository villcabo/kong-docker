# Kong Gateway Docker Setup

Este proyecto proporciona una configuración de Kong Gateway 3.11 con Docker Compose, optimizada para producción con PostgreSQL como base de datos.

## 📋 Tabla de Contenidos

- [Prerrequisitos](#-prerrequisitos)
- [Configuración](#-configuración)
- [Instalación](#-instalación)
- [Gestión de Servicios](#-gestión-de-servicios)
- [Configuración SSL/TLS](#-configuración-ssltls)
- [Variables de Entorno](#-variables-de-entorno)
- [Acceso a la Administración](#-acceso-a-la-administración)
- [Monitoreo y Salud](#-monitoreo-y-salud)
- [Solución de Problemas](#-solución-de-problemas)
- [Seguridad](#-seguridad)

## 🚀 Prerrequisitos

- Docker Engine 20.10+
- Docker Compose 2.0+
- Base de datos PostgreSQL externa configurada
- Certificados SSL/TLS (para HTTPS)

## ⚙️ Configuración

### 1. Variables de Entorno

Crea un archivo `.env` en el directorio raíz con las siguientes variables:

```bash
# Configuración de PostgreSQL
KONG_PG_HOST=your-postgres-host.example.com
KONG_PG_PORT=5432
KONG_PG_DATABASE=kong
KONG_PG_USER=kong
KONG_PG_SSL=on
KONG_PG_SSL_VERIFY=on

# Configuración de Kong
KONG_IMAGE=kong:3.11
KONG_LOG_LEVEL=notice
KONG_LOG_DIR=./kong-logs
KONG_ADMIN_GUI_URL=http://localhost:8002

# Configuración opcional para IPs confiables (si usas load balancer)
# KONG_TRUSTED_IPS=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
```

### 2. Configuración de Contraseña (Método Recomendado - Docker Secrets)

Crea el directorio de secrets y configura la contraseña de PostgreSQL:

```bash
mkdir -p secrets
echo "tu_super_pass_segura" > ./secrets/kong_pg_password.txt
chmod 600 ./secrets/kong_pg_password.txt
```

**Alternativa:** Si prefieres usar variables de entorno, modifica el `docker-compose.yml` comentando `KONG_PG_PASSWORD_FILE` y descomentando `KONG_PG_PASSWORD`.

### 3. Directorio de Logs

Kong genera logs que pueden ser útiles para monitoreo y debugging. Crea un directorio dedicado para logs:

```bash
# Crear directorio para logs
mkdir -p ./kong-logs

# Configurar permisos para el usuario Kong (UID 1337)
sudo chown -R 1337:1337 ./kong-logs

# Verificar permisos
ls -la kong-logs/
```

Para habilitar logs persistentes, configura la variable de entorno en tu archivo `.env`:

```bash
# Agregar al archivo .env
KONG_LOG_DIR=./kong-logs
```

> **💡 Nota:** Kong ejecuta con el usuario UID 1337 dentro del contenedor, por lo que es necesario configurar los permisos correctamente para que pueda escribir logs.

> **📁 Estructura de logs:** Con logs persistentes habilitados, Kong generará archivos como `access.log`, `error.log`, etc. en el directorio especificado.

### 4. Certificados SSL/TLS

#### Opción A: Certificados Autofirmados (Desarrollo/Testing)

Para generar certificados autofirmados para desarrollo o testing:

```bash
# Crear directorio SSL
mkdir -p ssl

# Generar clave privada
openssl genrsa -out ssl/tls.key 2048

# Generar certificado autofirmado válido por 365 días
openssl req -new -x509 -key ssl/tls.key -out ssl/tls.crt -days 365 \
  -subj "/C=ES/ST=Madrid/L=Madrid/O=Kong Dev/OU=IT/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,DNS:kong,IP:127.0.0.1,IP:0.0.0.0"

# Asegurar permisos correctos
chmod 600 ssl/tls.key
chmod 644 ssl/tls.crt

echo "✅ Certificados autofirmados generados en ./ssl/"
```

#### Opción B: Certificados de Producción

Para certificados de producción válidos, coloca tus certificados en el directorio `ssl/`:

```bash
mkdir -p ssl
# Copia tus certificados de producción
cp your-certificate.crt ssl/tls.crt
cp your-private-key.key ssl/tls.key
chmod 600 ssl/tls.key
chmod 644 ssl/tls.crt
```

> **⚠️ Nota:** Los certificados autofirmados mostrarán advertencias de seguridad en los navegadores. Para producción, usa certificados válidos de una CA reconocida (Let's Encrypt, etc.).

## 🔧 Instalación

### 1. Migraciones Iniciales (Solo la Primera Vez)

Ejecuta las migraciones de base de datos:

```bash
# Bootstrap inicial de la base de datos
docker compose run --rm kong-migrations

# Para actualizaciones futuras de Kong
docker compose run --rm kong-migrations-up
```

### 2. Iniciar Kong Gateway

```bash
# Levantar Kong en modo daemon
docker compose up -d kong

# Ver logs en tiempo real
docker compose logs -f kong
```

## 🛠️ Gestión de Servicios

```bash
# Iniciar servicios
docker compose up -d

# Detener servicios
docker compose down

# Reiniciar Kong
docker compose restart kong

# Ver estado de los servicios
docker compose ps

# Ver logs en tiempo real
docker compose logs -f kong

# Ver logs específicos del proxy
docker compose logs kong | grep proxy

# Ver logs de errores únicamente
docker compose logs kong | grep ERROR
```

### Logs Persistentes

Si configuraste el directorio `./kong-logs`, puedes acceder a los logs directamente:

```bash
# Ver logs de acceso del proxy
tail -f ./kong-logs/proxy_access.log

# Ver logs de errores del proxy  
tail -f ./kong-logs/proxy_error.log

# Ver logs de la Admin API
tail -f ./kong-logs/admin_access.log
```

## 🔒 Configuración SSL/TLS

Kong está configurado para manejar tráfico HTTPS en el puerto 443. 

### Configuración Básica

1. **Certificados en directorio `ssl/`** (ver sección de configuración)
2. **DNS configurado** para apuntar al servidor Kong
3. **Puertos 80 y 443 accesibles** desde internet

### Verificación de Certificados

```bash
# Verificar certificado generado
openssl x509 -in ssl/tls.crt -text -noout

# Verificar que la clave privada coincida con el certificado
openssl x509 -noout -modulus -in ssl/tls.crt | openssl md5
openssl rsa -noout -modulus -in ssl/tls.key | openssl md5

# Probar conexión HTTPS (después de iniciar Kong)
curl -k https://localhost:443/
```

### Troubleshooting SSL

Si tienes problemas con SSL:

1. **Verificar permisos de archivos:**
   ```bash
   ls -la ssl/
   # tls.key debe tener permisos 600
   # tls.crt debe tener permisos 644
   ```

2. **Verificar formato de certificados:**
   ```bash
   # El certificado debe comenzar con -----BEGIN CERTIFICATE-----
   head -n 1 ssl/tls.crt
   
   # La clave debe comenzar con -----BEGIN PRIVATE KEY----- o -----BEGIN RSA PRIVATE KEY-----
   head -n 1 ssl/tls.key
   ```

3. **Logs de Kong para errores SSL:**
   ```bash
   docker compose logs kong | grep -i ssl
   ```

## 🌐 Variables de Entorno

| Variable | Descripción | Valor por Defecto |
|----------|-------------|-------------------|
| `KONG_PG_HOST` | Host de PostgreSQL | - |
| `KONG_PG_PORT` | Puerto de PostgreSQL | `5432` |
| `KONG_PG_DATABASE` | Nombre de la base de datos | `kong` |
| `KONG_PG_USER` | Usuario de PostgreSQL | `kong` |
| `KONG_PG_SSL` | Habilitar SSL para PostgreSQL | `on` |
| `KONG_LOG_LEVEL` | Nivel de logging | `notice` |
| `KONG_LOG_DIR` | Directorio para logs persistentes | `./logs` |
| `KONG_ADMIN_GUI_URL` | URL pública del Kong Manager | `http://localhost:8002` |

## 🖥️ Acceso a la Administración

Por defecto, los puertos de administración NO están expuestos públicamente por seguridad.

### Acceso Local (Desarrollo)

Para acceder desde la misma máquina, descomenta las siguientes líneas en `docker-compose.yml`:

```yaml
ports:
  - "127.0.0.1:8001:8001"   # Admin API HTTP
  - "127.0.0.1:8002:8002"   # Kong Manager HTTP
  - "127.0.0.1:8444:8444"   # Admin API HTTPS
  - "127.0.0.1:8445:8445"   # Kong Manager HTTPS
  - "127.0.0.1:8100:8100"   # Status API
```

Luego reinicia el servicio:

```bash
docker compose restart kong
```

### URLs de Administración

- **Kong Manager:** http://localhost:8002
- **Admin API:** http://localhost:8001
- **Status API:** http://localhost:8100/status

### Acceso Remoto (Producción)

Para acceso remoto seguro, usa uno de estos métodos:

1. **Túnel SSH:**
   ```bash
   ssh -L 8002:localhost:8002 usuario@servidor-kong
   ```

2. **VPN** hacia la red privada

3. **Reverse Proxy** con autenticación (nginx, traefik, etc.)

## 📊 Monitoreo y Salud

Kong incluye endpoints de salud configurados:

```bash
# Verificar estado del gateway
curl http://localhost:8100/status

# Verificar preparación del servicio
curl http://localhost:8100/status/ready

# Información de la instancia Kong
curl http://localhost:8001/
```

## 🔍 Solución de Problemas

### Kong no inicia

1. Verificar conectividad a PostgreSQL:
   ```bash
   docker compose run --rm kong kong config db_import /dev/null
   ```

2. Revisar logs:
   ```bash
   docker compose logs kong
   ```

3. Verificar configuración:
   ```bash
   docker compose config
   ```

### Problemas de Conexión a Base de Datos

1. Verificar variables de entorno en `.env`
2. Comprobar que PostgreSQL esté accesible
3. Validar credenciales de acceso

### Problemas SSL/TLS

1. Verificar que los certificados existan en `ssl/`
2. Comprobar permisos de archivos (600)
3. Validar formato de certificados

## 🛡️ Seguridad

### Mejores Prácticas

1. **No expongas puertos de admin públicamente**
2. **Usa Docker secrets para contraseñas**
3. **Mantén Kong actualizado**
4. **Configura SSL/TLS apropiadamente**
5. **Limita acceso con IPs confiables cuando uses load balancer**
6. **Monitora logs regularmente**

### Configuración de Red

Kong está configurado con una red personalizada `kong-net` para aislar el tráfico.

### Estructura de Puertos

- **80:** HTTP Proxy (público)
- **443:** HTTPS Proxy (público)  
- **8001:** Admin API (privado)
- **8002:** Kong Manager (privado)
- **8100:** Status API (privado)
- **8444:** Admin API HTTPS (privado)
- **8445:** Kong Manager HTTPS (privado)

---

## 📚 Recursos Adicionales

- [Documentación oficial de Kong](https://docs.konghq.com/)
- [Kong Docker Hub](https://hub.docker.com/_/kong)
- [Kong Configuration Reference](https://docs.konghq.com/gateway/latest/reference/configuration/)

## 📄 Licencia

Este proyecto está bajo la licencia MIT. Ver archivo `LICENSE` para más detalles.