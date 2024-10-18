# Dudas sobre `tsconfig.json`
Dudas sobre `tsconfig.json` en la actual arquitectura hexagonal.
## Chat
### Introducción a `tsconfig.json`
El archivo `tsconfig.json` es el archivo de configuración de TypeScript que define cómo se debe compilar un proyecto. Proporciona opciones sobre el compilador TypeScript (`tsc`), como qué archivos incluir en la compilación, qué versión de JavaScript usar, y cómo gestionar la salida de archivos.

Cuando ejecutas `tsc` en un proyecto, el compilador busca automáticamente un archivo `tsconfig.json` en la carpeta raíz del proyecto para saber cómo configurar el proceso de compilación.

#### ¿Qué hace el archivo `tsconfig.json`?

1. **Especifica los archivos que TypeScript debe incluir/excluir en la compilación.**  
   Puedes definir explícitamente qué archivos o directorios incluir o excluir en el proceso de compilación usando las propiedades `include` y `exclude`.

2. **Configura opciones del compilador.**  
   Define qué características de JavaScript soportar, cómo manejar módulos, los archivos de salida, la verificación de tipos, entre otros, utilizando la clave `compilerOptions`.

3. **Permite la organización de proyectos grandes.**  
   En proyectos más grandes, `tsconfig.json` facilita dividir el código en módulos o paquetes utilizando referencias y configuraciones compartidas, por ejemplo, en una arquitectura monorepo.

#### Estructura básica del archivo `tsconfig.json`

Un archivo típico puede verse así:

```json
{
  "compilerOptions": {
    "target": "es6",                    // Especifica la versión de JS a la que se compila
    "module": "commonjs",                // Sistema de módulos a utilizar (CommonJS, ES6, etc.)
    "strict": true,                      // Activa estricta verificación de tipos
    "outDir": "./dist",                  // Carpeta donde se generará el JS compilado
    "rootDir": "./src",                  // Carpeta donde se encuentra el código fuente
    "esModuleInterop": true,             // Habilita la interoperabilidad entre módulos ES y CommonJS
    "skipLibCheck": true                 // Omite la verificación de tipos en archivos de declaración .d.ts
  },
  "include": ["src/**/*.ts"],            // Archivos que se incluyen en la compilación
  "exclude": ["node_modules", "dist"]    // Archivos que se excluyen de la compilación
}
```

#### Propiedades clave

1. **`compilerOptions`**: Configura el comportamiento del compilador. Aquí se encuentran la mayoría de las opciones de compilación, como el destino de ECMAScript, el tipo de módulo, la generación de archivos de declaración, etc.
2. **`include`** y **`exclude`**: Define qué archivos o carpetas deben incluirse o excluirse en la compilación.
3. **`files`**: Otra manera de definir archivos específicos a incluir, aunque menos usada porque se suele preferir `include`.

### `tsconfig.json` "global"

Dado que tienes una estructura modular con tres carpetas principales (core, infrastructure y backend-express), y que backend-express es la que tiene el servidor y depende de las otras dos, lo ideal sería usar **configuración por proyectos** en TypeScript para mantener cada parte de la aplicación independiente y bien modularizada.

#### Enfoque con múltiples `tsconfig.json` (por proyectos)

Podrías tener un `tsconfig.json` en la raíz del proyecto, que sirva como base o configuración común para los tres módulos, y luego tener un `tsconfig.json` específico en cada carpeta (`core`, `infrastructure`, `backend-express`), ajustado a las necesidades de cada una.

#### 1. `tsconfig.base.json` en la raíz

Este archivo contendrá la configuración general, compartida entre todas las carpetas, como opciones de compilador comunes:

```json
{
  "compilerOptions": {
    "target": "ES6",                    // Todas las partes compilan a ES6
    "module": "CommonJS",                // Utiliza CommonJS como módulo
    "strict": true,                      // Modo estricto de verificación
    "esModuleInterop": true,             // Interoperabilidad entre CommonJS y ES6
    "skipLibCheck": true,                // Omitir verificación de tipos en .d.ts
    "forceConsistentCasingInFileNames": true // Consistencia en los nombres de archivo
  }
}
```

#### 2. `tsconfig.json` para **core** y **infrastructure**

Tanto `core` como `infrastructure` solo contienen lógica de negocio o clases, sin un servidor. Aquí su `tsconfig.json` puede extender el archivo base, pero puedes especificar la carpeta fuente (`src`) y la carpeta de salida de compilación (`dist`):

```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",                 // Código fuente de esta carpeta
    "outDir": "./dist"                  // Archivos JS compilados aquí
  },
  "include": ["src/**/*.ts"],           // Incluir solo archivos TypeScript de la carpeta src
  "exclude": ["node_modules"]
}
```

#### 3. `tsconfig.json` para **backend-express**

Para **backend-express**, que es el servidor, también extenderás el archivo base. Sin embargo, aquí necesitarás importar funciones y clases de `core` e `infrastructure`. Para eso, usarás **path mappings** o referencias de proyecto, permitiendo que backend-express conozca dónde encontrar los módulos de las otras dos carpetas:

```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src", 
    "outDir": "./dist", 
    "baseUrl": ".",                    // Configura el punto base para importaciones
    "paths": {
      "@core/*": ["../core/src/*"],     // Mapea los archivos de core
      "@infrastructure/*": ["../infrastructure/src/*"] // Mapea los archivos de infrastructure
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules"]
}
```

Con `baseUrl` y `paths`, ahora puedes importar archivos de `core` e `infrastructure` usando rutas como:

```ts
import { MyClass } from "@core/my-module";
```

#### 4. Ejecutar los diferentes proyectos

Finalmente, para ejecutar cada parte del proyecto, puedes compilar los módulos usando `tsc -p`:

- Para compilar **core**:
  ```bash
  tsc -p core/tsconfig.json
  ```

- Para compilar **infrastructure**:
  ```bash
  tsc -p infrastructure/tsconfig.json
  ```

- Para compilar **backend-express**:
  ```bash
  tsc -p backend-express/tsconfig.json
  ```

También puedes automatizar esta compilación agregando scripts en un `package.json` principal.

#### Beneficios de este enfoque:
1. **Modularización clara**: Cada carpeta tiene su propia configuración independiente.
2. **Path mappings**: Facilita las importaciones cruzadas sin tener que usar rutas relativas complejas.
3. **Separación de responsabilidades**: El `core` y `infrastructure` son completamente independientes del servidor (`backend-express`), lo que hace más fácil el mantenimiento y pruebas individuales.

### ¿Porque sin `tsconfig.json` "global"?

Tienes razón en que **no es estrictamente necesario tener un `tsconfig.json` en la raíz** si cada subproyecto (core, infrastructure, backend-express) tiene su propio `tsconfig.json` bien configurado. Sin embargo, tener un `tsconfig.base.json` o un archivo de configuración en la raíz puede ofrecer algunas **ventajas** importantes:

### Ventajas de tener un `tsconfig.base.json` en la raíz:

1. **Consistencia**: 
   Al tener un archivo base, puedes definir las configuraciones comunes una vez (por ejemplo, el target, el módulo, o las reglas de strict mode), y luego extenderlo en los archivos `tsconfig.json` específicos de cada carpeta. Esto asegura que todas las partes del proyecto comparten las mismas reglas de compilación, sin tener que repetir las mismas opciones en cada carpeta.
   
   Ejemplo:
   ```json
   "target": "ES6",
   "module": "CommonJS",
   "strict": true
   ```

2. **Facilita el mantenimiento**: 
   Si en algún momento necesitas cambiar una configuración común, solo tendrías que modificarla en un solo lugar (en el archivo base) en lugar de hacerlo en cada archivo `tsconfig.json` individual de core, infrastructure y backend-express.

3. **Evitar duplicación de código**:
   Cada `tsconfig.json` de las carpetas podría volverse redundante si repites siempre la misma configuración. Usar un archivo base simplifica los archivos específicos de cada carpeta, reduciendo las opciones necesarias a las que son realmente particulares de cada módulo.

4. **Escalabilidad**:
   En proyectos grandes, si agregas más módulos, todos pueden extender el `tsconfig.base.json` y no tendrás que preocuparte por configurar cada nuevo subproyecto desde cero. Esto es útil especialmente en una arquitectura modular o monorepo.

5. **Control centralizado de opciones**:
   Si tienes varias configuraciones del compilador que afectan a todo el proyecto (como el nivel de verificación de tipos o la versión de ECMAScript a la que estás compilando), puedes asegurarte de que todas las partes del proyecto siguen esas reglas desde un solo punto.

### ¿Es obligatorio? No, pero:

Si prefieres tener configuraciones completamente independientes y más control directo en cada subproyecto, **puedes omitir el `tsconfig.json` de la raíz**. Cada carpeta podrá seguir teniendo su propio `tsconfig.json` autónomo. Sin embargo, al hacerlo, introduces el riesgo de que en algún momento las configuraciones de cada carpeta se desincronicen o se configuren de manera inconsistente.

### Caso sin `tsconfig.base.json` (configuración totalmente independiente):

Cada carpeta tendría su propio `tsconfig.json` completamente autónomo, sin extensión:

- **`core/tsconfig.json`**:

  ```json
  {
    "compilerOptions": {
      "target": "ES6",
      "module": "CommonJS",
      "strict": true,
      "rootDir": "./src",
      "outDir": "./dist"
    },
    "include": ["src/**/*.ts"],
    "exclude": ["node_modules"]
  }
  ```

- **`infrastructure/tsconfig.json`**:

  ```json
  {
    "compilerOptions": {
      "target": "ES6",
      "module": "CommonJS",
      "strict": true,
      "rootDir": "./src",
      "outDir": "./dist"
    },
    "include": ["src/**/*.ts"],
    "exclude": ["node_modules"]
  }
  ```

- **`backend-express/tsconfig.json`**:

  ```json
  {
    "compilerOptions": {
      "target": "ES6",
      "module": "CommonJS",
      "strict": true,
      "rootDir": "./src",
      "outDir": "./dist",
      "baseUrl": ".",
      "paths": {
        "@core/*": ["../core/src/*"],
        "@infrastructure/*": ["../infrastructure/src/*"]
      }
    },
    "include": ["src/**/*.ts"],
    "exclude": ["node_modules"]
  }
  ```

---

### Conclusión:
- Si quieres **evitar duplicación de código** y mantener la consistencia fácilmente, te convendría usar un archivo base en la raíz (`tsconfig.base.json`).
- **Si prefieres que cada módulo sea completamente autónomo, no necesitas el `tsconfig.json` en la raíz, aunque perderías algo de flexibilidad y centralización.**
