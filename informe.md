**![]()

72.40 Ingeniería de Software

Plataforma de Gestión de Activos Digitales (DAM) para Medios de Comunicación

Alumnos

Roman Berruti - 63533

Tomás Pinausig - 63167

Agostina Squillari - 64047

Docentes

Sotuyo Dodero, Juan Martin

Mogni, Guido Matias

Diciembre 2026

**

# Introducción

Este trabajo busca definir una arquitectura candidata para una Plataforma de Gestión de Activos Digitales (DAM) destinada a un grupo de medios que administra millones de archivos (video, audio e imágenes). El objetivo del sistema es permitir el almacenamiento a largo plazo de activos originales (masters), la generación y gestión de versiones derivadas (renditions) para distintos canales, la indexación inteligente mediante metadatos y análisis de contenido con IA, y una búsqueda rápida. Asimismo, la plataforma se encarga de automatizar la distribución multicanal (web, TV y redes sociales), garantizando recuperación eficiente de archivos grandes y escalabilidad ante el crecimiento continuo del volumen de datos.

La solución propuesta se diseña siguiendo lo visto en clase sobre documentación de arquitecturas (4+1), priorizando decisiones justificadas por atributos de calidad y considerando la restricción de la cátedra con respecto al vendor lock-in.

Dadas estas características, el sistema presenta desafíos arquitectónicos significativos en términos de escalabilidad, performance, durabilidad y desacoplamiento, que motivan la definición de una arquitectura orientada a componentes distribuidos y procesamiento asíncrono.

2. # Funcionalidad requerida

## 2.1 Requerimientos funcionales

El sistema debe cumplir con los siguientes requerimientos funcionales enumerados a continuación.

1. Permitir el almacenamiento de archivos digitales de gran tamaño, incluyendo videos, audios e imágenes, garantizando su conservación a largo plazo.
2. Permitir la recuperación y descarga de activos digitales, incluyendo archivos de gran tamaño, facilitando su acceso para el uso editorial.
3. Asociar metadatos descriptivos a cada activo digital y complementar dicha información mediante análisis automático del contenido utilizando técnicas de inteligencia artificial.
4. Permitir a los usuarios realizar búsquedas rápidas y eficientes sobre el conjunto de activos digitales, utilizando tanto metadatos como información derivada del análisis de contenido.
5. Automatizar la distribución de los activos digitales a múltiples canales de salida, tales como plataformas web, televisión y redes sociales.

## 2.2 Requerimientos no funcionales

El sistema deberá cumplir con los siguientes requerimientos no funcionales, que establecen restricciones y expectativas sobre su comportamiento, operación y evolución:

1. El sistema debe soportar millones de activos y un crecimiento continuo del volumen de datos, sin degradación significativa del servicio.
2. La plataforma debe ofrecer tiempos de respuesta adecuados para uso editorial, tanto en búsquedas como en accesos de activos.
3. La recuperación de archivos grandes debe realizarse de manera eficiente y confiable, evitando esperas excesivas en los flujos de trabajo de los editores.
4. El sistema debe ser escalable, permitiendo aumentar la capacidad de almacenamiento y procesamiento sin afectar la operación normal.
5. La plataforma debe garantizar la conservación a largo plazo de los activos, evitando pérdida o corrupción de información.
6. El sistema debe asegurar una alta disponibilidad del servicio para los usuarios internos, minimizando interrupciones y manteniendo continuidad operativa ante fallas.
7. Debe contar con mecanismos de seguridad acordes a un entorno corporativo: control de accesos, protección de datos y trazabilidad básica de acciones relevantes.
8. En caso de utilizar nube pública, la solución debe limitarse a infraestructura (IaaS), evitando dependencias de servicios gestionados o propietarios que generen vendor lock-in.

# 3. Atributos de Calidad

# TODO: ORDENAR POR PRIORIDAD

## 3.1 Disponibilidad

Para la plataforma de Gestión de Activos Digitales (DAM), la disponibilidad constituye uno de los atributos de calidad más críticos. La indisponibilidad del sistema impacta de manera directa en la capacidad de producir, editar y publicar contenidos, afectando los tiempos de respuesta del negocio.

La plataforma DAM es utilizada de forma intensiva por editores, productores y sistemas automatizados de publicación que dependen del acceso continuo a los activos digitales. Dado que el contenido puede ser requerido en distintos momentos del día y para múltiples canales, no existen períodos prolongados de inactividad en los que una caída del sistema resulte aceptable.

Adicionalmente, el sistema cumple un rol clave como origen del contenido distribuido automáticamente a plataformas web, televisión y redes sociales. Una indisponibilidad del DAM no solo afecta a los usuarios internos, sino que también interrumpe los flujos de distribución, dejando a los canales sin actualizaciones de contenido.

## 3.2 Interoperabilidad

La interoperabilidad es un atributo de calidad clave, ya que su función principal es integrarse con distintos sistemas del ecosistema del grupo de medios. El DAM debe poder comunicarse con plataformas externas como sistemas de gestión de contenidos web, sistemas de emisión televisiva y servicios de publicación en redes sociales.

Dado que la distribución de contenido es un proceso automatizado, resulta fundamental que el sistema exponga interfaces y servicios basados en estándares ampliamente utilizados, que permitan el intercambio de información y activos de manera fluida. Una adecuada interoperabilidad reduce la necesidad de intervenciones manuales, simplifica las integraciones y facilita la incorporación de nuevos canales de distribución a futuro.

## 3.3 Performance

El rendimiento es un atributo central para la plataforma DAM dado que condiciona directamente la operación editorial. Se exige una “búsqueda rápida” y se marca como esencial la “velocidad de recuperación de archivos grandes”, lo cual implica que el sistema debe responder en tiempos adecuados no solo en escenarios ideales, sino también bajo carga real y con millones de activos indexados.

Para el editor, el valor del sistema está en poder localizar material relevante y obtener una previsualización. Si la búsqueda tarda varios segundos o la recuperación de un video/imagen/audio se vuelve impredecible, el flujo de trabajo se interrumpe y el DAM deja de ser una herramienta habilitante para convertirse en un cuello de botella. Además, el sistema debe sostener un alto throughput en operaciones pesadas como la ingesta, el procesamiento de contenido (transcodificación y análisis con IA) y la distribución multicanal, ya que son procesos que pueden ocurrir en paralelo para muchos activos y compiten por recursos de cómputo, red y almacenamiento. En este contexto, resulta fundamental que el rendimiento del sistema sea estable y predecible, evitando degradaciones abruptas a medida que aumenta la carga operativa.

## 3.4 Confiabilidad

La confiabilidad es un atributo de calidad necesario, ya que la plataforma debe operar de manera consistente y predecible, minimizando fallas y errores a lo largo del tiempo. En un DAM, la confiabilidad se manifiesta especialmente en la correcta ejecución de flujos críticos como la ingesta de archivos, la generación de renditions, el análisis con IA, la indexación y la distribución multicanal. Dado que muchos de estos procesos son asíncronos y de larga duración, el sistema debe garantizar que los trabajos no se pierdan, que puedan reintentarse de forma segura (idempotencia) y que los resultados sean consistentes. Una baja confiabilidad genera fallas intermitentes difíciles de diagnosticar, retrabajo manual y pérdida de confianza de los editores en la plataforma.

## 3.5 Escalabilidad

La escalabilidad es un atributo de calidad prioritario para la plataforma, el sistema debe ser capaz de manejar un crecimiento continuo tanto en el volumen de datos almacenados como en la cantidad de usuarios y procesos concurrentes, sin requerir un rediseño de su arquitectura.

Se debe administrar millones de archivos y acompañar un incremento constante a lo largo del tiempo. En el contexto de un DAM para medios, los archivos no se eliminan, sino que se conservan como parte del patrimonio digital, lo que exige que la arquitectura pueda crecer de manera sostenida, especialmente en términos de almacenamiento y procesamiento asociado al análisis de contenido y transcodificación.

Asimismo, el sistema debe responder adecuadamente a picos de demanda generados por eventos de alta relevancia informativa, donde se incrementan de forma abrupta tanto las cargas de material como los accesos y descargas. La capacidad de escalar los recursos de procesamiento permite absorber estos picos sin degradar el servicio ni afectar los flujos de trabajo editoriales.

## 3.6 Seguridad

La seguridad es un atributo de calidad fundamental en una plataforma DAM destinada a un entorno corporativo de medios de comunicación, ya que el sistema administra activos que representan propiedad intelectual de alto valor, incluyendo material inédito o sensible previo a su publicación. Resulta crítico controlar de manera estricta quién puede acceder, descargar, modificar o distribuir cada activo, de modo de prevenir accesos no autorizados y fugas de información que puedan generar impactos económicos, legales o reputacionales para la organización.

Asimismo, la protección de la información durante su transmisión y almacenamiento, junto con mecanismos claros de autenticación y autorización, constituye un requisito indispensable para garantizar que los activos digitales se utilicen exclusivamente dentro de los límites definidos por el negocio y los derechos asociados a cada material.

## 3.7 Tolerancia a fallos

Dada la complejidad del sistema y la cantidad de componentes involucrados en una plataforma DAM de estas características, resulta fundamental que la falla de un componente individual no implique la caída total del sistema. La arquitectura debe permitir que el servicio continúe operando, aunque sea de manera degradada, ante fallas parciales de infraestructura o de software, preservando el acceso a las funcionalidades más críticas para los usuarios.

Este enfoque permite garantizar la continuidad del servicio y reducir el impacto operativo de incidentes inevitables, como la indisponibilidad temporal de nodos de procesamiento, servicios de análisis de contenido o recursos de almacenamiento. La tolerancia a fallos es especialmente relevante en un sistema que combina operaciones interactivas con procesos asíncronos de larga duración, donde la capacidad de aislar fallas y recuperar componentes sin interrumpir el resto del flujo resulta clave para sostener el funcionamiento global de la plataforma.

**
