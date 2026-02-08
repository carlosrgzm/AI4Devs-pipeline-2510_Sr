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
