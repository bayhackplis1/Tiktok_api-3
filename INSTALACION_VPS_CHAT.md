# 🚀 Guía Completa: Instalación del Chat Global en tu VPS

Esta guía te ayudará a instalar paso a paso el chat en tiempo real con mensajes persistentes en tu VPS.

---

## 📋 PARTE 1: PREPARAR EL SERVIDOR

### Paso 1.1: Actualizar el sistema
```bash
sudo apt update && sudo apt upgrade -y
```

### Paso 1.2: Instalar Node.js 20.x
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### Paso 1.3: Verificar instalación
```bash
node --version
npm --version
```
Deberías ver algo como: `v20.x.x` y `10.x.x`

---

## 🗄️ PARTE 2: INSTALAR Y CONFIGURAR POSTGRESQL

### Paso 2.1: Instalar PostgreSQL
```bash
sudo apt install -y postgresql postgresql-contrib
```

### Paso 2.2: Iniciar PostgreSQL
```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql
```
Presiona `q` para salir del status

### Paso 2.3: Crear la base de datos y usuario
```bash
sudo -u postgres psql
```

Ahora estás dentro de PostgreSQL. Ejecuta estos comandos **uno por uno**:

```sql
CREATE DATABASE tiktokdownloader;
CREATE USER tiktokuser WITH ENCRYPTED PASSWORD 'MiPassword123Seguro';
GRANT ALL PRIVILEGES ON DATABASE tiktokdownloader TO tiktokuser;
```

**Si usas PostgreSQL 15 o superior**, también ejecuta:
```sql
\c tiktokdownloader
GRANT ALL ON SCHEMA public TO tiktokuser;
GRANT ALL ON ALL TABLES IN SCHEMA public TO tiktokuser;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO tiktokuser;
```

Sal de PostgreSQL:
```sql
\q
```

---

## 📦 PARTE 3: INSTALAR DEPENDENCIAS

### Paso 3.1: Instalar yt-dlp (para descargas de TikTok)
```bash
sudo apt install -y python3-pip curl
sudo pip3 install -U yt-dlp
```

### Paso 3.2: Verificar instalación
```bash
yt-dlp --version
```

---

## 🎯 PARTE 4: SUBIR Y CONFIGURAR TU APLICACIÓN

### Paso 4.1: Crear directorio para la app
```bash
sudo mkdir -p /var/www/tiktok-app
sudo chown -R $USER:$USER /var/www/tiktok-app
cd /var/www/tiktok-app
```

### Paso 4.2: Subir tus archivos
Tienes 3 opciones:

**Opción A - Con Git:**
```bash
git clone https://github.com/tu-usuario/tu-repositorio.git .
```

**Opción B - Con SFTP/SCP:**
Usa FileZilla o WinSCP para subir todos los archivos a `/var/www/tiktok-app`

**Opción C - Con rsync (desde tu computadora):**
```bash
rsync -avz /ruta/local/del/proyecto/ usuario@tu-servidor.com:/var/www/tiktok-app/
```

### Paso 4.3: Instalar dependencias de Node.js
```bash
cd /var/www/tiktok-app
npm install
```

---

## 🔑 PARTE 5: CONFIGURAR VARIABLES DE ENTORNO

### Paso 5.1: Crear archivo .env.local
```bash
nano .env.local
```

### Paso 5.2: Agregar esta línea (cambia la contraseña por la que usaste):
```
DATABASE_URL=postgresql://tiktokuser:MiPassword123Seguro@localhost:5432/tiktokdownloader
```

### Paso 5.3: Guardar y cerrar
- Presiona `Ctrl + X`
- Presiona `Y`
- Presiona `Enter`

---

## 🏗️ PARTE 6: CREAR TABLAS EN LA BASE DE DATOS

### Paso 6.1: Ejecutar migración
```bash
npm run db:push
```

Deberías ver: `[✓] Changes applied`

### Paso 6.2: Verificar que las tablas se crearon
```bash
sudo -u postgres psql -d tiktokdownloader -c "\dt"
```

Deberías ver las tablas: `chat_messages` y `users`

---

## 🔨 PARTE 7: COMPILAR LA APLICACIÓN

### Paso 7.1: Compilar
```bash
npm run build
```

Espera a que termine (puede tomar 1-2 minutos)

### Paso 7.2: Probar que funciona (opcional)
```bash
npm run start
```

Presiona `Ctrl + C` para detener después de verificar que arranca sin errores

---

## 🔄 PARTE 8: INSTALAR PM2 (GESTOR DE PROCESOS)

### Paso 8.1: Instalar PM2 globalmente
```bash
sudo npm install -g pm2
```

### Paso 8.2: Iniciar la aplicación con PM2
```bash
pm2 start npm --name "tiktok-chat" -- run start
```

### Paso 8.3: Configurar PM2 para inicio automático
```bash
pm2 save
pm2 startup
```

**IMPORTANTE:** PM2 te mostrará un comando, **cópialo y ejecútalo**. Será algo como:
```bash
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u tu-usuario --hp /home/tu-usuario
```

### Paso 8.4: Verificar que está corriendo
```bash
pm2 status
pm2 logs tiktok-chat --lines 20
```

---

## 🌐 PARTE 9: INSTALAR Y CONFIGURAR NGINX

### Paso 9.1: Instalar Nginx
```bash
sudo apt install -y nginx
```

### Paso 9.2: Crear configuración del sitio
```bash
sudo nano /etc/nginx/sites-available/tiktok-chat
```

### Paso 9.3: Pegar esta configuración (REEMPLAZA `tu-dominio.com`):
```nginx
server {
    listen 80;
    server_name tu-dominio.com www.tu-dominio.com;

    client_max_body_size 100M;

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
        proxy_send_timeout 86400;
    }
}
```

### Paso 9.4: Guardar (Ctrl+X, Y, Enter)

### Paso 9.5: Habilitar el sitio
```bash
sudo ln -s /etc/nginx/sites-available/tiktok-chat /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
```

### Paso 9.6: Verificar configuración
```bash
sudo nginx -t
```

Deberías ver: `syntax is ok` y `test is successful`

### Paso 9.7: Reiniciar Nginx
```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```

---

## 🔐 PARTE 10: CONFIGURAR SSL (HTTPS)

### Paso 10.1: Instalar Certbot
```bash
sudo apt install -y certbot python3-certbot-nginx
```

### Paso 10.2: Obtener certificado SSL (REEMPLAZA `tu-dominio.com`)
```bash
sudo certbot --nginx -d tu-dominio.com -d www.tu-dominio.com
```

Sigue las instrucciones:
1. Ingresa tu email
2. Acepta los términos (Y)
3. Decide si quieres recibir emails (Y o N)
4. Selecciona opción 2 (redirect HTTP a HTTPS)

---

## 🔥 PARTE 11: CONFIGURAR FIREWALL

### Paso 11.1: Configurar UFW
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Confirma con `y`

### Paso 11.2: Verificar estado
```bash
sudo ufw status
```

---

## ✅ PARTE 12: VERIFICAR QUE TODO FUNCIONA

### Paso 12.1: Ver logs de la aplicación
```bash
pm2 logs tiktok-chat
```

### Paso 12.2: Verificar servicios
```bash
pm2 status
sudo systemctl status nginx
sudo systemctl status postgresql
```

### Paso 12.3: Probar en el navegador
Abre tu navegador y ve a: **https://tu-dominio.com**

### Paso 12.4: Probar el chat
1. Haz clic en "Chat Global"
2. Ingresa tu nombre y edad
3. Envía un mensaje
4. Ve a otra pestaña (Inicio)
5. Regresa al "Chat Global"
6. **¡Deberías ver tu mensaje guardado!**

---

## 🔧 COMANDOS ÚTILES PARA ADMINISTRAR

### Ver logs en tiempo real
```bash
pm2 logs tiktok-chat
```

### Reiniciar la aplicación
```bash
pm2 restart tiktok-chat
```

### Detener la aplicación
```bash
pm2 stop tiktok-chat
```

### Iniciar la aplicación
```bash
pm2 start tiktok-chat
```

### Ver uso de recursos
```bash
pm2 monit
```

### Actualizar la aplicación (después de hacer cambios)
```bash
cd /var/www/tiktok-app
git pull  # si usas git
npm install  # si hay nuevas dependencias
npm run build
pm2 restart tiktok-chat
```

### Ver mensajes en la base de datos
```bash
sudo -u postgres psql -d tiktokdownloader -c "SELECT * FROM chat_messages ORDER BY timestamp DESC LIMIT 10;"
```

### Limpiar mensajes antiguos (opcional)
```bash
sudo -u postgres psql -d tiktokdownloader -c "DELETE FROM chat_messages WHERE timestamp < NOW() - INTERVAL '30 days';"
```

---

## 🐛 SOLUCIÓN DE PROBLEMAS

### Problema: La aplicación no arranca
```bash
# Ver logs detallados
pm2 logs tiktok-chat --err

# Verificar que el puerto 5000 está libre
sudo lsof -i :5000

# Reiniciar todo
pm2 restart tiktok-chat
sudo systemctl restart nginx
```

### Problema: Los mensajes no se guardan
```bash
# Verificar conexión a PostgreSQL
sudo -u postgres psql -d tiktokdownloader -c "SELECT 1;"

# Verificar tablas
sudo -u postgres psql -d tiktokdownloader -c "\dt"

# Ver permisos del usuario
sudo -u postgres psql -c "\du tiktokuser"
```

### Problema: Error de permisos en PostgreSQL
```bash
sudo -u postgres psql
GRANT ALL PRIVILEGES ON DATABASE tiktokdownloader TO tiktokuser;
\c tiktokdownloader
GRANT ALL ON SCHEMA public TO tiktokuser;
GRANT ALL ON ALL TABLES IN SCHEMA public TO tiktokuser;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO tiktokuser;
\q
```

### Problema: WebSocket no conecta
```bash
# Verificar configuración de Nginx
sudo nginx -t
sudo nano /etc/nginx/sites-available/tiktok-chat
# Asegúrate de que la sección /ws esté configurada
sudo systemctl restart nginx
```

### Problema: Error "Cannot find module"
```bash
cd /var/www/tiktok-app
npm install
npm run build
pm2 restart tiktok-chat
```

---

## 📊 VERIFICACIÓN FINAL

Ejecuta este script para verificar que todo está configurado correctamente:

```bash
echo "=== VERIFICACIÓN DEL SISTEMA ==="
echo ""
echo "Node.js:"
node --version
echo ""
echo "NPM:"
npm --version
echo ""
echo "PostgreSQL:"
sudo systemctl status postgresql | grep Active
echo ""
echo "Nginx:"
sudo systemctl status nginx | grep Active
echo ""
echo "PM2:"
pm2 status
echo ""
echo "Tablas en la base de datos:"
sudo -u postgres psql -d tiktokdownloader -c "\dt"
echo ""
echo "Mensajes guardados:"
sudo -u postgres psql -d tiktokdownloader -c "SELECT COUNT(*) FROM chat_messages;"
```

---

## 🎉 ¡LISTO!

Tu chat en tiempo real está completamente funcional con:

- ✅ **Mensajes persistentes** guardados en PostgreSQL
- ✅ **Chat en tiempo real** con WebSocket
- ✅ **HTTPS seguro** con certificado SSL
- ✅ **Inicio automático** si el servidor se reinicia
- ✅ **Los mensajes NUNCA se borran** al cerrar el navegador o cambiar de pestaña

**URL de tu aplicación:** https://tu-dominio.com

---

## 📝 NOTAS IMPORTANTES

1. **Cambia la contraseña** de PostgreSQL por una segura
2. **Haz backups** regulares de la base de datos:
   ```bash
   sudo -u postgres pg_dump tiktokdownloader > backup-$(date +%Y%m%d).sql
   ```
3. **Monitorea los logs** regularmente:
   ```bash
   pm2 logs tiktok-chat
   ```
4. **Actualiza el sistema** periódicamente:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

## 💡 TIPS ADICIONALES

### Backup automático diario
```bash
# Crear script de backup
sudo nano /usr/local/bin/backup-chat.sh
```

Pega esto:
```bash
#!/bin/bash
sudo -u postgres pg_dump tiktokdownloader > /var/backups/tiktok-$(date +%Y%m%d-%H%M%S).sql
find /var/backups/tiktok-*.sql -mtime +7 -delete
```

Hacer ejecutable:
```bash
sudo chmod +x /usr/local/bin/backup-chat.sh
```

Agregar a crontab (diario a las 3am):
```bash
sudo crontab -e
```

Agregar:
```
0 3 * * * /usr/local/bin/backup-chat.sh
```

### Limpieza automática de mensajes antiguos (opcional)
```bash
sudo -u postgres psql -d tiktokdownloader
```

```sql
CREATE OR REPLACE FUNCTION delete_old_messages() RETURNS void AS $$
BEGIN
    DELETE FROM chat_messages WHERE timestamp < NOW() - INTERVAL '90 days';
END;
$$ LANGUAGE plpgsql;
```

---

¿Necesitas ayuda con algún paso específico? ¡Avísame!
