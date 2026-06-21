# Urban Grocery – Pruebas Web y de API

Proyecto de testing de API REST realizado como parte del programa de QA de **TripleTen LatAm** (junio 2026). El objetivo fue verificar la robustez y el comportamiento esperado de dos endpoints críticos del servicio Urban Grocery mediante pruebas funcionales y de validación de datos ejecutadas con Postman.

## Contexto de negocio

Urban Grocery es un servicio de compras y entregas a domicilio que permite a los usuarios armar **kits de productos** y calcular la viabilidad de una entrega exprés a través del servicio "Order and Go". Ambas funcionalidades dependen de una API REST que debe validar correctamente los datos de entrada antes de procesarlos: límites de cantidad, tipos de dato, campos obligatorios y rangos numéricos de negocio (horario, peso y número de artículos permitidos por pedido).

Una API que no valida sus entradas correctamente no solo es un problema técnico: puede generar pedidos inconsistentes, cobros incorrectos, entregas prometidas que no se pueden cumplir, o caídas del servicio ante datos malformados — todo esto impacta directamente la confianza del cliente y la operación logística.

## Endpoints evaluados

| Endpoint | Método | Función | Casos de prueba |
|---|---|---|---|
| `/api/v1/kits/{id}/products` | POST | Añadir productos a un kit existente | 19 |
| `/order-and-go/v1/delivery` | POST | Calcular viabilidad y costo de una entrega exprés | 22 |

## Metodología

Las pruebas se diseñaron aplicando técnicas de **análisis de valores límite** y **clases de equivalencia** (válidas e inválidas) sobre cada campo del cuerpo de la solicitud: tipos de dato correctos e incorrectos (string, null, booleano, caracteres especiales, alfabetos no latinos), presencia/ausencia de campos obligatorios, límites superior e inferior de rangos numéricos de negocio, y combinaciones de parámetros de ruta válidos e inválidos.

Cada caso se documentó en Excel con: datos de entrada, pasos de ejecución, resultado esperado, resultado obtenido y estatus (Passed/Failed). Cada defecto encontrado se reportó como un ticket independiente en Jira, con descripción, pasos de reproducción, resultado actual vs. esperado, configuración del entorno, evidencia visual de la petición/respuesta en Postman, prioridad y severidad.

## Resultados generales

| Métrica | Valor |
|---|---|
| Casos de prueba totales | 41 |
| Casos exitosos (Passed) | 22 (53.7%) |
| Casos fallidos (Failed) | 19 (46.3%) |
| Defectos reportados en Jira | 19 (QS4-1 a QS4-21, con QS4-9 y QS4-14 eliminados por duplicidad) |

| Endpoint | Total casos | Passed | Failed |
|---|---|---|---|
| `kits/{id}/products` (Requisito 1) | 19 | 10 | 9 |
| `order-and-go/v1/delivery` (Requisito 2) | 22 | 12 | 10 |

## Hallazgo principal

El patrón de falla más significativo y recurrente fue la **ausencia de validación de entrada (input validation)** en ambos endpoints, manifestada de dos formas distintas según el endpoint:

- **`kits/{id}/products`**: ante datos inválidos o malformados (campos faltantes, tipos de dato incorrectos, parámetros de ruta no numéricos), el sistema no responde con `400 Bad Request` como se esperaba, sino con **`500 Internal Server Error`**, en algunos casos exponiendo mensajes de error interno del servidor (ej. `"invalid input syntax for type integer: \"NaN\""`). Esto indica que la validación no ocurre antes de tocar la base de datos, sino que el error se produce más abajo en la capa de persistencia.

- **`order-and-go/v1/delivery`**: la validación de rangos de negocio (horario, peso, cantidad de artículos) y de campos obligatorios prácticamente **no existe**. El sistema responde de forma consistente con **`200 OK`** y `"isItPossibleToDeliver": true`, incluso cuando se envían valores fuera de rango, tipos de dato incorrectos (string, null, booleano) o se omiten campos obligatorios por completo. Este comportamiento es más grave que un error 500: el sistema no falla de forma visible, sino que **confirma una operación que en realidad no debería ser válida**, lo cual puede inducir a un cliente o a un sistema consumidor a actuar sobre una respuesta falsamente exitosa.

## Hallazgos por severidad

| Severidad | Cantidad | Criterio aplicado |
|---|---|---|
| 🔴 Crítica | 11 | El sistema responde `200 OK` confirmando una operación inválida ("falso éxito"): datos fuera de rango, tipo incorrecto o campos obligatorios ausentes son aceptados como si fueran válidos. |
| 🟠 Alta | 8 | El sistema responde `500 Internal Server Error` en vez de `400 Bad Request`/`404 Not Found`, exponiendo errores de servidor ante datos de entrada inválidos. |

### Detalle de defectos — Requisito 1: `kits/{id}/products`

| Ticket | Caso | Resultado esperado | Resultado obtenido | Severidad |
|---|---|---|---|---|
| QS4-1 | ID de producto inexistente (999) | 404 Not Found | 200 OK, respuesta incompleta (sin `id`) | 🔴 Crítica |
| QS4-2 | `productsList` enviado como array vacío | 400 Bad Request | 500 Internal Server Error | 🟠 Alta |
| QS4-3 | Campo `id` ausente en el objeto de la lista | 400 Bad Request | 500 Internal Server Error | 🟠 Alta |
| QS4-4 | Campo `quantity` ausente en el objeto de la lista | 400 Bad Request | 500 Internal Server Error (mensaje `NaN` expuesto) | 🟠 Alta |
| QS4-5 | Campo `id` con tipo de dato inválido (string) | 400 Bad Request | 500 Internal Server Error | 🟠 Alta |
| QS4-6 | Campo `quantity` con tipo de dato inválido (booleano) | 400 Bad Request | 500 Internal Server Error | 🟠 Alta |
| QS4-7 | Body vacío `{}`, sin objeto raíz `productsList` | 400 Bad Request | 500 Internal Server Error | 🟠 Alta |
| QS4-8 | Parámetro de ruta `id` con formato inválido (letras) | 400 Bad Request / 404 Not Found | 500 Internal Server Error | 🟠 Alta |
| QS4-10 | Campo obligatorio enviado como `null` | 400 Bad Request | 500 Internal Server Error | 🟠 Alta |

### Detalle de defectos — Requisito 2: `order-and-go/v1/delivery`

| Ticket | Caso | Resultado esperado | Resultado obtenido | Severidad |
|---|---|---|---|---|
| QS4-11 | `deliveryTime` por debajo del rango permitido | `isItPossibleToDeliver: false` | `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-12 | `deliveryTime` por encima del rango permitido | `isItPossibleToDeliver: false` | `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-13 | `productsCount` por debajo del rango permitido | `isItPossibleToDeliver: false` | `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-15 | `productsWeight` por debajo del rango permitido | `isItPossibleToDeliver: false` | `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-16 | `deliveryTime` con tipo de dato inválido (string) | 400 Bad Request | 200 OK, `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-17 | `deliveryTime` con valor `null` | 400 Bad Request | 200 OK, `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-18 | `deliveryTime` con tipo de dato inválido (booleano) | 400 Bad Request | 200 OK, `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-19 | Campo obligatorio `deliveryTime` ausente | 400 Bad Request | 200 OK, `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-20 | Campo obligatorio `productsCount` ausente | 400 Bad Request | 200 OK, `isItPossibleToDeliver: true` | 🔴 Crítica |
| QS4-21 | Campo obligatorio `productsWeight` ausente | 400 Bad Request | 200 OK, `isItPossibleToDeliver: true` | 🔴 Crítica |

> **Nota:** los tickets QS4-9 y QS4-14 fueron creados por error durante el reporte en Jira y eliminados posteriormente; por eso el consecutivo de numeración salta de QS4-8 a QS4-10 y de QS4-13 a QS4-15. No corresponden a ningún caso de prueba documentado.

## Estructura del repositorio

```
urban-grocery-qa/
├── README.md
├── casos-de-prueba/
│   └── casos-de-prueba-api-postman.xlsx     # Las 41 pruebas: datos, pasos, resultados, estatus
├── checklists/
└── evidencias/
    └── capturas-defectos/                    # 2 imágenes por defecto (38 archivos)
        ├── QS4-N-reporte-jira.png            # Ticket completo en Jira
        └── QS4-N-evidencia-postman.png        # Petición y respuesta en Postman
```

## Stack y herramientas

`Postman` `Jira` `Excel` `API REST` `Pruebas de API` `Análisis de valores límite` `Clases de equivalencia` `Validación de datos`

---

**Autora:** Deicy Hernández Zamora — QA Tester
[GitHub](https://github.com/deicyhernandez-qa)
