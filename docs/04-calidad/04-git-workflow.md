# 04 — Flujo de trabajo con Git

## 1. Estrategia de ramas

Este es un proyecto de una persona, así que no necesitamos GitFlow completo. Usamos
**trunk-based development con ramas cortas por fase**, que es lo más cercano a lo que se usa hoy en
la industria.

```
main ──●────●────●────●────●────●──────────►  (siempre desplegable)
        \      /      \      /
         ●────●        ●────●
      fase/03-persistencia  fase/04-clientes
```

| Rama | Uso |
|---|---|
| `main` | Siempre compila y pasa `mvn verify`. Cada fase completada se integra aquí |
| `fase/XX-nombre` | Una rama por fase del roadmap |
| `fix/descripcion` | Corrección puntual |
| `spike/descripcion` | Experimento; puede borrarse sin integrar |

**Regla:** nunca se trabaja directamente sobre `main`. Aunque estés solo, practicar el flujo de
ramas es parte del objetivo laboral.

## 2. Conventional Commits

```
<tipo>(<ámbito opcional>): <descripción en imperativo, minúscula, sin punto final>

<cuerpo opcional: el POR QUÉ, no el qué>

<pie opcional: referencias, BREAKING CHANGE>
```

### Tipos

| Tipo | Cuándo |
|---|---|
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de un error |
| `docs` | Solo documentación |
| `test` | Agregar o corregir pruebas |
| `refactor` | Cambio interno sin alterar comportamiento |
| `perf` | Mejora de rendimiento |
| `build` | Maven, dependencias, Dockerfile |
| `ci` | Pipelines |
| `chore` | Tareas de mantenimiento |
| `style` | Formato sin cambio funcional |

### Ámbitos de este proyecto

`clientes`, `productos`, `originacion`, `creditos`, `pagos`, `cartera`, `facturacion`,
`seguridad`, `plataforma`, `infra`, `docs`

### Ejemplos reales

```
feat(creditos): generar plan de pagos al desembolsar

Implementa RF-041 y las invariantes BR-040 y BR-041. El plan se genera en la
misma transacción del desembolso para garantizar consistencia.

Refs: RF-041, BR-040, BR-041
```

```
fix(pagos): evitar doble aplicación de webhooks duplicados

La comprobación previa en Java tenía una condición de carrera entre el SELECT
y el INSERT. Se apoya ahora en la restricción UNIQUE(psp, transaccion_id) y se
traduce la violación de integridad a "ya procesado".

Refs: BR-051, ADR-0009
```

```
test(creditos): cubrir imputación de pagos con mora

refactor(originacion): extraer MotorEvaluacion del servicio de solicitud

docs(adr): registrar decisión de no usar Lombok
```

### Reglas de los mensajes

1. Primera línea ≤ 72 caracteres.
2. En imperativo: "agrega", no "agregado" ni "agregando".
3. El **cuerpo explica el porqué**; el diff ya muestra el qué.
4. Referencia los identificadores (`RF-`, `BR-`, `ADR-`) cuando aplique.
5. Un commit = un cambio coherente. Si necesitas "y" para describirlo, son dos commits.

## 3. Ritmo de commits en este proyecto

**Un commit por unidad de trabajo comprensible**, no uno por fase completa.

Ejemplo de secuencia real de la Fase 04:

```
feat(clientes): agregar value objects NumeroDocumento y DatosContacto
test(clientes): cubrir invariantes de Cliente
feat(clientes): agregar agregado Cliente con sus invariantes
feat(clientes): agregar migración flyway de la tabla clientes
feat(clientes): agregar entidad JPA y adaptador de repositorio
test(clientes): verificar restricción de documento duplicado (BR-001)
feat(clientes): agregar caso de uso RegistrarCliente
feat(clientes): exponer POST /api/v1/clientes
feat(plataforma): manejar errores con Problem Details RFC 7807
test(clientes): pruebas de slice del controlador
docs(fases): cerrar fase 04 y actualizar bitácora
```

**Por qué importa:** un historial así se puede leer como la historia de construcción del sistema.
Es lo que un revisor —o tú dentro de seis meses— necesita.

## 4. Comandos habituales

```bash
# Iniciar una fase
git switch -c fase/04-clientes

# Ver el estado de forma compacta
git status -sb

# Revisar ANTES de commitear (obligatorio)
git diff
git diff --staged

# Añadir por partes (te obliga a revisar cada hunk)
git add -p

# Commit
git commit

# Enmendar el último commit (solo si NO se ha subido)
git commit --amend

# Ver el historial legible
git log --oneline --graph --decorate --all

# Ver qué cambió en un fichero, con contexto
git log -p --follow src/main/java/.../Credito.java

# Encontrar quién y cuándo introdujo una línea
git blame -L 40,60 src/main/java/.../Credito.java

# Encontrar el commit que introdujo un bug (búsqueda binaria)
git bisect start
git bisect bad
git bisect good <commit-que-funcionaba>

# Guardar trabajo a medias
git stash push -m "mitad del mapper"
git stash pop

# Cerrar la fase: limpiar el historial antes de integrar
git rebase -i main

# Integrar
git switch main
git merge --no-ff fase/04-clientes
```

> 💡 `git add -p` es probablemente el comando que más mejora la calidad de los commits: te obliga a
> mirar cada trozo de cambio antes de incluirlo, y descubre despistes (un `println` olvidado,
> una prueba comentada).

## 5. Antes de cada commit

```
[ ] mvn verify pasa
[ ] Revisé el diff completo (git diff --staged)
[ ] No hay println, código comentado, ni TODO sin identificador
[ ] No hay secretos, contraseñas ni rutas locales
[ ] El mensaje explica el POR QUÉ
[ ] Los ficheros generados no están incluidos (target/, .idea/)
```

## 6. Antes de cerrar una fase

```
[ ] Todos los criterios de aceptación de la fase pasan
[ ] La bitácora de la fase está escrita
[ ] El roadmap tiene la fase marcada
[ ] Los ADR nuevos están creados
[ ] El historial de la rama es legible (rebase interactivo si hace falta)
[ ] Etiqueta: git tag -a fase-04 -m "Fase 04 — CRUD de clientes"
```

## 7. Repositorio remoto

```bash
# Crear el repositorio (una sola vez)
gh repo create creditcore --private --source=. --remote=origin

# Subir
git push -u origin main
git push --tags
```

Se recomienda **privado** mientras el proyecto está en construcción, y hacerlo público al final
como portafolio — momento en el que se revisa que no quede ningún dato sensible en el historial.

## 8. Qué NUNCA se sube

- `target/`, `.idea/`, `*.iml`
- `.env`, credenciales, `*.p12`, `*.pem`
- `*.tfstate`
- Datos personales reales (usa datos sintéticos en las pruebas)
- Volcados de base de datos con información real

> Si un secreto llega a subirse: **rotarlo inmediatamente**. Borrarlo del historial con
> `git filter-repo` es necesario, pero no suficiente: hay que asumir que quedó comprometido.
