# forkmap

> **Repo:** [github.com/3zequiel3/forkmap](https://github.com/3zequiel3/forkmap)

Skill que deriva **`CHANGES.md`** — el índice operativo de changes de un proyecto — a partir de una base de conocimiento estructurada. Es la mitad consumidora de un par: **[`chronicle`](https://github.com/3zequiel3/chronicle) documenta el sistema y [`forkmap`](https://github.com/3zequiel3/forkmap) convierte esa documentación en un plan ejecutable.**

---

## Qué hace

Lee la base de conocimiento en `knowledge-base/` (el output de `chronicle`) **a través de su índice**, infiere el perfil del sistema y escribe `CHANGES.md` en la raíz del proyecto con:

- **Árbol de dependencias** (ASCII art) — **validado acíclico** antes de escribir (DAG + integridad de IDs).
- **Gates de paralelismo** con asignación sugerida de agentes.
- **Camino crítico** — el mínimo irreducible (MVP) para llegar a producción.
- **Plan de ejecución con N agentes** (tabla para proyectos chicos, cronograma por olas a escala).
- **Por cada change**: checkbox de estado, tamaño (S/M/L), scope operacional, dependencias, nivel de governance y los archivos de la KB a leer antes de implementar.

Consciente de **greenfield vs brownfield** (no genera scaffold para un sistema que ya existe), de **monorepos** (varios subsistemas/perfiles) y de **re-runs idempotentes** (no pisa tus `[x]` ni tus ediciones).

Es *fire-and-forget*: sin preguntas, directo al output. El idioma del output coincide con el de la KB.

---

## Por qué no es "solo un roadmap"

Un roadmap clásico es informativo. `CHANGES.md` es **operativo**: te dice cómo paralelizar el trabajo entre agentes, cuál es el mínimo irreducible si te falta tiempo, qué docs de la KB tiene que leer un agente antes de proponer cada change y cuánta revisión humana requiere cada uno.

| Aspecto | Roadmap simple | `CHANGES.md` |
|---------|----------------|--------------|
| Dependencias | Tabla 1 a 1 | Árbol jerárquico + gates |
| Paralelización | Implícita / nula | Explícita, asignada a agentes |
| Prioridad ante poco tiempo | Difícil de inferir | Camino crítico marcado (basado en MVP) |
| Contrato con la KB | Implícito | "Leer antes" por change, consciente de carpetas |
| Nivel de riesgo | No declarado | Governance: BAJO/MEDIO/ALTO/CRITICO |
| Scope por change | Descriptivo | Operacional (modelos, endpoints, migraciones) + códigos de la KB |
| Tracking de progreso | No | Checkboxes `[ ]` / `[x]` |
| Escala | Se rompe pasados ~20 | Promueve el detalle a `changes/` |

---

## Cómo se empareja con `chronicle`

[`forkmap`](https://github.com/3zequiel3/forkmap) lee lo que [`chronicle`](https://github.com/3zequiel3/chronicle) escribe — y nada más que eso. El contrato:

- **Descubrimiento por índice** — lee `knowledge-base/README.md` para saber qué nodos están activos y si cada uno es un **archivo o una carpeta**. Nunca asume un set fijo de archivos. Los nodos omitidos por perfil (por ejemplo, sin RBAC en una CLI) se respetan, no se marcan como faltantes.
- **Consciente del perfil** — el `system_type` de la KB (web_app, api, cli, mobile, saas_multi_tenant, library_sdk, data_pipeline) redefine la taxonomía de changes. El roadmap de una librería gira en torno a la superficie de API pública y el semver; un pipeline sigue el DAG.
- **Guiado por MVP** — los tags `[MVP]` / `[Post-MVP]` de `chronicle` manejan el camino crítico. Es exactamente para lo que esos tags fueron diseñados.
- **Trazable** — el Scope y el "Leer antes" citan los códigos de `chronicle` (`US-NNN`, `RN-{DOMINIO}-NN`, `DD-NN`) y paths reales, conscientes de carpetas.

Contrato completo: [`assets/kb-input-contract.md`](assets/kb-input-contract.md).

---

## Pre-requisitos

1. Una base de conocimiento en `knowledge-base/` (raíz) con un índice (`README.md`) y los nodos core. Generala con [`chronicle`](https://github.com/3zequiel3/chronicle).
2. OpenSpec inicializado:
   ```bash
   npx @fission-ai/openspec@latest init
   ```

Si falta cualquiera de los dos, la skill te avisa y se detiene **sin escribir nada**.

---

## Instalación

```bash
npx skills add https://github.com/3zequiel3/forkmap
```

Se empareja con [`chronicle`](https://github.com/3zequiel3/chronicle) — instalá ambas para el flujo completo documentar → planificar.

---

## Uso

```
tu-repo/
├── knowledge-base/      # KB generada por chronicle (con índice README.md)
└── openspec/            # OpenSpec inicializado

Le decís al agente:
"generá el CHANGES.md" / "armá el roadmap de la KB" / "forkmap"
```

→ El agente lee la KB a través de su índice y escribe `CHANGES.md` en la raíz (y `changes/` si el proyecto es grande).

---

## Escalabilidad

| Tamaño del proyecto | Output |
|---------------------|--------|
| ≤ 20 changes | Un solo `CHANGES.md`, detalle completo inline |
| > 20 changes | `CHANGES.md` como índice maestro + detalle por fase en `changes/FASE-NN-*.md` |

Misma filosofía que la promoción archivo↔carpeta de `chronicle`: no inflar la estructura en proyectos chicos, no ahogarse en los grandes.

---

## Output al cerrar

```
✅ CHANGES.md creado en la raíz con 10 changes en 4 fases (perfil: web_app).
Camino crítico: 7 changes (MVP)
Gates de paralelismo: 5
Primer change recomendado: C-01 (foundation-setup)
Para arrancar: /opsx:propose C-01-foundation-setup
```

---

## Archivos

- `SKILL.md` — contrato de runtime.
- `assets/kb-input-contract.md` — cómo forkmap lee una KB de chronicle (índice, perfil, modo, cobertura MVP).
- `assets/profiles-and-dependencies.md` — taxonomía por perfil y modo, monorepo, dependencias, governance, sizing, derivación por fases, escalabilidad.
- `assets/changes-template.md` — estructura exacta del output + checklist de validación de grafo.
- `assets/lifecycle.md` — re-runs idempotentes, drift KB↔CHANGES, sync con `openspec/changes/archive/` (carga lazy).

---

## Licencia

Apache-2.0
