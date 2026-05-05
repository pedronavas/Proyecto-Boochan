

<hr style="border: none; height: 5px; background: linear-gradient(to right, rgb(255, 215, 0), rgb(255, 69, 0), transparent); margin-bottom: 20px;">

- 1. **Instalación de Ubuntu Server 24.04:**
    
    - Descargamos la ISO oficial y configuramos la máquina virtual en VirtualBox con 4GB de RAM, un disco dinámico de 25GB y dos procesadores para una instalación óptima, para la instalación la haremos manualmente así que deshabilitamos la opción de instalación desatendida.
    
	- Antes de iniciar la máquina, modificaremos la configuración red y la pondremos en adaptador red.
        
    - Durante la instalación, seleccionamos el idioma español y configuramos el teclado adecuadamente para evitar errores con caracteres especiales en el futuro.
        
    - **Punto clave:** No instalamos el entorno gráfico todavía para mantener el sistema ligero y seguro desde el inicio.
        <div style="float: right; margin-left: 20px; margin-bottom: 10px; pointer-events: none;"> <img src="pikachu.png" style="width: 180px; filter: brightness(1.1) drop-shadow(10px 10px 10px rgba(0,0,0,0.3));"> </div>
- 2. **Configuración de Red Estática (Netplan):**  
    
    - Para que nuestro servidor sea localizable, necesitamos que su IP no cambie nunca.
        
    - Entramos en el fichero de configuración: `sudo nano /etc/netplan/*`.
        
    - Editamos/añadimos los parámetros para asignar la IP fija:
        
        - `addresses: [192.168.0.200/24]` <div style="float: right; margin-left: 20px; margin-bottom: 10px; pointer-events: none;"> <img src="pikachu2.png" style="width: 80px; filter: brightness(1.1) drop-shadow(10px 10px 10px rgba(0,0,0,0.3));"> </div>
            
        - `routes: - to: default, via: 192.168.0.1` (Puerta de enlace del router).
            
        - `nameservers: addresses: [8.8.8.8, 8.8.4.4]` (DNS de Google).
        
		- El resultado tiene que ser similar al de la foto, de lo contrario, no funcionara bien:<div style="pointer-events: none;"> <img src="netplan.png" style="width: 500px;"> </div>
            
    - Aplicamos los cambios con `sudo netplan apply`.
        
	- Confirmamos que la configuración se ha aplicado con `ip a`

<hr style="border: none; height: 5px; background: linear-gradient(to left, rgb(255, 215, 0), rgb(255, 69, 0), transparent); margin-bottom: 20px;">

.
Para ir a la siguiente Fase Click aqui →→ [[Fase 2 LINK (Gestión Remota e Híbrida de Administración (SOR + SR))]] 