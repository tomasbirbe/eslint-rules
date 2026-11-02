# `@tomasbirbe/eslint-rules`

Colección de configuraciones internas de linting y formateo, basadas en
[oxlint](https://oxc.rs/docs/guide/usage/linter/) y [oxfmt](https://oxc.rs/docs/guide/usage/formatter).

> Migrado de ESLint + Prettier a oxlint + oxfmt. El nombre del paquete se mantiene por
> compatibilidad con los consumidores existentes.

## Configuraciones

| Archivo               | Export      | Para                       |
| --------------------- | ----------- | -------------------------- |
| `base.oxlintrc.json`  | `./base`    | TypeScript / JS genérico   |
| `react.oxlintrc.json` | `./react`   | React (extiende `base`)    |
| `next.oxlintrc.json`  | `./next-js` | Next.js (extiende `react`) |
| `.oxfmtrc.json`       | `./oxfmt`   | Formateo (oxfmt)           |

Las capas se encadenan con `extends`, igual que antes con `defineConfig`.

## Uso

Instalar en el proyecto consumidor:

```bash
npm install -D @tomasbirbe/eslint-rules oxlint oxfmt
```

### Linting

`extends` de oxlint resuelve **rutas**, no especificadores de paquete, así que hay que apuntar
al archivo dentro de `node_modules`.

`.oxlintrc.json` en la raíz del proyecto:

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "extends": ["./node_modules/@tomasbirbe/eslint-rules/next.oxlintrc.json"]
}
```

Cambiá `next.oxlintrc.json` por `react.oxlintrc.json` o `base.oxlintrc.json` según el proyecto.

### Formateo

oxfmt **no** soporta `extends`, así que hay dos opciones.

Apuntar al config compartido con `-c` (recomendado, sin duplicar valores):

```json
{
  "scripts": {
    "format": "oxfmt -c ./node_modules/@tomasbirbe/eslint-rules/.oxfmtrc.json",
    "format:check": "oxfmt -c ./node_modules/@tomasbirbe/eslint-rules/.oxfmtrc.json --check"
  }
}
```

O copiar los valores a un `.oxfmtrc.json` propio en la raíz, si preferís que el editor los lea
sin configuración extra.

### CI

En CI usá `oxlint --deny-warnings` y `oxfmt --check`.

## JS Plugins

Algunas reglas no tienen equivalente nativo en oxlint y se cargan como
[JS Plugins](https://oxc.rs/docs/guide/usage/linter/js-plugins.html). Vienen como
dependencias de este paquete, no hace falta instalarlas a mano:

| Plugin                             | Alias en oxlint      | Reglas                                       |
| ---------------------------------- | -------------------- | -------------------------------------------- |
| `eslint-plugin-import`             | `import-order`       | `import-order/order`                         |
| `@stylistic/eslint-plugin`         | `@stylistic`         | `@stylistic/padding-line-between-statements` |
| `eslint-plugin-check-file`         | `check-file`         | convenciones de nombres en `src/`            |
| `eslint-plugin-better-tailwindcss` | `better-tailwindcss` | orden y validez de clases de Tailwind        |

El alias `import-order` es necesario porque `import` es un nombre reservado por el plugin nativo
de oxlint. Las reglas nativas `import/named`, `import/default`, `import/namespace` e
`import/export` sí se usan desde el plugin en Rust.

## Diferencias respecto a la configuración de ESLint

- **Prettier → oxfmt.** `prettier/prettier` quedó fuera del linter: en oxlint sólo funciona como
  JS Plugin y es muy lento. El formateo ahora corre aparte con `oxfmt`. oxfmt no acepta el valor
  `endOfLine: "auto"` (su default es `"lf"`); el resto de las opciones de `prettier-config.js` se
  migró tal cual.
- **`react/jsx-sort-props` eliminada.** No existe en oxlint y es una regla puramente estética.
  Si la necesitás, cargá `eslint-plugin-react` con un alias (`react` está reservado):
  ```json
  {
    "jsPlugins": [{"name": "react-x", "specifier": "eslint-plugin-react"}],
    "rules": {"react-x/jsx-sort-props": "warn"}
  }
  ```
- **`import/no-unresolved` eliminada.** Requiere `settings["import/resolver"]`, y en oxlint 1.90
  `settings` **no se hereda a través de `extends`**, así que desde un paquete compartido daría
  falsos positivos en todo import con alias (`@/**`). TypeScript ya reporta módulos inexistentes.
  Para reactivarla, declarala junto a `settings` en el `.oxlintrc.json` del proyecto:
  ```json
  {
    "jsPlugins": [{"name": "import-order", "specifier": "eslint-plugin-import"}],
    "settings": {
      "import/resolver": {
        "typescript": {"alwaysTryTypes": true, "project": "./tsconfig.json"},
        "node": true
      }
    },
    "rules": {"import-order/no-unresolved": "error"}
  }
  ```
  y agregá `eslint-import-resolver-typescript` como dependencia.
- **`globals` → `env`.** Los globals de browser/serviceworker se expresan con
  `"env": { "browser": true, "serviceworker": true }` en lugar de listarlos uno por uno.
- **`categories.correctness` en `off`.** Para no habilitar reglas que la config de ESLint no
  tenía. Podés subir categorías (`suspicious`, `pedantic`, `perf`, `style`) en tu proyecto.

## Editor

Instalá la extensión de oxc para tu editor: https://oxc.rs/docs/guide/usage/linter/editors.html
Acordate de desactivar la extensión de ESLint para este proyecto.

## Opcionales a futuro

oxfmt puede absorber dos cosas que hoy hace el linter, si en algún momento conviene:

- `"sortImports": true` reordena imports al formatear (algoritmo tipo
  `eslint-plugin-perfectionist/sort-imports`) y reemplazaría a `import-order/order`, eliminando el
  único JS Plugin pesado. No se activó porque habría que replicar el orden de grupos exacto
  (`type, builtin, object, external, internal, parent, sibling, index` + `@/**`) y el cambio pasa
  de avisar a reescribir.
- `"sortTailwindcss": true` usa el mismo algoritmo que `prettier-plugin-tailwindcss` y
  reemplazaría a `better-tailwindcss/enforce-consistent-class-order`.
