# Arquitectura de Microfrontends - Documentación Técnica Completa

> 💡 **Tip**:  Usa `Ctrl+F` o `Cmd+F` para buscar términos específicos en este documento.
> 
> 📖 Para comenzar rápido, ve al [README.md](README.md)

## Tabla de Contenidos

- [1. Arquitectura General](#1-arquitectura-general)
- [2. Componentes del Sistema](#2-componentes-del-sistema)
  - [2.1 Container Root](#21-container-root)
  - [2.2 Microfrontends](#22-microfrontends)
- [3. Flujo de Ejecución](#3-flujo-de-ejecución)
  - [3.1 Build Time (Tiempo de Construcción)](#31-build-time-tiempo-de-construcción)
  - [3.2 Runtime (Tiempo de Ejecución)](#32-runtime-tiempo-de-ejecución)
- [4. Configuración Técnica](#4-configuración-técnica)
  - [4.1 NGINX Container Root](#41-nginx-container-root)
  - [4.2 NGINX Microfrontend](#42-nginx-microfrontend)
  - [4.3 Webpack Configuration](#43-webpack-configuration)
- [5. Flujo de NGINX](#5-flujo-de-nginx)
  - [5.1 Arquitectura de Comunicación](#51-arquitectura-de-comunicación)
  - [5.2 Secuencia Detallada de Peticiones](#52-secuencia-detallada-de-peticiones)
  - [5.3 Análisis del Redirect](#53-análisis-del-redirect)
  - [5.4 Diagrama de Flujo Completo](#54-diagrama-de-flujo-completo)
  - [5.5 Responsabilidades de cada NGINX](#55-responsabilidades-de-cada-nginx)
- [6. Ventajas y Beneficios](#6-ventajas-y-beneficios)
- [7. Mejores Prácticas](#7-mejores-prácticas)
  - [7.1 Organización del Código](#71-organización-del-código)
  - [7.2 Naming Conventions](#72-naming-conventions)
  - [7.3 Variables de Entorno](#73-variables-de-entorno)
  - [7.4 Monitoreo y Logs](#74-monitoreo-y-logs)
  - [7.5 Testing](#75-testing)

---

## 1. Arquitectura General

```
┌─────────────────────────────────────────────────────┐
│              CONTAINER ROOT                         │
│  ┌──────────────────┐    ┌──────────────────┐     │
│  │   Single-SPA     │    │      NGINX       │     │
│  │  (Orquestador)   │    │ (Proxy Reverso)  │     │
│  └──────────────────┘    └──────────────────┘     │
└─────────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┬──────────────┐
        │             │             │              │
        ▼             ▼             ▼              ▼
  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
  │ LOGIN   │   │DASHBOARD│   │PRODUCTS │   │ PROFILE │
  │   MF    │   │   MF    │   │   MF    │   │   MF    │
  └─────────┘   └─────────┘   └─────────┘   └─────────┘
        │             │             │              │
        └─────────────┴─────────────┴──────────────┘
                      │
                      ▼
              ┌───────────────┐
              │  LIB-SHARED   │
              │      MF       │
              └───────────────┘
```

## 2. Componentes del Sistema

### 2.1 Container Root

**Responsabilidades:**
- Orquestación de microfrontends mediante Single-SPA
- Gestión del routing principal
- Proxy reverso a servicios de microfrontends
- Gestión de dependencias compartidas

**Tecnologías:**
- **Single-SPA**: Framework de orquestación de microfrontends
- **SystemJS**: Carga dinámica de módulos
- **NGINX**: Servidor web y proxy reverso
- **TypeScript**: Desarrollo con tipado estático

### 2.2 Microfrontends

**Estructura común:**

```
microfrontend/
├── src/
│   ├── components/
│   ├── services/
│   └── [nombre-mf]. tsx
├── webpack.config.js
├── Dockerfile
├── nginx.conf
├── entrypoint.sh
└── deployment/
    ├── dev/
    ├── staging/
    └── production/
```

## 3. Flujo de Ejecución

### 3.1 Build Time (Tiempo de Construcción)

#### Paso 1: Build Multi-Ambiente

Genera 3 builds con hashes únicos por ambiente:

```bash
npm run build
```

**Resultado:**
```
build_development/my-mf.[hash-dev].js
build_staging/my-mf.[hash-stg].js
build_production/my-mf.[hash-prod].js
```

#### Paso 2: Construcción de Imagen Docker

```dockerfile
FROM nginx:latest

# Copia todos los builds al contenedor
COPY build_development /usr/share/nginx/html_development
COPY build_staging /usr/share/nginx/html_staging
COPY build_production /usr/share/nginx/html_production

# Copia configuraciones
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY entrypoint.sh /entrypoint.sh

# Configura permisos
RUN chmod 755 /entrypoint.sh

# Ejecuta entrypoint que extrae hashes dinámicamente
ENTRYPOINT ["/entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

#### Paso 3: Extracción Dinámica de Hashes en Entrypoint

```bash
#!/bin/bash
set -e

# Determinar ambiente (inyectado por variables de entorno o Helm)
ENVIRONMENT=${ENVIRONMENT:-development}

# Función para extraer hash del archivo JS
get_hash() {
    local dir="$1"
    find "$dir" -maxdepth 1 -type f -name 'my-mf.*.js' -printf '%f\n' 2>/dev/null \
        | head -n1 \
        | sed -E 's/my-mf\.(. *)\.js/\1/' \
        || echo ""
}

NGINX_CONF="/etc/nginx/conf. d/default.conf"

if [ -f "$NGINX_CONF" ]; then
    # Configurar root según ambiente
    sed -i "s/set \$environment \"development\";/set \$environment \"$ENVIRONMENT\";/" "$NGINX_CONF"
    
    # Configurar hash según ambiente
    case "$ENVIRONMENT" in
        development)
            HASH=$(get_hash "/usr/share/nginx/html_development")
            sed -i "s/set \$hashDevelopment \"\"/set \$hashDevelopment \"$HASH\"/" "$NGINX_CONF"
            ;;
        staging)
            HASH=$(get_hash "/usr/share/nginx/html_staging")
            sed -i "s/set \$hashStaging \"\"/set \$hashStaging \"$HASH\"/" "$NGINX_CONF"
            ;;
        production)
            HASH=$(get_hash "/usr/share/nginx/html_production")
            sed -i "s/set \$hashProduction \"\"/set \$hashProduction \"$HASH\"/" "$NGINX_CONF"
            ;;
    esac
fi

exec "$@"
```

### 3.2 Runtime (Tiempo de Ejecución)

#### Secuencia de Carga

**1. Usuario accede al Container Root:**
```
GET https://container-root.example.com/
```

**2. Container sirve HTML con Import Maps:**
```html
<script type="systemjs-importmap">
{
  "imports": {
    "@org/dashboard-mf": "/mf/dashboard-mf/bundle",
    "@org/profile-mf": "/mf/profile-mf/bundle"
  }
}
</script>
```

**3. Single-SPA registra microfrontends:**
```javascript
registerApplication({
  name: "@org/dashboard-mf",
  app: () => System.import("@org/dashboard-mf"),
  activeWhen: ["/dashboard"],
  customProps: { 
    domElement: document.querySelector("#dashboard-container") 
  }
});
```

**4. Usuario navega a ruta específica:**
```
/dashboard → Single-SPA activa el MF correspondiente
```

**5. SystemJS resuelve y carga el módulo:**
```
System.import("@org/dashboard-mf")
    ↓
GET /mf/dashboard-mf/bundle
    ↓
NGINX proxy → https://dashboard-mf.example.com/my-mf. dev.js
    ↓
302 redirect → my-mf.[hash].js
    ↓
200 OK → archivo con cache busting
```

**6. Montaje del microfrontend:**
```javascript
export const bootstrap = () => Promise.resolve();

export const mount = (props) => {
  return ReactDOM.render(<App />, props.domElement);
};

export const unmount = (props) => {
  return ReactDOM.unmountComponentAtNode(props.domElement);
};
```

## 4. Configuración Técnica

### 4.1 NGINX Container Root

```nginx
# Proxy al bundle del microfrontend (punto de entrada)
location /mf/dashboard-mf/bundle {
    proxy_pass https://dashboard-mf.example.com/my-mf.dev.js;
    proxy_redirect ~^https://[^/]+(/.+)$ https://$host/mf/dashboard-mf$1;
    proxy_set_header Host $proxy_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

# Proxy a recursos estáticos del microfrontend
location /mf/dashboard-mf/ {
    proxy_pass https://dashboard-mf.example.com/;
    proxy_set_header Host $proxy_host;
}

# Proxy al BFF del microfrontend (opcional)
location /api/dashboard/ {
    proxy_pass https://dashboard-api.example.com/;
    proxy_set_header Host $proxy_host;
}
```

### 4.2 NGINX Microfrontend

```nginx
# Servir archivos estáticos según ambiente
location / {
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Content-Type' always;
    
    set $environment "development";
    root /usr/share/nginx/html_$environment;
    try_files $uri =404;
}

# Redirects con hash dinámico por ambiente
location /my-mf. dev.js {
    add_header 'Access-Control-Allow-Origin' '*' always;
    set $hashDevelopment "";
    return 302 https://$host/my-mf.$hashDevelopment.js;
}

location /my-mf.stg.js {
    add_header 'Access-Control-Allow-Origin' '*' always;
    set $hashStaging "";
    return 302 https://$host/my-mf.$hashStaging.js;
}

location /my-mf.js {
    add_header 'Access-Control-Allow-Origin' '*' always;
    set $hashProduction "";
    return 302 https://$host/my-mf.$hashProduction. js;
}

# Healthcheck
location /health {
    access_log off;
    add_header 'Content-Type' 'application/json';
    return 200 '{"status":"UP"}';
}
```

### 4.3 Webpack Configuration

```javascript
const path = require('path');

module.exports = (env) => {
  const environment = env. ENVIRONMENT || 'development';
  
  return {
    entry: './src/index.tsx',
    output: {
      filename: `my-mf.[contenthash].js`,
      path: path.resolve(__dirname, `build_${environment}`),
      libraryTarget: 'system',
      publicPath: '/',
    },
    externals: ['react', 'react-dom', 'single-spa'],
    module: {
      rules: [
        {
          test: /\.(ts|tsx)$/,
          use: 'ts-loader',
          exclude: /node_modules/,
        },
        {
          test: /\.css$/,
          use: ['style-loader', 'css-loader'],
        },
      ],
    },
    resolve:  {
      extensions: ['.ts', '.tsx', '.js', '. jsx'],
    },
  };
};
```

## 5. Flujo de NGINX

### 5.1 Arquitectura de Comunicación

```
┌──────────────┐
│  Navegador   │
└──────┬───────┘
       │ (1) GET /mf/dashboard-mf/bundle
       ▼
┌─────────────────────────────────┐
│   Container Root (NGINX Proxy)  │
│  location /mf/dashboard-mf/     │
│  {                              │
│    proxy_pass → microfrontend   │
│  }                              │
└──────────┬──────────────────────┘
           │ (2) GET /my-mf.dev.js
           ▼
┌─────────────────────────────────┐
│  Dashboard MF (NGINX)           │
│  - Redirect con hash            │
│  - Serve static files           │
└──────────┬──────────────────────┘
           │ (3) 302 → /my-mf.[hash].js
           ▼
┌─────────────────────────────────┐
│   Container Root (NGINX)        │
│   proxy_redirect reescribe URL  │
└──────────┬──────────────────────┘
           │ (4) 302 (reescrito)
           ▼
┌──────────────┐
│  Navegador   │ ← Sigue redirect
└──────┬───────┘
       │ (5) GET /mf/dashboard-mf/my-mf.[hash].js
       ▼
┌─────────────────────────────────┐
│   Container Root (NGINX)        │
│   proxy_pass al microfrontend   │
└──────────┬──────────────────────┘
           │ (6) GET /my-mf.[hash].js
           ▼
┌─────────────────────────────────┐
│  Dashboard MF (NGINX)           │
│  root /usr/share/nginx/html_dev │
│  try_files $uri =404            │
└──────────┬──────────────────────┘
           │ (7) 200 OK + [JS content]
           ▼
┌─────────────────────────────────┐
│   Container Root (NGINX)        │
│   Reenvía respuesta             │
└──────────┬──────────────────────┘
           │ (8) 200 OK + [JS content]
           ▼
┌──────────────┐
│  Navegador   │ ✅ Microfrontend cargado
└──────────────┘
```

### 5.2 Secuencia Detallada de Peticiones

#### Petición 1: Solicitud del bundle

```
(1) Navegador → Container Root
    GET https://container-root.example.com/mf/dashboard-mf/bundle
```

**Container Root procesa:**
```nginx
location /mf/dashboard-mf/bundle {
    proxy_pass https://dashboard-mf.example.com/my-mf. dev.js;
}
```

```
(2) Container Root → Dashboard
    GET https://dashboard-mf.example.com/my-mf.dev.js
```

#### Petición 2: Redirect con hash

**Dashboard responde:**
```nginx
location /my-mf. dev.js {
    set $hashDevelopment "ea1a373698777678cd69";
    return 302 https://$host/my-mf.$hashDevelopment.js;
}
```

```
(3) Dashboard → Container Root
    HTTP 302 Redirect
    Location: https://dashboard-mf.example.com/my-mf. ea1a373698777678cd69.js
```

**Container Root reescribe el redirect:**
```nginx
proxy_redirect ~^https://[^/]+(/.+)$ https://$host/mf/dashboard-mf$1;
```

```
(4) Container Root → Navegador
    HTTP 302 Redirect
    Location: https://container-root.example.com/mf/dashboard-mf/my-mf. ea1a373698777678cd69.js
```

#### Petición 3: Solicitud del archivo con hash

```
(5) Navegador → Container Root
    GET https://container-root.example.com/mf/dashboard-mf/my-mf. ea1a373698777678cd69.js
```

**Container Root procesa:**
```nginx
location /mf/dashboard-mf/ {
    proxy_pass https://dashboard-mf.example.com/;
}
```

**Transformación de URL:**
```
/mf/dashboard-mf/my-mf.ea1a373698777678cd69.js
→ Quita prefijo /mf/dashboard-mf/
→ /my-mf.ea1a373698777678cd69.js
```

```
(6) Container Root → Dashboard
    GET https://dashboard-mf.example.com/my-mf.ea1a373698777678cd69.js
```

#### Petición 4: Servir archivo estático

**Dashboard procesa:**
```nginx
location / {
    set $environment "development";
    root /usr/share/nginx/html_$environment;
    try_files $uri =404;
}
```

**Resolución:**
```
$uri = /my-mf.ea1a373698777678cd69.js
$environment = development (configurado por entrypoint)

Ruta completa: 
/usr/share/nginx/html_development/my-mf.ea1a373698777678cd69.js
```

```
(7) Dashboard → Container Root
    HTTP 200 OK
    Content-Type: application/javascript
    [contenido del archivo JS]

(8) Container Root → Navegador
    HTTP 200 OK
    [contenido del archivo JS]
```

### 5.3 Análisis del Redirect

**Input del microfrontend:**
```
https://dashboard-mf.example.com/my-mf.ea1a373698777678cd69.js
```

**Regex de proxy_redirect:**
```nginx
proxy_redirect ~^https://[^/]+(/.+)$ https://$host/mf/dashboard-mf$1;
```

**Análisis:**
```
~^https://[^/]+(/.+)$
  │       │     │
  │       │     └─ $1 = /my-mf.ea1a373698777678cd69.js (captura)
  │       └─ Dominio:  dashboard-mf.example.com
  └─ Protocolo: https://
```

**Output reescrito:**
```
https://container-root.example.com/mf/dashboard-mf/my-mf.ea1a373698777678cd69.js
```

### 5.4 Diagrama de Flujo Completo

```
    Navegador
    │
    │ ① GET /mf/dashboard-mf/bundle
    │
    v
Container Root (NGINX)
    location /mf/dashboard-mf/bundle
    {
        proxy_pass . ../dev.js
    }
    │
    │ ② proxy_pass → GET /my-mf.dev. js
    v
Dashboard (NGINX Microfrontend)
    location /my-mf.dev.js
    {
        return 302 . ../[hash].js
    }
    │
    │ ③ 302 Redirect
    │ Location: https://dashboard-mf.example.com/my-mf.ea1a... js
    v
Container Root (NGINX)
    proxy_redirect reescribe: 
    https://dashboard-mf.example.com/
    → https://container-root.example.com/mf/... 
    │
    │ ④ 302 Redirect (reescrito)
    │ Location: https://container-root.example.com/mf/dashboard-mf/my-mf.ea1a...js
    v
Navegador
    │
    │ ⑤ GET /mf/dashboard-mf/my-mf. ea1a373698777678cd69.js
    v
Container Root (NGINX)
    location /mf/dashboard-mf/
    {
        proxy_pass https://dashboard-mf.example.com/
    }
    │
    │ ⑥ proxy_pass → GET /my-mf. ea1a373698777678cd69.js
    v
Dashboard (NGINX Microfrontend)
    location /
    {
        root /usr/share/nginx/html_dev;
        try_files $uri =404;
    }
    │
    │ Busca archivo: 
    │ /usr/share/nginx/html_development/my-mf.ea1a... js
    │
    │ ⑦ 200 OK + [contenido JS]
    v
Container Root (NGINX)
    ⑧ Reenvía respuesta
    │
    │ 200 OK + [contenido JS]
    v
Navegador ✅ Microfrontend cargado
```

### 5.5 Responsabilidades de cada NGINX

#### Container Root
- ✅ No almacena archivos estáticos de los microfrontends
- ✅ Solo redirige peticiones al microfrontend correspondiente
- ✅ Reescribe URLs en los redirects para mantener todo bajo su dominio
- ✅ Actúa como punto único de entrada para el navegador

#### Microfrontend
- ✅ Primera petición:  Devuelve redirect (302) con hash
- ✅ Segunda petición: Sirve el archivo estático (200)
- ✅ Usa cache-busting mediante hashes en nombres de archivo
- ✅ Auto-configura el hash correcto al iniciar el contenedor

## 6. Ventajas y Beneficios

### Independencia de Despliegue
- ✅ Cada microfrontend se despliega de forma independiente
- ✅ No requiere rebuild del container root
- ✅ Rollback independiente por microfrontend
- ✅ Ciclos de release distintos por equipo

### Cache Busting Automático
- ✅ Hashes únicos por build garantizan cache busting
- ✅ No se requiere versionado manual
- ✅ Evita problemas de cache en producción
- ✅ Cada deploy genera nuevos hashes automáticamente

### Multi-Ambiente en una Sola Imagen
- ✅ Una imagen Docker contiene los 3 ambientes
- ✅ Selección de ambiente en runtime (no en build time)
- ✅ Reduce tiempos de CI/CD
- ✅ Elimina diferencias entre ambientes

### Escalabilidad
- ✅ Microfrontends pueden escalar independientemente
- ✅ Container root puede usar CDN para assets estáticos
- ✅ Load balancing por microfrontend
- ✅ Fácil agregar nuevos microfrontends

### Aislamiento
- ✅ Cada equipo es dueño de su microfrontend
- ✅ Tecnologías independientes (React, Vue, Angular)
- ✅ Errores en un MF no afectan a otros
- ✅ Dependencies isoladas por microfrontend

### Flexibilidad Tecnológica
- ✅ Cada microfrontend puede usar su propio stack
- ✅ Actualizaciones graduales de frameworks
- ✅ Testing de nuevas tecnologías en microfrontends aislados
- ✅ Migración progresiva sin rewrites completos

## 7. Mejores Prácticas

### 7.1 Organización del Código

```
organization/
├── container-root/
│   ├── src/
│   │   ├── index.html
│   │   ├── index.js
│   │   └── importmap.json
│   ├── nginx. conf
│   └── Dockerfile
├── dashboard-mf/
│   ├── src/
│   │   ├── components/
│   │   ├── services/
│   │   └── index.tsx
│   ├── nginx. conf
│   ├── entrypoint.sh
│   ├── webpack.config.js
│   └── Dockerfile
├── profile-mf/
│   ├── src/
│   ├── nginx.conf
│   ├── entrypoint.sh
│   └── Dockerfile
└── shared-lib/
    ├── src/
    │   ├── components/
    │   ├── utils/
    │   └── index.ts
    └── package.json
```

### 7.2 Naming Conventions

#### Microfrontends
```
Scoped package:  @org/nombre-mf
Ejemplo: @org/dashboard-mf, @org/profile-mf
```

#### Rutas en Container Root
```
Pattern: /mf/nombre-mf/
Ejemplo:  /mf/dashboard-mf/, /mf/profile-mf/
```

#### Archivos de Bundle
```
Pattern: nombre-mf.[hash].js
Ejemplo: dashboard-mf.ea1a373698777678cd69.js
```

#### Ambientes
```
development  → . dev.js
staging      → . stg.js
production   → . js
```

### 7.3 Variables de Entorno

#### Container Root
```bash
# Ambiente
ENVIRONMENT=production

# Configuración
PUBLIC_PATH=/
NODE_ENV=production
```

#### Microfrontend
```bash
# Ambiente
ENVIRONMENT=production

# API URLs
API_URL=https://api.example.com
AUTH_URL=https://auth.example.com

# Feature Flags
ENABLE_FEATURE_X=true
```

### 7.4 Monitoreo y Logs

#### NGINX Logs Estructurados

```nginx
# Formato JSON para logs
log_format json_combined escape=json
  '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"request":"$request",'
    '"status":  $status,'
    '"body_bytes_sent": $body_bytes_sent,'
    '"request_time": $request_time,'
    '"upstream_addr":"$upstream_addr",'
    '"upstream_status":"$upstream_status",'
    '"upstream_response_time":"$upstream_response_time"'
  '}';

access_log /var/log/nginx/access.log json_combined;
error_log /var/log/nginx/error.log warn;
```

#### Métricas Recomendadas

**Container Root:**
- Tiempo de respuesta del proxy
- Rate de errores 5xx por microfrontend
- Throughput de peticiones por MF
- Latencia de redirects

**Microfrontends:**
- Tiempo de carga del bundle
- Errores en mount/unmount
- Tamaño del bundle
- Cache hit rate

### 7.5 Testing

#### Unit Tests (Por Microfrontend)
```javascript
// dashboard-mf/src/components/Dashboard.test.tsx
import { render, screen } from '@testing-library/react';
import Dashboard from './Dashboard';

describe('Dashboard Component', () => {
  it('renders correctly', () => {
    render(<Dashboard />);
    expect(screen.getByText('Dashboard')).toBeInTheDocument();
  });
});
```

#### Integration Tests (Comunicación entre MFs)
```javascript
// tests/integration/mf-communication.test.js
describe('Microfrontend Communication', () => {
  it('should share state between dashboard and profile', async () => {
    // Test event bus or shared state
  });
});
```

#### E2E Tests (Desde Container Root)
```javascript
// tests/e2e/user-flow.spec.js
describe('User Flow', () => {
  it('navigates from dashboard to profile', async () => {
    await page.goto('https://container-root.example.com/dashboard');
    await page.click('[data-testid="profile-link"]');
    await expect(page).toHaveURL(/.*profile/);
  });
});
```

#### Performance Tests
```javascript
// tests/performance/bundle-size.test.js
describe('Bundle Size', () => {
  it('dashboard bundle should be under 500KB', async () => {
    const size = await getBundleSize('dashboard-mf');
    expect(size).toBeLessThan(500 * 1024);
  });
});
```

---

## Apéndice A: Comandos Útiles

### Build
```bash
# Build para un ambiente específico
npm run build -- --env ENVIRONMENT=development

# Build para todos los ambientes
npm run build: all
```

### Docker
```bash
# Build imagen
docker build -t my-mf: latest .

# Run con ambiente específico
docker run -e ENVIRONMENT=production -p 8080:8080 my-mf:latest

# Ver logs
docker logs -f container-id
```

### NGINX
```bash
# Test configuración
nginx -t

# Reload sin downtime
nginx -s reload

# Ver configuración activa
nginx -T
```

### Debugging
```bash
# Ver hash extraído en runtime
docker exec container-id cat /etc/nginx/conf.d/default.conf | grep hash

# Test redirect
curl -I https://dashboard-mf.example.com/my-mf.dev.js

# Ver archivos en contenedor
docker exec container-id ls -la /usr/share/nginx/html_development/
```

---

## Apéndice B:  Troubleshooting

### Problema: Microfrontend no carga

**Síntomas:**
```
Failed to fetch dynamically imported module
```

**Solución:**
1. Verificar CORS headers en NGINX del microfrontend
2. Verificar que el import map apunta a la URL correcta
3. Revisar logs de NGINX del container root

### Problema: Hash no se actualiza

**Síntomas:**
```
302 redirect apunta a archivo con hash antiguo
```

**Solución:**
1. Verificar que entrypoint. sh se ejecuta correctamente
2. Verificar permisos del script:  `chmod 755 entrypoint.sh`
3. Revisar logs del contenedor al iniciar

### Problema: 404 en archivo con hash

**Síntomas:**
```
GET /my-mf.abc123.js → 404 Not Found
```

**Solución:**
1. Verificar que el archivo existe en el contenedor
2. Verificar variable `$environment` en NGINX
3. Verificar que el build se copió correctamente

---

## Referencias

- [Single-SPA Documentation](https://single-spa.js.org/)
- [SystemJS Documentation](https://github.com/systemjs/systemjs)
- [NGINX Documentation](https://nginx.org/en/docs/)
- [Module Federation](https://webpack.js.org/concepts/module-federation/)
- [Micro Frontends by Martin Fowler](https://martinfowler.com/articles/micro-frontends.html)

---

**Última actualización:** 2025-12-30  
**Versión:** 1.0.0
