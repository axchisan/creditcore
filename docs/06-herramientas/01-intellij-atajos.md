# 01 — IntelliJ IDEA: atajos y productividad (macOS)

> Objetivo: que el IDE deje de ser un editor de texto con colores y se convierta en una herramienta.
> **Regla del proyecto: cada vez que uses el ratón para algo repetitivo, busca el atajo.**
>
> Símbolos: `⌘` Command · `⌥` Option/Alt · `⌃` Control · `⇧` Shift · `⏎` Enter · `⌫` Backspace

---

## 1. Los diez imprescindibles (automatízalos primero)

| Atajo | Acción |
|---|---|
| `⇧⇧` | **Search Everywhere** — clases, archivos, acciones, símbolos, todo |
| `⌘E` | Archivos recientes (`⌘E` dos veces: archivos editados recientemente) |
| `⌘B` / `⌘click` | Ir a la declaración |
| `⌥⌘B` | Ir a la **implementación** (interfaz → clase concreta) |
| `⌥⏎` | **Show Context Actions** — el atajo más potente del IDE |
| `⌘N` | Generate (constructor, getters, `equals`, `toString`, override) |
| `⌃T` | Menú de refactorizaciones |
| `⌘F12` | Estructura del archivo (navegación dentro de la clase) |
| `⇧F10` / `⌃R` | Ejecutar lo último |
| `⌘⇧F` | Buscar en todo el proyecto |

---

## 2. Navegación

| Atajo | Acción |
|---|---|
| `⇧⇧` | Search Everywhere |
| `⌘O` | Ir a clase |
| `⌘⇧O` | Ir a archivo |
| `⌥⌘O` | Ir a símbolo (método, campo) |
| `⌘B` | Declaración / usos |
| `⌥⌘B` | Implementaciones |
| `⌃⇧B` | Ir al tipo de la declaración |
| `⌥⌘U` | Diagrama UML de la clase |
| `⌘U` | Ir al método de la superclase |
| `⌘E` | Archivos recientes |
| `⌘⇧E` | Ubicaciones editadas recientemente |
| `⌘[` / `⌘]` | Atrás / adelante en la navegación |
| `⌘⇧⌫` | Al último punto de edición |
| `⌘F12` | Estructura del archivo |
| `⌃H` | Jerarquía de tipos |
| `⌃⌥H` | Jerarquía de llamadas (**quién llama a este método**) |
| `⌘G` | Ir a línea |
| `F2` / `⇧F2` | Siguiente / anterior error |
| `⌘1` | Panel de proyecto |
| `⌘⇧A` | Buscar una acción por nombre |
| `⌥F7` | Buscar usos |
| `⌘⌥F7` | Usos en ventana emergente |

> 💡 `⌃⌥H` (Call Hierarchy) es oro cuando llegas a un código que no conoces: te dice quién usa ese
> método y desde dónde. Es la forma rápida de entender un flujo ajeno.

---

## 3. Edición

| Atajo | Acción |
|---|---|
| `⌥⏎` | Acciones de contexto / quick fix |
| `⌘⇧⏎` | Completar sentencia (pone `;` y llaves) |
| `⌃␣` | Autocompletado básico |
| `⌃⇧␣` | Autocompletado inteligente (por tipo esperado) |
| `⌘P` | Ver parámetros del método |
| `F1` | Documentación rápida |
| `⌘D` | Duplicar línea |
| `⌘⌫` | Borrar línea |
| `⌥⇧↑` / `⌥⇧↓` | Mover línea |
| `⇧⌘↑` / `⇧⌘↓` | Mover bloque/método completo |
| `⌥↑` / `⌥↓` | Expandir / contraer selección **semántica** |
| `⌘/` | Comentario de línea |
| `⌥⌘/` | Comentario de bloque |
| `⌥⌘L` | Formatear código |
| `⌃⌥O` | Optimizar imports |
| `⌃⇧J` | Unir líneas |
| `⌘⇧V` | Historial del portapapeles |
| `⌃G` | Selección múltiple del siguiente igual |
| `⌥ + arrastrar` | Cursores múltiples |
| `⌘J` | Insertar live template |
| `⌘⌥J` | Envolver con live template (`try/catch`, `if`, bucle) |

> 💡 `⌥↑` (Extend Selection) es la forma correcta de seleccionar: expande por expresión → argumento
> → llamada → sentencia → bloque. Mucho más preciso que arrastrar el ratón.

---

## 4. Refactorizaciones

| Atajo | Acción | Cuándo la usarás aquí |
|---|---|---|
| `⇧F6` | Renombrar (en todo el proyecto) | Constantemente |
| `⌥⌘M` | Extraer método | Cuando un método pasa de 30 líneas |
| `⌥⌘V` | Extraer variable | Dar nombre a una expresión compleja |
| `⌥⌘C` | Extraer constante | Eliminar números mágicos |
| `⌥⌘F` | Extraer campo | |
| `⌥⌘P` | Extraer parámetro | |
| `F6` | Mover clase/método | Reorganizar módulos |
| `F5` | Copiar clase | |
| `⌘⌥N` | Inline (lo contrario a extraer) | Eliminar indirecciones inútiles |
| `⌃T` | Menú con todas | Cuando no recuerdas el atajo |
| `⌘⌥⇧T` | Refactor this (igual que `⌃T`) | |

**Refactorizaciones específicas del menú `⌃T` que usaremos mucho:**
- *Change Signature* — cambiar parámetros propagando a todas las llamadas.
- *Extract Interface* — crear un puerto a partir de una clase existente.
- *Extract Delegate* — dividir una clase que hace dos cosas.
- *Replace Constructor with Factory Method* — para los factory methods del dominio.
- *Introduce Parameter Object* — cuando un método tiene 4+ parámetros.

---

## 5. Generación de código (`⌘N`)

Sin Lombok (ADR-0004), `⌘N` es tu generador. Dentro de una clase:

| Opción | Genera |
|---|---|
| Constructor | Con los campos que elijas |
| Getter / Setter / both | Accessors |
| equals() and hashCode() | Con plantilla configurable |
| toString() | Cuidado: **excluye campos sensibles** |
| Override Methods (`⌃O`) | Implementar métodos de la interfaz o superclase |
| Implement Methods (`⌃I`) | Los abstractos pendientes |
| Test... | Crea la clase de prueba correspondiente |
| Delegate Methods | Delegación a un campo |

> 💡 `⌥⏎` sobre el nombre de una clase inexistente → *Create class*. Escribe primero el uso y deja
> que el IDE cree la clase. Es la forma natural de trabajar dirigido por el diseño.

---

## 6. Ejecución y pruebas

| Atajo | Acción |
|---|---|
| `⌃⇧R` | Ejecutar lo que está bajo el cursor (clase o método de prueba) |
| `⌃R` | Re-ejecutar lo último |
| `⌃⇧D` | Depurar lo que está bajo el cursor |
| `⌃D` | Re-depurar lo último |
| `⌘F2` | Detener |
| `⇧⌘T` | Saltar entre clase y su prueba (**crea la prueba si no existe**) |
| `⌘4` | Ventana de ejecución |
| `⌘5` | Ventana de depuración |

> 💡 `⇧⌘T` es el atajo que convierte "debería escribir la prueba" en "ya estoy escribiéndola".

---

## 7. Búsqueda

| Atajo | Acción |
|---|---|
| `⌘F` | Buscar en el archivo |
| `⌘R` | Reemplazar en el archivo |
| `⌘⇧F` | Buscar en el proyecto |
| `⌘⇧R` | Reemplazar en el proyecto |
| `⌥F7` | Buscar usos |
| `⌘F7` | Usos en el archivo actual |
| `⇧⌘F7` | Resaltar usos en el archivo |
| `⌘⇧A` | Buscar acción (cuando no recuerdas dónde está algo del IDE) |

En `⌘⇧F` usa los filtros: por tipo de archivo, por directorio, expresión regular. Y **Structural
Search** (`⌘⇧A` → "Structural Search") para buscar **patrones de código**, no texto:
por ejemplo, todos los `catch (Exception e) {}` vacíos.

---

## 8. Git dentro del IDE

| Atajo | Acción |
|---|---|
| `⌘9` | Ventana de Git |
| `⌘K` | Commit |
| `⌘⇧K` | Push |
| `⌘T` | Update (pull) |
| `⌥⌘Z` | Revertir cambios del archivo |
| `⌘⌥⇧↓` / `↑` | Ir al siguiente/anterior cambio |
| `⌃V` → | Menú de operaciones VCS |
| `⌥⇧C` | Cambios recientes |

> 💡 **Annotate** (clic derecho en el margen → *Annotate with Git Blame*): muestra quién escribió
> cada línea y en qué commit. Imprescindible para entender por qué algo está como está.

---

## 9. Ventanas y layout

| Atajo | Acción |
|---|---|
| `⌘1` Proyecto · `⌘4` Ejecución · `⌘5` Depuración · `⌘6` Problemas · `⌘7` Estructura · `⌘9` Git |
| `⇧⎋` | Cerrar la ventana activa |
| `⌘⇧F12` | Maximizar el editor (esconder todo) |
| `⌃⇧F12` | Restaurar layout |
| `⌘⌥[` / `]` | Dividir el editor |
| `⌘W` | Cerrar pestaña |
| `⌘⇧⏎` | Modo distracción cero |

---

## 10. Análisis y calidad

| Acción | Cómo |
|---|---|
| Inspeccionar código | `Code → Inspect Code` |
| Ver problemas del archivo | `⌘6` |
| Analizar dependencias | `Analyze → Analyze Dependencies` |
| Buscar duplicados | `Analyze → Locate Duplicates` |
| Ver complejidad | Plugin *MetricsReloaded* |
| Diagrama de dependencias de Maven | Panel Maven → *Show Dependencies* (`⌘⇧A` → "Show Dependencies") |

---

## 11. El HTTP Client integrado

IntelliJ trae un cliente HTTP en archivos `.http`. **Lo usaremos en vez de Postman**: vive en el
repositorio, se versiona y se ejecuta sin salir del IDE.

```http
### Variables de entorno (http-client.env.json)
@baseUrl = http://localhost:8080/api/v1

### Registrar cliente
POST {{baseUrl}}/clientes
Content-Type: application/json

{
  "tipoDocumento": "CC",
  "numeroDocumento": "1098765432",
  "nombres": "Ana María",
  "apellidos": "Pérez Gómez",
  "fechaNacimiento": "1995-04-12",
  "email": "ana@example.com",
  "celular": "3001234567"
}

> {%
    client.test("responde 201", function() {
        client.assert(response.status === 201, "Se esperaba 201");
    });
    client.global.set("clienteId", response.body.id);
%}

### Consultar el cliente creado (usa la variable capturada arriba)
GET {{baseUrl}}/clientes/{{clienteId}}
```

Ejecutar: `⌃⏎` sobre la petición, o el icono ▶ del margen.

---

## 12. Plugins recomendados

| Plugin | Para qué |
|---|---|
| **Key Promoter X** | Te avisa del atajo cada vez que haces algo con el ratón. **Instálalo hoy.** |
| **SonarQube for IDE** | Detecta olores y bugs mientras escribes |
| **Database Tools** (incluido en Ultimate) | Alternativa a DBeaver dentro del IDE |
| **Maven Helper** | Analiza conflictos de dependencias visualmente |
| **String Manipulation** | Conversiones de formato de texto |
| **Rainbow Brackets** | Legibilidad en expresiones anidadas |
| **GitToolBox** | Blame en línea, estado de la rama |

> 🎯 **Key Promoter X es literalmente una herramienta de aprendizaje de atajos.** Es el plugin que
> más impacto tendrá en tu velocidad durante este proyecto.

---

## 13. Plan de entrenamiento

| Semana | Objetivo |
|---|---|
| 1 | Los diez imprescindibles. Prohibido navegar con el ratón. |
| 2 | Refactorizaciones: renombrar, extraer método, extraer variable/constante. |
| 3 | Depurador completo (ver el documento siguiente). |
| 4 | Generación (`⌘N`), live templates propios, `⇧⌘T`. |
| 5 | Búsqueda avanzada, Structural Search, análisis. |
| 6+ | Git desde el IDE, HTTP Client, perfiles de ejecución. |

**Autoevaluación**: si al terminar el proyecto no usas el ratón para navegar, refactorizar ni
ejecutar pruebas, esta parte del objetivo está cumplida.
