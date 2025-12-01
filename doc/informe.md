# Distributed Machine Learning (DML):

## Introducción

**DML**: es un sistema que permite a los usuarios entrenar modelos de machine learning de forma distribuida y realizar predicciones utilizando modelos previamente entrenados.

El sistema expone una **API REST** que ofrece los servicios de entrenamiento y predicción, y además incluye una aplicación de consola que encapsula toda la lógica necesaria para que el funcionamiento del sistema sea completamente transparente para el usuario.

## Alcance del Sistema

1. El sistema ofrece diversos modelos de machine learning para tareas de ***regresión*** y ***clasificación.***

2. Permite a los usuarios subir archivos ****.csv**** con los datasets de entrenamiento o predicción.

3. Los usuarios pueden crear procesos de entrenamiento indicando:
    - el tipo de entrenamiento (regresión o clasificación). 
    - el dataset que se usará.
    - los modelos a entrenar.

4. El sistema permite consultar el estado de los entrenamientos en curso.

5. Los usuarios pueden ejecutar predicciones utilizando modelos previamente entrenados.

6. Se pueden consultar los resultados generados por las predicciones.

7. Los usuarios pueden descargar los modelos ya entrenados.

## Especificaciones del Sistema:

El sistema utiliza una arquitectura modular y desacoplada basada en dos tipos de nodos (Workers y DataBases)

## Workers:

Los nodos Worker son responsables de procesar las peticiones de los usuarios y ejecutar las tareas de entrenamiento y predicción de los modelos. Cada Worker es una instancia del backend (`dml-service`) que expone una API REST construida con FastAPI.

Cada Worker funciona de forma totalmente independiente, sin necesidad de comunicarse ni coordinarse con otros Workers. Su única interacción es con los nodos DataBase, que actúan como fuente de datos y como mecanismo de control del trabajo distribuido.


## Procesos de los Workers:

Cada Worker ejecuta 4 tipos de procesos distintos:

### 1. REST (API Endpoints):

Procesos que manejan las peticiones HTTP a través de los endpoints definidos en `app/api/endpoints/`. Se encargan de:

- **`/training/train`**: Crear sesiones de entrenamiento, generar `training_id` único e inicializar modelos en background
- **`/predictions/predict`**: Registrar sesiones de predicción para un modelo y dataset
- **`/datasets`**: Gestionar datasets disponibles
- **`/model_registry`**: Consultar información de modelos
- **`/health`**: Verificar el estado del Worker
- **`/leader/*`**: Endpoints para elección de líder entre Workers (`/compare`, `/announce`, `/get`, `/start-election`)

Estos endpoints procesan las peticiones de los clientes, validan los datos de entrada mediante esquemas Pydantic (`app/schemas/`), y realizan las operaciones necesarias sobre los nodos DataBase a través del `DatabaseManager`.

### 2. GET_MODEL (Polling de Tareas):

Proceso implementado en la clase `Runner` (`app/models/runner.py`) que ejecuta un hilo de polling continuo mediante el método `_poll_for_models()`. Su funcionamiento:

1. Verifica si hay capacidad disponible (controlado por `MAX_CONCURRENT_RUNNING_MODELS`)
2. Llama a `database_manager.get_model_to_run()` para obtener modelos disponibles
3. Recibe un objeto `ModelToRun` con:
   - **`model_id`**: Identificador del modelo
   - **`running_type`**: Tipo de tarea (`training` o `prediction`)
   - **`dataset_id`**: Dataset asociado
4. Inicia la ejecución del modelo mediante `_start_model_execution()`
5. Limpia modelos completados con `_cleanup_completed_models()`

El polling se ejecuta cada **2 segundos**.

### 3. Training/Prediction (Ejecución de ML):

Procesos gestionados por la clase `Runner` que crea dos hilos por cada modelo:

**Hilo de Ejecución (`_run_model`):**
- Obtiene los datos completos del modelo desde el DataBase
- Instancia la clase del modelo correspondiente (ej: `RandomForestClassifierModel`)
- Deserializa el estado previo del modelo
- Ejecuta `_execute_training()` o `_execute_prediction()` según el `running_type`
- Serializa y guarda los resultados

**Hilo de Status (`_send_status_notifications`):**
- Envía heartbeats cada **10 segundos** mediante `database_manager.send_model_running_status()`
- Se detiene cuando el hilo de ejecución termina

Las clases de modelos heredan de `MLModel` (`app/models/ml_model.py`) e implementan los métodos `train()`, `predict()`, `serialize()` y `deserialize()`.

### 4. Descubrimiento (Discovery):

Hilo implementado en la clase `Middleware` (`app/middleware/middleware.py`) mediante el método `_discover_ips()`:

- Se inicia con `start_monitoring()` al arrancar la aplicación
- Ejecuta `_refresh_service_ip_cache()` cada **10 segundos**
- Resuelve IPs mediante DNS con `_resolve_domain_ips()`
- Mantiene un `ip_cache` con todos los nodos descubiertos
- Verifica la salud de cada IP con `check_ip_alive()` haciendo peticiones a `/api/v1/health`
- Elimina IPs que no responden del caché
- Usa `cache_lock` (threading.Lock) para acceso thread-safe al caché

## Tolerancia a Fallas de los Workers:

Los Workers están diseñados con total independencia:

- **No conocen** la existencia de otros Workers
- **No comparten estado** ni intercambian mensajes entre sí
- **No requieren coordinación** directa

### Garantías:

| Característica | Descripción |
|----------------|-------------|
| **Escalabilidad Horizontal** | Agregar nuevos Workers sin cambios de configuración |
| **Tolerancia a Fallas** | K-1, donde K es la cantidad de Workers activos |
| **Resistencia a Particiones** | El sistema funciona mientras exista al menos un Worker activo |
| **Recuperación Automática** | Workers pueden reconectarse sin intervención manual |

## Coordinación entre Workers:

Para evitar que dos Workers procesen el mismo modelo simultáneamente, el sistema utiliza un mecanismo de **heartbeat** basado en el campo `health` almacenado en los nodos DataBase.

### Funcionamiento del Mecanismo:
┌─────────────┐ ┌─────────────┐
│ Worker │ │ DataBase │
│ (Runner) │ │ │
└──────┬──────┘ └──────┬──────┘
│ │
│ 1. get_model_to_run() │
│ (health > 20 segundos) │
│───────────────────────────────────>│
│ │
│ 2. ModelToRun(model_id, │
│ running_type, dataset_id) │
│<───────────────────────────────────│
│ │
│ 3. send_model_running_status() │
│ (cada 10s desde status thread) │
│───────────────────────────────────>│
│ │
│ 4. Actualiza health = now() │
│ │

### Etapas del Proceso:

1. **Selección del modelo disponible**:
   - Los nodos DataBase registran el `health` (timestamp) de cada modelo
   - El método `get_model_to_run()` retorna modelos cuyo `health` sea mayor a **20 segundos** (modelo libre o abandonado)

2. **Heartbeat del Worker**:
   - El hilo `_send_status_notifications()` envía señales cada **10 segundos**
   - Llama a `database_manager.send_model_running_status(model_id, dataset_id)`
   - El DataBase actualiza: `health = timestamp_actual`

3. **Detección de fallo o abandono**:
   - Si no se recibe heartbeat en más de **20 segundos**, el modelo se considera abandonado
   - El modelo queda disponible para reasignación

4. **Reasignación automática**:
   - Cualquier Worker puede tomar un modelo libre inmediatamente
   - No requiere bloqueos distribuidos ni coordinación entre Workers

### Ventajas del Mecanismo:

| Ventaja | Descripción |
|---------|-------------|
| **Sin conflictos** | Ningún modelo será procesado por dos Workers simultáneamente |
| **Alta disponibilidad** | Si un Worker falla, el trabajo se reasigna automáticamente |
| **Escalabilidad simple** | Más Workers = más capacidad sin cambios estructurales |
| **Desacoplamiento total** | La lógica distribuida se delega a los nodos DataBase |
| **Sin bloqueos distribuidos** | El campo `health` elimina la necesidad de locks complejos |


## DataBases:

Los nodos DataBase emplean una arquitectura peer to peer (p2p), comunicándose entre ellos, sin necesidad de un líder que controle el flujo de datos.

Cada DataBase expone una API REST para responder a las solicitudes que reciben desde los Workers o el resto de DataBases. 

Cada nodo mantiene su propio almacenamiento, procesa consultas de lectura y escritura, y participa en el mecanismo de replicación distribuida para lograr la consistencia del sistema.

### Procesos de los DataBases:

Los DataBase realizan 5 procesos distintos:

#### 1. RESTs: 

Encargados de procesar y darle respuesta a las peticiones de los workers u otros DataBases. ( son los procesos detrás de los endpoints que exponen).

Cuando un DataBase recibe una petición primeramente mira si el es capaz de realizarla (si tiene los datos necesarios guardados localmente) en caso de no poder busca un nodo que si pueda (a través de las tablas de rutas) y le realiza la petición a ese nodo.

####  2. Descubrimiento: 

Hilo que se encarga a través del DNS de Docker de saber que nodos DataBase están activos.

Mantiene una tabla con todos los nodos del sistemas (incluso los que no están activos), y a cada uno le asocia una variable health (bool que dice si está activo o no).

#### 3. Coordinación: 

Para coordinarse, los DataBases necesitan saber que nodos guardan cada dato, y esto lo hacen compartiendo unas tablas (llamadas tablas de rutas), las cuales precisamente relacionan los **ids** de los datos con las **ips** de los nodos que almacenan la información de dichos datos.

El sistema busca que entodo momento los nodos tengan las tablas lo más actualizado posible, por lo que cada vez que ocurre un cambio en alguna de las tablas el nodo que realizó el cambio la comparte con el resto de nodos.

Notemos que las tablas de ruta solo cambian en dos situaciones específicas:
- Cuando se agrega un nuevo dato al sistema.
- Cuando falla algún nodo.

Para el caso en que se agrega un nuevo dato al sistema el nodo DataBase que recibió la petición simplemente guarda el dato e inicia un proceso de replicación del dato. Una vez terminadas las replicaciones actualiza sus tablas de rutas y se las comparte a los demás nodos, para que la actualicen.

Para el caso en que falla un nodo (o varios nodos):

Primero definamos como **"dueño de un dato"** al nodo de la red que está activo y es el de menor ip que almacena el dato.

Cada nodo DataBase mantiene en ejecución un hilo que con ayuda de la lista de ips activos que proporciona el descubrimiento y con las tablas de rutas, se encarga de verificar si un dato no está replicado la cantidad de veces necesarias en nodos activos, y en caso de que un dato no lo esté solo el dueño de dicho dato iniciará un proceso de replicación, actualizará sus tablas de rutas y las compartirá con el resto de nodos.


#### 4. Replicacion:

El proceso de replicación está directamente vinculado al proceso de coordinación ya que solo lo inicia dicho proceso.

Este proceso recibe un dato y emplea las tablas de rutas y en los nodos activos para copiar el dato a nodos activos que no lo tengan mientras sea posible o hasta que alcance la cantidad de replicaciones necesarias para garantizar la tolerancias a fallos del sistema.

A medida que va copiando los datos actualiza las tablas de rutas para que luego el proceso de coordinación se encargue de compartirlas.


#### 5. Consistencia:

La cosistencia se basa en tratar de que todos los nodos que almacenan un dato tengan la misma versión del mismo.

Para ello cada nodo DataBase tiene un hilo que verifica cada cierto tiempo si es el dueño de un dato, y en caso de serlo verifica las versiones del dato en los nodos que lo tienen replicado, si todas las versiones son las mismas no hay problema, pero en caso de haber diferencias (solo pueden haber en los modelos ya que los trainings y los datasets no se cambian nunca) realiza un "merge".

El "merge" de los modelos se hace teniendo en cuenta la información de los batches por los que se quedaron entrenando y por los que se quedaron prediciendo y prioriza siempre la información del modelo que se quedó por el mayor batch; ya que los entrenamientos y las predicciones se realizan secuencialmente sobre los batches y todos los modelos del mismo tipo se inicializan usando la misma semilla.

Una vez hecho el "merge" le enváa el nuevo dato al resto de los que tenian las réplicas.

### Tolerancia a fallas de los DataBases:

Los DataBases presentan una tolerancia 2(por defecto) a fallos o desconexiones temporales de nodos, pero si se aumenta el factor de replicación también aumentaría el factor de tolerancia a fallos.

Para los DataBases una partición de la red no representa ningún problema ya que sería igual a que fallen muchos nodos a la vez, es decir siempre que no fallen todos los nodos que tienen un dato guardado el sistema seguirá funcionando de manera correcta.

Al desconectarse un nodo temporalmente, si en el tiempo que se desconectó, el líder de los datos que almacenaba el nodo caído se da cuenta inicia automaticamente el proceso de replicación y al volver nuevamente el nodo habrán más nodos con el dato de los que había antes de la caída del nodo, lo cual no es malo ya que a partir de ese momento dichos datos tendrán mas tolerancia a fallos, para si en el futuro el nodo (u otro nodo) presenta fallos ya no habria que replicar nada. 

En caso de que hubiera un cambio en los datos mientras el nodo no estaba disponible, al volver a unirse a la red el proceso de consistencia eventualmente se dará cuenta y el líder hara el merge de los datos y luego la replicación del mismo, con lo cual se lograría la consistencia nuevamente. Con esto se cubriría tambien el caso de que ocurra una partición de la red ya que es análogo a que muchos nodos se desconecten temporalmente a la vez.

Cuando un nodo se agrega nuevo a la red, eventualmente le tocará replicar un dato o le tocará agregar un nuevo dato al sistema, pero mientras tanto sirve como router para las peticiones de los Workers, por lo que tampoco presenta ningún problema para el sistema.

## Comunicación

La comunicación entre todos los componentes del sistema (workers, nodos DataBase y clientes) se realiza exclusivamente mediante peticiones HTTP. 

Esto debido a que cada nodo (excepto los clientes) expone una API REST que permite intercambiar datos y coordinar las operaciones distribuidas sin necesidad de mantener conexiones persistentes. Y tambien gracias a que todos los procesos del sistema son independientes y no necesitan mantener una conexión.

Gracias a este enfoque, los nodos se mantienen desacoplados, pueden operar de forma independiente y es posible reemplazarlos, escalarlos o reiniciarlos sin afectar el funcionamiento global del sistema.

## Nombrado y Localización 

Para que cada nodo del sistema pueda localizar a otros nodos activos dentro de la red, se utilizan dos mecanismos de descubrimiento complementarios. Esto garantiza que, si uno falla, el otro pueda actuar como respaldo, manteniendo la comunicación y la operatividad del sistema.

### Domain Name Service (DNS)

El sistema se ejecuta sobre una red de Docker utilizando Docker Swarm, que proporciona un DNS interno capaz de resolver automáticamente los nombres de servicio de cada nodo.

Gracias a este DNS:

- Los nodos no necesitan conocer direcciones IP específicas.
- Un worker puede comunicarse con los nodos DataBase simplemente usando su dominio o nombre de servicio.
- Docker gestiona la resolución y el balanceo entre instancias activas.
- Los fallos de nodos individuales quedan aislados del resto del sistema.
- Este mecanismo es la forma principal de descubrimiento dentro de la red.

### IP Cache

Como mecanismo de respaldo, cada nodo mantiene una caché local de direcciones IP correspondientes a otros nodos conocidos. Está técnica funciona incluso si la resolución por DNS falla.

### El funcionamiento es el siguiente:

**IPs fijas al iniciar el nodo:**

Cuando un nodo se levanta, carga en su caché un conjunto de IPs fijas preconfiguradas.
Esto le permite tener acceso inmediato a nodos potenciales, incluso si el DNS presenta fallos desde el inicio.

**Actualización dinámica:**

A medida que el nodo interactúa con otros, añade nuevas IPs a la caché cuando detecta nodos que responden correctamente.

**Uso como respaldo:**

Si el DNS no funciona o la red presenta inconsistencias, el nodo reutiliza las IPs guardadas en la caché para intentar conectarse a otros nodos activos.

**Depuración automática:**

Si una IP en la caché deja de ser válida, se descarta al detectar fallos consecutivos.

Gracias a está combinación, el sistema puede arrancar, operar y recuperarse incluso en momentos donde la infraestructura de red esté parcialmente degradada.

## Distribución de servicios en ambas redes de docker

Dicha arquitectura permite que el sistema funcione en una red de docker siempre que exista al menos un nodo de cada tipo.
