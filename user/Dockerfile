FROM node:14-alpine

WORKDIR /opt/server

RUN addgroup -S app && \
    adduser -S app -G app

COPY package*.json ./

RUN npm ci --only=production 

COPY --chown=app:app . .

RUN chown -R app:app /opt/server/node_modules

USER app

EXPOSE 8080

CMD ["node", "server.js"]