# Stage 1: build the React app
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG REACT_APP_API_URL=http://localhost:8080/api/todos
ENV REACT_APP_API_URL=$REACT_APP_API_URL
RUN npm run build

# Stage 2: small non-root web server
FROM nginxinc/nginx-unprivileged:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 8080
