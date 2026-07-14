---
tipo: historias-qa
modulo: Formularios (Capa 4)
proposito: Historias de prueba en formato Dado/Cuando/Entonces, AJUSTABLES por el usuario. Son la fuente de la que salen los tests automatizados. El QA las ejecuta de forma adversarial (busca romperlas).
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Formularios

> Historias de usuario orientadas a QA. **Ajustalas libremente** (datos, montos, nombres,
> tenants): lo que quede aqui es el contrato que el QA convierte en tests. Formato
> Dado/Cuando/Entonces. Cada historia lleva su severidad si falla y si hoy esta cubierta.
> Politica y DoD: [[00 - Politicas QA y Definition of Done]]. Veredicto:
> [[00 - Plan y veredicto QA - Formularios]].

## F1 - Lookups / autocompletado

### H1.1 - Autocompletar cliente desde el Directorio (camino feliz)
- **Dado** un formulario con un campo "Cliente" (Origen=Directorio, Presentacion=Autocompletar)
- **Cuando** el usuario escribe "and" en el campo
- **Entonces** el sistema muestra terceros del tenant que coinciden (busqueda server-side)
- **Y** al elegir "ANDINA S.A.S" el campo guarda el **id** del tercero
- **Y** se **copian** NIT y Ciudad a sus campos destino (autollenado por copia)
- Severidad si falla: Alto | Cubierta hoy: manual (falta test)

### H1.2 - Autocompletar item desde Inventario
- **Dado** un campo "Item" (Origen=Inventario)
- **Cuando** el usuario elige un item
- **Entonces** se copian precio y unidad a sus campos
- Severidad: Alto | Cubierta: **no** (adaptador no verificado)

### H1.3 - Lista desde Contenedor de datos
- **Dado** un campo (Origen=Contenedor de datos, Presentacion=Lista)
- **Cuando** se abre el formulario
- **Entonces** la lista se llena desde el contenedor del tenant
- Severidad: Alto | Cubierta: **no**

### H1.4 - Aislamiento cross-tenant (adversarial, CRITICO)
- **Dado** el tenant A con clientes y el tenant B con otros clientes
- **Cuando** un usuario del tenant B usa el lookup de "Cliente"
- **Entonces** NUNCA ve ni puede elegir clientes del tenant A (ni escribiendo su nombre exacto)
- Severidad: **Critico** | Cubierta: **no (PENDIENTE)**

### H1.5 - No traer el catalogo completo (rendimiento/seguridad)
- **Dado** un catalogo con 50.000 terceros
- **Cuando** el usuario escribe en el lookup
- **Entonces** el servidor devuelve solo una pagina acotada (no los 50.000 al cliente)
- Severidad: Alto | Cubierta: **no**

### H1.6 - Crear al vuelo (dato faltante)
- **Dado** que el cliente buscado no existe
- **Cuando** el usuario pulsa "Crear"
- **Entonces** el sistema lo lleva al modulo de creacion (deep-link); al volver, re-busca y aparece
- Severidad: Medio | Cubierta: **no**

## F2 - Calculo y agregacion

### H2.1 - Subtotal por formula
- **Dado** un campo "Subtotal" = `{cantidad} * {precio}`
- **Cuando** cantidad=3 y precio=540000
- **Entonces** Subtotal muestra 1.620.000, en vivo y solo lectura
- Severidad: Alto | Cubierta: unit OK + manual

### H2.2 - Totales y roll-up en GridDetail
- **Dado** una tabla con lineas y una columna Subtotal con agregado Suma
- **Cuando** hay filas 2x1500 y 3x1000
- **Entonces** subtotales 3000 y 3000, fila de totales 6000, y el campo "Total general" del
  encabezado = 6000 (roll-up)
- Severidad: Alto | Cubierta: unit OK + manual

### H2.3 - El servidor corrige montos manipulados (adversarial)
- **Dado** un campo calculado
- **Cuando** el cliente envia a mano un total falso (ej. 1)
- **Entonces** el servidor **recalcula** y guarda el valor correcto, ignorando el del cliente
- Severidad: **Critico** (integridad) | Cubierta: **no explicito**

### H2.4 - Allow-list rechaza expresion peligrosa
- **Dado** el evaluador de formulas
- **Cuando** una formula intenta algo fuera de la allow-list (acceso a tipos, SQL, etc.)
- **Entonces** se rechaza y no ejecuta (sin RCE)
- Severidad: **Critico** | Cubierta: unit (confirmar cobertura de casos peligrosos)

## F3 - Transaccionalidad

### H3.1 - Consecutivo consume al confirmar
- **Dado** un formulario transaccional FRM-021 con identidad = Consecutivo
- **Cuando** se confirma el envio
- **Entonces** el registro recibe FRM-021-000001, estado Confirmed, fecha de transaccion, y la
  secuencia queda consumida
- Severidad: Alto | Cubierta: manual (falta test)

### H3.2 - Concurrencia: dos confirmaciones no repiten numero (adversarial)
- **Dado** dos usuarios confirmando el mismo formulario a la vez
- **Cuando** ambos guardan simultaneamente
- **Entonces** reciben numeros distintos (sin duplicado)
- Severidad: **Critico** | Cubierta: **no**

### H3.3 - Anular no libera el numero
- **Dado** un registro Confirmed con numero
- **Cuando** se anula (Void con motivo)
- **Entonces** queda Voided y su numero NO se reutiliza (queda el hueco)
- Severidad: Medio | Cubierta: **no**

### H3.4 - Clave natural unica
- **Dado** identidad = Clave natural (campo NIT)
- **Cuando** se intenta guardar dos registros con el mismo NIT
- **Entonces** el segundo se rechaza por unicidad (por tenant)
- Severidad: Alto | Cubierta: **no**

### H3.5 - Los dos estados no se confunden
- **Dado** un registro con `status` (envio) y `record_status` (transaccional)
- **Cuando** el envio esta en Draft pero el registro fue Confirmed (o viceversa)
- **Entonces** cada estado refleja su ciclo sin pisar al otro
- Severidad: Medio | Cubierta: **no**

## Transversales (aplican a todo el modulo)

### HT.1 - Paridad DAL dual
- **Dado** las suites de F1/F2/F3
- **Cuando** corren en Testcontainers
- **Entonces** pasan en **PostgreSQL y en SQL Server** (no solo PG local)
- Severidad: Alto | Cubierta: **no**

### HT.2 - Migracion reversible
- **Dado** una migracion de estas olas
- **Cuando** se aplica `up` y luego `down` sobre una copia del esquema real
- **Entonces** aplica y revierte limpio, con `__EFMigrationsHistory` consistente
- Severidad: Medio | Cubierta: **no**

> Ajusta o agrega historias aqui; el QA las tomara como contrato. Marca con severidad y, si
> quieres, con el tenant/datos exactos que deba usar.
