# 02 — Método de aprendizaje

## 1. Contrato de trabajo

| Rol | Responsabilidad |
|---|---|
| **Estudiante** | Escribe **todo** el código de producción a mano. Ejecuta, prueba, depura, commitea. Hace preguntas. |
| **Claude (profesor)** | Explica, diseña, muestra ejemplos, revisa, corrige, pregunta, mantiene la documentación. **No escribe el código del proyecto.** |

Las reglas técnicas exactas de esta separación están en [`../../CLAUDE.md`](../../CLAUDE.md).

### Por qué el código se transcribe y no se genera

Escribir a mano fuerza tres procesos que copiar-pegar elimina:

1. **Codificación motora**: el gesto de teclear `public Optional<Cliente> findById(UUID id)` fija
   la sintaxis mucho más que leerla.
2. **Detección de errores**: al escribir, aparecen errores de compilación tuyos, y corregirlos es
   donde realmente se aprende el sistema de tipos.
3. **Atención al detalle**: te obliga a notar el `;`, el genérico, el `final`, la anotación.

Por eso los ejemplos de la documentación están pensados para ser **leídos, entendidos y reescritos**,
no copiados. Adaptarlos (cambiar nombres, agregar un campo) es aún mejor que transcribirlos literal.

## 2. Ciclo de una sesión

```
┌─ 1. ENCUADRE ───────────────────────────────────────────┐
│  ¿En qué fase estamos? ¿Qué quedó de la sesión anterior?│
│  ¿Qué vamos a lograr hoy? (objetivo verificable)        │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─ 2. CONCEPTO ───────────────────────────────────────────┐
│  Teoría mínima necesaria. Se responde el "¿por qué?"    │
│  antes del "¿cómo?".                                    │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─ 3. SINTAXIS ───────────────────────────────────────────┐
│  El patrón exacto, mostrado como ejemplo funcional,     │
│  con el atajo de IntelliJ que lo acelera.               │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─ 4. CONSTRUCCIÓN ───────────────────────────────────────┐
│  El estudiante escribe. Compila. Falla. Corrige.        │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─ 5. VERIFICACIÓN ───────────────────────────────────────┐
│  mvn test / levantar la app / curl / consultar la BD.   │
│  Los criterios de aceptación de la fase deben pasar.    │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─ 6. DEPURACIÓN ─────────────────────────────────────────┐
│  Si falla: hipótesis → breakpoint → evidencia → arreglo.│
│  Claude pregunta primero, responde después.             │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─ 7. CIERRE ─────────────────────────────────────────────┐
│  Commit · Bitácora de la fase · Roadmap actualizado     │
│  · Preguntas de control respondidas                     │
└─────────────────────────────────────────────────────────┘
```

## 3. Las tres preguntas obligatorias

Antes de escribir cualquier clase, el estudiante debe poder responder:

1. **¿Qué responsabilidad tiene esta clase?** (una sola frase, sin la palabra "y")
2. **¿Quién la usa y quién la conoce?** (dirección de las dependencias)
3. **¿Cómo la voy a probar?** (si no sabes probarla, probablemente esté mal diseñada)

## 4. Protocolo de depuración

Prohibido el "cambiar cosas hasta que funcione". El protocolo es:

```
1. LEER EL ERROR COMPLETO
   - ¿Qué excepción? ¿Qué mensaje? ¿Cuál es la línea de MI código más profunda del stacktrace?
   - "Caused by" al final del stacktrace suele ser la causa raíz.

2. FORMULAR UNA HIPÓTESIS EXPLÍCITA
   - "Creo que el repositorio devuelve null porque la transacción ya se cerró."
   - Escribirla. Sin hipótesis no se toca el código.

3. DISEÑAR LA OBSERVACIÓN
   - Breakpoint (condicional si aplica), Evaluate Expression, log, consulta SQL.

4. OBSERVAR Y CONFIRMAR/REFUTAR
   - Si se refuta, volver al paso 2. NO cambiar código todavía.

5. CORREGIR LA CAUSA, NO EL SÍNTOMA
   - Un try/catch que esconde el error no es una corrección.

6. ESCRIBIR UNA PRUEBA QUE FALLE SIN LA CORRECCIÓN
   - Es la única garantía de que el bug no vuelve.

7. ANOTAR EN LA BITÁCORA
   - Qué falló, por qué, cómo se detectó. Es el activo de aprendizaje más valioso.
```

## 5. Preguntas de control

Cada fase termina con preguntas de comprobación. Reglas:

- Se responden **sin mirar la documentación**.
- Si no se puede responder al menos el 80%, no se avanza de fase.
- Las respuestas se anotan en la bitácora de la fase.

## 6. Bitácora

Cada documento de fase tiene una sección final `## Bitácora` donde el estudiante registra:

```markdown
### Sesión <n> — <fecha>
- **Hecho:** ...
- **Concepto que costó:** ...
- **Error interesante:** <mensaje> → <causa real> → <cómo lo encontré>
- **Atajo nuevo aprendido:** ...
- **Duda abierta:** ...
```

Esta bitácora es material de repaso y es, en la práctica, la prueba del aprendizaje.

## 7. Regla del "modo examen"

Cada 3 fases hay un **ejercicio en frío**: el estudiante escribe una funcionalidad pequeña
**sin documentación abierta y sin ayuda**, con tiempo límite. Sirve para medir la fluidez real,
no la capacidad de seguir instrucciones.

## 8. Gestión del ritmo

- Sesión recomendada: 60–120 minutos.
- Si una fase se atasca más de 3 sesiones, se divide o se simplifica el alcance.
- Prohibido saltar fases: el orden es dependencia técnica, no capricho.
- Permitido volver atrás: releer una fase anterior nunca es retroceso.
