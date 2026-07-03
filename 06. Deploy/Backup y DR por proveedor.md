---
tipo: nota-deploy
capa: transversal
proposito: backup, restore y DR del sistema destino ECOREX Tareas (.NET 10) - procedimientos diferenciados PostgreSQL / SQL Server
estado: planeado (destino de migracion)
---

# Backup y Disaster Recovery por proveedor

> ECOREX Tareas destino corre sobre DAL dual: cada tenant vive en PostgreSQL O
> SQL Server. El backup y la recuperacion difieren por motor. Este documento es el
> runbook. Complementa [[Deploy a Produccion - Docker e hibrido]] y [[CICD Pipeline]].

## 1. Objetivos RTO / RPO

| Escenario | RTO (recuperacion) | RPO (perdida max) |
|---|---|---|
| Corrupcion de dato por bug | 15 min | 5 min |
| Perdida de un tenant | 30 min | 1 h |
| Fallo del host primario | 2 h | 15 min |
| Catastrofe multi-region | 4 h | 24 h |

## 2. Estrategia diferenciada por proveedor

### 2.1 Tenants PostgreSQL (default SaaS)

**Hosting**: VPS Contabo o nube gestionada (Railway/Supabase/Neon).

- **WAL archiving continuo** -> almacenamiento por tenant cada 5 min.
- **Snapshot diario** (3 AM UTC) con retencion 30 dias.
- **Snapshot semanal** a cold storage, retencion 6 meses.
- **Snapshot mensual** a archive, retencion 5 anos (compliance Ley 1581).

```bash
# Backup nocturno por tenant
pg_dump --format=custom --compress=9 \
  --file=/backups/{tenant_id}-$(date +%F).dump "$POSTGRES_URL"
# Subir a almacenamiento
aws s3 cp /backups/{tenant_id}-$(date +%F).dump s3://ecorex-backups/postgres/{tenant_id}/
```

Restore + point-in-time con `pg_restore` + WALs hasta el timestamp deseado.

### 2.2 Tenants SQL Server (enterprise on-prem)

**Hosting**: SQL Server 2022 del cliente o Azure SQL MI. Es el mismo motor del
origen (`db3dev` -> `db3prod`), asi que la infraestructura de backup del legacy se
reutiliza y se formaliza.

- **Full backup nocturno** (2 AM UTC) cifrado.
- **Differential cada 6h**.
- **Transaction log cada 15 min** (habilita PITR).
- Retencion 30d hot / 6m warm / 5y cold.

```sql
BACKUP DATABASE [ecorex_{tenant}]
TO URL = 'https://.../full-{fecha}.bak'
WITH COMPRESSION, ENCRYPTION (ALGORITHM = AES_256, SERVER CERTIFICATE = BackupCert);

BACKUP LOG [ecorex_{tenant}]
TO URL = 'https://.../log-{fecha-hora}.trn' WITH COMPRESSION;
```

Restore con `RESTORE DATABASE ... WITH NORECOVERY` + diffs + logs y `STOPAT` para
PITR. **AlwaysOn Availability Groups** para HA (replica sincrona local + asincrona
en region secundaria) en clientes que lo exijan.

## 3. Backups aplicativos (comunes a ambos motores)

- **Object Storage** (adjuntos de tareas, archivos de formularios): versionado +
  soft-delete 30 dias + replicacion cross-region para tenants Pro/Enterprise.
- **Redis**: NO se respalda estado transaccional (es cache y locks). SI se respalda
  el estado de sesiones/refresh tokens via dump AOF nocturno.
- **Secretos**: en Key Vault con soft-delete + purge protection 90 dias.
- **Codigo**: GitHub + mirror semanal + imagenes Docker en registro dual.

## 4. Playbook - corrupcion de dato en un tenant

Ej.: una regla mal configurada o un bug sobrescribe campos de tareas.

1. **Aislar**: pausar el tenant (`TenantStatus = Suspended`) para frenar cambios.
2. **Restore paralelo** a una BD temporal (segun el motor del tenant).
3. **Extraer** los registros correctos de la BD temporal.
4. **Merge selectivo** solo de lo afectado.
5. **Verificar** con la auditoria (`AdminAuditLog`) + el responsable del tenant.
6. **Reactivar**: `TenantStatus = Active`.
7. **Postmortem**: registrar incidente, agregar test de regresion.

Tiempo estimado: 30-60 min.

## 5. Playbook - perdida del host primario

1. **Detectar** (alertas de health check).
2. **Cambiar DNS/reverse proxy** al host secundario.
3. **Promover replicas**: Postgres `pg_ctl promote`; SQL Server `ALTER AVAILABILITY GROUP FAILOVER`.
4. **Verificar** migraciones aplicadas + smoke test por tenant piloto.
5. **Reactivar** servicio para todos los tenants.
6. **Comunicar** a los clientes.

Tiempo objetivo: 2 horas.

## 6. Aislamiento del backup por tenant

Como el aislamiento es una invariante (ver [[Gestion de Empresas - Admin multi-tenant]]),
los backups tambien se segregan por `tenant_id`: un restore de un tenant nunca
puede traer datos de otro. Los backups van a rutas/containers por tenant, cifrados.

## 7. Test de restore - cadencia

- **Diario**: smoke automatico — restaura un backup al azar en sandbox y valida integridad.
- **Semanal**: restore completo de 1 tenant a un entorno DR, mide performance.
- **Trimestral**: simulacro de DR completo (medir RTO real).
- **Anual**: pentest sobre los backups (que un atacante no pueda descargarlos).

## 8. Monitoreo de backups

Dashboard `Backup Health`: ultimo backup exitoso por tenant y por motor, tamano
(detecta anomalias), tiempo del restore de smoke diario, cobertura (todos los
tenants activos deben aparecer). Alerta si: 2 fallos consecutivos, cambio de tamano
>50%, o smoke de restore fallido.

## 9. Compliance

Ley 1581 (Colombia): retencion minima de datos de negocio, cifrado en reposo,
prueba de restore mensual documentada, auditoria de acceso a backups. Cada tenant
enterprise firma DPA con la politica de RTO/RPO y notificacion de incidentes (72h).

## 10. Enlaces

- [[Deploy a Produccion - Docker e hibrido]] — infra donde corren los backups
- [[CICD Pipeline]] — deploys que respetan backups pre-migracion
- [[00 - Visión MotherData]] — DAL dual (proveedores diferenciados)
- [[Gestion de Empresas - Admin multi-tenant]] — aislamiento por tenant
