# Arquitectura de Microfrontends

> Documentación de la arquitectura de microfrontends basada en Single-SPA con Container Root como orquestador.

## 🎯 Descripción

Sistema de microfrontends que permite: 
- ✅ Despliegue independiente de aplicaciones frontend
- ✅ Integración en runtime sin necesidad de rebuilds
- ✅ Soporte multi-framework (React, Vue, Angular)
- ✅ Cache busting automático
- ✅ Multi-ambiente en una sola imagen Docker

## 🏗️ Arquitectura

```
Container Root (Single-SPA + NGINX Proxy)
    ├── Login MF
    ├── Dashboard MF
    ├── Products MF
    ├── Profile MF
    └── Shared Library
```

## 📚 Documentación Completa

Lee **[ARCHITECTURE.md](ARCHITECTURE.md)** para documentación técnica detallada que incluye: 

- 🏛️ Arquitectura general y componentes
- ⚙️ Configuración de NGINX (Container + Microfrontends)
- 🔄 Flujo de build y runtime
- 🚀 Proceso de deployment multi-ambiente
- 📦 Configuración de Webpack
- 🔍 Flujo completo de peticiones HTTP

## 🚀 Quick Start

### Crear un nuevo microfrontend

```bash
# Copia la plantilla
cp -r examples/microfrontend-template my-new-mf

# Instala dependencias
cd my-new-mf
npm install

# Desarrolla
npm start

# Build para todos los ambientes
npm run build
```

### Estructura de un microfrontend

```
my-new-mf/
├── src/
│   └── index.tsx          # Exporta bootstrap, mount, unmount
├── nginx. conf             # Config con redirects por ambiente
├── entrypoint.sh          # Extrae hashes dinámicamente
├── webpack.config.js      # Output:  libraryTarget: 'system'
└── Dockerfile             # Imagen con 3 ambientes
```

### Registrar en Container Root

```javascript
// container-root/src/index.js
registerApplication({
  name: "@org/my-new-mf",
  app: () => System.import("@org/my-new-mf"),
  activeWhen: ["/my-route"],
  customProps: { domElement: document.querySelector("#app") }
});
```

```html
<!-- container-root/src/index.html -->
<script type="systemjs-importmap">
{
  "imports": {
    "@org/my-new-mf": "/mf/my-new-mf/bundle"
  }
}
</script>
```

```nginx
# container-root/nginx. conf
location /mf/my-new-mf/bundle {
    proxy_pass https://my-new-mf.example.com/my-new-mf. dev. js;
    proxy_redirect ~^https://[^/]+(/.+)$ https://$host/mf/my-new-mf$1;
}

location /mf/my-new-mf/ {
    proxy_pass https://my-new-mf.example.com/;
}
```

## 🛠️ Tecnologías

| Componente | Tecnología |
|------------|------------|
| Orquestador | Single-SPA |
| Module Loader | SystemJS |
| Proxy Reverso | NGINX |
| Frameworks | React, Vue, Angular |
| Build Tool | Webpack |
| Containerización | Docker |

## 📁 Ejemplos

### [Container Root](examples/container-root/)
Configuración completa del container root con: 
- NGINX proxy configuration
- Import maps de SystemJS
- Registro de aplicaciones Single-SPA

### [Microfrontend Template](examples/microfrontend-template/)
Plantilla lista para usar que incluye:
- Configuración de Webpack multi-ambiente
- NGINX con redirects por ambiente
- Entrypoint para extracción dinámica de hashes
- Dockerfile multi-stage

## 🎯 Casos de Uso

- ✅ Múltiples equipos trabajando en diferentes features
- ✅ Migración gradual de monolito a microfrontends
- ✅ Aplicaciones con diferentes ciclos de release
- ✅ Sistemas que requieren hot-deploy de módulos
- ✅ Plataformas multi-tenant con módulos opcionales

## 📖 Recursos

- [Single-SPA Documentation](https://single-spa.js.org/)
- [SystemJS Documentation](https://github.com/systemjs/systemjs)
- [NGINX Documentation](https://nginx.org/en/docs/)

## 🤝 Contribución

Las contribuciones son bienvenidas.  Por favor:

1. Fork el proyecto
2. Crea una rama para tu feature (`git checkout -b feature/AmazingFeature`)
3. Commit tus cambios (`git commit -m 'Add some AmazingFeature'`)
4. Push a la rama (`git push origin feature/AmazingFeature`)
5. Abre un Pull Request

## 📝 Licencia

MIT License - ver [LICENSE](LICENSE) para más detalles.

## 👥 Autores

- **Tu Nombre** - [@FacundoBettella](https://github.com/FacundoBettella)

---

⭐ Si te resultó útil, considera darle una estrella al proyecto
