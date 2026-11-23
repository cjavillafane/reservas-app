# Informe de funcionamiento - Variante B

En este informe dejo las pruebas que realicé con la aplicación de reservas utilizando Kubernetes y Helm. La idea fue comprobar que la aplicación funcionara correctamente, que pudiera aumentar sus réplicas cuando hubiera mucha carga, que Helm permitiera volver a una versión anterior y que no se generaran reservas duplicadas.

## 1. Funcionamiento mediante Ingress

Primero revisé que los pods estuvieran funcionando:

```kubectl get pods -n reservas-app```

El resultado mostró los pods de la API, el frontend y la base de datos en estado `Running`, todos con `1/1` contenedores listos.

También comprobé los servicios y el Ingress:

```kubectl get svc,ingress -n reservas-app```

El Ingress estaba configurado con el dominio `reservas.local`.

Después probé el acceso a la aplicación:

```curl -i http://reservas.local```

**Resultado:**

```
HTTP/1.1 200 OK
Server: nginx/1.31.6
Content-Type: text/html
```

La aplicación también abrió correctamente desde Firefox utilizando `http://reservas.local`.

Desde el navegador pude ver las salas, los horarios y las reservas existentes. También se realizó una reserva para comprobar que el frontend se comunicara con la API.

Para verificar los datos guardados ejecuté:

```curl -i http://reservas.local/api/reservas```

**Resultado:**

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-Powered-By: Express
```

La API devolvió las reservas en formato JSON. Entre ellas aparecía una reserva creada desde la aplicación:

```
Sala: Laboratorio 1
Fecha: 2026-11-19
Franja: 3
Responsable: Curso Docker
Motivo: Clase Teorica
```

Con esta prueba comprobé que el Ingress permite acceder al frontend, que las solicitudes a la API funcionan y que las reservas se guardan y se pueden consultar.

---

## 2. Prueba del HPA

Para comprobar el autoescalado de la API primero consulté el estado del HPA:

```kubectl get hpa -n reservas-app```

Al comenzar, la API tenía 2 réplicas y el consumo de CPU era del 1%, con un objetivo configurado del 60%.

Después dejé el HPA en observación:

```kubectl get hpa -n reservas-app -w```

En otra terminal generé carga con el siguiente comando:

```kubectl run carga-hpa \
  -n reservas-app \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c 'while true; do wget -q -O /dev/null http://reservas-api:3000/api/reservas; done'
```

Mientras se ejecutaban las solicitudes, el consumo de CPU comenzó a aumentar.

**Resultado observado:**

| CPU | Réplicas |
|---|---|
| 1% | 2 |
| 81% | 2 |
| 164% | 3 |
| 112% | 6 |
| 83% | 6 |
| 70% | 6 |
| 63% | 6 |
| 57% | 6 |
| 47% | 6 |
| 5% | 6 |

Se puede ver que la API comenzó con 2 réplicas y llegó hasta 6, que era el máximo configurado.

Después detuve la carga eliminando el pod que había creado:

```kubectl delete pod carga-hpa -n reservas-app```

También comprobé las réplicas con:

```kubectl get deployment reservas-api -n reservas-app```

En ese momento el resultado fue:

```
NAME           READY   UP-TO-DATE   AVAILABLE
reservas-api   6/6     6            6
```

Al principio las réplicas no bajaron inmediatamente, porque el HPA espera un tiempo antes de reducirlas.

Más adelante volví a consultar:

```kubectl get hpa -n reservas-app```

Y el resultado fue:

```
NAME               REFERENCE                 TARGETS     MINPODS   MAXPODS   REPLICAS
reservas-api-hpa   Deployment/reservas-api   cpu: 1%/60% 2         6         2
```

Con esta prueba comprobé que el HPA aumenta las réplicas cuando sube el consumo de CPU y después las vuelve a reducir cuando termina la carga.



---

## 3. Actualización y rollback con Helm

En el TP8 trabajamos con dos versiones de la API.

Primero se realizó la actualización de la imagen `v1` a `v2` utilizando Helm y después se volvió a una versión anterior mediante rollback.

Para revisar las revisiones ejecuté:

```helm history reservas-app -n reservas-app```

**Resultado:**

| Revisión | Estado | Versión | Descripción |
|---|---|---|---|
| 1 | failed | v1 | Error inicial del PVC |
| 2 | failed | v1 | Tiempo de espera agotado |
| 3 | superseded | v1 | Tiempo de espera agotado |
| 4 | superseded | v1 | Upgrade complete |
| 5 | superseded | v1 | Upgrade complete |
| 6 | deployed | v1 | Rollback to 1 |

En el historial se observan algunos errores que aparecieron durante las primeras instalaciones, principalmente por el PVC y por tiempos de espera.

Después se pudieron completar las actualizaciones.

Para comprobar que la revisión  había utilizado la imagen `v2`, ejecuté:

```helm get manifest reservas-app -n reservas-app --revision 5 \
  | grep 'image:.*reservas-api'
```

**Resultado:**

```
image: 'cristianvillafane/reservas-api:v2'
```

Luego comprobé la imagen que estaba utilizando la API después del rollback:

```
kubectl get deployment reservas-api -n reservas-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

**Resultado:**

```
cristianvillafane/reservas-api:v1
```

Finalmente consulté el estado de Helm:

```
helm status reservas-app -n reservas-app
```

**Resultado:**

```
NAME: reservas-app
NAMESPACE: reservas-app
STATUS: deployed
REVISION: 6
```

Con estas comprobaciones pude verificar que la revisión 5 tenía la imagen `v2` y que después del rollback la aplicación volvió a utilizar `v1`.



---

## 4. Prueba de los valores de Helm

Para comprobar que los valores configurados en Helm realmente se aplicaban, rendericé la plantilla del HPA utilizando dos porcentajes diferentes de CPU.

Primero probé con un objetivo del 60%:

```
helm template reservas-app ./helm/reservas-app \
  -n reservas-app \
  -f ~/Curso_Docker/privado/values-tp8.yaml \
  --show-only templates/hpa-api.yaml \
  --set hpa.targetCPUUtilization=60
```

**Resultado:**

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: reservas-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: reservas-api
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

Después ejecuté el mismo comando, pero cambiando el objetivo al 75%:

```
helm template reservas-app ./helm/reservas-app \
  -n reservas-app \
  -f ~/Curso_Docker/privado/values-tp8.yaml \
  --show-only templates/hpa-api.yaml \
  --set hpa.targetCPUUtilization=75
```

**Resultado:**

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: reservas-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: reservas-api
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
```

Al comparar los resultados se puede ver que `averageUtilization` cambió de 60 a 75, mientras que el resto de la configuración se mantuvo igual.

Con esto comprobé que el porcentaje se obtiene desde los valores de Helm y no está escrito de forma fija en la plantilla.



---

## 5. Prueba de concurrencia del TP6

Para comprobar que no se generaran reservas duplicadas, envié 30 solicitudes al mismo tiempo intentando reservar la misma sala, fecha y franja horaria.

Los datos utilizados fueron:

```
Sala: Laboratorio 1
sala_id: 1
Fecha: 2026-12-15
Franja: 5
Cantidad de solicitudes: 30
```

Primero comprobé que la API estuviera funcionando:

```
curl -i http://reservas.local/api/reservas
```

La respuesta fue `HTTP/1.1 200 OK`.

Después ejecuté las solicitudes simultáneas desde la terminal:

```
FECHA="2026-12-15"

for i in $(seq 1 30); do
  (
    curl -s -o /dev/null -w "Solicitud $i: HTTP %{http_code}\n" \
      -X POST "http://reservas.local/api/reservas" \
      -H "Content-Type: application/json" \
      -d "{\"sala_id\":1,\"fecha\":\"$FECHA\",\"franja\":5,\"responsable\":\"Prueba $i\",\"motivo\":\"Concurrencia\"}"
  ) &
done

wait
```

**Resultado:**

Las solicitudes repetidas devolvieron el código:

```
HTTP 409
```

Esto indica que la aplicación rechazó los intentos de reservar una sala que ya tenía ocupado ese horario.

Después consulté nuevamente las reservas:

```
curl -s http://reservas.local/api/reservas | python3 -m json.tool
```

En la respuesta apareció una reserva con los siguientes datos:

```
{
  "id": 36,
  "sala_id": 1,
  "sala": "Laboratorio 1",
  "fecha": "2026-12-15",
  "franja": 5,
  "responsable": "Prueba 14",
  "motivo": "Concurrencia"
}
```

Finalmente ejecuté una consulta para contar cuántas reservas habían quedado guardadas con esos mismos datos:

```
curl -s http://reservas.local/api/reservas | \
python3 -c '
import json,sys
datos=json.load(sys.stdin)
reservas=[r for r in datos if int(r["sala_id"])==1 and r["fecha"]=="2026-12-15" and int(r["franja"])==5]
print("Reservas encontradas:",len(reservas))
'
```

**Resultado:**

```
Reservas encontradas: 1
```

Con esta prueba comprobé que, aunque se enviaron 30 solicitudes al mismo tiempo, solamente se guardó una reserva para esa combinación de sala, fecha y franja.

Las demás solicitudes fueron rechazadas con HTTP 409, evitando que se generaran reservas duplicadas.

---

## 6. Conclusión

Con las pruebas realizadas pude comprobar que la aplicación de reservas funciona correctamente en Kubernetes.

El Ingress permite acceder al frontend y a la API desde `reservas.local`, y las reservas se pueden guardar y consultar.

También comprobé que el HPA aumenta las réplicas de la API cuando hay mucha carga, pasando de 2 a 6, y después vuelve a reducirlas.

Con Helm pude verificar las revisiones, la actualización a `v2` y el regreso a `v1` mediante rollback. Además, comprobé que los valores de CPU del HPA se pueden modificar desde Helm.

Por último, la prueba de concurrencia permitió comprobar que no se guardan reservas duplicadas cuando llegan varias solicitudes al mismo tiempo.

De esta manera quedaron realizadas las pruebas de funcionamiento solicitadas para el proyecto final de la Variante B.
