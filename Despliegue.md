# Despliegue de PokeDex en Azure

Este documento explica cómo se desplegó la aplicación **PokeDex** (Angular) en la nube pública **Microsoft Azure** usando el servicio **Azure Static Web Apps**, con la suscripción *Azure for Students*.

- **Repositorio:** https://github.com/CamiloProgramadix/pokedex
- **URL de la aplicación:** https://delightful-pond-0c6454810.3.azurestaticapps.net
- **Código fuente original:** https://github.com/keilermora/pokedex-angular

---

## 1. Servicio elegido y justificación

Se eligió **Azure Static Web Apps** porque:

- La aplicación es un front-end Angular que se compila a archivos estáticos (HTML, CSS, JS).
- Tiene un **plan gratuito** (Free), adecuado para la suscripción de estudiante.
- Se integra con **GitHub**: crea automáticamente un workflow de GitHub Actions que compila y publica la app en cada `git push`.
- Entrega **HTTPS** automáticamente.
- Permite configurar **encabezados HTTP de seguridad** mediante el archivo `staticwebapp.config.json`.

## 2. Requisitos previos

- Cuenta de Azure for Students activa (ver `README.md`).
- Cuenta de GitHub con el repositorio `pokedex`.
- Git instalado en el computador.
- Código fuente de PokeDex descargado en la carpeta local del proyecto.

## 3. Subir el código al repositorio

1. Clonar o enlazar el repositorio de GitHub con la carpeta del proyecto.
2. Verificar que el código de la app esté en la raíz del repositorio (`angular.json`, `package.json`, carpeta `src`, etc.).
3. Subir los archivos:

```bash
git add .
git commit -m "Agrega codigo fuente de PokeDex"
git push origin main
```

## 4. Crear el recurso Static Web App en Azure

1. Ingresar a [https://portal.azure.com](https://portal.azure.com).
2. Buscar **Static Web Apps** y hacer clic en **Create**.
3. Completar la pestaña **Basics**:

| Campo | Valor |
|---|---|
| Subscription | Azure for Students |
| Resource group | Nuevo, por ejemplo `rg-pokedex` |
| Name | `pokedex-app` |
| Plan type | Free |
| Region | La más cercana disponible (por ejemplo, East US 2) |

4. En **Deployment details**:
   - **Source:** GitHub
   - Autorizar la cuenta de GitHub.
   - **Organization:** usuario de GitHub
   - **Repository:** `pokedex`
   - **Branch:** `main`

5. En **Build Details**:

| Campo | Valor |
|---|---|
| Build Presets | Angular |
| App location | `/` |
| Api location | (vacío) |
| Output location | Carpeta de salida del build de Angular (ver nota) |

> **Nota sobre Output location:** el valor depende de la versión de Angular. Se revisa en `angular.json`, en la propiedad `outputPath`. Ejemplos habituales: `dist/pokedex-angular` o `dist/pokedex-angular/browser`.

6. Clic en **Review + create** y luego en **Create**.

## 5. Workflow de GitHub Actions

Al crear el recurso, Azure agrega automáticamente un archivo en el repositorio:

```
.github/workflows/azure-static-web-apps-<nombre>.yml
```

Este workflow se ejecuta en cada push a `main`: instala dependencias, compila la app con Angular y publica el resultado en Azure.

El progreso se verifica en la pestaña **Actions** del repositorio. El despliegue es exitoso cuando el workflow termina en verde.

### Problema encontrado: push rechazado

Como Azure agregó el archivo de workflow directamente al repositorio remoto, el siguiente `git push` desde el computador fue rechazado con el mensaje `failed to push some refs ... fetch first`, porque el remoto tenía un commit que el repositorio local no tenía.

**Solución:**

```bash
git pull --rebase origin main
git push origin main
```

## 6. Ajuste de configuración para Azure: ruta de imágenes

Al abrir la aplicación desplegada, la interfaz cargaba, pero **las imágenes de los Pokémon no aparecían**.

**Causa:** en el archivo `src/environments/environment.prod.ts`, la propiedad `imagesPath` tenía el valor:

```ts
imagesPath: '/pokedex-angular/assets/images',
```

Ese prefijo `/pokedex-angular/` corresponde a **GitHub Pages**, donde las aplicaciones se publican en un subdirectorio con el nombre del repositorio. Azure Static Web Apps publica la aplicación en la **raíz del dominio**, por lo que esa ruta devolvía error 404.

**Solución:** se eliminó el prefijo:

```ts
imagesPath: '/assets/images',
```

Luego se subió el cambio con `git add`, `git commit` y `git push`, y el workflow volvió a desplegar la aplicación.

## 7. Configuración de encabezados de seguridad

Para cumplir los requisitos de seguridad de la compañía, se creó el archivo `staticwebapp.config.json` en la raíz del proyecto (en la carpeta definida como *App location*):

```json
{
  "globalHeaders": {
    "Content-Security-Policy": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' https://pokeapi.co https://beta.pokeapi.co; font-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
    "X-Frame-Options": "DENY",
    "X-Content-Type-Options": "nosniff",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=()"
  }
}
```

### Función de cada encabezado

| Encabezado | Función |
|---|---|
| Content-Security-Policy | Define desde qué orígenes puede cargar recursos la página. Mitiga ataques XSS. |
| Strict-Transport-Security | Obliga al navegador a usar siempre HTTPS. |
| X-Frame-Options | Impide que la app se cargue dentro de un iframe (clickjacking). |
| X-Content-Type-Options | Evita que el navegador interprete archivos con un tipo MIME distinto al declarado. |
| Referrer-Policy | Limita la información de origen enviada al navegar a otros sitios. |
| Permissions-Policy | Desactiva funciones del navegador que la app no usa. |

### Ajuste de la política CSP

La política se adaptó a los recursos que realmente usa la aplicación:

- `connect-src` incluye `https://pokeapi.co` y `https://beta.pokeapi.co`, porque la app consulta la API REST y el endpoint GraphQL de PokeAPI.
- `img-src 'self' data:` es suficiente porque las imágenes se sirven desde el propio dominio (`/assets/images`).

Para verificar que la CSP no bloquea recursos necesarios, se abrió la aplicación con las herramientas de desarrollador (F12) y se revisó la pestaña **Console**. Cualquier error del tipo `Refused to load ...` indica un dominio que debe agregarse a la directiva correspondiente.

Después de crear el archivo se publicó con:

```bash
git add staticwebapp.config.json
git commit -m "Agrega encabezados de seguridad"
git push origin main
```

## 8. Verificación del despliegue

1. En la pestaña **Actions** de GitHub, el workflow terminó en verde.
2. En el portal de Azure, dentro del recurso Static Web App, se copió la **URL** generada.
3. Se abrió la URL y se comprobó que la aplicación carga, muestra los Pokémon con sus imágenes y los filtros (versión, nombre, tipo) funcionan sin errores.

## 9. Escaneo de seguridad

Se escaneó la aplicación en [https://securityheaders.com](https://securityheaders.com) ingresando la URL pública.

| Momento | Calificación |
|---|---|
| Antes de agregar encabezados | _(completar)_ |
| Después de agregar encabezados | _(completar)_ |

_(Agregar capturas de pantalla de ambos escaneos)_

## 10. Auditorías adicionales (opcional)

- **SSL Labs:** https://www.ssllabs.com/ssltest/ — calificación: _(completar)_
- **OWASP ZAP:** escaneo automatizado sobre la URL de la aplicación — resultado: _(completar)_

## 11. Actualizaciones futuras

Cada `git push` a la rama `main` dispara automáticamente un nuevo despliegue mediante GitHub Actions. No se requiere ninguna acción manual en Azure.

## 12. Solución de problemas

| Problema | Causa probable | Solución |
|---|---|---|
| El workflow falla en la pestaña Actions | *App location* u *Output location* incorrectos | Revisar el log del workflow y verificar `outputPath` en `angular.json` |
| `git push` rechazado (`fetch first`) | El remoto tiene commits que no están en local | `git pull --rebase origin main` y luego `git push origin main` |
| Las imágenes no cargan (404) | Prefijo de GitHub Pages en `imagesPath` | Cambiar a `/assets/images` en `environment.prod.ts` |
| Faltan datos o estilos tras agregar encabezados | La CSP bloquea un recurso | Revisar la consola (F12) y agregar el dominio faltante a la directiva correspondiente |
| La página no refleja los cambios | Caché del navegador | Recargar con Ctrl + Shift + R |
