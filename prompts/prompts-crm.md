# Prompts utilizados para generar el pipeline

Este documento recoge los prompts utilizados para generar cada paso del pipeline de GitHub Actions.

---

## 1. Tests de backend

**Prompt:**

```
Crea un job de GitHub Actions que ejecute los tests del backend de un proyecto Node.js/TypeScript.

Requisitos:
- El backend está en la carpeta backend/
- Usa npm ci para instalar dependencias
- El comando de tests es npm run test
- Usa Node.js 20
- Configura cache de npm para acelerar la instalación
- El job debe llamarse "Test Backend"
```

**Resultado:** Job `test` que instala dependencias con `npm ci` y ejecuta `npm run test` en el directorio backend.

---

## 2. Generación del build del backend

**Prompt:**

```
Crea un job de GitHub Actions que genere el build del backend de un proyecto Node.js/TypeScript con Prisma.

Requisitos:
- El job debe depender del job de tests (needs: test)
- El backend está en backend/
- Hay que ejecutar npx prisma generate antes del build
- El comando de build es npm run build
- Sube los artefactos del build (dist, node_modules, package.json, package-lock.json, prisma) para usarlos en el despliegue
- Usa actions/upload-artifact para subir los artefactos
```

**Resultado:** Job `build` que genera el cliente Prisma, compila el proyecto y sube los artefactos necesarios para el despliegue.

---

## 3. Despliegue del backend en EC2

**Prompt:**

```
Crea un job de GitHub Actions que despliegue el backend en un servidor EC2 usando SSH.

Requisitos:
- El job debe depender del job de build (needs: build)
- Descargar los artefactos del build anterior
- Usar SSH para conectarse al EC2 con los siguientes secrets: EC2_HOST, EC2_USER, EC2_SSH_KEY
- Copiar los archivos del backend al servidor en ~/backend-app
- Ejecutar npm install --production en el servidor
- Iniciar la aplicación con npm run start:prod (o pm2 restart si está configurado)
- Si los secrets no están configurados, saltar el despliegue sin fallar el pipeline
- Usar rsync para copiar los archivos de forma eficiente
```

**Resultado:** Job `deploy` que descarga artefactos, configura SSH, sincroniza archivos con rsync y ejecuta la aplicación en EC2.

---

## Trigger del pipeline

**Prompt:**

```
Configura el trigger del pipeline para que se dispare con un push a una rama que tenga un Pull Request abierto.

Usa el evento pull_request de GitHub Actions, que se activa cuando:
- Se abre un Pull Request
- Se hace push a una rama que tiene un Pull Request abierto
```

**Resultado:** `on: pull_request` con branches `[main, master]`.

---

# Iteraciones y corrección de fallos

Documentación de los errores encontrados durante el despliegue y las soluciones aplicadas.

## 1. "No event triggers defined in `on`"

**Error:** Anotación en GitHub Actions indicando que no hay triggers definidos.

**Causa:** El archivo `ci.yml` estaba vacío. GitHub parsea todos los `.yml` en `.github/workflows/` y un archivo sin `on:` válido provoca este error.

**Solución:** Añadir un workflow mínimo con `on: push` en `ci.yml`, o eliminar el archivo si no se usa.

---

## 2. `tsc: command not found` en EC2

**Error:** Al ejecutar `npm run start:prod` en el servidor, falla con `sh: tsc: command not found`.

**Causa:** `start:prod` ejecuta `npm run build && npm start`. El build usa `tsc` (TypeScript), que es devDependency. Con `npm install --omit=dev` no se instalan devDependencies.

**Solución:** No compilar en EC2. El `dist/` ya viene precompilado del pipeline. Usar `pm2 start dist/index.js --name backend` en lugar de `npm run start:prod`.

---

## 3. "Process or Namespace backend not found"

**Error:** PM2 falla al ejecutar `pm2 restart backend`.

**Causa:** En el primer deploy no existe un proceso PM2 llamado "backend".

**Solución:** Usar `pm2 restart backend 2>/dev/null || pm2 start dist/index.js --name backend` para iniciar si no existe y reiniciar si ya está en ejecución.

---

## 4. "Script not found: dist/index.js"

**Error:** PM2 no encuentra `~/backend-app/dist/index.js` en el EC2.

**Causa:** La estructura del artifact al subir `backend/` completo no conservaba bien la carpeta `dist/` o el artefacto se extraía en una ubicación distinta.

**Solución:** Crear un paquete de deploy explícito en el job de build:
- Crear carpeta `deploy/` con `dist/`, `package.json`, `package-lock.json`, `prisma/`
- Subir `deploy/` como artifact
- Hacer rsync de `deploy/` a EC2

---

## 5. "rsync: change_dir deploy failed: No such file or directory"

**Error:** `rsync` falla porque la carpeta `deploy/` no existe en el workspace del job de deploy.

**Causa:** Según cómo se extrae el artifact, la estructura puede variar: `deploy/` en raíz, `backend-build/deploy/`, o `dist/` y `package.json` sueltos en la raíz.

**Solución:** Añadir el paso "Prepare deploy folder" que normaliza la estructura:
- Si existe `deploy/` → usarlo
- Si existe `backend-build/deploy/` → copiar a `deploy/`
- Si existe `dist/` en raíz → crear `deploy/` y mover los archivos necesarios

---

## 6. Versión de Node en EC2 (Node 16)

**Error:** Warnings `EBADENGINE` por paquetes que requieren Node 18+ o 20+.

**Causa:** Amazon Linux 2023 puede traer Node 16 por defecto.

**Solución:** Añadir en el deploy: `nvm use 20` si nvm está instalado. Documentar que en EC2 debe estar Node 20 (por ejemplo con nvm o NodeSource).

---

## 7. `npm install --production` deprecado

**Error:** `npm WARN config production Use --omit=dev instead`.

**Solución:** Sustituir `--production` por `--omit=dev`.

---

## 8. Tests ejecutándose desde `dist/`

**Error:** 4 tests fallaban al ejecutarse desde los archivos compilados en `dist/`.

**Causa:** Jest ejecutaba tanto los tests en `src/` como en `dist/`, y los compilados se comportaban distinto.

**Solución:** Añadir `testPathIgnorePatterns: ['/node_modules/', '/dist/']` en `backend/jest.config.js`.
