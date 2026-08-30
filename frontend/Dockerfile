FROM node:18-alpine AS build
WORKDIR /app
ARG ENV_FILE=.env.dev
COPY package*.json ./
RUN npm install
COPY . .
COPY ${ENV_FILE} .env
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
