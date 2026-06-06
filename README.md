# Recompresor seguro de buzones Maildir en cPanel/Dovecot

## Nota importante antes de comenzar

Este repositorio documenta un caso de estudio técnico. La implementación productiva completa no se publica, pero dispongo del flujo completo, probado y preparado para su adaptación a entornos reales en producción.

Si una persona, organización o empresa necesita una solución similar para recomprimir buzones Maildir históricos en cPanel/Dovecot en producción, puede contactarme directamente. Los medios de contacto están disponibles en el README.md de mi repositorio personal "v80000".

Toda consulta o posible implementación se tratará con privacidad y confidencialidad, el cual como se sabe de mí, siempre es mi prioridad.

## Descripción

Este proyecto documenta un flujo de automatización que diseñé y desarrollé para recomprimir buzones Maildir en un entorno cPanel/Dovecot donde la compresión de correo había estado deshabilitada durante mucho tiempo, mi flujo completo está pensado ya para actuar diréctamente en producción, ya que lo desarrollé en una empresa para solucionar este problema, tiene muchas, muchas pruebas hechas, y ya ha sido muy utilizado en producción sin problema alguno, pero por ello se realizan validaciones previas, ya que son 4 scripts, uno para validación individual, otro para validación en lotes, que usa el validación individal como worker, y luego los propios de recompresión, que funcionan igual, recompresión individual, y luego recompresión de buzones por lotes, sigue leyendo para más información y entender el como funciona.

El problema era bastante común en infraestructuras de correo: aunque después se habilitase la compresión en el servidor, todos los correos antiguos que ya habían entrado en los buzones seguían almacenados sin comprimir. En una empresa que gestiona correo para muchos clientes, esto puede acabar suponiendo una gran cantidad de espacio ocupado innecesariamente, backups más pesados, ventanas de mantenimiento más largas y mayor coste operativo.

La solución no era simplemente “comprimir ficheros”. Había que hacerlo con cuidado, respetando la estructura Maildir, evitando pérdida de mensajes, sin romper buzones de Dovecot, permitiendo ejecución por lotes y manteniendo siempre validación, logs y capacidad de rollback.

Este repositorio es una documentación técnica de mi proyecto y de su arquitectura.
Los scripts productivos reales no se publican por motivos de confidencialidad, seguridad operativa y protección de propiedad intelectual, para implementación, contactarme diréctamente.

---

## Nota rápida sobre los ejemplos

Aunque este repositorio no incluye la implementación productiva real, recomiendo revisar la carpeta `examples/`, donde incluyo salidas anonimizadas del flujo: entrada por lotes, validación, ejecución del recompresor, logs de una recompresión correcta y ejemplo de rollback.

Estos ejemplos ayudan a entender cómo se comporta el sistema en la práctica, qué información queda registrada y cómo se puede auditar cada operación realizada sobre un buzón.

---

## Problema que resuelve

En servidores cPanel/Dovecot puede darse el caso de que la compresión de correo se habilite tarde, cuando ya existen años de correos almacenados sin comprimir.

Esto provoca:

* Uso excesivo de disco.
* Backups más grandes.
* Ventanas de backup más lentas.
* Mayor coste de almacenamiento.
* Mayor dificultad para mover, copiar o mantener buzones grandes.
* Riesgo al intentar corregir el problema manualmente buzón por buzón.
* Falta de trazabilidad si no se registra cada operación.

En entornos donde se gestionan buzones de muchas empresas, el impacto puede multiplicarse rápidamente.

---

## Objetivo del proyecto

El objetivo fue diseñar un sistema automatizado para:

* Validar buzones antes de tocarlos.
* Separar buzones seguros de buzones bloqueados.
* Reprocesar buzones Maildir para que el contenido quedase almacenado con compresión.
* Usar staging para no modificar directamente el buzón original.
* Minimizar la ventana crítica de corte.
* Detectar actividad inesperada durante el cutover.
* Aplicar rollback si era necesario.
* Guardar backup del buzón anterior.
* Generar logs por buzón.
* Ejecutar el proceso de forma individual o por lotes.

---

## Arquitectura general

El flujo se dividió en dos partes principales:

1. Validación previa.
2. Recompresión controlada.

La idea era no ejecutar nunca el proceso de recompresión sobre un buzón que no hubiese pasado antes unas comprobaciones mínimas.

```text
Listado de buzones
        ↓
Validación individual
        ↓
Clasificación OK / BLOCK
        ↓
Ejecución por lotes sobre buzones válidos
        ↓
Staging Maildir
        ↓
Sincronización con doveadm backup
        ↓
Kick de sesiones activas
        ↓
Sincronización final
        ↓
Cutover controlado
        ↓
Detección de actividad durante ventana crítica
        ↓
Rollback o finalización
        ↓
Backup del buzón antiguo
        ↓
Logs y resumen
```

---

## Scripts diseñados

El sistema se diseñó en cuatro piezas principales.

### 1. Validador individual de buzón

Este componente valida un buzón concreto antes de permitir su recompresión.

Comprueba, entre otras cosas:

* Que existe el buzón original.
* Que no existe un staging previo.
* Que Dovecot puede listar correctamente las carpetas del buzón.
* Que se pueden consultar los GUIDs de las carpetas.
* Que no existen GUIDs duplicados.
* Que el buzón se encuentra en un estado seguro para ser procesado.

El resultado puede ser:

```text
OK|correo@dominio.com|usuario_cpanel|dominio.com|validacion limpia
```

O:

```text
BLOCK|correo@dominio.com|usuario_cpanel|dominio.com|motivo del bloqueo
```

La idea es sencilla: si un buzón tiene un estado raro, inconsistente o peligroso, se bloquea y se revisa manualmente.

---

### 2. Validador por lotes

El validador por lotes recibe un fichero de texto con buzones a comprobar.

Formato de entrada:

```text
correo@dominio.com|usuario_cpanel|dominio.com
otro@empresa.net|usuario2|empresa.net
```

Este script recorre línea por línea, ignora líneas vacías o comentadas, valida el formato y llama al validador individual como worker.

Los resultados se separan en dos ficheros:

```text
ok.txt
block.txt
```

De esta forma se puede preparar una tanda de buzones aptos para procesar y dejar apartados los buzones problemáticos.

---

### 3. Recompresor individual de buzón

Este es el componente principal del flujo.

Su función es reprocesar un buzón concreto usando una zona de staging, en vez de modificar el buzón original directamente.

El proceso general es:

1. Localizar el buzón original.
2. Crear una ruta de staging.
3. Ejecutar una primera sincronización con `doveadm backup`.
4. Ejecutar una segunda sincronización para reducir diferencias.
5. Expulsar sesiones activas del usuario con `doveadm kick`.
6. Esperar unos segundos.
7. Ejecutar una sincronización final.
8. Crear una marca temporal de cutover.
9. Mover el buzón original a una ruta temporal.
10. Mover el staging al lugar del buzón original.
11. Comprobar si entraron mensajes durante la ventana crítica.
12. Aplicar rollback si se detecta actividad o si falla algún movimiento.
13. Guardar el buzón antiguo como backup.
14. Ajustar permisos.
15. Registrar el resultado en logs.

El uso de `doveadm backup` permite reconstruir el buzón en staging pasando por Dovecot, evitando una manipulación directa e insegura de los ficheros originales.

---

### 4. Recompresor por lotes

El recompresor por lotes permite procesar muchos buzones a partir de un fichero `.txt`.

Usa el mismo formato:

```text
correo@dominio.com|usuario_cpanel|dominio.com
```

Por cada línea válida, llama al recompresor individual como worker.

Al final muestra un resumen:

```text
TOTAL: 100
OK: 97
FAIL: 3
```

Esto permite ejecutar el proceso de forma controlada sobre tandas de buzones, sin tener que lanzar manualmente cada recompresión.

---

## Estrategia de seguridad

El flujo se diseñó pensando en producción.

Algunas medidas importantes:

* No se toca un buzón sin validación previa.
* No se trabaja directamente sobre el buzón original.
* Se usa staging.
* Se hacen varias sincronizaciones antes del corte.
* Se expulsa la sesión activa del usuario antes de la sincronización final.
* Se marca temporalmente la ventana crítica.
* Se detectan mensajes nuevos durante el cutover.
* Se aplica rollback si algo no cuadra.
* Se conserva el buzón anterior como backup.
* Se generan logs por buzón.
* Se separan los buzones correctos de los bloqueados.

---

## Rollback

El rollback era una parte central del diseño.

Se contemplaron casos como:

* Fallo al mover el buzón original.
* Fallo al mover el staging al destino final.
* Aparición de contenido inesperado en la ruta original.
* Entrada de mensajes durante la ventana crítica.
* Fallos al archivar el buzón antiguo.

Cuando se detecta una situación insegura, el sistema intenta restaurar el buzón original y preservar el estado problemático para revisión manual.

El objetivo no era solo automatizar, sino automatizar sin perder capacidad de recuperación.

---

## Logs y trazabilidad

Cada ejecución genera un log por buzón.

Los logs permiten revisar:

* Buzón procesado.
* Ruta original.
* Ruta de staging.
* Ruta de backup.
* Hora de inicio.
* Tamaño inicial.
* Pasos ejecutados.
* Errores.
* Resultado final.
* Tamaño final.
* Estado de Dovecot.

Esto permite auditar qué se ha hecho y facilita la revisión en caso de incidencia.

---

## Ejemplo de entrada

```text
usuario1@example.com|cpuser1|example.com
usuario2@example.net|cpuser2|example.net
usuario3@example.org|cpuser3|example.org
```

---

## Ejemplo de salida de validación

```text
OK|usuario1@example.com|cpuser1|example.com|validacion limpia
BLOCK|usuario2@example.net|cpuser2|example.net|existe STAGE previo
OK|usuario3@example.org|cpuser3|example.org|validacion limpia
```

---

## Ejemplo de salida de ejecución

```text
Procesando: usuario1@example.com
OK: usuario1@example.com

Procesando: usuario3@example.org
OK: usuario3@example.org

TOTAL: 2
OK: 2
FAIL: 0
```

---

## Tecnologías utilizadas

* Linux
* Bash
* Dovecot
* `doveadm`
* cPanel mail layout
* Maildir / Maildir++
* Procesamiento por lotes
* Staging
* Rollback
* Logging
* Validación operacional

---

## Por qué es útil para empresas

Esta solución es especialmente útil para proveedores de hosting, empresas de soporte IT o administradores de servidores cPanel/Dovecot que hayan habilitado tarde la compresión de correo.

En esos casos, puede haber años de mensajes antiguos ocupando mucho más espacio del necesario.

Un flujo como este permite:

* Reducir uso de disco.
* Reducir tamaño de backups.
* Mejorar tiempos de copia y mantenimiento.
* Evitar trabajo manual repetitivo.
* Procesar buzones en lotes.
* Reducir riesgo operativo.
* Mantener trazabilidad de la operación.
* Separar automáticamente buzones seguros y problemáticos.

---

## Alcance de este repositorio

Este repositorio no contiene los scripts productivos reales.

Incluye documentación técnica, arquitectura y ejemplos anonimizados de entrada, salida y logs para explicar el diseño técnico del flujo sin publicar la implementación productiva.

La implementación completa se mantiene privada por motivos de:

* Confidencialidad.
* Seguridad operativa.
* Protección de propiedad intelectual.
* Evitar uso incorrecto en entornos de producción.
* Evitar exposición de detalles internos de infraestructura.

---

## Estado del proyecto

Proyecto realizado en un contexto real de administración de sistemas, orientado a resolver un problema operativo en infraestructura de correo.

Este repositorio actúa como caso de estudio técnico y portfolio profesional.

---

## Aviso

Este repositorio no debe usarse como herramienta lista para producción, son solo explicaciones de como funciona todo, mi flujo personalizado, el cual SI está listo para aplicar en producción, quien se encuentre con este problema, como digo, que me contacte.

Operaciones sobre Maildir, Dovecot y buzones reales pueden causar pérdida de datos si se ejecutan sin pruebas, backups, validación y conocimiento del entorno.

Antes de aplicar un flujo similar en producción, es imprescindible adaptarlo, probarlo en laboratorio y disponer de una estrategia de recuperación.


