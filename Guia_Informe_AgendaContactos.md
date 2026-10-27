# Guía para el informe – AgendaContactos (Grupo 7)

## 1. Plantilla rápida de decisiones
**Estilo:** Arquitectura en capas (Presentación → Lógica/Dominio → Persistencia).
**Patrones arquitectónicos:** MVC (Vista = `vista/`, Controlador = `controlador/`, Modelo = `modelo/` + `estrategias/`) · Repository (`repositorio/RepositorioContactos.js`) · DTO (`modelo/ContactoDTO.js`).
**Patrones GoF:**

| Categoría | Patrón | Dónde vive | Problema que resuelve en este caso |
|---|---|---|---|
| Creacional | Singleton | `RepositorioContactos.obtenerInstancia()` | Una sola copia de la lista en memoria; dos repositorios se sobrescribirían los datos. |
| Creacional | Factory | `modelo/FabricaContactos.js` | Un Contacto nace de dos fuentes (formulario o almacenamiento); la fábrica centraliza la normalización. |
| Estructural | Adapter | `repositorio/AdaptadorAlmacenamiento.js` | `localStorage` solo maneja texto; el repositorio necesita objetos. |
| Comportamiento | Observer | `modelo/Observable.js` → `AgendaServicio` (Subject) / `ControladorAgenda` (Observador) | La pantalla se repinta sola cuando el Modelo cambia, sin que el Modelo conozca la Vista. |
| Comportamiento | Strategy | `estrategias/` (`ResumenPorCategoria`, `CumpleanosDelMes`, `ContextoResumen`) | Dos resúmenes (por categoría en el panel; cumpleaños del mes en la torta) que se piden por id al contexto, sin `if/switch`. |

## 2. Árbol de archivos
```
AgendaContactos/
├── index.html
├── styles.css
├── img/                      (recursos gráficos)
└── js/
    ├── main.js                       Composición: crea y conecta todo
    ├── util/utilidades.js            Funciones puras
    ├── modelo/
    │   ├── Categorias.js             Catálogo (categorías, meses)
    │   ├── Contacto.js               Entidad + reglas de validación
    │   ├── ContactoDTO.js            DTO
    │   ├── FabricaContactos.js       Factory
    │   ├── Observable.js             Observer (Subject)
    │   └── AgendaServicio.js         Lógica de aplicación (única puerta al Modelo)
    ├── repositorio/
    │   ├── AdaptadorAlmacenamiento.js  Adapter
    │   └── RepositorioContactos.js     Repository + Singleton
    ├── estrategias/                  Strategy (interfaz, 2 concretas, contexto)
    ├── vista/
    │   ├── VistaAgenda.js            Vista pasiva
    │   ├── DecoracionTarjetas.js     Recurso visual
    │   └── EfectosVisuales.js        Loader, topbar, menú móvil
    ├── controlador/ControladorAgenda.js   Controlador delgado + Observador
    └── datos/contactosDemo.js        Datos de ejemplo (opcional)
```
**Justificación:** una carpeta por responsabilidad; las dependencias van en un solo sentido (Vista → Controlador → Servicio → Repositorio → Adaptador). El Modelo no importa nada de `vista/` ni del DOM.

## 3. Flujo de "agregar un contacto" (para la sustentación)
1. El usuario pulsa **Guardar** → `VistaAgenda` llama a `manejadores.guardar()`.
2. `ControladorAgenda.#guardar()` lee los valores crudos de la Vista y arma un **`ContactoDTO`**.
3. Llama a `AgendaServicio.agregar(dto)`, que ejecuta `Contacto.validar()` (reglas de negocio en el Modelo).
4. Si hay errores → devuelve `{ok:false, errores}` y la Vista solo los muestra.
5. Si es válido → **`FabricaContactos.desdeDTO()`** crea el Contacto → **`RepositorioContactos.guardar()`** → **`AdaptadorAlmacenamiento`** lo escribe en `localStorage`.
6. El servicio hace `notificar('contactoAgregado')` (**Observer**) → el controlador se repinta pidiendo datos al Modelo y entregándoselos a la Vista (incluido el resultado de la **Strategy** activa).

## 4. Reglas MVC: dónde comprobarlas
- **Modelo no toca el DOM:** `grep -rn "document\|window" js/modelo js/estrategias js/repositorio` → solo `window.localStorage` se pasa desde `main.js`.
- **Vista no valida reglas de negocio:** las regex están en `Contacto.validar()`; la Vista solo recibe `{campo: mensaje}`.
- **Controlador delgado:** no tiene HTML ni regex; solo coordina y guarda estado de pantalla (filtros, página).

## 5. Casos de prueba manuales (Fase 7)
| # | Acción | Resultado esperado |
|---|---|---|
| 1 | Abrir la app por primera vez | Carga 30 contactos de ejemplo, ordenados A→Z. |
| 2 | Guardar el formulario vacío | Errores en nombre, teléfono, email y categoría; no se guarda. |
| 3 | Agregar contacto válido (con y sin fecha de nacimiento) | Aparece en la lista; el contador sube en 1. |
| 4 | Fecha de nacimiento futura | Error "no puede ser futura". |
| 5 | Buscar "mar" | Solo contactos cuyo nombre contiene "mar" (ignora mayúsculas y tildes). |
| 6 | Editar un contacto | Formulario precargado; el cambio se refleja en la tarjeta. |
| 7 | Eliminar un contacto y confirmar | Desaparece y el contador baja en 1. |
| 8 | Recargar la página | Los cambios persisten (no se duplican los de ejemplo). |
| 9 | Strategy "Por categoría" → clic en tarjeta Familia | La lista se filtra a esa categoría. |
| 10 | Abrir la torta 🎂 de la barra superior | Lista TODOS los cumpleaños del mes actual, ordenados por día, con insignia ¡Hoy! / Ya pasó / En N días. |
| 11 | Día en que alguien cumple años | La torta se mueve, aparece el punto rojo y esa persona se marca con aro morado en la lista. Si hoy nadie cumple, la torta queda quieta y sin punto. |
| 12 | Paginación (más de 10 contactos) | Página 2 muestra "Mostrando 11 - 20 de 30". |

## 6. Checklist de patrones (¿está de verdad en el código?)
- [x] MVC — carpetas `vista/`, `controlador/`, `modelo/`
- [x] Repository — `RepositorioContactos`
- [x] DTO — `ContactoDTO` (usado por formulario y demo)
- [x] Singleton — `RepositorioContactos.obtenerInstancia()` (el constructor lanza error si se usa `new` dos veces)
- [x] Factory — `FabricaContactos` (nadie más hace `new Contacto`, salvo las pruebas)
- [x] Adapter — `AdaptadorAlmacenamiento`
- [x] Observer — `Observable` + `agenda.suscribir()` en el controlador
- [x] Strategy — `EstrategiaResumen` + 2 concretas + `ContextoResumen`
