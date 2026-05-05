---
tags: [smrtech, active-directory, softether, vpn, windows-server]
---
# Integración SoftEther VPN + Active Directory

## 1. Preparación del servidor

Instalar el rol **Active Directory Domain Services (AD DS)** desde Server Manager y promover el servidor a controlador de dominio.

Pasos en *Usuarios y equipos de Active Directory*:
- Crear una nueva **Unidad Organizativa (OU)**
- Dentro de la OU, crear el grupo `g_principal`
- Crear un usuario y añadirlo al grupo

> [!tip] Verificar nombre de dominio
> En el servidor: **Server Manager → Local Server → campo Domain**

---

## 2. Configuración de SoftEther (servidor)

1. Abrir **SoftEther VPN Server Manager** y conectarse a `localhost`
2. Ir a **Manage Virtual Hub → Manage Users → New**
3. Rellenar los campos:
   - **Nombre de usuario:** igual que en Active Directory
   - **Nombre completo:** nombre real del usuario
   - **Auth Type:** `NT Domain Authentication`
4. Guardar el usuario

> [!warning] Requisito previo
> El servidor SoftEther debe estar unido al mismo dominio de Windows para que la autenticación NT funcione correctamente.

---

## 3. Configuración del cliente VPN

### 3.1 SoftEther Client

- Abrir **SoftEther VPN Client**
- Clic derecho en la última conexión configurada
- Activar **Set as Startup Connection**

> [!info] ¿Por qué Startup Connection?
> Permite que la VPN funcione como **servicio del sistema**, estableciendo el túnel antes del inicio de sesión del usuario, sin intervención manual.

### 3.2 Unir el equipo al dominio

Ruta: **Configuración → Sistema → Información → Dominio o grupo de trabajo → Cambiar...**

- Seleccionar la opción **Dominio**
- Introducir el nombre del dominio (ej. `smrtech.local`)
- Aceptar todo y **reiniciar** el equipo cuando se solicite

Al reiniciar, en la pantalla de inicio de sesión:
1. Seleccionar **Otro usuario**
2. Introducir credenciales del usuario de Active Directory

---

## 4. Comprobaciones

### 4.1 Verificar usuario de dominio

En el menú Inicio, el usuario debe aparecer como:



> [!note]
> Este usuario **no es administrador local** por defecto; ciertas acciones solicitarán credenciales de administrador.

### 4.2 Verificar con ipconfig

Abrir CMD y ejecutar:

```cmd
ipconfig /all
```

En *Configuración IP de Windows*, el campo **Lista de búsqueda de sufijos DNS** debe mostrar el nombre del dominio.

---

## 5. Acceso a recursos compartidos (complementario)

1. Abrir el **Explorador de archivos**
2. En la barra de direcciones escribir: `\\smrtech.local`
3. Aparecerán las carpetas compartidas (incluidas 2 predeterminadas de Windows Server)

Para mapear una unidad de red permanente:
- Clic derecho sobre la carpeta compartida
- Seleccionar **Conectar a unidad de red**
- Asignar una letra de unidad → visible en **Este equipo**