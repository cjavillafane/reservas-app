# Arquitectura - Proyecto Final Variante B

## 1. Preguntas obligatorias

### a) ¿Por qué el frontend se configura con API_HOST en tiempo de ejecución y no con VITE_API_URL en tiempo de construcción? Si fuera al revés, ¿qué habría que rehacer para pasar de desarrollo a producción?

Utilicé `API_HOST` porque me permite cambiar a qué API se conecta el frontend sin tener que volver a construir la imagen Docker.

Con `VITE_API_URL`, la dirección queda guardada cuando se compila el frontend con Vite. Entonces, si quiero pasar de desarrollo a producción y cambia la dirección de la API, tendría que volver a compilar el frontend y generar otra imagen.

En cambio, con `API_HOST` puedo utilizar la misma imagen y cambiar la configuración cuando arranca el contenedor.

### b) ¿Por qué esta arquitectura no necesita CORS? ¿Qué habría que agregar si la aplicación de página única se comunicara directamente con la API?

No se necesita configurar CORS porque el navegador se comunica con el frontend y nginx se encarga de enviar las peticiones `/api` al backend.

Para el navegador, todo se hace desde la misma dirección.

Si React se conectara directamente a la API usando otra dirección o puerto, ahí sí tendría que configurar CORS en Express para permitir las peticiones desde el frontend.

### c) ¿Por qué hay un solo autoescalador entre los doce objetos? ¿Qué ocurriría si se le asignara uno a reservas-db? ¿Y por qué tampoco lo tiene el frontend?

El único HPA que configuré es para la API, porque es la parte que procesa las peticiones y puede necesitar más réplicas cuando aumenta la carga.

En MySQL no puedo agregar réplicas de esa manera, porque cada instancia necesita manejar correctamente los datos. Si simplemente escalo el Deployment, no significa que tenga una base de datos replicada. Incluso podría tener problemas si varias instancias intentan utilizar los mismos archivos.

Al frontend tampoco le configuré un HPA porque solamente sirve los archivos de React mediante nginx. Para este proyecto dejé una cantidad fija de réplicas, ya que no necesitaba escalarlo automáticamente.

## 2. Pregunta de la Variante B

### ¿Qué deja de estar garantizado al pasar de una réplica de la API a tres?

Cuando tengo una sola API, las peticiones llegan al mismo proceso. Al pasar a tres réplicas, varias peticiones pueden atenderse al mismo tiempo en procesos distintos.

El problema aparece cuando dos personas intentan reservar la misma sala, el mismo día y en el mismo horario.

Las dos peticiones podrían consultar la base de datos antes de que alguna guarde la reserva. En ese momento, ambas pueden encontrar el horario libre.

Por eso, entendí que consultar primero si está disponible no alcanza para evitar que se duplique.

Lo que realmente evita el problema es la restricción `UNIQUE` de MySQL sobre `sala_id`, `fecha` y `franja`.

Aunque dos réplicas intenten guardar la misma reserva, MySQL permite una sola y rechaza la otra.


## 3. Problemas que tuve con MySQL

### Primer problema: las credenciales y la conexión con la base de datos

Durante los prácticos tuve que configurar las credenciales que utiliza MySQL, incluyendo la contraseña de root.

Después, cuando pasé la configuración a Kubernetes, también tuve problemas con las variables del Secret. En uno de los casos, la clave de la contraseña estaba mal escrita como `MYSQL_PASWORD`, cuando tenía que ser `MYSQL_PASSWORD`.

Esto provocaba un error `CreateContainerConfigError`, porque Kubernetes buscaba una clave que no existía.

Lo revisé consultando el estado y la descripción del Pod. La solución fue corregir el nombre de la variable para que coincidiera con la clave guardada en el Secret.

También tuve errores de conexión de la API a MySQL, como `ECONNREFUSED` al puerto 3306. Para revisarlo utilicé los logs de los Pods y comprobé si MySQL había iniciado correctamente.

### Segundo problema: el almacenamiento persistente de MySQL

En el TP7 trabajé con un PVC para que las reservas no se pierdan cuando se elimina el Pod de MySQL.

Primero escribí un archivo dentro del volumen usando BusyBox. Después intenté montar ese volumen directamente en `/var/lib/mysql`.

MySQL no podía inicializarse porque el directorio ya contenía archivos.

Revisé los logs y la solución fue agregar `subPath: mysql` en el montaje del volumen.

De esta manera, MySQL utiliza una carpeta propia dentro del PVC y no tiene problemas con otros archivos.

Más adelante también tuve inconvenientes con el PVC en estado `Pending`, por lo que tuve que revisar el almacenamiento disponible en el clúster y si el volumen podía asignarse al Pod.

## 4. ¿Qué faltaría para usar este proyecto en producción?

El proyecto sirve para lo que me pidieron en el curso, pero considero que para utilizarlo realmente en producción todavía faltarían varias cosas.

**Monitoreo:** necesitaría tener un sistema que me permita ver el estado de los servicios, los errores y el consumo de recursos, y que me avise cuando algo falla.

**CI/CD:** sería conveniente automatizar las pruebas y los despliegues para no tener que hacer todo manualmente cada vez que modifico la aplicación.

**Secretos:** las contraseñas no deberían quedar en archivos del proyecto. Lo correcto sería utilizar un sistema de gestión de secretos con permisos y controles de acceso.

**Backups:** también necesitaría hacer copias de seguridad automáticas de MySQL y comprobar que se puedan restaurar.

**Base de datos:** en mi proyecto utilicé MySQL dentro de un Deployment con un PVC. Esto me permite conservar los datos si se recrea el Pod, pero no significa que tenga alta disponibilidad.

Para un sistema real, considero que sería mejor utilizar una base de datos administrada o una solución preparada para replicación, recuperación ante fallos y copias de seguridad.
