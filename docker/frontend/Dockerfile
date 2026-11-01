# Use an official Node.js runtime as a parent image
FROM node:24-alpine AS builder
# Set the working directory in the container
WORKDIR /app
#copy package.json and package-lock.json to the working directory
COPY package*.json ./
# Install dependencies
RUN npm ci
# Copy the rest of the application code to the working directory
COPY . .
# API URL for Docker deployment
ARG VITE_API_URL=/api
ENV VITE_API_URL=${VITE_API_URL}
# Build the application
RUN npm run build

# Use an official Nginx image to serve the built application

FROM nginx:alpine
# Set the working directory in the container
WORKDIR /app
# Copy the Nginx configuration file
COPY nginx.conf /etc/nginx/conf.d/default.conf
# Copy the built application files
COPY --from=builder /app/dist /usr/share/nginx/html
# Expose the port
EXPOSE 80
# Start Nginx
CMD ["nginx", "-g", "daemon off;"]