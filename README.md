# Guía de Inicio: Proyecto Angular 22 con pnpm y Backend Spring Boot

Paso a paso para arrancar y levantar un proyecto con **Angular 22** usando **pnpm**, configurado para conectarse con un backend en **Spring Boot**, junto con la estructura de carpetas para que sepas dónde va cada cosa.

---

## 1. Requerimientos

Antes de arrancar, tenés que tener instalado sí o sí:

- **Node.js**: Versión `v24.x` o superior.
  - Chequeá la versión:
    ```bash
    node -v
    ```
- **pnpm**: Versión `v11.x` o superior.
  - Chequeá si lo tenés:
    ```bash
    pnpm -v
    ```
  - Si no lo tenés, instalalo con:
    ```bash
    npm install -g pnpm@latest
    ```
- **Git**: Instalá Git para el control de versiones y punto.
  - Chequeá que esté instalado:
    ```bash
    git --version
    ```
- **Backend Spring Boot**: Tené corriendo tu API de Spring Boot (por defecto en `http://localhost:8080`).

---

## 2. Aclaración clave sobre SSR y Spring Boot

**Tener backend en Spring Boot NO te obliga a meter SSR (Server-Side Rendering).**

- **¿Qué es SSR?** Levanta un servidor intermedio en **Node.js** que renderiza el HTML antes de mandarlo al navegador. Sirve casi exclusivamente si necesitás **SEO** (que Google indexe tus páginas) o redes sociales (previews de links), como en un e-commerce o una landing pública.
- **¿Qué pasa con Spring Boot?** Tu backend en Java ya maneja la lógica de negocio, base de datos y expone endpoints REST (JSON).
  - Si tu proyecto es un panel, sistema de gestión, dashboard o app detrás de login: **Poné que NO a SSR**. Te ahorrás desplegar un server Node.js al cuete además del server de Spring Boot.
  - Si es una página pública que necesita SEO sí o sí: **Poné que SÍ a SSR**. En ese caso Angular te genera `server.ts` y `app.config.server.ts` para que Node.js haga el pre-renderizado.

---

## 3. Comandos para Crear y Levantar el Proyecto

Meté estos comandos en la terminal en este orden:

### Paso 1: Crear el proyecto con Angular CLI y pnpm

Corré este comando para generar el proyecto indicándole que use `pnpm`:

```bash
pnpm dlx @angular/cli@22 new mi-proyecto --package-manager=pnpm
```

Cuando el CLI te pregunte:
- **Formato de estilos**: Elegí `SCSS`.
- **SSR / Prerendering**: 
  - Si es una SPA estándar que consume la API de Spring Boot: Poné **`N`** (No).
  - Si necesitás posicionamiento en buscadores (SEO): Poné **`Y`** (Sí).

### Paso 2: Entrar a la carpeta del proyecto

```bash
cd mi-proyecto
```

### Paso 3: Instalar las dependencias

Asegurate de bajar todos los paquetes:

```bash
pnpm install
```

### Paso 4: Levantar el servidor de desarrollo

Para compilar y correr el proyecto en local:

```bash
pnpm start
```

Una vez que levantó, abrilo en el navegador:
👉 **`http://localhost:4200`**

---

## 4. Conexión con el Backend Spring Boot

Para que Angular se conecte con Spring Boot sin renegar con CORS en desarrollo y manejando seguridad de forma limpia, tenés que hacer lo siguiente:

### A. Configurar el Proxy de Desarrollo (evita problemas de CORS)

Creá el archivo `proxy.conf.json` en la raíz del proyecto para reenviar las llamadas de `/api` a Spring Boot (`http://localhost:8080`):

```json
{
  "/api": {
    "target": "http://localhost:8080",
    "secure": false,
    "changeOrigin": true,
    "logLevel": "debug"
  }
}
```

En tu `package.json`, asegurate de que el script `start` use este proxy:

```json
"scripts": {
  "start": "ng serve --proxy-config proxy.conf.json",
  ...
}
```

Así, desde Angular llamás a `/api/usuarios` y Angular se lo pide directo a Spring Boot en el puerto 8080 sin que el navegador te bloquee por CORS.

### B. Habilitar `HttpClient` en `src/app/app.config.ts`

Angular 22 no usa módulos. Registrá el cliente HTTP con `withFetch()` en la configuración global:

```typescript
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withFetch(), withInterceptors } from '@angular/common/http';
import { routes } from './app.routes';
import { jwtInterceptor } from './core/interceptors/jwt.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideHttpClient(
      withFetch(),
      withInterceptors([jwtInterceptor]) // Inyecta el token en las peticiones a Spring Boot
    )
  ]
};
```

### C. Interceptor para Spring Security (JWT)

En `src/app/core/interceptors/jwt.interceptor.ts`, creá el interceptor funcional para mandar el token en cada request:

```typescript
import { HttpInterceptorFn } from '@angular/common/http';

export const jwtInterceptor: HttpInterceptorFn = (req, next) => {
  const token = localStorage.getItem('token');

  if (token) {
    const cloned = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`
      }
    });
    return next(cloned);
  }

  return next(req);
};
```

### D. Servicio de ejemplo para pegarle a Spring Boot

Generá un servicio para la entidad que necesites:

```bash
pnpm ng generate service core/services/usuario
```

Y adentro usás el `HttpClient` inyectado con `inject()`:

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface Usuario {
  id: number;
  nombre: string;
  email: string;
}

@Injectable({
  providedIn: 'root'
})
export class UsuarioService {
  private http = inject(HttpClient);
  private apiUrl = '/api/usuarios';

  getUsuarios(): Observable<Usuario[]> {
    return this.http.get<Usuario[]>(this.apiUrl);
  }
}
```

---

## 5. Estructura del Scaffolding (Dónde va cada cosa)

Con la integración de Spring Boot y Angular 22 (Standalone), la estructura ordenada del proyecto queda así:

```text
mi-proyecto/
├── .angular/                  # Caché interna del CLI de Angular
├── .vscode/                   # Ajustes recomendados para VS Code
├── node_modules/              # Dependencias gestionadas por pnpm
├── public/                    # Archivos estáticos servidos directamente (imágenes, favicon)
├── src/                       # Todo el código fuente de tu app
│   ├── app/                   # Componentes, lógica y estado de la aplicación
│   │   ├── core/              # Código transversal de instancia única (singleton)
│   │   │   ├── guards/        # Protección de rutas (ej: auth.guard.ts)
│   │   │   ├── interceptors/  # Interceptores HTTP (ej: jwt.interceptor.ts, error.interceptor.ts)
│   │   │   ├── models/        # Modelos e interfaces de datos de Spring Boot (ej: usuario.model.ts)
│   │   │   └── services/      # Servicios de comunicación con la API (ej: auth.service.ts)
│   │   ├── shared/            # Componentes y pipes reutilizables en varias pantallas
│   │   │   ├── components/    # Botones, tablas, modales, loaders comunes
│   │   │   └── pipes/         # Pipes comunes (formateo de fechas, moneda, etc.)
│   │   ├── features/          # Pantallas y vistas divididas por funcionalidad
│   │   │   ├── auth/          # Login, registro
│   │   │   └── dashboard/     # Panel principal con sus componentes locales
│   │   ├── app.component.ts   # Componente raíz (Standalone)
│   │   ├── app.component.html # Template HTML principal
│   │   ├── app.component.scss # Estilos propios del componente raíz
│   │   ├── app.config.ts      # Providers globales (provideRouter, provideHttpClient)
│   │   ├── app.config.server.ts # Solo si elegiste SSR: configuración del server
│   │   └── app.routes.ts      # Definición y lazy loading de rutas
│   ├── environments/          # Variables de entorno (URLs del backend según ambiente)
│   │   ├── environment.ts     # Producción (ej: apiUrl: 'https://api.tudominio.com/api')
│   │   └── environment.development.ts # Desarrollo local
│   ├── index.html             # HTML base donde se monta la app (<app-root>)
│   ├── main.ts                # Entrada que levanta la app en el browser
│   ├── server.ts              # Solo si elegiste SSR: servidor Node.js que ejecuta Angular
│   └── styles.scss            # Estilos globales de la aplicación
├── .editorconfig              # Reglas de formato entre editores
├── .gitignore                 # Archivos ignorados por Git
├── angular.json               # Configuración central del CLI de Angular
├── package.json               # Dependencias y scripts
├── pnpm-lock.yaml             # Versiones exactas fijadas por pnpm (subilo a Git)
├── proxy.conf.json            # Configuración de proxy para desarrollo local con Spring Boot
├── tsconfig.app.json          # Configuración de TypeScript para la app
├── tsconfig.json              # Configuración base de TypeScript
└── tsconfig.spec.json         # Configuración de TypeScript para los tests
```

---

### Comandos que vas a usar en el día a día

| Comando | Para qué sirve |
| :--- | :--- |
| `pnpm start` | Levanta el servidor local con el proxy hacia Spring Boot (`ng serve --proxy-config proxy.conf.json`). |
| `pnpm run build` | Compila todo para producción en la carpeta `dist/`. |
| `pnpm test` | Corre los tests unitarios. |
| `pnpm ng generate component <nombre>` | Crea un componente standalone nuevo. |
| `pnpm ng generate service <nombre>` | Crea un servicio nuevo para pegarle a la API. |
