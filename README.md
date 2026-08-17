# Sistema-de-Gesti-n-de-Cajas-Bancarias-Empresa-ABC
Proyecto final del curso **Estructuras de Datos (SC-304)**, Ingeniería en Sistemas de Computación — Universidad Fidélitas. La empresa ficticia **ABC**, dedicada a brindar soluciones de TI, encarga una herramienta para gestionar las cajas de atención de un banco: configuración inicial, emisión y atención de tiquetes por prioridad, balanceo de carga entre cajas, recomendación de productos complementarios y reportes históricos.

Desarrollado en **Java** con interfaz por `JOptionPane` (Swing), sin frameworks ni librerías de colecciones de Java (`LinkedList`, `TreeMap`, `PriorityQueue`, etc.) — todo el enlazado de nodos, recorridos e inserciones ordenadas está implementado desde cero, como lo exige el enunciado del curso.

## Requisitos del sistema

Según el enunciado, el sistema debe:

- Funcionar para cualquier nombre de banco y cualquier cantidad de cajas (mínimo 3), configurables una única vez y persistidas en `prod.txt`.
- Contar con una caja preferencial (discapacidad, embarazo, adultos mayores, empresarial), una caja rápida (un solo trámite) y las demás para dos o más trámites no preferenciales.
- Requerir login de al menos dos usuarios para acceder al sistema.
- Emitir tiquetes con nombre, ID, edad, hora de creación, hora de atención, trámite (Depósitos, Retiros, Cambio de Divisas) y tipo (P/A/B), asignándolos a la caja con menor cantidad de clientes en fila.
- Registrar cada atención en un histórico para generar reportes (caja con más clientes atendidos, total atendidos, mejor tiempo promedio, promedio general).
- Sugerir productos complementarios según el trámite (ej. depósitos → seguros) mediante un grafo, mostrando la sugerencia como recordatorio interno al cajero, no directamente al cliente.
- Persistir la fila de clientes ante un cierre inesperado del programa, y consultar el tipo de cambio del dólar en tiempo real vía el Web Service del Banco Central de Costa Rica.

Este repositorio se enfoca en el **núcleo de estructuras de datos** del proyecto, que es la parte que sostiene todos los módulos anteriores; los módulos de configuración, login, persistencia en archivo y consumo del web service del BCCR se apoyan sobre estas mismas estructuras pero viven en otras clases del proyecto.

## Motivación del diseño

El reto de fondo era resolver varios problemas de atención al cliente — priorización, balanceo de carga entre cajas, clasificación automática de tipo de cliente y recomendación de productos — combinando varias estructuras de datos clásicas en una sola arquitectura, en vez de forzar una sola estructura para todo. Cada estructura se eligió según el tipo de operación que debía resolver de forma más eficiente.

## Estructuras implementadas

### 1. Cola de prioridad (`ColaPrioridad`)
Lista enlazada simple, ordenada por prioridad de tipo de cliente en el momento de la inserción (`P` > `A` > `B`). Cada inserción recorre la cola para ubicar al cliente en la posición correcta según su prioridad, y devuelve cuántas personas quedaron adelante — útil para mostrarle al cliente su posición estimada en fila. Mantiene además un contador de tiquetes emitidos y el tamaño real de la fila en todo momento.

### 2. Estructura híbrida de cajas (`GestionCajaHibridaEstructura`)
La pieza central del proyecto. Un mismo conjunto de nodos (`NodoCaja`) participa simultáneamente en dos estructuras distintas mediante punteros independientes:

- **Lista circular doblemente enlazada**, ordenada por cantidad de clientes en fila — permite encontrar en O(1) la caja menos ocupada (`ini`) para asignar al siguiente cliente, y reordenar una caja cuando su carga cambia sin reconstruir toda la lista.
- **Árbol binario de búsqueda**, ordenado por número de caja — permite ubicar una caja específica por su identificador sin recorrer la lista completa.

Sobre esta base opera un **árbol de decisión** independiente (`NodoArbolDecision`), que clasifica a cada cliente recorriendo preguntas binarias (¿es preferencial? ¿tiene 2+ trámites? ¿es de caja rápida?) hasta llegar a una hoja con la categoría de caja que le corresponde (`PREFERENCIAL`, `COMUN`, `RAPIDA`). El flujo completo (`recibirYAsignarCliente`) combina las tres: clasifica al cliente con el árbol de decisión, busca la caja menos ocupada de ese tipo en la lista circular, lo encola con `ColaPrioridad`, y reordena la caja en la lista.

### 3. Grafo de productos (`GrafoProductos`)
Grafo dirigido implementado con listas de adyacencia enlazadas manualmente (`NodoProducto` → `NodoArista`). Cada trámite es un vértice, y sus aristas apuntan a los productos financieros que se le pueden ofrecer en venta cruzada. Se inicializa con relaciones predeterminadas (ej. *Depósitos* → *Plan de Ahorro Programado*) y expone `obtenerRecomendaciones` para generar sugerencias en tiempo real durante la atención.

### 4. Módulo de reportes (`GestorReportes`)
No es una estructura nueva en sí, pero consume una lista enlazada propia (`ListaEstadisticasCaja`) para acumular estadísticas por caja a partir de un archivo histórico de atenciones. El parseo de cada línea del archivo (`extraerCampoManual`) se hace carácter por carácter, sin usar `String.split()`, para practicar manipulación manual de cadenas. Genera tres reportes: resumen general (promedios y totales), desglose por tipo de trámite, y búsqueda de atenciones por nombre de cliente.

## Estructura del proyecto

```
com.mycompany.proyectoestructuraempresaabc.Estructura/
├── ColaPrioridad.java                  # Cola de prioridad por tipo de cliente
├── GestionCajaHibridaEstructura.java   # Lista circular + BST + árbol de decisión
├── GrafoProductos.java                 # Grafo de recomendaciones de venta cruzada
└── GestorReportes.java                 # Estadísticas e histórico desde archivo
```

> Nota: estas cuatro clases dependen de otras clases de modelo del proyecto (`Caja`, `NodoCliente`, `NodoProducto`, `NodoArista`, `ListaEstadisticasCaja`, `NodoEstadisticaCaja`) que forman parte del paquete completo pero no se incluyen en este extracto.

## Decisiones técnicas destacadas

- **Doble indexación sobre los mismos nodos** en la estructura híbrida de cajas: en vez de mantener dos colecciones separadas (una lista y un árbol) que habría que sincronizar manualmente, los punteros de ambas estructuras conviven en el mismo objeto `NodoCaja`. Esto evita duplicar información y mantiene ambas vistas siempre consistentes.
- **Árbol de decisión desacoplado del árbol de cajas**: la clasificación del cliente (qué tipo de caja necesita) y la búsqueda de la caja física son problemas distintos, resueltos con estructuras distintas, aunque ambas sean "árboles".
- **Sin dependencias de la librería estándar de colecciones**: todo el enlazado (`siguiente`, `anterior`, `izq`, `der`) se maneja a mano, lo que expone explícitamente la complejidad de cada operación (inserción, extracción, reordenamiento) en lugar de ocultarla detrás de una API.
- **Parsing manual de archivos** en `GestorReportes` en lugar de `split(";")`, como ejercicio deliberado de manipulación de cadenas carácter por carácter.

## Cómo ejecutar

Proyecto Java estándar (NetBeans). Compilar y ejecutar la clase principal del paquete `com.mycompany.proyectoestructuraempresaabc` (no incluida en este extracto), que maneja login, configuración inicial y el menú principal por `JOptionPane`. El módulo de reportes requiere un archivo `historico_reportes.txt` en el directorio de ejecución con registros previos de atención en formato `numCaja;nombre;tramite;tiempo`.

## Nota

Este README es para el repositorio de GitHub. El `readme.txt` que exige la entrega del curso (número de grupo e integrantes) es un archivo aparte, específico para el .zip que se sube al campus virtual — no reemplaza a este documento ni viceversa.

---

Proyecto desarrollado para el curso de Estructuras de Datos (SC-304) — Ingeniería en Sistemas de Computación, Universidad Fidélitas. Prof. Jorge Andrés Mora M.
