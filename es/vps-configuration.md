# 🚀 Guía de Configuración Profesional de VPS: Seguridad, Redes y Despliegue

[🇺🇸 Leer en Inglés](../en/vps-configuration.md)

Esta guía detalla el proceso para transformar un VPS en una infraestructura de producción segura y eficiente. Está diseñada y probada específicamente para **Ubuntu 24.04 LTS**, aunque la mayoría de los pasos son compatibles con otras distribuciones basadas en Debian.

---

## 📋 Resumen de la Arquitectura

- **Sistema Operativo:** Ubuntu 24.04 LTS (o distribuciones similares como Debian).
- **Acceso:** SSH mediante llaves criptográficas (RSA / ED25519).  
- **Red privada:** Tailscale (Mesh VPN) + MagicDNS.  
- **Seguridad:** Firewall UFW bloqueando todo el tráfico administrativo fuera de la VPN.  
- **Despliegue:** Docker + Dokploy (PaaS auto-hosteado).

---

## Fase 1: Hardening Inicial y Gestión de Usuarios (En el VPS)

Nunca operes un servidor como usuario root. El primer paso es crear un usuario con privilegios sudo.

### 1. Actualización del Sistema (VPS)

```bash
apt update && apt upgrade -y
```

### 2. Creación del Usuario de Sistema (VPS)

Sustituye `nombre_usuario` por tu alias preferido.

```bash
adduser nombre_usuario
usermod -aG sudo nombre_usuario
```

### 3. Transferencia de llaves SSH (VPS)

Copiamos la llave que ya usas como root al nuevo usuario para no perder el acceso.

```bash
mkdir -p /home/nombre_usuario/.ssh
cp /root/.ssh/authorized_keys /home/nombre_usuario/.ssh/
chown -R nombre_usuario:nombre_usuario /home/nombre_usuario/.ssh
chmod 700 /home/nombre_usuario/.ssh
chmod 600 /home/nombre_usuario/.ssh/authorized_keys
```

> Nota: en este punto, abre una nueva terminal en tu **dispositivo local** y verifica el acceso:
> `ssh nombre_usuario@IP_PUBLICA`

---

## Fase 2: Conectividad con Tailscale y MagicDNS

Tailscale crea una red privada virtual (mesh VPN) basada en WireGuard que permite que tus dispositivos se comuniquen de forma segura como si estuvieran en la misma red local, sin necesidad de abrir puertos en el firewall público.

### 1. Preparación del Dispositivo Local

Antes de configurar el VPS, tu dispositivo de trabajo debe estar dentro de la red privada:

1. **Instalación:** Descarga e instala Tailscale en tu dispositivo local desde [tailscale.com/download](https://tailscale.com/download).
2. **Autenticación:** Inicia sesión con su cuenta elegida (Google, Microsoft, GitHub, etc.). Este será el administrador de su "Tailnet".
3. **Estado:** Asegúrate de que Tailscale figure como "Connected" en tu dispositivo.

### 2. Instalación en el VPS

Ahora vincularemos el servidor a tu misma cuenta de Tailscale.

```bash
# Descarga e instala el binario de Tailscale (VPS)
curl -fsSL https://tailscale.com/install.sh | sh

# Inicia el servicio y genera el enlace de autenticación (VPS)
sudo tailscale up
```

**Acción requerida:**

- Al ejecutar `sudo tailscale up`, aparecerá una URL en la terminal del VPS.
- Copia y pega esa URL en el navegador de tu **dispositivo local**.
- Autoriza la conexión haciendo clic en **Connect**. El VPS ahora es parte de tu red privada.

### 3. Verificación de Conectividad (Dispositivo Local)

En la terminal de tu **dispositivo local**, ejecuta:

```bash
tailscale status
```

Deberías ver listado tu VPS con una dirección IP interna (rango `100.x.y.z`). Prueba la conexión básica:

```bash
ping 100.x.y.z_DE_TU_VPS
```

### 4. Configuración de MagicDNS (Opcional - Desde el navegador)

#### Paso 1: Activar MagicDNS en el Panel de Tailscale

1. Entra a tu [Consola de Tailscale](https://login.tailscale.com/admin/dns).
2. Ve a la pestaña **DNS**.
3. Busca la sección **MagicDNS** y haz clic en el botón **Enable MagicDNS**.

#### Paso 2: Personalizar el nombre de tu VPS (Navegador)

Por defecto, Tailscale usa el nombre que tiene el servidor en el sistema (ej. `ubuntu-24-04-lts`). Vamos a ponerle uno más corto:

1. En la consola de Tailscale, ve a la pestaña **Machines**.
2. Busca tu VPS en la lista.
3. Haz clic en los tres puntos (`...`) al final de la fila y selecciona **Edit machine name**.
4. Desactiva "Use OS hostname" y escribe un nombre corto, por ejemplo: `servidor`.
5. Haz clic en **Save**.

#### Paso 3: Simplificar el acceso SSH en tu dispositivo local

Ahora vamos a configurar tu dispositivo local (PC, Laptop, etc.) para que reconozca ese nombre y sepa qué usuario usar automáticamente.

- En tu **dispositivo local**, abre la terminal y edita (o crea) tu archivo de configuración SSH:

```bash
nano ~/.ssh/config
```

- Pega el siguiente bloque al principio del archivo (reemplaza `tu_usuario` por el nombre de usuario que creaste en el VPS):

```plaintext
Host vps
    HostName servidor
    User tu_usuario
```

- Guarda con `Ctrl+O`, `Enter` y sal con `Ctrl+X`.

**Resultado:** Ahora, para entrar a tu servidor desde tu dispositivo, solo tienes que escribir:
`ssh vps`

---

## Fase 3: Seguridad Perimetral (Firewall y SSH en el VPS)

Configuraremos el firewall para que solo "escuche" a tus dispositivos a través de Tailscale.

> Advertencia crítica: no actives `ufw` hasta que verifiques desde otro dispositivo que `tailscale up` funciona correctamente y que puedes acceder por la interfaz Tailscale. Habilitar `ufw` sin comprobarlo puede bloquearte.

### 1. Configuración del Firewall (UFW - VPS)

La regla clave es permitir todo el tráfico que venga de la interfaz virtual de Tailscale.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir tráfico web público (Opcional si usas Cloudflare o similar)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Permitir TODO desde la red privada Tailscale (SSH, Dokploy, etc.)
sudo ufw allow in on tailscale0

sudo ufw enable
```

### 2. Hardening de SSH (VPS)

Desactivamos el acceso por contraseña y el acceso de root.

```bash
sudo nano /etc/ssh/sshd_config
```

Modifica o añade las siguientes líneas:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Reinicia el servicio:

```bash
sudo systemctl restart ssh
```

### 3. Cambio del Puerto SSH (Opcional - VPS)

Si deseas una capa extra de "seguridad por oscuridad":
En `sshd_config`, cambia `Port 22` a uno personalizado (ej. `Port 5522`).

#### Paso 1: Elegir un puerto (VPS)

Elige un número entre 1024 y 65535. Evita los comunes (8080, 3000, 8000).

- _Sugerencia:_ Usemos el **5522**.

#### Paso 2: Editar la configuración SSH (VPS)

- Edita el archivo de configuración:

```bash
sudo nano /etc/ssh/sshd_config
```

- Busca la línea `#Port 22` (si tiene un #, bórralo).

- Cámbiala por:

```plaintext
Port 5522
```

- Guarda (`Ctrl+O`, `Enter`) y sal (`Ctrl+X`).

#### Paso 3: Ajustar el Firewall (UFW - VPS)

- Abre el nuevo puerto en el firewall:

```bash
sudo ufw allow 5522/tcp
```

>Nota: Si quieres mantener la seguridad máxima, confía en la regla de Tailscale (`tailscale0`) que ya permite el tráfico.

- Borra la regla antigua del puerto 22 (Opcional):

```bash
sudo ufw delete allow 22/tcp
```

#### Paso 4: Reiniciar SSH (VPS)

```bash
sudo service ssh restart
```

#### Paso 5: Prueba de Fuego (Testing - Dispositivo Local)

1. **NO CIERRES TU TERMINAL ACTUAL DEL VPS.**
2. Abre una **nueva** terminal en tu **dispositivo local**.
3. Intenta conectar especificando el puerto con `-p`:

```bash
ssh -p 5522 tu_usuario@IP_TAILSCALE
```

Si entraste exitosamente, ¡felicitaciones! Ya cambiaste el puerto.

---

## Fase 4: Infraestructura de Contenedores y Dokploy (En el VPS)

Dokploy actuará como tu panel de control para desplegar proyectos desde GitHub.

### 1. Instalación de Docker (VPS)

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ${USER}
```

Nota: después de añadir el usuario al grupo `docker` hace falta cerrar sesión y volver a iniciarla, o ejecutar `newgrp docker`.

### 2. Instalación de Dokploy (VPS)

```bash
curl -sSL https://dokploy.com/install.sh | sudo sh
```

### 3. Acceso al Dashboard (Navegador - Dispositivo Local)

Gracias a la configuración anterior, el panel de Dokploy (puerto 3000) no es accesible públicamente. Solo tú puedes entrar usando:

- `http://servidor:3000` (usando MagicDNS)  
- o `http://100.x.y.z:3000` (IP de Tailscale)

### 4. Deshabilitar el acceso mediante `IP:puerto` (Opcional - VPS)

Para mejorar la seguridad, se recomienda restringir el acceso a Dokploy mediante la dirección IP y el puerto del servidor.

>**Importante**: Antes de este paso, asegúrese de haber configurado un dominio con HTTPS funcionando correctamente.

Una vez verificado, ejecuta este comando en el **servidor (VPS)**:

```bash
docker service update --publish-rm "published=3000,target=3000,mode=host" dokploy
```

---

## Fase 5: Optimización del Flujo de Trabajo (Dispositivo Local)

Para que tu experiencia como desarrollador sea fluida desde tu dispositivo local u otros dispositivos de confianza.

### 1. Acceso desde dispositivos móviles

- Instala la app de Tailscale en tu dispositivo móvil.  
- Instala un cliente SSH (ej. Termius).  
- Crea una nueva llave en el dispositivo y añade la llave pública al archivo `~/.ssh/authorized_keys` del **servidor**.

---

## 🛠 Mantenimiento y Buenas Prácticas

- **Actualizaciones automáticas (VPS):** instala `unattended-upgrades` para parches de seguridad automáticos.  
- **Backups (Proveedor):** activa los snapshots semanales en tu proveedor de VPS.  
- **Docker (VPS):** no instales bases de datos ni lenguajes directamente en el host; usa siempre contenedores gestionados por Dokploy.

---

## Notas rápidas de seguridad y operativa

- Antes de `ufw enable` — verifica `tailscale up` y acceso remoto desde otra máquina.  
- Si cambias el puerto SSH, deja una sesión abierta hasta confirmar que puedes reconectar.  
- Después de `usermod -aG docker` realiza relogin o `newgrp docker`.  
- Asegura que Dokploy no esté expuesto públicamente: bind a `tailscale0` o detrás de proxy.

---
