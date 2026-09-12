# 05 — Trazabilidad, Definition of Ready y Definition of Done

## 1. Cadena de trazabilidad

```
Necesidad de negocio
        │
        ▼
  Requerimiento (RF/RNF) ────► Regla de negocio (BR)
        │                              │
        ▼                              ▼
  Historia de usuario (HU)      Implementada en una clase
        │                              │
        ▼                              ▼
  Escenario Gherkin ──────────► Método de prueba con el ID en el nombre
        │
        ▼
     Commit ──► Fase ──► Bitácora
```

**Regla operativa:** ningún código entra al repositorio sin poder responder:
*"¿qué RF o BR está implementando esto y qué prueba lo verifica?"*

## 2. Matriz de trazabilidad (resumen por fase)

| Fase | Requerimientos principales | Reglas principales | Entregable verificable |
|---|---|---|---|
| F00 | — | — | Entorno funcionando; `mvn -v`, `psql`, `docker` responden |
| F01 | — | — | Ejercicios de Java resueltos y probados |
| F02 | RNF-024, RNF-041, RNF-042 | — | App arranca; ArchUnit verde |
| F03 | RNF-050..RNF-054 | BR-043 | Migraciones aplicadas; esquema validado |
| F04 | RF-001..RF-006, RF-113, RNF-005, RNF-035, RNF-037 | BR-001..BR-005 | CRUD de clientes con pruebas |
| F05 | RNF-040 | — | Pirámide de pruebas completa; JaCoCo con umbral |
| F06 | RF-010..RF-014, RF-044, RF-047 | BR-010..BR-015, BR-040..BR-045 | Motor de amortización por TDD |
| F07 | RF-020..RF-031, RF-040..RF-045 | BR-020..BR-035 | Originación y desembolso end-to-end |
| F08 | RF-046, RF-050..RF-052, RF-058..RF-062, RF-070..RF-076 | BR-050..BR-065 | Pagos y cierre de día |
| F09 | RF-027, RF-042, RF-100..RF-104 | BR-025, BR-033, BR-080..BR-084 | Seguridad y auditoría |
| F10 | RF-053..RF-057, RF-105 | BR-051, BR-082 | Integración de pagos con idempotencia |
| F11 | RF-080..RF-090 | BR-070..BR-076 | Facturación electrónica |
| F12 | RF-074, RF-110, RF-111 | — | Observabilidad y rendimiento |
| F13 | RF-114, RF-115 | — | Asincronía, outbox, SQS |
| F14 | RF-112 | — | OpenAPI publicada |
| F15 | RNF-033, RNF-036, RNF-043, RNF-082 | — | CI/CD verde con calidad |
| F16 | RNF-080, RNF-083 | — | Despliegue en AWS |
| F17 | RNF-001..RNF-004 | — | Pruebas de carga y hardening |
| F18 | — | — | Migración a Spring Boot 4.x |

*(La matriz detallada RF ↔ HU ↔ prueba se mantiene actualizada en cada fase.)*

---

## 3. Definition of Ready (DoR)

Una historia está lista para construirse cuando:

- [ ] Tiene rol, capacidad y beneficio claros.
- [ ] Tiene al menos tres escenarios Gherkin: feliz, borde y error.
- [ ] Las reglas de negocio aplicables están identificadas por ID.
- [ ] Las dependencias técnicas existen (no depende de algo aún no construido).
- [ ] El contrato de API está definido (verbo, ruta, request, response, códigos de error).
- [ ] Se sabe cómo se va a probar.
- [ ] Cabe en una o dos sesiones; si no, se divide.

---

## 4. Definition of Done (DoD)

Una historia está terminada cuando **todo** lo siguiente es cierto:

### Funcionalidad
- [ ] Todos los escenarios Gherkin pasan como pruebas automatizadas.
- [ ] Los casos de error devuelven el código HTTP y el Problem Details correctos.
- [ ] Las reglas de negocio referenciadas están implementadas y probadas por su ID.

### Código
- [ ] Escrito a mano por el estudiante.
- [ ] Compila sin advertencias nuevas.
- [ ] Cumple las convenciones de `docs/04-calidad/03-convenciones-de-codigo.md`.
- [ ] No hay código muerto, `TODO` sin ticket, ni `System.out.println`.
- [ ] El dominio no importa nada de infraestructura (verificado por ArchUnit).

### Pruebas
- [ ] Pruebas unitarias del dominio.
- [ ] Prueba de integración del caso de uso completo.
- [ ] Al menos una prueba de caso negativo por regla de negocio.
- [ ] `mvn verify` pasa en limpio.
- [ ] La cobertura no bajó respecto al commit anterior.

### Datos
- [ ] Las migraciones Flyway están escritas y son reversibles o compensables.
- [ ] Las restricciones de integridad están en la base de datos, no solo en Java.

### Documentación
- [ ] El documento de la fase está actualizado.
- [ ] Si hubo una decisión relevante, hay un ADR.
- [ ] La bitácora de la sesión está escrita.

### Comprensión (específico de este proyecto)
- [ ] El estudiante puede explicar, sin mirar el código, qué hace cada clase nueva y por qué existe.
- [ ] El estudiante puede responder las preguntas de control de la fase.
- [ ] El estudiante sabe dónde pondría un breakpoint para depurar ese flujo.

### Entrega
- [ ] Commit con mensaje conforme a Conventional Commits.
- [ ] Roadmap actualizado.

---

## 5. Criterios de aceptación del proyecto completo

El proyecto se acepta como terminado cuando:

1. Un cliente puede ser registrado, evaluado, aprobado, desembolsado, pagar en línea, entrar en mora
   y quedar a paz y salvo — todo end-to-end, con pruebas automatizadas que lo demuestren.
2. La facturación electrónica emite, firma, envía, recibe respuesta y emite notas crédito.
3. `mvn verify` pasa en limpio, con cobertura sobre el umbral y sin violaciones de arquitectura.
4. La aplicación se despliega en AWS mediante Terraform y responde correctamente.
5. El pipeline de CI/CD ejecuta build, pruebas, calidad y despliegue.
6. Existe documentación completa y actualizada de las 18 fases, con bitácoras.
7. **El estudiante puede reconstruir desde cero, sin ayuda, una rodaja vertical completa**
   (entidad → migración → repositorio → caso de uso → controlador → pruebas) en menos de dos horas.

El punto 7 es el verdadero criterio de éxito. Los otros seis son el vehículo.
