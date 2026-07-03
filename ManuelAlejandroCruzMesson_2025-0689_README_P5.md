# IPSec IKEv1: VPN Site-to-Site Punto a Punto Basada en Políticas

**Asignatura:** Seguridad de Redes
**Estudiante:** Manuel Cruz
**Docente:** Jonathan Rondón
**Fecha:** 29 de junio de 2026
---

## Tabla de Contenidos

- [1. Resumen y Objetivos](#1-resumen-y-objetivos)
- [2. Topología de Red y Direccionamiento](#2-topología-de-red-y-direccionamiento)
- [3. Parámetros de Seguridad Utilizados (Políticas)](#3-parámetros-de-seguridad-utilizados-políticas)
- [4. Puntos Críticos de la Configuración](#4-puntos-críticos-de-la-configuración)
- [5. Verificación](#5-verificación)

---

## 1. Resumen y Objetivos

Este documento detalla la implementación y verificación de una **Red Privada Virtual (VPN) Site-to-Site** utilizando el protocolo **IPsec** (Internet Protocol Security) entre dos sedes remotas (**PEER A** y **PEER B**), interconectadas a través de un proveedor de servicios de Internet simulado (**R-ISP**).

### Objetivos del Proyecto

- **Confidencialidad e Integridad:** asegurar el tráfico de datos entre la LAN de PEER A (`172.16.1.0/24`) y la LAN de PEER B (`172.16.2.0/24`) mediante cifrado robusto de extremo a extremo.
- **Enrutamiento Óptimo:** configurar rutas estáticas que permitan alcanzar las subredes remotas a través del direccionamiento público asignado por el ISP.

---

## 2. Topología de Red y Direccionamiento

| Dispositivo / Sede | Interfaz | Dirección IP | Máscara de Subred | Propósito / Rol |
|---|---|---|---|---|
| PEER A | Gi0/0 | 10.0.0.60 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER A | Gi0/1 | 172.16.1.1 | 255.255.255.0 | Gateway LAN A / NAT Inside |
| PEER B | Gi0/0 | 10.0.0.70 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER B | Gi0/1 | 172.16.2.1 | 255.255.255.0 | Gateway LAN B / NAT Inside |
| R-ISP (Gateway) | N/A | 10.0.0.1 | 255.255.255.0 | Puerta de enlace predeterminada de la WAN |

---

## 3. Parámetros de Seguridad Utilizados (Políticas)

Para el establecimiento seguro del túnel, se ha unificado la suite de criptografía en ambos peers bajo los estándares de la **Fase 1 (ISAKMP)** y la **Fase 2 (IPsec)**.

### Fase 1: IKEv1 (ISAKMP Policy)

| Parámetro | Valor |
|---|---|
| Algoritmo de Cifrado | AES-256 (Advanced Encryption Standard, 256 bits) |
| Algoritmo de Hash | SHA-256 (Secure Hash Algorithm, 256 bits) |
| Método de Autenticación | Pre-Shared Key (Clave precompartida) |
| Grupo Diffie-Hellman | Group 14 (Modular Exponentiation de 2048 bits) |
| Tiempo de Vida (Lifetime) | 86,400 segundos (24 horas) |
| Pre-Shared Key Empleada | `ClaveSegura123` |

### Fase 2: IPsec (Transform Set)

| Parámetro | Valor |
|---|---|
| Nombre del Transform Set | `MI_SET` |
| Encapsulación de Cifrado | esp-aes 256 |
| Autenticación de Datos | esp-sha256-hmac |
| Modo de Operación | mode tunnel |

---

## 4. Puntos Críticos de la Configuración

### A. Prevención de NAT (Exclusión del Tráfico de la VPN)

Para evitar que el router aplique PAT (Overload) al tráfico inter-sede y rompa el *match* de la VPN, se configuró una regla de denegación previa en la ACL de NAT.

> La sentencia `deny` le indica al proceso de NAT que ignore el tráfico que va hacia la LAN remota, permitiendo que conserve sus direcciones IP originales. La sentencia `permit` posterior gestiona la salida tradicional a Internet.

### B. Enrutamiento Estático

La ruta por defecto gestiona la navegación general. La ruta estática hacia la subred remota fuerza al router a dirigir el tráfico por la interfaz WAN hacia el peer destino, permitiendo que la crypto map intercepte los paquetes.

### C. Crypto Map

Define el **tráfico interesante**. Al aplicarse en la interfaz externa (Gi0/0), el router evalúa los paquetes salientes: si coinciden con los rangos definidos, se inicia la negociación y el cifrado IPsec.

---

## 5. Verificación

### Validación de Fase 1 (ISAKMP)

```
show crypto isakmp sa
```

Debe mostrar el estado **QM_IDLE**. Esto confirma que la fase de negociación de la clave precompartida y los algoritmos fue exitosa.

### Validación de Fase 2 (IPsec)

```
show crypto ipsec sa
```

Verifica las líneas `#pkts encaps: X`, `#pkts encrypt: X`, `#pkts digest: X` y sus contrapartes de `decap`/`decrypt`. Deben ser mayores a 0 tras generar tráfico.

### Prueba de Conectividad End-to-End

Desde el VPC de PEER A:

```
ping 172.16.2.10
```

Una respuesta exitosa (**Ping exitoso**) demuestra que el tráfico cruza el R-ISP de manera transparente y cifrada.
