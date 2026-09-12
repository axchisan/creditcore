# 03 — Requerimientos no funcionales

Cada RNF incluye **cómo se verifica**. Un requerimiento no funcional que no se puede medir es un
deseo, no un requerimiento.

---

## A. Rendimiento

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-001 | Las consultas de un crédito individual responden en < 200 ms (p95) con 50.000 créditos en base. | Prueba de carga con datos sintéticos + métrica de Micrometer | F12 |
| RNF-002 | El listado paginado de clientes responde en < 300 ms (p95) con 100.000 registros. | Prueba de carga + `EXPLAIN ANALYZE` | F12 |
| RNF-003 | No debe existir ningún problema N+1 en los endpoints de consulta. | Contador de consultas Hibernate en pruebas de integración | F12 |
| RNF-004 | El cierre de día procesa 50.000 créditos en menos de 10 minutos. | Medición con dataset sintético | F12 |
| RNF-005 | Toda consulta que devuelva colecciones debe estar paginada; no existen endpoints que devuelvan listas completas. | Revisión de código + prueba | F04 |

## B. Escalabilidad

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-010 | La aplicación es *stateless*: cualquier instancia puede atender cualquier petición. | Despliegue con 2+ réplicas y pruebas | F15 |
| RNF-011 | Los procesos batch soportan ejecución en una sola instancia sin duplicarse (bloqueo distribuido o *shedlock*). | Prueba con dos instancias simultáneas | F13 |
| RNF-012 | El pool de conexiones está dimensionado y monitoreado (HikariCP). | Métricas de Actuator | F12 |

## C. Disponibilidad y resiliencia

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-020 | La caída de la pasarela de pago no impide operar el resto del sistema. | Prueba con adaptador que falla + circuit breaker | F10 |
| RNF-021 | Toda llamada saliente tiene timeout explícito (conexión y lectura). | Revisión de configuración + prueba | F10 |
| RNF-022 | Las llamadas salientes idempotentes reintentan con backoff exponencial. | Prueba con Resilience4j | F10 |
| RNF-023 | Los eventos no procesados se reintentan y, tras N fallos, van a una cola de mensajes fallidos (DLQ). | Prueba con LocalStack SQS | F13 |
| RNF-024 | El arranque de la aplicación falla rápido si falta configuración obligatoria. | Prueba de contexto | F03 |

## D. Seguridad

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-030 | Toda la API, salvo salud y login, requiere autenticación. | Pruebas de seguridad con caso negativo por endpoint | F09 |
| RNF-031 | Los tokens de acceso expiran en ≤ 15 minutos. | Configuración + prueba | F09 |
| RNF-032 | Las contraseñas se almacenan con BCrypt (o Argon2), nunca en claro ni con hash simple. | Revisión + prueba | F09 |
| RNF-033 | No existe ningún secreto en el repositorio. | Escaneo en CI (gitleaks) | F15 |
| RNF-034 | Los datos sensibles se enmascaran en los logs. | Prueba que inspecciona la salida de log | F12 |
| RNF-035 | La API valida y sanea toda entrada; no se concatena SQL. | Revisión + análisis estático | F04 |
| RNF-036 | Las dependencias se analizan por vulnerabilidades conocidas en cada build. | OWASP Dependency-Check en CI | F15 |
| RNF-037 | Los mensajes de error no revelan detalles internos (stacktraces, SQL, versiones). | Prueba de integración sobre respuestas de error | F04 |

## E. Mantenibilidad y calidad

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-040 | Cobertura de pruebas ≥ 80% en el paquete de dominio y ≥ 60% global. | JaCoCo con umbral que rompe el build | F05 |
| RNF-041 | El dominio no depende de Spring, JPA ni de ninguna librería de infraestructura. | Prueba de arquitectura con ArchUnit | F03 |
| RNF-042 | Las dependencias entre paquetes respetan la regla de la arquitectura hexagonal. | ArchUnit | F03 |
| RNF-043 | El código cumple el estilo definido; el build falla si no. | Checkstyle/Spotless en CI | F15 |
| RNF-044 | Ningún método supera 30 líneas ni complejidad ciclomática 10, salvo excepción justificada. | Análisis estático | F15 |
| RNF-045 | Toda regla de negocio `BR-xxx` tiene al menos una prueba que la referencia por identificador. | Matriz de trazabilidad + revisión | continuo |
| RNF-046 | Toda decisión arquitectónica relevante está registrada como ADR. | Revisión | continuo |

## F. Datos y consistencia

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-050 | El esquema evoluciona únicamente por migraciones versionadas; `ddl-auto` está en `validate`. | Configuración + arranque | F03 |
| RNF-051 | Las invariantes monetarias están protegidas también por restricciones en base de datos (`CHECK`, `UNIQUE`, `NOT NULL`). | Revisión de migraciones + prueba | F03 |
| RNF-052 | Toda tabla tiene clave primaria y las relaciones tienen clave foránea con acción definida. | Revisión | F03 |
| RNF-053 | Los importes se almacenan como `NUMERIC(19,2)` y las tasas como `NUMERIC(12,8)`. | Revisión de migraciones | F03 |
| RNF-054 | Las fechas con hora se almacenan en UTC (`timestamptz`); la presentación usa `America/Bogota`. | Prueba | F03 |
| RNF-055 | Ninguna operación de negocio borra datos históricos. | Revisión + ausencia de `DELETE` en repositorios de movimientos | F08 |

## G. Observabilidad

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-060 | Cada petición tiene un identificador de correlación propagado a logs y a llamadas salientes. | Prueba de integración | F12 |
| RNF-061 | Se exponen métricas de negocio: solicitudes por estado, desembolsos del día, pagos aplicados, documentos rechazados. | Endpoint de métricas | F12 |
| RNF-062 | Los logs son JSON en entornos distintos de desarrollo. | Configuración por perfil | F12 |
| RNF-063 | Existe un endpoint de readiness que verifica base de datos y dependencias críticas. | Actuator + prueba | F12 |

## H. Cumplimiento

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-070 | Los documentos fiscales se conservan por el período legal exigido. | Política de retención en S3 | F16 |
| RNF-071 | La auditoría es inmutable: no existe operación de actualización ni borrado sobre ella. | Revisión + permisos de base de datos | F09 |
| RNF-072 | El tratamiento de datos personales cumple la Ley 1581 de 2012 (habeas data): consentimiento, finalidad, supresión. | Documentación + endpoint de gestión de datos | F09 |
| RNF-073 | No se almacena información completa de tarjetas (PCI DSS). | Revisión + prueba | F10 |

## I. Portabilidad y entorno

| ID | Requerimiento | Verificación | Fase |
|---|---|---|---|
| RNF-080 | La aplicación se ejecuta idénticamente en local, LocalStack y AWS cambiando solo configuración. | Despliegue en los tres entornos | F16 |
| RNF-081 | Toda la configuración sensible proviene de variables de entorno o gestor de secretos. | Revisión | F15 |
| RNF-082 | La imagen de contenedor no ejecuta como root y pesa menos de 400 MB. | Inspección de imagen | F15 |
| RNF-083 | La infraestructura se define como código y es reproducible. | `terraform plan` limpio tras `apply` | F16 |

---

## Cómo se usan estos RNF

- No se marcan como "cumplidos" por opinión: cada uno tiene una **verificación ejecutable** o una
  revisión explícita registrada.
- En cada fase se revisan los RNF asignados a esa fase, y no se cierra la fase sin ellos.
- Un RNF que resulte irreal se **renegocia y se documenta**, no se ignora en silencio.
