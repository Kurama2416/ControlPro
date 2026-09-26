# 🛡️ Segmentación de Red e Implementación de IDS - ControlPro S.A.S.

Estrategia de **Defensa en Profundidad** desplegada sobre Fedora Linux. Este proyecto combina el filtrado perimetral e inter-VLAN en capas 3/4 mediante **iptables/Netfilter** con la inspección profunda de paquetes (DPI) en capa 7 mediante **Suricata IDS**.

---

## 📐 Arquitectura de Seguridad

### 1. Modelo de Defensa en Profundidad

* **Capa 3 / 4 (Netfilter / iptables):** Control de acceso perimetral e inter-VLAN bajo la política de *Mínimo Privilegio* (`FORWARD DROP`).
* **Capa 7 (Suricata DPI Sensor):** Auditoría pasiva sobre la interfaz perimetral (`wlp2s0`) vinculada mediante `AF_PACKET`.

---

### 2. Matriz de VLANs y Direccionamiento IP

| Subinterfaz | Departamento / Zona | Segmento IP | Política iptables | Propósito |
| --- | --- | --- | --- | --- |
| `wlp2s0.10` | Administración | `192.168.10.0/24` | `ACCEPT` | Acceso autorizado al Servidor BD |
| `wlp2s0.20` | Contabilidad | `192.168.20.0/24` | `DROP` | Bloqueo registrado con traza `FW-DROP` |
| `wlp2s0.30` | Recursos Humanos | `192.168.30.0/24` | `DROP` | Bloqueo registrado con traza `FW-DROP` |
| `wlp2s0.40` | Soporte Técnico | `192.168.40.0/24` | `ACCEPT` | Acceso SSH (`22`) y MySQL (`3306`) a BD |
| `wlp2s0.50` | Servidores / BD | `10.0.50.0/24` | `ZONA BD` | Servidor objetivo MySQL (`10.0.50.10:3306`) |

---

## 🚀 Uso del Script Automatizado (`controlpro-net.sh`)

El script `controlpro-net.sh` gestiona todo el ciclo de vida del laboratorio.

```bash
# 1. Desplegar subinterfaces VLAN, IP Forwarding e iptables
sudo ./controlpro-net.sh install

# 2. Consultar el estado de las VLANs y reglas cargadas
sudo ./controlpro-net.sh status

# 3. Ejecutar simulación automatizada de tráfico e intrusiones
sudo ./controlpro-net.sh test

# 4. Eliminar subinterfaces y restaurar el sistema
sudo ./controlpro-net.sh uninstall
```

---

## 🧪 Pruebas Manuales de Auditoría (Paso a Paso)

Si deseas realizar la validación de auditoría directamente en la consola sin depender del script, sigue este procedimiento:

### Paso 1: Configurar Reglas Temporales

```bash
sudo iptables -A OUTPUT -s 192.168.20.10 -d 10.0.50.10 -j LOG --log-prefix "FW-DROP-CONTABILIDAD: " --log-level 4
sudo iptables -A OUTPUT -s 192.168.20.10 -d 10.0.50.10 -j DROP
sudo iptables -A OUTPUT -s 192.168.30.10 -d 10.0.50.10 -j LOG --log-prefix "FW-DROP-RRHH: " --log-level 4
sudo iptables -A OUTPUT -s 192.168.30.10 -d 10.0.50.10 -j DROP
sudo iptables -A OUTPUT -s 192.168.10.10 -d 10.0.50.1 -j ACCEPT
sudo iptables -A OUTPUT -s 192.168.40.10 -d 10.0.50.1 -j ACCEPT
```

---

### Paso 2: Inyectar Paquetes de Prueba (`hping3`)

```bash
sudo hping3 -c 1 -S -a 192.168.20.10 -p 3306 10.0.50.10
sudo hping3 -c 1 -S -a 192.168.30.10 -p 3306 10.0.50.10
sudo hping3 -c 1 -S -a 192.168.10.10 -p 80 10.0.50.1
sudo hping3 -c 1 -S -a 192.168.40.10 -p 3306 10.0.50.1
```

---

### Paso 3: Verificar Evidencias

```bash
# Ver trazas de bloqueos registrados en el kernel
sudo journalctl -k --grep="FW-DROP" -n 2 --no-pager

# Consultar contadores de tráfico permitido
sudo iptables -L OUTPUT -v -n --line-numbers | grep -E "192.168.10.10|192.168.40.10"
```

---

### Paso 4: Limpiar Reglas Temporales

```bash
sudo iptables -D OUTPUT -s 192.168.20.10 -d 10.0.50.10 -j LOG --log-prefix "FW-DROP-CONTABILIDAD: " --log-level 4
sudo iptables -D OUTPUT -s 192.168.20.10 -d 10.0.50.10 -j DROP
sudo iptables -D OUTPUT -s 192.168.30.10 -d 10.0.50.10 -j LOG --log-prefix "FW-DROP-RRHH: " --log-level 4
sudo iptables -D OUTPUT -s 192.168.30.10 -d 10.0.50.10 -j DROP
sudo iptables -D OUTPUT -s 192.168.10.10 -d 10.0.50.1 -j ACCEPT
sudo iptables -D OUTPUT -s 192.168.40.10 -d 10.0.50.1 -j ACCEPT
```

---

## 🔍 Prueba de Validación de IDS (Suricata)

Para comprobar la detección en Capa 7, dispara la firma de prueba conocida mediante `curl`:

```bash
# 1. Ejecutar petición de prueba
curl http://testmyids.com

# 2. Monitorear las alertas en tiempo real
sudo tail -f /var/log/suricata/fast.log
```
