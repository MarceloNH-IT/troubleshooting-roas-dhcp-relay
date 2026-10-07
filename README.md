# troubleshooting-roas-dhcp-relay
# 🛠️ Resolución de Incidentes (Troubleshooting): Inter-VLAN Routing (ROAS) y DHCP Relay Agent

## 📋 Contexto y Ticket de Soporte (Gestión de Incidentes)
* **ID del Ticket:** INC-84092
* **Prioridad:** Alta (Afectación operativa en Sucursal)
* **Escalamiento:** El ticket fue escalado por el equipo de Soporte Nivel 1 (HelpDesk) al área de Ingeniería de Redes. El N1 verificó que los cables físicos estuvieran conectados y reinició los equipos finales sin éxito.
* **Descripción del Incidente:** Los usuarios del departamento de Operaciones (VLAN 20) reportan pérdida total de conectividad a la red local e internet. El departamento de Administración (VLAN 10), ubicado en la misma sucursal, opera con total normalidad.
* **Síntomas Iniciales:** Las PCs de Operaciones no logran contactar al Servidor DHCP Central, recibiendo por defecto un direccionamiento de falla APIPA (`169.254.x.x`).

---

## 🗺️ Arquitectura y Topología de Red
La infraestructura consta de un **Servidor DHCP Central (R1)** conectado mediante un enlace WAN a un **Router de Sucursal (R2)**. Este último utiliza la técnica **Router-on-a-Stick (ROAS)** para enrutar el tráfico de las VLANs 10 y 20 que provienen del **Switch de Acceso (S1)**.

![Topología de Red ROAS y DHCP Relay](./Topologia.jpg)

*(Nota: Configuración base del Servidor DHCP Central)*
![Configuración DHCP Central](./DHCP-CENTRAL.jpg)

---

## 🔍 Metodología de Diagnóstico (Troubleshooting Paso a Paso)

Para aislar el problema y encontrar la causa raíz (Root Cause), se aplicó un enfoque deductivo de abajo hacia arriba (*Bottom-Up*), siguiendo el Modelo OSI.

### Paso 1: Verificación de Capa 2 (Switching y VLANs)
Primero se descartó una mala configuración en los puertos de acceso o en el enlace troncal del switch de la sucursal.
* **Comando:** `S1_Sucursal# show vlan brief`
* **Comando:** `S1_Sucursal# show interfaces trunk`

![Verificación de Capa 2 en Switch](./Swicht-S1.jpg)

* **Conclusión Nivel 2:** La capa de Switching funciona sin errores. Los puertos están en las VLANs correctas y el enlace troncal opera en modo 802.1Q. El problema reside en el enrutamiento.

### Paso 2: Verificación de Capa 3 (Inter-VLAN Routing)
Dado que el Servidor DHCP está en una red remota, el Router 2 debe actuar como agente de retransmisión (*DHCP Relay Agent*). Se inspeccionaron las subinterfaces lógicas:
* **Comando:** `R2_Sucursal# show running-config`

**⚠️ Hallazgo (Root Cause Analysis):**
Al revisar la subinterfaz `GigabitEthernet0/1.20` (Gateway de la VLAN 20), se detectó la ausencia crítica del comando `ip helper-address`. 
Al no estar este parámetro, el router frena por defecto los paquetes *Broadcast* (peticiones DHCP) de la PC de Operaciones, impidiendo que lleguen a la IP del Servidor Central (`10.0.0.1`). 

![Análisis de Causa Raíz en Router de Sucursal](./R2-Sucursal.jpg)

---

## 🛠️ Resolución del Incidente

Se inyectó el parámetro faltante en la subinterfaz afectada del router de la sucursal, habilitando el reenvío de las peticiones DHCP en formato *Unicast* hacia el servidor central.

**Comandos de mitigación aplicados:**
```text
R2_Sucursal> enable
R2_Sucursal# configure terminal
R2_Sucursal(config)# interface GigabitEthernet0/1.20
R2_Sucursal(config-subif)# ip helper-address 10.0.0.1
R2_Sucursal(config-subif)# end
R2_Sucursal# write memory
```

✅ Verificación y Pruebas Post-Resolución
Una vez aplicado el parche de configuración, se validó la mitigación del incidente directamente desde el equipo del usuario final.

1. Obtención de Direccionamiento (DHCP Request)
Se forzó una renovación de la interfaz de red en la PC_Operaciones. El equipo logró comunicarse exitosamente con el Servidor DHCP remoto, abandonando la IP APIPA y recibiendo los parámetros correctos de su segmento (192.168.20.11).

[📸 INSERTAR IMAGEN AQUÍ: ![Ping-Sin-perdida](./Ping-Sin-perdida.jpg)" con la IP 192.168.20.11]

2. Conectividad End-to-End (ICMP Ping)
Se comprobó la estabilidad del enrutamiento ROAS enviando paquetes ICMP hacia una PC de otra red (VLAN 10). La tabla ARP resolvió la dirección y los paquetes llegaron con un 0% de pérdida.

[📸 INSERTAR IMAGEN AQUÍ: Captura de la ventana del Command Prompt mostrando el segundo ping exitoso (Sent = 4, Received = 4, Lost = 0)]

Estado del Ticket: CERRADO Y DOCUMENTADO 🟢

