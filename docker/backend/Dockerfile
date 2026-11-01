# Use an official Node.js runtime as a parent image
FROM node:24-alpine
# Set the working directory in the container
WORKDIR /app
#copy package.json and package-lock.json to the working directory
COPY package*.json ./
# Install dependencies
RUN npm ci --omit=dev
# Copy the rest of the application code to the working directory
COPY . .
# Expose the port the app runs on
EXPOSE 5000
# Start the application
CMD ["npm", "start"]