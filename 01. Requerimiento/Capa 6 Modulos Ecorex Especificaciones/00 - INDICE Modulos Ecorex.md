---
tipo: indice-carpeta
proposito: 5 specs de modulos Ecorex para reconstruccion en Claude Design
fecha: 2026-07-02
auditado: si
---

# Capa 6 - Modulos Ecorex - Especificaciones para reconstruccion

> Carpeta con 5 specs auto-contenidos, cada uno permite reconstruir un modulo
> ASP.NET WebForms de la solucion Bootstrap/GestionMovil sin acceso al codigo
> original.

## Los 5 modulos documentados

| # | Modulo | Codigo | Carpeta | Spec | Prototipo HTML |
|---|--------|--------|---------|------|----------------|
| 1 | NEWFRONT_tar_conceptos | 000270 | Tareas | [[NEWFRONT_tar_conceptos - Spec para reconstruir\|spec]] | [prototipo](../Prototipo/conceptos-claude-design/proto_tar_conceptos.html) |
| 2 | gen_reglas | 000802 | General | [[gen_reglas - Spec para reconstruir\|spec]] | [prototipo](../Prototipo/conceptos-claude-design/proto_gen_reglas.html) |
| 3 | NEWFRONT_adm_empresas | 000072 | Configuracion | [[NEWFRONT_adm_empresas - Spec para reconstruir\|spec]] | [prototipo](../Prototipo/conceptos-claude-design/proto_adm_empresas.html) |
| 4 | NEWFRONT_web_scraping | 000730 | Utilidades | [[NEWFRONT_web_scraping - Spec para reconstruir\|spec]] | [prototipo](../Prototipo/conceptos-claude-design/proto_web_scraping.html) |
| 5 | comer_ContactLoader | 000873 | Comercial | [[comer_ContactLoader - Spec para reconstruir\|spec]] | [prototipo](../Prototipo/conceptos-claude-design/proto_contact_loader.html) |

**Nota**: los prototipos HTML son visuales navegables con datos demo realistas
(Soldarco, Andina Vidrios, TeamTravels, BitCode IT, FacturaTech). Ver directorio
`01. Requerimiento/Prototipo/` o abrir directamente cada HTML en el navegador.

## Auditoria (2026-07-02)

Se corrieron **2 subagentes en paralelo** para validar los 5 specs:

### Subagente 1 - Fidelidad vs codigo real
- Verifico codigos de modulo, tablas SQL, IDs de controles en designer
- **Resultado**: 10/10 - ninguna falla critica
- Cero afirmaciones desmentidas: los 5 codigos de modulo coinciden con `topBar.Modulo`, las tablas mencionadas aparecen en los `.vb`, los IDs de controles existen en los `.designer.vb`
- Bugs documentados verificados: `Session("Emmpresa")` linea 412 (web_scraping), ausencia real de `CONTROL_REGLAS_D` (gen_reglas), verbos mDATA/Execute no implementados (gen_reglas), base `dbn8n` hardcoded (ContactLoader)

### Subagente 2 - Completitud estructural
- Reviso estructura (11+ secciones), ASCII puro, wireframes, tablas de controles, snippets SQL, riesgos, reconstruibilidad
- **Resultado**: 9/10 global

**Ranking por reconstruibilidad**:
1. comer_ContactLoader (10/10) - snippets JSON avanzados + mapeo exhaustivo
2. NEWFRONT_tar_conceptos (9/10) - snippets ejecutables + RQ07 rastreable
3. NEWFRONT_adm_empresas (9/10) - 40+ controles, 15 tablas
4. NEWFRONT_web_scraping (9/10) - excelente contexto runtime/configurador
5. gen_reglas (9/10) - buena arquitectura, pocos SQL literales

### Recomendaciones menores (no bloqueantes)
1. Estandarizar snippets SQL literales en docs 2 y 4
2. Wireframe explicito de acordeones en doc 3
3. Renombrar handlers genericos (`LinkButton1_Click` -> nombre semantico) transversalmente
4. Documentar contratos XML (PARAMXML 0202, PARAM_XML CorXml, XML_CONFIG) con schema formal
5. Consolidar seccion "Cross-base access" (COLLATE cross-catalog) en todos los docs

## Convenciones aplicadas

- **Solo ASCII**: sin acentos, sin enye, sin emojis (regla del proyecto)
- **Frontmatter YAML**: tipo, modulo, carpeta, proposito
- **Estructura**: 11-14 secciones numeradas
- **Auto-contenido**: cada doc se lee sin dependencia de los otros
- **Enfoque reconstructivo**: incluye "Puntos de reconstruccion en Claude Design"

## Enlaces relacionados

- [[00 - INDICE|INDICE maestro OBSIDIAN.tareas]]
- [[gen_reglas - Spec para reconstruir en Claude Design|Spec previa de gen_reglas]] (dentro de contexto Flujos; la de esta carpeta es auto-contenida)
- [[ctrTareasII - Spec para reconstruir en Claude Design|Spec ctrTareasII]]
- [[ctrVertareasII - Spec para reconstruir en Claude Design|Spec ctrVertareasII]]

## Capitulo nuevo (2026-07-11) - Agente Conector On-Prem (Contenedor de datos)

Proyecto NUEVO (no reconstruccion): un agente de escritorio VB.NET + WPF que se instala en la
red del cliente y alimenta el modulo "Contenedor de datos" por WebSocket (SignalR), disparado
por los horarios del sistema web o por refresco inmediato. Documentacion auto-contenida para
que un sub-agente lo construya. Ver subcarpeta `Agente Conector On-Prem/`:

- [[00 - INDICE - Agente Conector On-Prem|Indice del proyecto]]
- [[01 - Vision, arquitectura y decisiones]]
- [[02 - Protocolo SignalR (mensajes, handshake, secuencias)]]
- [[03 - Servidor (Hub + Scheduler + Ingesta)]]
- [[04 - Cliente VB.NET WPF (Servicio Windows + config)]]
- [[05 - Plan de trabajo por olas (para sub-agente)]]
