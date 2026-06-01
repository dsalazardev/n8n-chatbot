
## Cómo hacer fork y trabajar con upstream

### 1. Hacer fork del repositorio

Ve a `https://github.com/YanethM/n8n-chatbot` y haz clic en el botón **Fork** (esquina superior derecha). Esto crea una copia del repositorio en tu cuenta de GitHub.

### 2. Clonar tu fork localmente

```bash
git clone https://github.com/TU-USUARIO/n8n-chatbot.git
cd n8n-chatbot
```

### 3. Agregar el repositorio original como upstream

El **upstream** es el repositorio original desde el cual hiciste el fork. Configurarlo te permite sincronizar cambios futuros del proyecto original.

```bash
git remote add upstream https://github.com/YanethM/n8n-chatbot.git
```

Verifica que los remotos quedaron bien configurados:

```bash
git remote -v
```

Deberías ver:

```
origin    https://github.com/TU-USUARIO/n8n-chatbot.git (fetch)
origin    https://github.com/TU-USUARIO/n8n-chatbot.git (push)
upstream  https://github.com/YanethM/n8n-chatbot.git (fetch)
upstream  https://github.com/YanethM/n8n-chatbot.git (push)
```

### 4. Configurar el proyecto

Copia el archivo de ejemplo y agrega tu URL de webhook:

```bash
cp config.example.js config.js
```

Abre `config.js` y reemplaza la URL con la de tu propio webhook de n8n:

```js
const CONFIG = {
    N8N_WEBHOOK_URL: 'https://TU-INSTANCIA.app.n8n.cloud/webhook/TU-UUID'
};
```

> `config.js` está en `.gitignore` para que no subas tu URL privada al repositorio.

### 5. Crear una rama para tus cambios

Nunca trabajes directamente en `main`. Crea una rama con un nombre descriptivo:

```bash
git checkout -b feature/nombre-de-tu-feature
```

### 6. Subir tu rama a tu fork

```bash
git push origin feature/nombre-de-tu-feature
```

---

## Ramas disponibles

| Rama                  | Descripcion                                      |
| --------------------- | ------------------------------------------------ |
| `main`                | Formulario base de reporte de incidentes         |
| `feature/chat-widget` | Widget de chatbot flotante en esquina inferior   |

---

## Sincronizar cambios del repositorio original

Cuando el repositorio original tenga actualizaciones, sincronizalas a tu fork así:

```bash
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## Estructura del proyecto

```
n8n-chatbot/
├── index.html         # Interfaz principal
├── config.js          # URL del webhook (no se sube al repo)
└── config.example.js  # Plantilla de configuracion
```
