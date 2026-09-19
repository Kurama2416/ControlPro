
---

# **INFORME TÉCNICO DE CONFIGURACIÓN Y SUSTENTACIÓN DE RED**

## **PROYECTO: CONTROLPRO S.A.S.**

---

### **1. RESUMEN EJECUTIVO**

El presente documento detalla la implementación, arquitectura de seguridad y análisis técnico comando por comando de la infraestructura de red para la empresa **ControlPro S.A.S.**.

La solución fue desplegada utilizando **Fedora Linux** configurado como un **Router-on-a-Stick** con etiquetado **IEEE 802.1Q**, segmentación de tráfico mediante subinterfaces VLAN, control de acceso perimetral mediante **iptables** y auditoría de eventos de seguridad centralizada en el log del kernel (`journalctl`).

---

### **2. TOPOLOGÍA Y MAPA DE SUBREDES**

| Departamento / Área | ID VLAN | Interfaz Virtual | Segmento de Red (CIDR) | Dirección IP Gateway | IP Host Simulado |
| --- | --- | --- | --- | --- | --- |
| **Administración** | VLAN 10 | `wlp2s0.10` | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.10` |
| **Contabilidad** | VLAN 20 | `wlp2s0.20` | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.10` |
| **Recursos Humanos (RRHH)** | VLAN 30 | `wlp2s0.30` | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.10` |
| **Soporte Técnico** | VLAN 40 | `wlp2s0.40` | `192.168.40.0/24` | `192.168.40.1` | `192.168.40.10` |
| **Servidores / Base de Datos** | VLAN 50 | `wlp2s0.50` | `10.0.50.0/24` | `10.0.50.1` | `10.0.50.10` *(Servidor BD)* |

---

### **3. ANÁLISIS DETALLADO DE COMANDOS DEL SCRIPT (`controlpro-net.sh`)**

#### **FASE 1: Inicialización de Módulos y Enrutamiento**

1. **`modprobe 8021q`**
* **Descripción:** Carga en el kernel de Linux el módulo controlador del estándar IEEE 802.1Q.


* **Función:** Permite al sistema operativo empaquetar, leer e interpretar las etiquetas VLAN (VLAN Tags) insertadas en las tramas Ethernet a través del enlace troncal (*Trunk*) sobre la tarjeta Wi-Fi/física `wlp2s0`.




2. **`sysctl -w net.ipv4.ip_forward=1`**
* **Descripción:** Modifica el parámetro del kernel para habilitar el reenvío de paquetes IP en tiempo real.


* **Función:** Transforma la máquina Linux en un router funcional, permitiendo que el tráfico que entra por una subinterfaz virtual pueda ser conmutado y enrutado hacia otra subred distinta.





---

#### **FASE 2: Despliegue de Subinterfaces Virtuales (Router-on-a-Stick)**

Para cada departamento de la empresa, se ejecuta una secuencia de cuatro comandos para configurar la interfaz lógica sobre el adaptador físico `wlp2s0`:

* **`ip link add link wlp2s0 name wlp2s0.10 type vlan id 10`**
* *Función:* Crea la subinterfaz lógica asignándole el ID de etiqueta VLAN 10.




* **`ip addr flush dev wlp2s0.10`**
* *Función:* Elimina cualquier dirección IP previa para evitar conflictos de enrutamiento.




* **`ip addr add 192.168.10.1/24 dev wlp2s0.10`**
* *Función:* Asigna la IP privada que sirve como puerta de enlace predeterminada (*Default Gateway*) para los equipos de dicha subred.




* **`ip link set dev wlp2s0.10 up`**
* *Función:* Enciende el adaptador virtual para comenzar a transmitir y recibir datos.





*(Esta secuencia se aplica idénticamente para las subinterfaces `wlp2s0.20`, `wlp2s0.30`, `wlp2s0.40` y `wlp2s0.50`)*.

---

#### **FASE 3: Limpieza y Políticas de Seguridad Base (`iptables`)**

1. **`iptables -F && iptables -X && iptables -Z && iptables -t nat -F`**
* **Descripción:** Realiza un vaciado (*Flush*) y reinicio completo del cortafuegos.


* **Función:** Elimina todas las reglas previas (`-F`), borra cadenas personalizadas (`-X`), reinicia los contadores de paquetes a cero (`-Z`) y limpia las reglas de NAT (`-t nat -F`).




2. **`iptables -P FORWARD DROP`**
* **Descripción:** Establece la política por defecto de la cadena `FORWARD` en **Descartar** (`DROP`).


* **Función:** Aplica el principio de seguridad de **Mínimo Privilegio (Deny-by-Default)**. Ningún paquete puede transitar entre subredes salvo que exista una regla explícita que lo apruebe.




3. **`iptables -P INPUT ACCEPT` y `iptables -P OUTPUT ACCEPT`**
* **Función:** Permiten el tráfico propio de la máquina host para mantener operativa la conectividad a internet personal y los sockets locales del sistema operativo.




4. **`iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT`**
* **Descripción:** Implementa la inspección de estado (*Stateful Firewall*).


* **Función:** Permite el retorno automático de paquetes pertenecientes a conexiones legítimas previamente iniciadas por usuarios autorizados.





---

#### **FASE 4: Listas de Control de Acceso Inter-VLAN (ACLs)**

1. **Reglas de Bloqueo y Auditoría para Contabilidad (VLAN 20):**
```bash
iptables -A FORWARD -s 192.168.20.0/24 -d 10.0.50.10 -j LOG --log-prefix "FW-DROP-CONTABILIDAD: " --log-level 4
iptables -A FORWARD -s 192.168.20.0/24 -d 10.0.50.10 -j DROP

```


* **Análisis:** La primera regla captura la cabecera del paquete no autorizado y genera una alerta con la etiqueta `"FW-DROP-CONTABILIDAD: "` en los logs del kernel (`journalctl`). La segunda regla destruye inmediatamente el paquete antes de que alcance el Servidor de Base de Datos.




2. **Reglas de Bloqueo y Auditoría para Recursos Humanos (VLAN 30):**
```bash
iptables -A FORWARD -s 192.168.30.0/24 -d 10.0.50.10 -j LOG --log-prefix "FW-DROP-RRHH: " --log-level 4
iptables -A FORWARD -s 192.168.30.0/24 -d 10.0.50.10 -j DROP

```


* **Análisis:** Aplica la misma directiva de auditoría y denegación estricta al segmento de RRHH hacia el servidor crítico.




3. **Reglas de Permiso Selectivo para Soporte Técnico (VLAN 40):**
```bash
iptables -A FORWARD -s 192.168.40.0/24 -d 10.0.50.10 -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -s 192.168.40.0/24 -d 10.0.50.10 -p tcp --dport 3306 -j ACCEPT

```


* **Análisis:** Autoriza el acceso desde Soporte Técnico únicamente hacia los servicios esenciales de administración remota SSH (puerto TCP 22) y el motor de Base de Datos MySQL/MariaDB (puerto TCP 3306).




4. **Reglas de Permiso Total para Administración (VLAN 10) y Soporte (VLAN 40):**
```bash
iptables -A FORWARD -s 192.168.10.0/24 -j ACCEPT
iptables -A FORWARD -s 192.168.40.0/24 -j ACCEPT

```


* **Análisis:** Otorga paso libre y total a los segmentos encargados de la gestión operativa hacia cualquier destino dentro de la infraestructura corporativa.





---

#### **FASE 5: Batería Automatizada de Pruebas e Inyección de Tráfico (`test_security`)**

1. **Inyección de Tráfico Simulado con `hping3`:**
```bash
hping3 -c 1 -S -a 192.168.20.10 -p 3306 10.0.50.10

```


* **`-c 1`:** Envía exactamente un paquete de prueba.


* **`-S`:** Establece la bandera TCP SYN (petición de inicio de conexión).


* **`-a 192.168.20.10`:** Realiza una falsificación de IP de origen (*IP Spoofing*) para simular la petición desde la subred deseada.


* **`-p 3306`:** Apunta al puerto objetivo de la Base de Datos.




2. **Captura de Evidencias de Seguridad:**
```bash
journalctl -k --grep="FW-DROP" -n 4 --no-pager

```


* **Función:** Filtra los registros del kernel para desplegar las marcas de tiempo y direcciones IP de los intentos de intrusión bloqueados.




```bash
iptables -L OUTPUT -v -n --line-numbers | grep -E "192.168.10.10|192.168.40.10"

```


* **Función:** Muestra el contador de la regla de acceso permitido en la tabla de `iptables` incrementando de `0` a `1` paquete (`pkts: 1`), demostrando que las redes autorizadas atraviesan el cortafuegos exitosamente.
