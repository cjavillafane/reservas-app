# Etapa 1: construccion
FROM node:20-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci

COPY . .

RUN npm run build


# Etapa 2: imagen final con nginx sin privilegios
FROM nginxinc/nginx-unprivileged:1.27-alpine

COPY --from=build /app/dist /usr/share/nginx/html

COPY nginx/templates/default.conf.template /etc/nginx/templates/default.conf.template

ENV API_HOST=reservas-api \
    API_PORT=3000 \
    NGINX_ENVSUBST_FILTER=^API_

EXPOSE 8080
