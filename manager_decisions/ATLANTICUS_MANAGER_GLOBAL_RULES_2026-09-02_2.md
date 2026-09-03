# Atlanticus — Reglas Globales del Manager

**Estado:** APROBADO / CONGELADO  
**Fecha:** 2026-09-02  
**Ámbito:** Atlanticus Manager  
**Naturaleza:** Regla global reutilizable para ADA y otros proyectos

## 1. Principio general

El Manager es una capacidad genérica de Atlanticus. No debe acoplarse a ADA ni a un dominio particular.

La navegación, la Home, el sidebar, la paginación visual, el estado global y el workflow administrativo pertenecen al Manager.

La configuración concreta de cada módulo pertenece a su dominio.

## 2. Manager Home

La ruta raíz `/manager` representa una Home real e independiente. No debe resolver implícitamente a un módulo por defecto.

La Home debe contener:

- Encabezado y contexto general del Manager.
- Resumen global del estado de los módulos.
- Cards de módulos visibles para el usuario.
- Acceso directo desde cada card hacia la página del módulo.
- Paginación visual de las cards.
- Botón flotante del Manager para navegación rápida.

### Cards

Las cards responden a:

> ¿Qué puedo administrar y en qué estado está?

Cada card debe mantener información acotada:

- Nombre.
- Descripción.
- Estado.
- Acción para abrir el módulo.

Las cards no deben contener acciones de workflow como Guardar, Publicar o Proyectar.

### Paginación de Home

La Home utiliza paginación puramente de presentación.

Reglas:

- `page_size = 6`.
- La paginación permanece visible incluso cuando existe una sola página.
- Cuando existe una sola página, los controles anterior/siguiente permanecen deshabilitados.
- No se introduce paginación backend para la lista de módulos.
- El tamaño de página no cambia según viewport; la adaptación responsive corresponde al grid CSS.

## 3. Páginas de módulos

Las páginas internas, por ejemplo:

- `/manager/users`
- `/manager/navigation`
- `/manager/tools`

deben concentrarse exclusivamente en la configuración y operación de su módulo.

Reglas:

- Deben incluir una acción explícita para volver a `Manager Home`.
- Deben conservar el botón flotante del Manager.
- No deben repetir el resumen global del Manager.
- Deben aprovechar el espacio liberado para el editor/configuración específica.
- No deben mezclar información global del Manager con controles propios del dominio.

## 4. Separación obligatoria: Configuración vs Workflow

Esta es una regla global del Manager.

La superficie de configuración y la superficie de workflow administrativo deben mantenerse separadas.

### Configuración

Responde a:

> ¿Cómo está configurado este objeto?

Incluye únicamente controles propios del dominio.

Ejemplos:

- Sources.
- Structure.
- Campos de configuración.
- Relaciones.
- Parámetros específicos.

### Workflow

Responde a:

> ¿Qué hago con esta configuración?

Incluye el ciclo de vida administrativo, por ejemplo:

- Guardar borrador.
- Validar.
- Publicar.
- Proyectar.
- Estado de revisión.
- Revisión publicada/proyectada.
- Otras acciones de lifecycle cuando correspondan.

El workflow no debe mezclarse dentro del formulario de configuración si eso satura la ventana.

La UX actual de Guardar/Publicar/Proyectar debe conservarse mientras siga siendo funcional. Cambiar la Home no autoriza un rediseño del workflow.

## 5. Botón flotante

El botón flotante derecho del Manager se conserva como elemento permanente de navegación rápida.

Debe estar disponible tanto en:

- Manager Home.
- Páginas internas de módulos.

Su propósito es diferente al de las cards:

- Cards: visión de estado + entrada contextual.
- Botón flotante: navegación rápida entre módulos.

## 6. Sidebar

El sidebar del botón flotante debe escalar sin crecer indefinidamente.

Estructura conceptual:

```text
Sidebar
├── Header
│   └── Manager Home
├── Filtro
├── Lista de módulos
│   └── scroll vertical
└── Controles fijos
```

Reglas:

- `Manager Home` permanece disponible como acceso fijo.
- El filtro permanece visible.
- Solo la lista de módulos debe desplazarse verticalmente.
- El módulo activo debe distinguirse visualmente.
- No se utiliza paginación dentro del sidebar.
- No se introducen agrupaciones/categorías hasta que exista una necesidad real.

### Filtro

El filtro es local y simple.

Debe aplicarse únicamente sobre módulos que el usuario ya puede ver.

Orden obligatorio:

```text
ManagerModuleRegistry
        ↓
visible_modules(principal)
        ↓
filter(query)
        ↓
render
```

Nunca se debe filtrar primero sobre todos los módulos y comprobar permisos después.

El filtro inicial puede ser `contains`, case-insensitive, sobre identidad/nombre del módulo. No requiere fuzzy search, scoring ni backend search.

## 7. Fuente única de navegación

Las cards de Home y la lista del sidebar deben derivarse de la misma fuente de verdad:

`ManagerModuleRegistry`

No debe existir:

- Una lista hardcodeada para Home.
- Otra lista hardcodeada para sidebar.

Registrar un módulo debe permitir que aparezca automáticamente en las representaciones correspondientes, sujeto a visibilidad/permisos.

## 8. Responsabilidades

```text
ManagerModuleRegistry
        │
        ├── Home
        │   ├── Cards
        │   ├── Estado
        │   └── Paginación
        │
        └── Sidebar
            ├── Visibilidad
            ├── Filtro
            ├── Scroll
            └── Estado activo
```

Cada módulo mantiene aparte:

```text
Manager Module
├── Configuration Surface
│   └── Controles propios del dominio
└── Manager Workflow
    ├── Estado
    ├── Revisión
    └── Acciones de lifecycle
```

## 9. Restricciones de diseño

No introducir en este incremento, salvo finding real:

- Búsqueda backend.
- Fuzzy search.
- Favoritos.
- Orden manual.
- Categorías configurables.
- Agrupaciones artificiales.
- `page_size` configurable.
- Acciones de workflow en cards.
- Resumen global dentro de páginas internas.
- Rediseño innecesario del workflow existente.

## 10. Decisión de implementación

Esta validación define la regla global para futuros incrementos del Manager.

Los incrementos que afecten Home, navegación, sidebar, módulos o workflow deben respetarla y cualquier cambio posterior debe discutirse explícitamente antes de modificar este contrato.
