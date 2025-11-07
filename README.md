## Auth Microservice

A NestJS microservice that provides user authentication (register/login) for the launcher project. It uses Prisma with MongoDB as the persistence layer and NATS for inter-service messaging.

## Features

- User registration and login (email/password)
- JWT-based authentication (token signing/verification)
- Prisma ORM for MongoDB access (generated client in `generated/prisma`)
- NATS transport for microservice messaging
- Input validation with class-validator

## Project layout

```
auth-ms/
├── prisma/                 # Prisma schema
├── generated/prisma/       # Prisma client (generated)
├── src/
│   ├── auth/               # auth controller, service, dto
│   ├── config/             # env/config helpers
│   └── transports/         # nats transport
├── .env                    # local env file
├── dockerfile
├── docker-compose (root)   # launched at repo root
└── package.json
```

## Important environment variables

Copy `.env.template` to `.env` and update values.

Minimum variables used by this service (example):

```env
PORT=3004
NATS_SERVERS=nats://localhost:4222
DATABASE_URL=mongodb://auth-db:27017/AuthDB?directConnection=true
JWT_SECRET=your_jwt_secret_here
```

## Prisma setup

This project uses Prisma with a custom generator output into `generated/prisma`.

If you added/changed `prisma/schema.prisma`, run:

```powershell
npx prisma generate
```

## Running locally (development)

1. Install dependencies:

```powershell
npm install
```

2. Start required infra:
- NATS (locally or via docker)
- MongoDB (see Docker recipe below or use Atlas)

3. Start the service in watch mode:

```powershell
npm run start:dev
```

The service listens on `PORT` and connects to NATS via `NATS_SERVERS`.

## Running with Docker (recommended for local integration)

This repo provides a root-level `docker-compose.yml` that defines `auth-db` (Mongo), `nats-server` and microservices. The root compose file lives at the repo root; start services from there:

```powershell
# stop & remove volumes if you want a clean start
docker-compose down -v
# start the db first so replica set initialization can complete
docker-compose up -d auth-db
# follow logs and wait until the replica set is initialized
docker-compose logs -f auth-db
# start the rest
docker-compose up -d auth-ms nats-server
```

Notes:
- If you use the single-node replica set approach, the compose file starts Mongo with `--replSet` and a healthcheck that initializes `rs0`.
- For development only you can run Mongo without authentication and without keyfile (easier), but do not use that in production.

## API / Message patterns

This service primarily exposes functionality through NestJS message patterns (NATS). It also contains controller code for local testing.

- NATS (Message Patterns):
	- `auth.register` (or similar) — register user (check `auth.controller.ts` / `auth.service.ts` for exact patterns)
	- `auth.login` — login user and return JWT

If you need HTTP endpoints (client-gateway proxies requests), check `client-gateway` which calls this service via NATS.

## Security & JWT

- JWT secret lives in `JWT_SECRET` (or similarly named variable in your `.env`).
- Tokens should be signed and verified using the same secret.
