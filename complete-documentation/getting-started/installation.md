# Installation

Get started with RamAPI by installing it in your project. This guide covers prerequisites, installation steps, and verifying your setup.

## Prerequisites

Before installing RamAPI, ensure you have the following:

### Node.js

RamAPI requires **Node.js 20.0.0 or higher**.

Check your Node.js version:

```bash
node --version
```

If you need to install or update Node.js:
- Download from [nodejs.org](https://nodejs.org/)
- Or use a version manager like [nvm](https://github.com/nvm-sh/nvm)

```bash
# Using nvm
nvm install 20
nvm use 20
```

### Package Manager

RamAPI works with any Node.js package manager:
- **npm** (comes with Node.js)
- **yarn**
- **pnpm**
- **bun**

## Installation

### Create a New Project

If starting from scratch, create a new directory:

```bash
mkdir my-ramapi-project
cd my-ramapi-project
npm init -y
```

### Install RamAPI

Install RamAPI and its peer dependency Zod:

```bash
npm install ramapi zod
```

Or with your preferred package manager:

```bash
# Yarn
yarn add ramapi zod

# pnpm
pnpm add ramapi zod

# Bun
bun add ramapi zod
```

### Install TypeScript (Recommended)

RamAPI is built with TypeScript and provides full type safety. Install TypeScript and type definitions:

```bash
npm install -D typescript tsx @types/node
```

Initialize TypeScript configuration:

```bash
npx tsc --init
```

Update your `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### Update package.json

Add the following to your `package.json`:

```json
{
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

## Optional Dependencies

### uWebSockets.js (High Performance)

For 2-3x better performance, install uWebSockets.js:

```bash
npm install uWebSockets.js@uNetworking/uWebSockets.js#v20.48.0
```

RamAPI automatically detects and uses uWebSockets when available, with intelligent fallback to Node.js HTTP.

### OpenTelemetry (Observability)

For distributed tracing and observability features:

```bash
npm install @opentelemetry/api @opentelemetry/sdk-trace-node @opentelemetry/exporter-trace-otlp-http
```

These are already included in RamAPI's dependencies, but you may want to install additional exporters.

## Create Your First App

Create a `src/index.ts` file:

```typescript
import { createApp } from 'ramapi';

const app = createApp();

app.get('/', async (ctx) => {
  ctx.json({ message: 'Hello, RamAPI!' });
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

## Run Your App

### Development Mode

Run with hot reload:

```bash
npm run dev
```

Your server is now running at `http://localhost:3000`

### Test Your App

Open your browser or use curl:

```bash
curl http://localhost:3000
```

Expected response:

```json
{
  "message": "Hello, RamAPI!"
}
```

## Production Build

Build your TypeScript code:

```bash
npm run build
```

Run the compiled code:

```bash
npm start
```

## Verification

Verify your installation is working correctly:

### Check Version

Create a simple version check:

```typescript
import { createApp } from 'ramapi';

const app = createApp();

app.get('/health', async (ctx) => {
  ctx.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    ramapi: 'installed'
  });
});

app.listen(3000);
```

Test it:

```bash
curl http://localhost:3000/health
```

## Directory Structure

After installation, your project should look like this:

```
my-ramapi-project/
├── node_modules/
├── src/
│   └── index.ts
├── package.json
├── package-lock.json
└── tsconfig.json
```

## Common Installation Issues

### Issue: Node.js Version Too Old

**Error**: `RamAPI requires Node.js >= 20.0.0`

**Solution**: Upgrade Node.js to version 20 or higher.

### Issue: TypeScript Errors

**Error**: TypeScript compilation errors

**Solution**: Ensure your `tsconfig.json` has `"moduleResolution": "node"` and `"esModuleInterop": true`

### Issue: Module Not Found

**Error**: `Cannot find module 'ramapi'`

**Solution**:
1. Verify installation: `npm list ramapi`
2. Reinstall: `rm -rf node_modules package-lock.json && npm install`
3. Check `package.json` includes `"type": "module"`

### Issue: uWebSockets.js Installation Failed

**Error**: Build errors when installing uWebSockets.js

**Solution**: This is optional. RamAPI works perfectly with Node.js HTTP. Skip uWebSockets if build fails on your platform.

## Next Steps

Now that RamAPI is installed, you can:

1. [Follow the Quick Start guide](quick-start.md) to build your first API
2. [Learn about project structure](project-structure.md) for larger applications
3. [Explore core concepts](../core-concepts/context-and-handlers.md) to understand RamAPI fundamentals

## Additional Resources

- [GitHub Repository](https://github.com/shanmukhram/RamAPI)
- [NPM Package](https://www.npmjs.com/package/ramapi)
- [Example Applications](../examples/todo-api.md)
- [Troubleshooting Guide](../guides/troubleshooting.md)

## Platform-Specific Notes

### Windows

RamAPI works on Windows, but uWebSockets.js may require build tools:

```bash
npm install --global windows-build-tools
```

### macOS

macOS users with Apple Silicon (M1/M2) may need Rosetta for some dependencies:

```bash
softwareupdate --install-rosetta
```

### Linux

Most Linux distributions work out of the box. Ensure you have build essentials:

```bash
# Ubuntu/Debian
sudo apt-get install build-essential

# Fedora/RHEL
sudo dnf install gcc-c++ make
```

## Docker Installation

If you prefer Docker, create a `Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

Build and run:

```bash
docker build -t my-ramapi-app .
docker run -p 3000:3000 my-ramapi-app
```

---

**Ready to build your first API?** Continue to the [Quick Start Guide](quick-start.md)
