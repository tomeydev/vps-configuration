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

#### Paso 1: Activar MagicDNS en el Panel de Tailscale

1. Entra a tu [Consola de Tailscale](https://login.tailscale.com/admin/dns).
2. Ve a la pestaña **DNS**.
3. Busca la sección **MagicDNS** y haz clic en el botón **Enable MagicDNS**.

#### Paso 2: Personalizar el nombre de tu VPS

Por defecto, Tailscale usa el nombre que tiene el servidor en el sistema (ej. `ubuntu-24-04-lts`). Vamos a ponerle uno más corto:

1. En la consola de Tailscale, ve a la pestaña **Machines**.
2. Busca tu VPS en la lista.
3. Haz clic en los tres puntos (`...`) al final de la fila y selecciona **Edit machine name**.
4. Desactiva "Use OS hostname" y escribe un nombre corto, por ejemplo: `servidor`.
5. Haz clic en **Save**.

#### Paso 3: Simplificar el acceso SSH en tu Mac Mini

Ahora vamos a configurar tu Mac para que reconozca ese nombre y sepa qué usuario usar automáticamente.

- En tu Mac Mini, abre la terminal y edita (o crea) tu archivo de configuración SSH:

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

#### Paso 1: Elegir un puerto

Elige un número entre 1024 y 65535. Evita los comunes (8080, 3000, 8000).

- _Sugerencia:_ Usemos el **5522** (fácil de recordar, pero no estándar).

#### Paso 2: Editar la configuración SSH

- Edita el archivo de configuración:

```bash
sudo nano /etc/ssh/sshd_config
```

- Busca la línea que dice `Include /etc/ssh/sshd_config.d/*.conf`. A veces la configuración real está ahí dentro, pero generalmente podemos sobrescribirla aquí o editar la línea `Port 22`.

- Busca la línea `#Port 22` (si tiene un #, bórralo).

- Cámbiala por:

```plaintext
Port 5522
```

- Guarda (`Ctrl+O`, `Enter`) y sal (`Ctrl+X`).

#### Paso 3: Ajustar el Firewall (UFW)

Aunque la regla de Tailscale (`allow in on tailscale0`) permite todo el tráfico por la VPN (incluido el nuevo puerto), es bueno ser explícito por si algún día necesitas entrar por la IP pública de emergencia.

- Abre el nuevo puerto en el firewall:

```bash
sudo ufw allow 5522/tcp
```

>Nota: Si quieres mantener la seguridad máxima, ignora este paso. Tu regla de Tailscale ya permitirá el tráfico por el puerto 5522 automáticamente porque permitimos todo tráfico en la interfaz (`tailscale0`).

- Borra la regla antigua del puerto 22 (Opcional, para limpieza):

```bash
sudo ufw delete allow 22/tcp
```

#### Paso 4: Reiniciar SSH

Aplica los cambios.

```bash
sudo service ssh restart
```

#### Paso 5: Prueba de Fuego (Testing)

1. **NO CIERRES TU TERMINAL ACTUAL.**
2. Abre una **nueva** terminal en tu Mac.
3. Intenta conectar especificando el puerto con `-p`:

```bash
ssh -p 5522 tu_usuario@IP_TAILSCALE
```

Si entraste exitosamente, ¡felicitaciones! Ya cambiaste el puerto.

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

Para que tu experiencia como desarrollador sea fluida desde tu dispositivo principal o cualquier otro dispositivo.

### 1. Acceso desde dispositivos móviles

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
