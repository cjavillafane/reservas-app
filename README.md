# Reservas App

Aplicación para la reserva de salas y laboratorios por franja horaria.

Proyecto integrador de la asignatura Contenedores, Docker y Orquestación con Kubernetes.

## Arquitectura

    navegador ──► reservas-frontend ──────► reservas-api ──────► reservas-db
                  React + Vite             Node 20 + Express     MySQL 8.4
                  nginx :8080              :3000                 :3306
                  (host 3000)              (host 3001, solo      (sin puerto
                                            depuración)           publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
| --- | --- | --- | --- |
| `reservas-frontend` | React + Vite servido por nginx | 8080 | 3000 |
| `reservas-api` | Node 20 + Express | 3000 | 3001 (solo depuración) |
| `reservas-db` | MySQL 8.4 | 3306 | — |

El frontend es el único punto de entrada: nadie le habla a la base
directamente, y a la API le habla el frontend.

## TP2 - Construcción de imágenes Docker

### Comparación de imágenes

| Imagen | Imagen base | Tamaño | Reducción respecto de API ingenua |
| --- | --- | ---: | ---: |
| API ingenua | `node:20` | 402 MB | — |
| API definitiva v1 | `node:20-alpine` | 50 MB | 87,56 % |
| Frontend v1 | `nginxinc/nginx-unprivileged:1.27-alpine` | 21 MB | 94,78 % |

### Decisiones adoptadas

- **Construcción multi-etapa:** permite separar las herramientas necesarias durante la construcción de los componentes requeridos durante la ejecución.
- **`node:20-alpine`:** utiliza una imagen base mínima, reduciendo considerablemente el tamaño respecto de `node:20`.
- **`npm ci --omit=dev`:** instala únicamente las dependencias de producción del backend y respeta exactamente las versiones definidas en `package-lock.json`.
- **Orden de `COPY`:** copiar primero `package.json` y `package-lock.json` permite reutilizar la capa de dependencias mientras dichos archivos no cambien.
- **Usuario no privilegiado:** la API se ejecuta utilizando el usuario `node` provisto por la imagen oficial, evitando ejecutar el proceso como `root`.
- **Frontend multi-etapa:** Node.js se utiliza únicamente para generar `dist/`. La imagen final contiene nginx y los archivos compilados del frontend, pero no Node.js, npm, `node_modules` ni el código fuente.
- **`.dockerignore`:** evita enviar archivos innecesarios o sensibles al contexto de construcción.
