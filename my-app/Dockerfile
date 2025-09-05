# -------- Stage 1: Build React app --------
FROM node:18-alpine AS build
WORKDIR /app

# install deps
COPY package*.json ./
RUN npm ci

# build app
COPY . .
RUN npm run build

# -------- Stage 2: Serve with Nginx --------
FROM nginx:alpine
WORKDIR /usr/share/nginx/html

# clean default nginx html
RUN rm -rf ./*

# copy build output from previous stage
COPY --from=build /app/build .

# expose port
EXPOSE 80

# default nginx startup
CMD ["nginx", "-g", "daemon off;"]