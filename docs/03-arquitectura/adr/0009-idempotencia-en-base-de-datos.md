# ADR-0009 — La idempotencia se garantiza en la base de datos

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F10

## Contexto
Los webhooks de las pasarelas de pago se entregan **al menos una vez**: llegan duplicados de forma
rutinaria. Aplicar dos veces un pago es un error con impacto económico directo (BR-051).

## Opciones consideradas
1. **Comprobación previa en Java** (`if (repository.existePorReferencia(...))`) — legible, pero tiene
   una condición de carrera entre la consulta y la inserción. Con dos entregas simultáneas del mismo
   webhook, ambas pueden pasar la comprobación.
2. **Bloqueo distribuido (Redis, advisory locks)** — funciona, pero añade una dependencia y un punto
   de fallo para un problema que la base de datos ya resuelve.
3. **Restricción `UNIQUE` sobre la referencia externa + manejo de la violación** — la base de datos
   garantiza la unicidad de forma atómica, sin importar la concurrencia.

## Decisión
**Restricción `UNIQUE (psp, transaccion_id)` en `pagos_recibidos`**, capturando la violación de
integridad como señal de "ya procesado" y respondiendo `200` sin reprocesar.

El mismo principio se aplica en todo el sistema:
- `creditos.solicitud_id UNIQUE` → una solicitud produce a lo sumo un crédito (BR-035).
- `documentos_fiscales (prefijo, consecutivo) UNIQUE` → sin consecutivos repetidos (BR-071).
- Índice único parcial sobre solicitudes en estudio (BR-020).

## Consecuencias
- La regla se cumple aunque el código Java tenga un error de lógica: es la última línea de defensa.
- Hay que traducir `DataIntegrityViolationException` a semántica de negocio, distinguiendo **qué**
  restricción se violó (por el nombre de la restricción, que por eso se nombra explícitamente).
- La comprobación previa en Java se mantiene igualmente, por claridad y para dar un mensaje mejor en
  el caso normal — pero **no** es lo que garantiza la regla.
- Requiere pruebas de concurrencia reales (dos hilos, misma referencia) — que se escriben en F10.

> Lección general del proyecto: **una invariante de negocio crítica se protege en el dominio y se
> garantiza en la base de datos.** Los dos, no uno.
