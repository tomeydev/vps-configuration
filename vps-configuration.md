# 🚀 Guía de Configuración Profesional de VPS: Seguridad, Redes y Despliegue

Esta guía detalla el proceso para transformar un VPS en una infraestructura de producción segura y eficiente.

---

## 📋 Resumen de la Arquitectura

- **Acceso:** SSH mediante llaves criptográficas (RSA / ED25519).  
- **Red privada:** Tailscale (Mesh VPN) + MagicDNS.  
- **Seguridad:** Firewall UFW bloqueando todo el tráfico administrativo fuera de la VPN.  
- **Despliegue:** Docker + Dokploy (PaaS auto-hosteado).

---

## Fase 1: Hardening Inicial y Gestión de Usuarios

Nunca operes un servidor como usuario root. El primer paso es crear un usuario con privilegios sudo.

### 1. Actualización del Sistema

```bash
apt update && apt upgrade -y
```

### 2. Creación del Usuario de Sistema

Sustituye `nombre_usuario` por tu alias preferido.

```bash
adduser nombre_usuario
usermod -aG sudo nombre_usuario
```

### 3. Transferencia de llaves SSH

Copiamos la llave que ya usas como root al nuevo usuario para no perder el acceso.

```bash
mkdir -p /home/nombre_usuario/.ssh
cp /root/.ssh/authorized_keys /home/nombre_usuario/.ssh/
chown -R nombre_usuario:nombre_usuario /home/nombre_usuario/.ssh
chmod 700 /home/nombre_usuario/.ssh
chmod 600 /home/nombre_usuario/.ssh/authorized_keys
```

> Nota: en este punto, abre una nueva terminal en tu PC local y verifica el acceso:
> `ssh nombre_usuario@IP_PUBLICA`

---

## Fase 2: Conectividad con Tailscale y MagicDNS

Tailscale crea una red privada virtual que permite acceder al servidor sin exponer puertos críticos a internet.

### 1. Instalación de Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Autentica el link que aparecerá en consola.

### 2. Configuración de MagicDNS (Opcional pero recomendado)

- En el Admin Console de Tailscale, activa MagicDNS.  
- En la lista de máquinas, renombra tu servidor a algo simple (ej. `servidor`).

Resultado: podrás acceder vía `ssh nombre_usuario@servidor` siempre que MagicDNS esté activado y tu cliente Tailscale tenga habilitada la resolución DNS de Tailscale. Si no funcionan los nombres cortos, usa la dirección con el dominio completo de Tailscale o la IP del túnel (`100.x.y.z`).

---

## Fase 3: Seguridad Perimetral (Firewall y SSH)

Configuraremos el firewall para que solo "escuche" a tus dispositivos a través de Tailscale.

> Advertencia crítica: no actives `ufw` hasta que verifiques desde otro dispositivo que `tailscale up` funciona correctamente y que puedes acceder por la interfaz Tailscale. Habilitar `ufw` sin comprobarlo puede bloquearte.

### 1. Configuración del Firewall (UFW)

La regla clave es permitir todo el tráfico que venga de la interfaz virtual de Tailscale.

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir tráfico web público
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Permitir TODO desde la red privada Tailscale (SSH, Dokploy, etc.)
# Dependiendo de la versión de ufw, la sintaxis alternativa es:
# sudo ufw allow in on tailscale0 from any to any
sudo ufw allow in on tailscale0

sudo ufw enable
```

### 2. Hardening de SSH

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

### 3. Cambio del Puerto SSH (Opcional)

Si deseas una capa extra de "seguridad por oscuridad":
En `sshd_config`, cambia `Port 22` a uno personalizado (ej. `Port 5522`).
Recuerda permitir ese puerto en UFW y mantener una sesión activa mientras pruebas para no perder acceso.

```bash
sudo ufw allow 5522/tcp
sudo systemctl.restart ssh
```

---

## Fase 4: Infraestructura de Contenedores y Dokploy

Dokploy actuará como tu panel de control para desplegar proyectos desde GitHub.

### 1. Instalación de Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ${USER}
```

Nota: después de añadir el usuario al grupo `docker` hace falta cerrar sesión y volver a iniciarla para que los cambios surtan efecto, o ejecutar `newgrp docker` en la sesión actual.

### 2. Instalación de Dokploy

```bash
curl -sSL https://dokploy.com/install.sh | sudo sh
```

### 3. Acceso al Dashboard

Gracias a la configuración anterior, el panel de Dokploy (puerto 3000) no es accesible públicamente. Solo tú puedes entrar usando:

- `http://servidor:3000` (usando MagicDNS)  
- o `http://100.x.y.z:3000` (IP de Tailscale)

Recomendación: limita Dokploy para que escuche solo en la interfaz Tailscale o pon un proxy inverso con autenticación. Evita exponer el puerto 3000 directamente a Internet.

---

## Fase 5: Optimización del Flujo de Trabajo (Local)

Para que tu experiencia como desarrollador sea fluida desde tu Mac Mini o cualquier otro dispositivo.

### 1. Alias de SSH en tu PC local

Edita tu archivo local `~/.ssh/config`:

```text
Host vps
    HostName servidor
    User nombre_usuario
    # Port 5522 (Descomenta si cambiaste el puerto)
```

Ahora solo necesitas ejecutar `ssh vps` para entrar.

### 2. Acceso desde dispositivos móviles

- Instala la app de Tailscale en tu iPhone/iPad.  
- Instala un cliente SSH como Termius.  
- Crea una nueva llave en el iPhone y añade la llave pública al archivo `~/.ssh/authorized_keys` del servidor.

---

## 🛠 Mantenimiento y Buenas Prácticas

- **Actualizaciones automáticas:** instala `unattended-upgrades` para parches de seguridad automáticos.  
- **Backups:** activa los snapshots semanales en tu proveedor de VPS.  
- **Docker:** no instales bases de datos ni lenguajes directamente en el host; usa siempre contenedores gestionados por Dokploy.

---

## Notas rápidas de seguridad y operativa

- Antes de `ufw enable` — verifica `tailscale up` y acceso remoto desde otra máquina.  
- Si cambias el puerto SSH, deja una sesión abierta hasta confirmar que puedes reconectar.  
- Después de `usermod -aG docker` realiza relogin o `newgrp docker`.  
- Asegura que Dokploy no esté expuesto públicamente: bind a `tailscale0` o detrás de proxy.

---

Archivo original con typo renombrado a `vps-configuration.md` para evitar confusiones.
