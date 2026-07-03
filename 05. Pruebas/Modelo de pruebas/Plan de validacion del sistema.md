---
tipo: plan-pruebas
proposito: Checklist de validacion manual del sistema Tareas (origen) por modulo
---

# Plan de validacion del sistema Tareas [ORIGEN]

> **Nota de ORIGEN.** Este es el checklist de validacion MANUAL del sistema legacy
> (WebForms sobre `db3dev`), util antes de migrar y como base de los casos del
> destino. El sistema origen NO tiene tests automatizados (no hay framework de
> testing; `Enc_3dev_tester` es una herramienta manual de encriptacion).
>
> Las pruebas AUTOMATIZADAS del sistema DESTINO (.NET 10) — piramide, matriz
> Testcontainers dual, aislamiento cross-tenant, ETL, E2E Playwright — estan en
> [[Estrategia de Testing (.NET 10)]].

## Entorno de pruebas

- **BD**: `db3dev` (tenant de desarrollo)
- **App local**: `http://localhost:47640/GestionMovil/`
- **Produccion** (solo lectura para validar): `https://app.bitcode.com.co/`
- Credenciales: gestionadas fuera del vault (no se documentan aqui)

## Checklist por modulo

### Tareas basicas (Capa 2)
- [ ] Crear tarea desde ctrTareasII (wizard 5 pasos completo)
- [ ] Ver detalle en ctrVertareasII: timer worklog inicia/detiene
- [ ] Cambiar estado y verificar seguimiento en `TAR_SEGUIMIENTO`
- [ ] Kanban: arrastrar tarjeta entre columnas

### Flujos BPMN (Capa 3)
- [ ] Crear flujo en NEWFRONT_doc_procesos y guardar XML
- [ ] Disparar tarea con flujo asociado: nodo inicial correcto
- [ ] `SiguienteEstado` avanza sin superar 50 iteraciones
- [ ] Reglas por nodo se ejecutan (verbo Ensamblado)
- [ ] XML exportado abre en demo.bpmn.io sin errores

### Formularios (Capa 4)
- [ ] Crear formulario con al menos 5 tipos de control distintos
- [ ] Responder via visor por token (docu_viewform)
- [ ] Respuestas quedan en `FORX_DATA` (patron EAV)
- [ ] Union formulario-flujo via `FORX_DATA_FLUJO`

### Reglas (gen_reglas 000802)
- [ ] Crear regla modo Ensamblado con PARAM_XML valido
- [ ] Ejecutar y verificar historial en `CONTROL_REGLAS_H`
- [ ] Confirmar que mDATA/Execute NO ejecutan (limitacion conocida)

### Gestion de empresas (Capa 1 - 000072)
- [ ] Crear empresa de prueba y habilitar 2 modulos
- [ ] Copiar formularios entre empresas
- [ ] Verificar aislamiento: usuario de empresa A no ve datos de B

## Escenarios E2E

Usar las narrativas de [[05 - Un dia en SKY SYSTEM (narrativa E2E)]] como
guion de regresion manual: cubre 17 modulos en un flujo realista.

## Riesgos conocidos a re-verificar tras cualquier cambio

- Inyeccion SQL por concatenacion (transversal — ver specs en Capa 6)
- Typo `Session("Emmpresa")` en NEWFRONT_web_scraping linea 412
- Handler de UPDATE que nunca ejecuta en NEWFRONT_tar_conceptos

## Enlaces

- [[00 - INDICE|INDICE maestro]]
- [[00 - Indice y personas del sistema|Historias de usuario]]
