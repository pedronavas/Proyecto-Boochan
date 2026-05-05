# 📑 DOCUMENTACIÓN MAESTRA: PROYECTO BOOCHAN V2

**Infraestructura Híbrida Cloud: Samba AD DC, VPN WireGuard y Seguridad ABE**

---

## 🏗️ Fase 1: Infraestructura Cloud (Azure IaaS)

El proyecto se despliega sobre una **Virtual Machine (VM)** en Microsoft Azure, utilizando un modelo de Infraestructura como Servicio (IaaS).

### 1.1. Configuración de la Instancia

- **Sistema Operativo:** Ubuntu Server 22.04 LTS.
    
- **Networking (NSG):** Firewall perimetral de Azure configurado con las siguientes reglas de entrada:

| **Prioridad** | **Nombre**              | **Puerto**                  | **Protocolo** | **Descripción Técnica**                                                                   |
| ------------- | ----------------------- | --------------------------- | ------------- | ----------------------------------------------------------------------------------------- |
| **100**       | `Router`                | 9090                        | TCP           | Acceso al panel de administración del Router/Cockpit.                                     |
| **300**       | `SSH`                   | 22                          | TCP           | Gestión remota segura de la consola Linux.                                                |
| **310**       | `AllowWireGuardInbound` | 51820                       | UDP           | **Vital:** Es la puerta de entrada para el túnel VPN.                                     |
| **410**       | `AD_Global`             | 88, 135, 389, 445, 464, 636 | TCP           | Puertos globales de AD (88:Kerberos, 135:RPC, 389:LDAP).                                  |
| **420**       | `AD_Servicios_UDP`      | 88, 123, 389, 464           | UDP           | Replicación de servicios y Kerberos vía UDP.                                              |
| **430**       | `AD_RPC_Dinamico`       | 49152-65535                 | TCP           | **Clave:** Rango de puertos dinámicos que usa Windows para la gestión de usuarios y GPOs. |
| **440**       | `AD_Essential_Ports`    | 53,88,135,445,1024-5000     | Any           | Puertos críticos como el 53 (DNS) y 445 (SMB).                                            |

---

## 🧹 Fase 2: Purga y Preparación del Entorno

Para configurar un **Active Directory Domain Controller (AD DC)**, es imperativo eliminar cualquier configuración previa de Samba que funcione como servidor de archivos independiente (Standalone).

### 2.1. Limpieza Total

Bash

```
sudo systemctl stop smbd nmbd winbind
sudo apt-get purge samba* -y
sudo apt-get autoremove -y
sudo rm -rf /etc/samba/ /var/lib/samba/ /var/cache/samba/ /run/samba/
```

### 2.2. Instalación de Dependencias AD DC

Bash

```
sudo apt update && sudo apt install acl attr samba krb5-user winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config wireguard resolvconf -y
```

En la primera pantalla de configuracion de Kerberos pondremos nuestro dominio .local
ej: (boochan.local)
En las siguientes pantallas simplemente le daremos al enter.

### 2.3. Configuracion de hosts

** Editar /etc/hosts para que el sistema se reconozca a sí mismo **

Bash

```
10.0.0.1  UbuntuServer.elche.local  UbuntuServer
```
---

## 🔒 Fase 3: Conectividad VPN (WireGuard)

WireGuard establece el túnel seguro entre la red local (Home/Office) y la Virtual Private Cloud (VPC) de Azure.

### 3.1. Creacion de las claves

Lista de comandos:

```
# 1. Entrar al directorio con sudo
sudo -i
cd /etc/wireguard

# 2. Generar las llaves (ahora sí te dejará)
umask 077
wg genkey | tee privatekey | wg pubkey > publickey

# 3. Ver las llaves para copiarlas
cat privatekey
cat publickey

# 4. Salir del modo root para volver a tu usuario normal
exit
```

### 3.2. Archivo de Configuración del Servidor (Azure)

Ubicación: `/etc/wireguard/wg0.conf`

Ini, TOML

```
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <CLAVE_PRIVADA_SERVIDOR>

[Peer]
PublicKey = <CLAVE_PUBLICA_ROUTER_O_PC>
AllowedIPs = 10.0.0.2/32
```

### 3.3. Archivo de Configuración del Cliente (PC/Router)

Ubicación: `C:\Program Files\WireGuard\Data\Configurations\wg0.conf` o Interfaz del Router.

Ini, TOML

```
[Interface]
Address = 10.0.0.2/24
PrivateKey = <CLAVE_PRIVADA_CLIENTE>
DNS = 10.0.0.1

[Peer]
PublicKey = <CLAVE_PUBLICA_SERVIDOR>
Endpoint = <IP_PUBLICA_AZURE>:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```
![[Pasted image 20260211113142.png]]
---

### 3.4 Encender el tunel
```
# Levantar el túnel ahora
sudo wg-quick up wg0

# Comprobar que el túnel está activo y ver el estado
sudo wg show

# Habilitar para que encienda solo al reiniciar el servidor
sudo systemctl enable wg-quick@wg0
```


## 👑 Fase 4: Aprovisionamiento del Dominio (Samba AD DC)

### 4.1. Preparación y Limpieza de Conflictos (CRÍTICO)

Antes de provisionar, debemos eliminar cualquier servicio que esté "secuestrando" los puertos que Samba necesita (53 para DNS y 445 para archivos).

Bash

```
# 1. Detener servicios de archivos estándar (Modo zombie)
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind

# 2. Liberar el puerto 53 (DNS de Ubuntu)
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# 3. Eliminar configuración residual
sudo rm /etc/samba/smb.conf
```

### 4.2. Provisión del Dominio

Bash

```
# Lanzar provisión con soporte RFC2307 (Activa el esquema para IDs de Linux)
sudo samba-tool domain provision --use-rfc2307 --interactive
```

- **Realm:** BOOCHAN.SPACE
    
- **Domain:** BOOCHAN
    
- **Role:** dc
    
- **DNS Backend:** SAMBA_INTERNAL
    
- **DNS Forwarder:** 8.8.8.8
    

### 4.3. Configuración de DNS local (CRÍTICO)

Azure sobrescribe el DNS. Debemos forzar al servidor a consultarse a sí mismo para encontrar el dominio:

Bash

```
sudo rm /etc/resolv.conf
sudo nano /etc/resolv.conf
```

**Contenido:**

Plaintext

```
nameserver 127.0.0.1
domain boochan.space
search boochan.space
```

### 4.4. Activación de Servicios y Kerberos

Bash

```
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
sudo systemctl unmask samba-ad-dc
sudo systemctl enable --now samba-ad-dc
```

### 4.5. Ajuste de Escucha de Red (Binding)

Editar `/etc/samba/smb.conf`. En la sección `[global]`:

Ini, TOML

```
interfaces = 127.0.0.1 10.0.0.1 
bind interfaces only = no
```

- **IMPORTANTE:** Si tras reiniciar el puerto sigue en `127.0.0.1`, forzar el reinicio total: `sudo killall -9 samba smbd && sudo systemctl start samba-ad-dc`
    
- **Verificación:** `sudo ss -tulpn | grep :445` debe mostrar `0.0.0.0:445`.
### 4.6. Integración de Identidad (NSS & RFC2307)

Este paso construye el puente entre el AD y Linux.

1. **Configurar NSS:** `sudo nano /etc/nsswitch.conf` -> Añadir `winbind` en las líneas de `passwd` y `group`.
    
2. **Activar Atributos Unix en el Grupo Base:**
    
    Bash
    
    ```
    # Asignar ID al grupo universal "Domain Users" que crea Samba por defecto
    sudo samba-tool group addunixattrs "Domain Users" 3000
    ```
    
	**Reiniciar servicios:**  

	```
	sudo net cache flush sudo systemctl restart samba-ad-dc
	```
	   
 

---

## 👥 Fase 5: Gestión de Identidades (Usuarios y Grupos)

Creación de la jerarquía basada en los grupos `pichoncillo`, `policia` y `Domain Admins`. Cada usuario se crea vinculado a su grupo y con su ID de Linux (UID) integrado.

### 5.1. Configuración de Grupos

Asignamos IDs a los grupos para que Linux pueda gestionar los permisos de las carpetas.

Bash

```
# 1. Asignar ID al grupo de Administradores (Ya existe por defecto)
sudo samba-tool group addunixattrs "Domain Admins" 3001

# 2. Crear grupos nuevos y asignarles ID de Linux
sudo samba-tool group add pichoncillo
sudo samba-tool group addunixattrs pichoncillo 3002

sudo samba-tool group add policia
sudo samba-tool group addunixattrs policia 3003
```

### 5.2. Creación de Usuarios con Identidad Integrada

Creamos los usuarios asignando directamente su UID y su grupo principal (`--gid-number`).

Bash

```
# User1: Pertenece a Pichoncillo (3002)
sudo samba-tool user create user1 --uid=10001 --gid-number=3002

# User2: Pertenece a Policia (3003)
sudo samba-tool user create user2 --uid=10002 --gid-number=3003

# User3: Pertenece a Administradores (3001)
sudo samba-tool user create user3 --uid=10003 --gid-number=3001
```

### 5.3. Verificación de Integración

Es vital comprobar que Linux "ve" los nombres y los IDs correctamente antes de aplicar permisos.

Bash

```
# Comprobar que los grupos responden con su ID
getent group pichoncillo
getent group policia

# Verificar que user1 es reconocido por el sistema
id user1
```
## 💾 Fase 6: Almacenamiento Virtual (Cuotas con Loop Devices)

Simulación de límites de disco físicos mediante archivos de imagen `.img` formateados en ext4.

### 6.1. Creación de Discos Virtuales

Bash

```
sudo mkdir -p /srv/samba
sudo dd if=/dev/zero of=/samba_p1.img bs=1M count=5120
sudo dd if=/dev/zero of=/samba_p2.img bs=1M count=10240
sudo dd if=/dev/zero of=/samba_p3.img bs=1M count=15360

sudo mkfs.ext4 /samba_p1.img
sudo mkfs.ext4 /samba_p2.img
sudo mkfs.ext4 /samba_p3.img
```

### 6.2. Persistencia en `/etc/fstab`

Plaintext

```
/samba_p1.img  /srv/samba/prueba1  ext4  loop,defaults  0  0
/samba_p2.img  /srv/samba/prueba2  ext4  loop,defaults  0  0
/samba_p3.img  /srv/samba/prueba3  ext4  loop,defaults  0  0
```

---

## 🕵️ Fase 7: Seguridad Avanzada y ABE

Implementación de permisos a nivel de Kernel y visibilidad selectiva de carpetas en red mediante **Access Based Enumeration**.

**📂 Paso previo necesario: Creación de puntos de montaje**

Antes de configurar permisos o montar los archivos `.img`, debes asegurar que la estructura existe y está vinculada al grupo del dominio.

Bash

```
# Crear la estructura de carpetas (si no se hizo en la Fase 6)
sudo mkdir -p /srv/samba/prueba{1,2,3,4}

# Vincular la propiedad al grupo del dominio (Mapeado en Fase 4.6)
sudo chgrp -R "Domain Users" /srv/samba/prueba*
```

###  **7.1. Permisos Linux (ACLs)**

Utilizamos `chmod` para los permisos básicos y `setfacl` para permisos granulares por grupo de AD.

Bash

```
# PRUEBA 1 y 2: Acceso total para todos los usuarios del dominio
sudo chmod -R 770 /srv/samba/prueba1
sudo chmod -R 770 /srv/samba/prueba2

# PRUEBA 3: Solo grupo 'acosador' y 'Domain Admins'
sudo chmod 700 /srv/samba/prueba3
sudo setfacl -b /srv/samba/prueba3
sudo setfacl -m g:policia:rwx /srv/samba/prueba3
sudo setfacl -m g:"Domain Admins":rwx /srv/samba/prueba3

# PRUEBA 4: Restringida solo a Administradores
sudo chmod 700 /srv/samba/prueba4
sudo setfacl -b /srv/samba/prueba4
sudo setfacl -m g:"Domain Admins":rwx /srv/samba/prueba4
```

###  **7.2. Configuración Samba ABE (smb.conf)**

Edita `/etc/samba/smb.conf` para añadir las definiciones de los recursos. El parámetro `access based share enum` hará que las carpetas sean invisibles para quienes no tengan permiso.

Ini, TOML

```
[Prueba1]
    path = /srv/samba/prueba1
    read only = no
    guest ok = no
    access based share enum = yes

[Prueba2]
    path = /srv/samba/prueba2
    read only = no
    guest ok = no
    access based share enum = yes

[Prueba3]
    path = /srv/samba/prueba3
    read only = no
    access based share enum = yes
    vfs objects = dfs_samba4 acl_xattr

[Prueba4]
    path = /srv/samba/prueba4
    read only = no
    access based share enum = yes
    vfs objects = dfs_samba4 acl_xattr
```

### **7.3. Aplicación de cambios**

Bash

```
sudo net cache flush
sudo systemctl restart samba-ad-dc
```
---

## 💻 Fase 8: Configuración del Cliente Windows 11

### 8.1. Bypass de Cuenta Microsoft

Durante la instalación (OOBE):

1. Pulsar `Shift + F10`.
    
2. Ejecutar: `OOBE\BYPASSNRO`.
    
3. Tras el reinicio, seleccionar "No tengo internet" para crear una cuenta local.
    

### 8.2. Conexión al Recurso

En un CMD con permisos:

DOS

```
net use Z: \\10.0.0.1\Prueba1 /user:BOOCHAN\hector <password> /persistent:no
```

### 8.3. Troubleshooting de Conexión y Unión al Dominio
**Si el dominio no se encuentra o da "Acceso Denegado":

1. **Limpieza de sesiones:** Ejecutar `net use * /delete /y` para evitar el error de "conexiones múltiples".
    
2. **Desactivar IPv6:** En las propiedades del adaptador WireGuard de Windows, desmarcar IPv6 para evitar el "DNS Timeout".
    
3. **Sincronización horaria:** Asegurar que la hora de Windows coincide con la de Azure (tolerancia máx. 5 min para Kerberos).
    
4. **Reset de Password:** Si se desconoce la clave de administrador del dominio: `sudo samba-tool user setpassword administrator`


> [!IMPORTANT] **🔄 Protocolo de Reinicio de Servicios (Si algo falla)** Si los clientes no ven las carpetas, ejecutar en el servidor en este orden:
> 
> 1. `sudo wg-quick down wg0 && sudo wg-quick up wg0` (Reinicia el túnel).
>     
> 2. `sudo systemctl restart samba-ad-dc` (Vincula Samba a la red activa).
>     
> 3. `sudo ss -tulpn | grep -E ':53|:445'` (Verificar que aparece `0.0.0.0` y no `127.0.0.1`).



Cerrar usuarios desde CMD:
**Cerrar:** `net use * /delete /y` (Limpia conexiones).
Documento de windows a modificar:
C:\Windows\System32\drivers\etc -> hosts