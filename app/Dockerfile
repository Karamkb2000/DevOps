FROM node:20-alpine AS builder


# Why using working directory?
# without using workdir, files will go to the / (root). risk

WORKDIR /app

# Why copy package.json first?
# because of caching. if we copy all files, then npm install will run every time.

COPY package*.json ./

# Why using npm ci instead of npm install?

RUN npm ci --omit=dev


# Why copy src after npm install?
# because of caching. if we copy all files, then npm install will run every time.
COPY src/ ./src/


# using new stage so we can have smaller image size 
# we will reduce the image size by discarding  npm cache build tools.
FROM node:20-alpine AS production

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/src         ./src
COPY --from=builder /app/package.json ./

USER node



EXPOSE 3000




HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1




CMD ["node", "src/index.js"]
