# Instrucciones para Instalar Chat en Tiempo Real en tu VPS

## 📋 Requisitos Previos
- Un VPS con Ubuntu 20.04 o superior
- Acceso root o sudo
- Dominio apuntando a tu VPS (opcional pero recomendado)

## 🔧 Paso 1: Instalar Node.js y npm

```bash
# Actualizar el sistema
sudo apt update && sudo apt upgrade -y

# Instalar Node.js 20.x (versión LTS)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verificar instalación
node --version
npm --version
```

## 🗄️ Paso 2: Instalar PostgreSQL

```bash
# Instalar PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Iniciar el servicio
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Verificar que está corriendo
sudo systemctl status postgresql
```

## 🔐 Paso 3: Configurar PostgreSQL

```bash
# Cambiar a usuario postgres
sudo -u postgres psql

# Dentro de PostgreSQL, ejecutar:
CREATE DATABASE tiktokdownloader;
CREATE USER tiktokuser WITH ENCRYPTED PASSWORD 'tu_contraseña_segura_aqui';
GRANT ALL PRIVILEGES ON DATABASE tiktokdownloader TO tiktokuser;

# Si usas PostgreSQL 15+, también ejecutar:
\c tiktokdownloader
GRANT ALL ON SCHEMA public TO tiktokuser;

# Salir de PostgreSQL
\q
```

## 📦 Paso 4: Instalar yt-dlp (para descargas de TikTok)

```bash
# Instalar yt-dlp
sudo apt install -y python3-pip
sudo pip3 install -U yt-dlp

# Verificar instalación
yt-dlp --version
```

## 🚀 Paso 5: Clonar y Configurar la Aplicación

```bash
# Ir al directorio donde quieres instalar la app
cd /var/www  # o el directorio que prefieras

# Clonar tu repositorio (reemplaza con tu URL de Git)
git clone <tu-repositorio-git>
cd <nombre-carpeta>

# O si subes archivos manualmente, asegúrate de tener todos los archivos del proyecto

# Instalar dependencias
npm install
```

## 🔑 Paso 6: Configurar Variables de Entorno

```bash
# Crear archivo .env.local
nano .env.local

# Agregar esta línea (reemplaza con tus datos):
DATABASE_URL=postgresql://tiktokuser:tu_contraseña_segura_aqui@localhost:5432/tiktokdownloader

# Guardar y cerrar (Ctrl+X, luego Y, luego Enter)
```

## 🏗️ Paso 7: Crear las Tablas en la Base de Datos

```bash
# Ejecutar migración de la base de datos
npm run db:push
```

## 🔨 Paso 8: Compilar la Aplicación

```bash
# Compilar el frontend y backend
npm run build
```

## 🌐 Paso 9: Instalar PM2 (Process Manager)

```bash
# Instalar PM2 globalmente
sudo npm install -g pm2

# Iniciar la aplicación con PM2
pm2 start npm --name "tiktok-chat" -- run start

# Guardar configuración de PM2
pm2 save

# Configurar PM2 para iniciar automáticamente al reiniciar el servidor
pm2 startup
# Copiar y ejecutar el comando que te muestre PM2

# Ver logs de la aplicación
pm2 logs tiktok-chat

# Ver estado
pm2 status
```

## 🔒 Paso 10: Configurar Nginx (Proxy Inverso)

```bash
# Instalar Nginx
sudo apt install -y nginx

# Crear configuración del sitio
sudo nano /etc/nginx/sites-available/tiktok-chat

# Pegar esta configuración (reemplaza tu-dominio.com con tu dominio):
```

```nginx
server {
    listen 80;
    server_name tu-dominio.com www.tu-dominio.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /ws {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 86400;
    }
}
```

```bash
# Guardar y cerrar (Ctrl+X, Y, Enter)

# Crear enlace simbólico para habilitar el sitio
sudo ln -s /etc/nginx/sites-available/tiktok-chat /etc/nginx/sites-enabled/

# Eliminar configuración por defecto si existe
sudo rm /etc/nginx/sites-enabled/default

# Verificar configuración
sudo nginx -t

# Reiniciar Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## 🔐 Paso 11: Configurar SSL con Let's Encrypt (HTTPS)

```bash
# Instalar Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtener certificado SSL (reemplaza tu-dominio.com)
sudo certbot --nginx -d tu-dominio.com -d www.tu-dominio.com

# Seguir las instrucciones en pantalla
# El certificado se renovará automáticamente
```

## 🔥 Paso 12: Configurar Firewall

```bash
# Permitir SSH, HTTP y HTTPS
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443

# Habilitar firewall
sudo ufw enable

# Ver estado
sudo ufw status
```

## ✅ Verificar que Todo Funciona

```bash
# Ver logs de la aplicación
pm2 logs tiktok-chat

# Ver estado de todos los servicios
pm2 status
sudo systemctl status postgresql
sudo systemctl status nginx

# Probar la aplicación
# Abre tu navegador y ve a: https://tu-dominio.com
```

## 📊 Comandos Útiles para Administrar la Aplicación

```bash
# Reiniciar la aplicación
pm2 restart tiktok-chat

# Detener la aplicación
pm2 stop tiktok-chat

# Iniciar la aplicación
pm2 start tiktok-chat

# Ver logs en tiempo real
pm2 logs tiktok-chat --lines 100

# Ver uso de recursos
pm2 monit

# Actualizar la aplicación después de hacer cambios
cd /var/www/<nombre-carpeta>
git pull  # si usas Git
npm install  # si hay nuevas dependencias
npm run build
pm2 restart tiktok-chat
```

## 🔍 Solución de Problemas

### El chat no se conecta
```bash
# Verificar que la app esté corriendo
pm2 status

# Ver logs de errores
pm2 logs tiktok-chat --err

# Verificar WebSocket en Nginx
sudo nginx -t
sudo systemctl restart nginx
```

### Los mensajes no se guardan
```bash
# Verificar conexión a PostgreSQL
sudo -u postgres psql -d tiktokdownloader -c "SELECT * FROM chat_messages LIMIT 10;"

# Verificar que la tabla existe
sudo -u postgres psql -d tiktokdownloader -c "\dt"
```

### Error de permisos en PostgreSQL
```bash
sudo -u postgres psql
GRANT ALL PRIVILEGES ON DATABASE tiktokdownloader TO tiktokuser;
\c tiktokdownloader
GRANT ALL ON SCHEMA public TO tiktokuser;
GRANT ALL ON ALL TABLES IN SCHEMA public TO tiktokuser;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO tiktokuser;
\q
```

## 🎉 ¡Listo!

Tu chat en tiempo real ya está funcionando en tu VPS con:
- ✅ Mensajes persistentes en PostgreSQL
- ✅ Chat en tiempo real con WebSocket
- ✅ HTTPS seguro
- ✅ Reinicio automático si el servidor se reinicia
- ✅ Los mensajes NO se borran al cerrar el navegador o cambiar de sección

Accede a tu aplicación en: **https://tu-dominio.com**

---

## 📝 Notas Importantes

1. **Seguridad**: Cambia `tu_contraseña_segura_aqui` por una contraseña fuerte
2. **Backups**: Configura backups automáticos de PostgreSQL
3. **Actualizaciones**: Mantén el sistema actualizado con `sudo apt update && sudo apt upgrade`
4. **Monitoreo**: Revisa los logs regularmente con `pm2 logs`
