

# Links de apoyo
```
https://www.zabbix.com/la/download?zabbix=7.4&os_distribution=ubuntu&os_version=24.04&components=proxy&db=pgsql&ws=
Documentación Oficial : https://www.zabbix.com/documentation/current/es/manual
Implementation of TimescaleDB in Zabbix: Benefits, Key Tables, and Installation -> https://www.initmax.com/wiki/implementation-of-timescaledb-in-zabbix/
Zabbix 7.4 – instructions for installation in 5 minutes -> https://www.initmax.com/wiki/zabbix-7-4-instructions-for-installation-in-5-minutes/

Curso de Zabbix [Dmitry Lambert] - https://www.youtube.com/watch?v=4CJQTaJKxww&list=PLF7Hh_ikyQDpHiHhXwLtw57OCDF9jn7zL
```
 

## 🎯 Objetivo de arquitectura

*   **Servidor Zabbix Server (core):** Ejecuta el motor de monitoreo, procesa checks, almacena/recupera datos en la BD y atiende a agentes/proxys.
*   **Servidor Base de Datos (PostgreSQL):** Almacena historial, eventos, configuración. Carga intensiva en I/O y memoria.
*   **Servidor Interfaz Web (Frontend):** Provee la UI (PHP + Apache/Nginx) para administrar y visualizar Zabbix.

> **Beneficios:** aislamiento de cargas (DB vs. PHP vs. daemon), escalabilidad por capas, seguridad (menos superficie expuesta), y facilidad de tuning por rol.

***

## 🧩 ¿Qué hace cada paquete?

*   `zabbix-server-pgsql`  
    Servicio principal de Zabbix **compilado para usar PostgreSQL** como backend.
*   `zabbix-frontend-php`  
    Archivos del **frontend web** de Zabbix (PHP).
*   `php8.3-pgsql`  
    Extensión PHP para conectar **PHP → PostgreSQL** (necesaria para que el frontend hable con la BD).
*   `zabbix-apache-conf`  
    Configuración de **Apache** para publicar el frontend Zabbix (vhost, alias, etc.).
*   `zabbix-sql-scripts`  
    **Esquemas y datos iniciales** SQL (creación de base, tablas, índices) que usarás para inicializar la BD de Zabbix.
*   `zabbix-agent`  
    **Agente** que instalas en los hosts monitoreados (incluyendo, si quieres, estos tres servidores).
*   `zabbix-proxy-pgsql`  
    **Proxy** Zabbix que agrega/almacena datos temporalmente y los reenvía al server. Útil en sitios remotos o para escalar. (Opcional; no es necesario en arquitectura básica de 3 nodos).

***

## 🏗️ Arquitectura propuesta (3 servidores)

                     +--------------------+
                     |  Interfaz Web      |
                     |  (PHP + Apache)    |
                     |  zabbix-frontend   |
                     +----------+---------+
                                |
                                | HTTP(S) para UI
                                v
    +----------------+      +---+-----------------+      +----------------------+
    | Dispositivos   |-->---|  Zabbix Server      |--->--|  Base de Datos       |
    | y Agentes      |      |  (daemon + worker)  |      |  PostgreSQL          |
    | zabbix-agent   |<--\  | zabbix-server-pgsql |<--->-| schema Zabbix        |
    +----------------+   \--|                      |      +----------------------+
                          \-> (opcional) Zabbix Proxy
                              zabbix-proxy-pgsql (sitios remotos)

**Puertos clave:**

*   Agente → Server/Proxy: **10051/TCP** (server escucha), **10050/TCP** (agent escucha para checks activos).
*   Frontend (Apache/Nginx): **80/443**.
*   Server ↔ Base de Datos (PostgreSQL): **5432/TCP**.

***

## 📦 ¿Qué instalar en cada servidor y por qué?

### 1) **Servidor Zabbix Server**

**Paquetes:**

```bash
sudo apt install zabbix-server-pgsql zabbix-sql-scripts zabbix-agent
```

*   `zabbix-server-pgsql`: núcleo del monitor.
*   `zabbix-sql-scripts`: para inicializar la BD (lanzar los scripts desde aquí o desde el servidor BD).
*   `zabbix-agent`: opcional pero recomendable para **monitorear el propio Zabbix Server** (CPU, RAM, discos, procesos).

> **Por qué aquí:** el daemon del server no debe compartir host con la BD o el frontend para no competir por CPU/IO; además, mejora seguridad (expone solo 10051).

***

### 2) **Servidor Base de Datos (PostgreSQL)**

**Paquetes (del sistema):**

```bash
sudo apt install postgresql postgresql-contrib
```

*(No necesitas paquetes Zabbix específicos aquí, salvo que decidas ejecutar los scripts SQL desde este host.)*

> **Por qué aquí:** separación de responsabilidades; la BD requiere tuning independiente (memoria compartida, autovacuum, I/O, backups, PITR). Mantenerla aislada reduce ruido de otras cargas.

***

### 3) **Servidor Interfaz Web (Frontend)**

**Paquetes:**

```bash
sudo apt install zabbix-frontend-php php8.3-pgsql zabbix-apache-conf
```

*   `zabbix-frontend-php`: UI.
*   `php8.3-pgsql`: PHP → PostgreSQL.
*   `zabbix-apache-conf`: publica el frontend rápidamente (puedes optar por Nginx si prefieres).

> **Por qué aquí:** el frontend consume CPU (PHP-FPM) y necesita su propio escalado/ caché. Separarlo evita que consultas pesadas del UI afecten al motor o a la BD.

***

### (Opcional) **Servidor Zabbix Proxy**

**Paquetes (si tienes sedes remotas o gran escala):**

```bash
sudo apt install zabbix-proxy-pgsql zabbix-agent
```

*   Proxy almacena temporalmente y reenvía al server; **reduce latencia** y **aisla redes** remotas.
*   `zabbix-agent` para monitorizar el proxy mismo.

> **Si no tienes sitios remotos o miles de hosts**, puedes omitir el proxy en esta primera fase.

***

## ⚙️ Pasos de configuración mínimos

### A) Inicializar la Base de Datos (en el servidor PostgreSQL)

1.  Crear usuario/BD:

```sql
-- como usuario postgres (psql)
CREATE USER zabbix WITH ENCRYPTED PASSWORD 'TuClaveFuerte';
CREATE DATABASE zabbix OWNER zabbix;
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
```

2.  Ajustar `pg_hba.conf` para permitir acceso desde las IPs de **Zabbix Server** y **Frontend**:

<!---->

    host    zabbix    zabbix    <IP_ZABBIX_SERVER>/32    md5
    host    zabbix    zabbix    <IP_FRONTEND>/32         md5

3.  Reiniciar PostgreSQL.

### B) Cargar el esquema Zabbix (desde el servidor Zabbix Server o desde la BD)

> Ubicación típica en Debian/Ubuntu:

```bash
# En el servidor Zabbix Server
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | \
  PGPASSWORD=TuClaveFuerte psql -h <IP_DB> -U zabbix -d zabbix
```

### C) Configurar `zabbix_server.conf` (en Zabbix Server)

```ini
DBHost=<IP_DB>
DBName=zabbix
DBUser=zabbix
DBPassword=TuClaveFuerte
# Ajustes de performance (ejemplos):
StartPollers=10
CacheSize=256M
HistoryCacheSize=256M
TrendCacheSize=128M
```

Reinicia y habilita:

```bash
sudo systemctl enable --now zabbix-server
```

### D) Configurar Frontend (en Interfaz Web)

*   Verifica que Apache publique el sitio (URL /zabbix o según el vhost instalado).
*   En el **wizard** de instalación, apunta al **host** y credenciales de PostgreSQL.
*   `php.ini` (o pool de PHP-FPM): ajusta `memory_limit`, `max_execution_time`, `date.timezone`, etc.

### E) Instalar y registrar agentes (en los tres servidores y otros hosts)

```bash
sudo apt install zabbix-agent
# /etc/zabbix/zabbix_agentd.conf
Server=<IP_ZABBIX_SERVER>
ServerActive=<IP_ZABBIX_SERVER>
Hostname=<nombre_del_host>
sudo systemctl enable --now zabbix-agent
```

***

## 🔐 Buenas prácticas de seguridad y rendimiento

*   **SSL/TLS en el frontend** (HTTPS) y, si es posible, **túneles cifrados** o VPN para tráfico a 10050/10051.
*   **Usuarios mínimos y contraseñas fuertes** en PostgreSQL; usa `pg_hba.conf` con CIDR restringido.
*   **Tuning PostgreSQL**: `shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size` según RAM; habilitar `autovacuum` agresivo para tablas de historia.
*   **Rotación y compresión de logs** (server, agent, Apache).
*   **Backups y PITR** de la BD; prueba restore (no sólo dump).
*   **Housekeeper y trends**: ajusta retenciones en Zabbix (ej. 90 días en history, 365 en trends) para controlar tamaño.
*   **Proxy** en sitios remotos o enlaces inestables para resiliencia.

***

## 📋 Resumen rápido de instalación por servidor

**Zabbix Server**

```bash
apt install zabbix-server-pgsql zabbix-sql-scripts zabbix-agent
```

**Base de Datos (PostgreSQL)**

```bash
apt install postgresql postgresql-contrib
# crear usuario/BD zabbix, cargar /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz
```

**Interfaz Web (Frontend)**

```bash
apt install zabbix-frontend-php php8.3-pgsql zabbix-apache-conf
# configurar PHP, apuntar al host PostgreSQL en el wizard
```

**(Opcional) Proxy**

```bash
apt install zabbix-proxy-pgsql zabbix-agent
```

***

Si quieres, te armo **playbooks Ansible** o **scripts Bash** para desplegar todo en tu entorno (con variables para IPs, usuarios y retenciones) y un checklist de validación post‑instalación. ¿Usarás **Nginx** en lugar de Apache para el frontend o te quedas con `zabbix-apache-conf`?




```
--- Levantar Zabbix
systemctl  enable zabbix-server.service
systemctl  enable zabbix-agent
systemctl  start zabbix-server.service

```
