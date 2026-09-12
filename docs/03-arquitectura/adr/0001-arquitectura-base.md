# ADR-0001 — Arquitectura base del sistema

**Estado:** Aceptado · **Fecha:** 2026-09-12 · **Fase:** F00

## Contexto
CreditCore gestiona dinero, tiene reglas de negocio densas (amortización, imputación, mora),
normativa (DIAN, usura, habeas data) e integraciones externas que cambiarán (pasarelas, proveedor
tecnológico, centrales de riesgo). Lo desarrolla una sola persona con fines de aprendizaje profundo,
y debe parecerse a un sistema laboral real.

## Opciones consideradas

1. **Monolito por capas** — rápido y familiar. El dominio quedaría acoplado a JPA y Spring; la lógica
   financiera se diluiría en `@Service` grandes; probar reglas exigiría base de datos.
2. **Microservicios** — despliegue independiente. Costo operativo (gateway, discovery, trazas,
   sagas, contratos) injustificable para un equipo de una persona y un volumen de decenas de miles
   de créditos. Además, las operaciones críticas se benefician de transacciones locales.
3. **Monolito modular + hexagonal + DDD táctico** — un despliegue, fronteras explícitas por módulo,
   dominio aislado de la tecnología, verificable automáticamente.

## Decisión
Opción 3: **monolito modular con arquitectura hexagonal, DDD táctico y organización por vertical
slices**, con CQRS ligero para consultas.

Razones determinantes:
- El dominio es el activo del sistema y debe poder probarse en milisegundos, sin infraestructura.
- Las integraciones externas son volátiles; los puertos las aíslan.
- Las fronteras entre módulos se pueden verificar con ArchUnit, así que no dependen de la disciplina.
- Si algún día hiciera falta extraer un servicio (`facturacion` es la candidata natural), la
  arquitectura lo permite sin reescribir el negocio.

## Consecuencias

### Positivas
- Dominio testeable sin Spring ni base de datos.
- Cambiar de pasarela o de proveedor DIAN no toca reglas de negocio.
- Estructura que enseña explícitamente la inversión de dependencias.

### Negativas (aceptadas)
- Doble modelo (dominio y entidad JPA) y mappers que mantener.
- Más clases e interfaces que en una arquitectura por capas.
- Curva de aprendizaje mayor — que en este proyecto es un objetivo, no un costo.

### Mitigaciones
- Excepción documentada: módulos de pura parametrización pueden usar una capa simple.
- ArchUnit hace cumplir las reglas; no dependen de recordarlas.

### Cómo se revertiría
Colapsar `dominio` + `aplicacion` en un paquete de servicios y usar entidades JPA como modelo.
Coste alto una vez avanzado el proyecto; por eso se decide al inicio.
