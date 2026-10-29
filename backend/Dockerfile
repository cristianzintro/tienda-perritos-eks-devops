FROM node:25-alpine
WORKDIR /app
COPY package.json .
COPY server.js .
RUN npm install --production
EXPOSE 3001
CMD ["node","server.js"]