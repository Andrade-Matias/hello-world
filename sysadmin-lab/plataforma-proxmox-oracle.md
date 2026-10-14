# Plataforma de lab — Proxmox + Debian + Oracle + SQL Developer

Este plan asume que virtualizás en **Proxmox VE** (basado en Debian) sobre tu PC (**i5-12400f, 32 GB RAM**) y que la capa de datos es **Oracle Database** (edición Free/XE en lab), administrada desde **SQL Developer** en tu máquina de escritorio.

---

## Roles de cada capa

| Capa | Qué es | Tu trabajo como sysadmin |
|------|--------|---------------------------|
| **Proxmox (host)** | Hipervisor en el metal | VMs, redes, snapshots, recursos CPU/RAM/disco |
| **VM Debian 12** | Servidores Linux (`srv-core`, `srv-web`) | SSH, systemd, Nginx, PHP-FPM, firewall |
| **VM Oracle** | Motor Oracle (recomendado **Oracle Linux 9**) | Listener, instancia, tablespaces, backups, puerto **1521** |
| **SQL Developer** | Cliente gráfico en tu PC | Conexiones, pruebas, diagnóstico (no reemplaza `systemctl` en el servidor) |

> **Por qué Oracle Linux para `srv-oracle`:** instalar Oracle Database en Debian es posible pero frágil para un lab. En empresas con Oracle suele haber **Linux + Oracle DB**; Debian queda para app/web. Si insistís en todo Debian, usá la [Oracle Database Free para Linux](https://www.oracle.com/database/free/) siguiendo la guía oficial para tu distro.

---

## Red en Proxmox (lab `10.10.10.0/24`)

1. En Proxmox: **Datacenter → tu nodo → Network** — bridge `vmbr0` unido a la NIC física (o NAT solo para lab).
2. **Gateway del lab:** `10.10.10.1` (puede ser la IP del host Proxmox en esa VLAN, o el router de tu red si las VMs están en LAN doméstica).
3. **DNS:** el de tu router, `1.1.1.1`, o un resolver en `srv-core` más adelante.

| VM | Hostname | IP | RAM (mín.) | Disco | SO |
|----|----------|-----|------------|-------|-----|
| `srv-core` | srv-core | 10.10.10.10 | 2 GB | 20 GB | Debian 12 |
| `srv-web` | srv-web | 10.10.10.20 | 4 GB | 30 GB | Debian 12 |
| `srv-oracle` | srv-oracle | 10.10.10.30 | 8 GB | 60 GB | Oracle Linux 9 (o Debian + Free DB) |

Con **32 GB** en el host: dejá ~4 GB para Proxmox; el resto repartido. No levantes las tres a full carga al mismo tiempo hasta validar RAM.

---

## Crear una VM Debian en Proxmox (checklist)

1. Descargar ISO [Debian 12 netinst](https://www.debian.org/CD/netinst/).
2. **Create VM:** ID libre, nombre `srv-core`, ISO, **VirtIO SCSI**, disco **qcow2** 20G, **Qemu agent** activado (instalá `qemu-guest-agent` en Debian después).
3. **CPU:** tipo `host` si está estable; **cores:** 2.
4. **Network:** bridge `vmbr0`, modelo **VirtIO**.
5. Instalar Debian: solo **SSH server**, sin escritorio.
6. IP estática en la VM (ver semana 2) o DHCP reservado en tu router.
7. **Snapshot:** `clean-install` antes de romper cosas en tickets.

Documentá en bitácora: **VMID**, IP, usuario SSH.

---

## Oracle en `srv-oracle` (resumen instalación lab)

1. VM con **8 GB RAM** y swap acorde (Oracle suele pedir ≥2 GB swap en docs de pre-requisitos).
2. Instalar **Oracle Database 23c Free** (o 21c XE) desde [oracle.com/database/free](https://www.oracle.com/database/free/get-started/).
3. Anotar:
   - **SID / servicio:** ej. `FREE` / `FREEPDB1` (según versión; en 23c Free suele usarse PDB `FREEPDB1`).
   - **Puerto listener:** `1521`.
   - Usuario admin de lab (no uses `SYS` en apps).

**Servicios en Linux (Oracle Linux):**

```bash
sudo systemctl status oracle-free-23c  # nombre puede variar; usar systemctl list-units '*oracle*'
lsnrctl status
```

**Archivos útiles:**

- `$ORACLE_HOME/network/admin/listener.ora`
- `$ORACLE_HOME/network/admin/tnsnames.ora`
- Alert log: bajo `$ORACLE_BASE/diag/...` (ruta según instalación)

---

## SQL Developer en tu PC

1. Descargar [Oracle SQL Developer](https://www.oracle.com/database/sqldeveloper/technologies/download/) (requiere JDK).
2. Nueva conexión:
   - **Tipo:** Basic o TNS.
   - **Hostname:** `10.10.10.30`
   - **Port:** `1521`
   - **Service name:** `FREEPDB1` (ajustar a tu instalación)
   - Usuario de app: ej. `APPUSER` creado en el PDB.
3. Probar **Test** antes de los tickets de semana 3.

**Cadena equivalente (para `tnsnames.ora` en tu PC):**

```
LAB_ORACLE =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.10.10.30)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = FREEPDB1))
  )
```

---

## Snapshots Proxmox (convención)

| Nombre | Cuándo |
|--------|--------|
| `clean-install` | Post SO, pre tickets |
| `pre-week-N` | Domingo antes de simulacro |
| `broken-T-00X` | Opcional: estado roto para repetir ticket |

Revertir snapshot **apaga** la VM un instante; coordiná con tickets abiertos.

---

## Puertos a recordar (firewall semana 2 + Oracle)

| Puerto | Servicio |
|--------|----------|
| 22 | SSH |
| 80 / 443 | Nginx |
| 1521 | Oracle Listener |
| 5500 | EM Express (opcional en XE/Free; a veces deshabilitado) |

En `srv-oracle`, abrir **1521/tcp** solo desde `10.10.10.0/24` (ufw/firewalld), no a Internet.

---

## Relación con el stack PHP

- **srv-web** sirve PHP (Nginx + PHP-FPM).
- **srv-oracle** solo base de datos.
- Conexión desde PHP: extensión **OCI8** (`php-oci8`) o API intermedia; en semana 3 el ticket T-007 se resuelve primero con **SQL Developer** y listener; la parte PHP es opcional (ver semana 3).
