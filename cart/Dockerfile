FROM node:14-alpine

WORKDIR /opt/server

RUN addgroup -S app && \
    adduser -S app -G app

<<<<<<< HEAD
COPY package*.json ./
=======
COPY package*.json .\
>>>>>>> 40e358d (Update CI pipelines and infra setup)

RUN npm ci --only=production

COPY --chown=app:app . .

RUN chown -R app:app /opt/server/node_modules

USER app

EXPOSE 8080

CMD ["node","server.js"]
