---
title: Code Ghostwriter
description: Genera automáticamente documentación completa y comentarios para bases de código existentes mediante análisis estático y plantillas.
version: "1.0.0"
author: OpenClaw Engineering
tags:
  - documentation
  - writing
  - automation
maintainer: ops@openclaw.io
dependencies:
  - openclaw-core>=2.0.0
  - tree-sitter>=0.20.0
  - jinja2>=3.0.0
---

# OpenClaw Skill: Code Ghostwriter

## Propósito

Code Ghostwriter aborda el problema común de las bases de código no documentadas o deficientemente documentadas. Inyecta automáticamente comentarios de documentación y docstrings limpios, consistentes y compatibles con el estilo en archivos fuente existentes sin alterar la lógica subyacente.

Casos de uso reales:

- **Modernización de código legado**: Agregar documentación a bases de código antiguas que carecen de comentarios.
- **Generación de documentación de API**: Crear docs en estilo JSDoc/Sphinx/Google para APIs públicas.
- **Incorporación de equipo**: Mejorar la legibilidad del código para nuevos desarrolladores que se unen a un proyecto.
- **Cumplimiento normativo**: Satisfacer requisitos de documentación para industrias reguladas.
- **Integración CI/CD**: Generar documentación automáticamente como parte del pipeline de build.

## Alcance

Code Ghostwriter proporciona los siguientes comandos CLI bajo el namespace `openclaw code-ghostwriter`:

### `openclaw code-ghostwriter scan <path>>
Escanea el directorio o archivo especificado para identificar elementos no documentados.

Opciones:
- `--language=<lang>`: Filtrar por lenguaje (python, javascript, typescript, go, rust, java). Por defecto: detección automática.
- `--exclude=<pattern>`: Excluir archivos que coincidan con el patrón glob (ej., `tests/*`, `*.d.ts`).
- `--min-coverage=<pct>`: Solo reportar archivos con cobertura de documentación por debajo del umbral (0-100). Por defecto: 0.
- `--output=<format>`: Formato de salida (`json`, `yaml`, `text`). Por defecto: `text`.

Ejemplo:
```bash
openclaw code-ghostwriter scan src/ --language=python --min-coverage=50 --output=json
```

### `openclaw code-ghostwriter generate [--apply]`
Genera documentación para la base de código escaneada.

Opciones:
- `--format=<style>`: Estilo de documentación (google, numpy, sphinx, jsdoc, java). Por defecto: detección automática del proyecto.
- `--dry-run`: Mostrar diff sin modificar archivos.
- `--backup=<suffix>`: Crear archivos de backup con el sufijo dado antes de aplicar. Por defecto: `.ghostwriter.bak`.
- `--max-line-length=<cols>`: Ajustar docstrings al ancho de columna. Por defecto: 88 (PEP 257).
- `--include-tests`: Generar docs también para funciones de test.

Ejemplo:
```bash
openclaw code-ghostwriter generate --format=google --dry-run
openclaw code-ghostwriter generate --apply --backup=.bak
```

### `openclaw code-ghostwriter revert`
Revierte la generación de documentación más reciente aplicada por este skill.

Opciones:
- `--commit`: Si el repositorio está bajo Git, crear automáticamente un commit de reversión.
- `--files=<list>`: Revertir solo archivos específicos (separados por comas).

Ejemplo:
```bash
openclaw code-ghostwriter revert --commit
```

### `openclaw code-ghostwriter config`
Gestionar perfiles de configuración.

Subcomandos:
- `config set <key>=<value>`: Establecer una opción de configuración.
- `config get <key>`: Obtener un valor de configuración.
- `config list`: Mostrar toda la configuración.
- `config load <profile>`: Cargar un perfil predefinido (ej., `strict`, `fast`, `minimal`).

La configuración global se almacena en `~/.openclaw/code-ghostwriter/config.json`.

## Proceso de Trabajo

1. **Descubrimiento**
   - El skill recorre recursivamente el árbol de directorios proporcionado.
   - Identifica archivos fuente por extensión (`.py`, `.js`, `.ts`, `.go`, `.rs`, `.java`).
   - La detección de lenguaje usa tanto la extensión del archivo como la línea shebang.

2. **Análisis**
   - Para cada archivo, un parser específico de lenguaje (Tree-sitter) construye un Árbol de Sintaxis Abstracta (AST).
   - El AST se recorre para extraer definiciones: funciones, clases, métodos, módulos.
   - Se detectan docstrings/comentarios existentes y se infiere su estilo.

3. **Análisis de Cobertura**
   - Para cada definición, se determina si la documentación está presente y es suficiente.
   - La cobertura se calcula como (elementos documentados / total de elementos) * 100.
   - Se genera un reporte mostrando cobertura por archivo y elementos faltantes.

4. **Generación**
   - Para elementos no documentados, un generador de documentación produce cadenas de comentario:
     - Python: Genera docstrings en el estilo elegido (Google/Napoleon, NumPy, Sphinx, reST).
     - JavaScript/TypeScript: Genera bloques JSDoc con `@param`, `@returns`, `@throws`.
     - Go: Genera comentarios GoDoc precediendo la declaración.
     - Rust: Genera comentarios de documentación `///` con ejemplos.
     - Java: Genera estilo Javadoc `/** ... */`.
   - El generador usa plantillas y, opcionalmente, asistencia de IA (cuando se conecta a un LLM) para inferir descripciones de parámetros a partir de nombres y tipos.

5. **Validación**
   - Los docs generados se verifican para cumplir con la guía de estilo seleccionada.
   - Se verifica longitud y completitud (ej., todos los parámetros documentados, tipo de retorno descrito).
   - Opcionalmente, un linter (ej., `pydocstyle`, `eslint-plugin-jsdoc`) se ejecuta en modo dry-run para detectar problemas.

6. **Aplicación**
   - En modo `--apply`, el skill inserta los comentarios generados en los archivos originales.
   - La inserción ocurre en la ubicación correcta (inmediatamente antes de la definición de función/clase).
   - Los archivos se reescriben atómicamente: escribir en archivo temporal y luego renombrar.
   - Si se establece `--backup`, se copian los archivos originales antes de la modificación.

7. **Chequeos post-aplicación**
   - El skill puede opcionalmente ejecutar un build/lint para asegurar que no se introdujeron errores de sintaxis.
   - Se imprime un resumen mostrando total de archivos modificados, adiciones y mejora de cobertura.

## Reglas de Oro

1. **Nunca alterar la semántica del código** – El skill solo agrega comentarios; no debe reformatear, renombrar o mover código.
2. **Preservar documentación existente** – Si una función ya tiene un docstring, no sobrescribir a menos que se use explícitamente el flag `--force`.
3. **Respetar convenciones del proyecto** – Detectar automáticamente el estilo desde docs existentes; fallback a configuración del proyecto (`.docstyle`, `tsconfig.json`, `pyproject.toml`).
4. **Política de archivos de test** – Por defecto, omitir `test_*.py`, `*.spec.js`, directorios `__tests__` a menos que se pase `--include-tests`.
5. **Escrituras atómicas** – Siempre escribir en un archivo temporal y renombrar para evitar escrituras parciales en caso de interrupción.
6. **Backup antes de aplicar** – A menos que se desactive con `--no-backup`, mantener una copia del archivo original con sufijo `.ghostwriter.bak`.
7. **Sin alucinación de IA** – Al usar IA para generar descripciones, restringirse a nombres y tipos de parámetros; nunca inventar comportamiento no evidente a partir del código.
8. **Codificaciones** – Preservar la codificación original del archivo (detectar BOM, usar UTF-8 por defecto).
9. **Finales de línea** – Preservar los finales de línea originales (CRLF vs LF).
10. **Conciencia de Git** – Si se detecta un repositorio Git, preparar cambios solo de documentación separadamente de cambios de código para facilitar la revisión.

## Ejemplos

### Ejemplo 1: Generación básica de docstrings en Python

**Antes** (`src/utils.py`):
```python
def format_currency(amount, currency='USD'):
    if currency == 'USD':
        return f'${amount:.2f}'
    elif currency == 'EUR':
        return f'€{amount:.2f}'
    return str(amount)
```

**Comando**:
```bash
openclaw code-ghostwriter scan src/utils.py
openclaw code-ghostwriter generate --format=google --dry-run
```

**Salida (diff)**:
```diff
--- a/src/utils.py
+++ b/src/utils.py
@@ -1,6 +1,13 @@
 def format_currency(amount, currency='USD'):
+    \"\"\"Formats a numeric amount as a currency string.
+
+    Args:
+        amount (float): The monetary amount to format.
+        currency (str): Currency code, e.g., 'USD', 'EUR'. Defaults to 'USD'.
+
+    Returns:
+        str: Formatted currency string with symbol.
+    \"\"\"
     if currency == 'USD':
         return f'${amount:.2f}'
     elif currency == 'EUR':
```

**Aplicar**:
```bash
openclaw code-ghostwriter generate --apply
```

### Ejemplo 2: JSDoc en JavaScript/TypeScript con inferencia de tipos

**Antes** (`lib/math.ts`):
```typescript
export function sum(...nums: number[]): number {
    return nums.reduce((a, b) => a + b, 0);
}
```

**Comando**:
```bash
openclaw code-ghostwriter generate --language=typescript --format=jsdoc
```

**Después** (`lib/math.ts`):
```typescript
/**
 * Calculates the sum of provided numbers.
 *
 * @param nums - Rest parameter of numbers to sum.
 * @returns The total sum as a number.
 */
export function sum(...nums: number[]): number {
    return nums.reduce((a, b) => a + b, 0);
}
```

### Ejemplo 3: Documentación en Go

**Antes** (`cmd/server.go`):
```go
func startServer(port int, env string) error {
    // ...
}
```

**Comando**:
```bash
openclaw code-ghostwriter generate --language=go
```

**Después**:
```go
// startServer initializes and starts the HTTP server.
//
// port is the TCP port to listen on. env is the environment name (dev/staging/prod).
// Returns an error if the server fails to start.
func startServer(port int, env string) error {
    // ...
}
```

## Rollback

Si la generación de documentación produce resultados no deseados, revertir usando:

```bash
# Revertir la operación de aplicación más reciente (usa backups o Git)
openclaw code-ghostwriter revert
```

Si el repositorio usa Git y los cambios se commitearon automáticamente (con `--commit`), revertir via:

```bash
openclaw code-ghostwriter revert --commit
# Equivalente a: git revert HEAD
```

Si necesitas revertir archivos específicos manualmente:
```bash
# Restaurar desde backup
cp src/module.py.ghostwriter.bak src/module.py
# O si se usó Git y aún no has commiteado:
git checkout -- src/module.py
```

El skill almacena metadatos de los cambios aplicados en `~/.openclaw/code-ghostwriter/last_run.json`. Asegúrate que este archivo exista antes de revertir.

## Dependencias y Requisitos

- **Python**: 3.8 o superior.
- **openclaw-core**: El núcleo del framework OpenClaw (se instala automáticamente con el skill).
- **tree-sitter**: Biblioteca de análisis multi-lenguaje (se descarga automáticamente).
- **Jinja2**: Motor de plantillas para formatos de docstring personalizados.
- **IA Opcional**: Integración con proveedores de LLM (OpenAI, Anthropic, Ollama local) para descripciones más ricas; configurar `CODE_GHOSTWRITER_LLM_API_KEY` y `CODE_GHOSTWRITER_LLM_MODEL`.
- **Herramientas del sistema**: Se recomienda `git` para rollback fácil; `sed`, `awk` para procesamiento (usados internamente).

Instalación:
```bash
openclaw skill install code-ghostwriter
```

Configuración:
```bash
openclaw code-ghostwriter config set llm.provider=ollama
openclaw code-ghostwriter config set llm.model=llama3
openclaw code-ghostwriter config set style.default=google
```

## Verificación

Después de generar documentación, verificar la calidad:

1. **Chequeo de cobertura**:
   ```bash
   openclaw code-ghostwriter scan . --min-coverage=80
   ```
   Debería mostrar al menos 80% de funciones públicas documentadas.

2. **Validación de estilo**:
   ```bash
   # Para Python
   pydocstyle src/
   # Para JavaScript
   npx eslint src/ --plugin jsdoc
   ```

3. **Integridad de build**:
   ```bash
   # Asegurar que no se introdujeron errores de sintaxis
   python -m py_compile src/*.py
   npm run build
   ```

4. **Test funcional** (si aplica):
   Ejecutar tests unitarios para confirmar que no hay cambio de comportamiento:
   ```bash
   pytest
   npm test
   ```

## Solución de Problemas

### Problema: Error "Unsupported language"
- **Causa**: Extensión de archivo no reconocida o parser faltante.
- **Solución**: Instalar gramáticas Tree-sitter adicionales. Lista disponible: `tree-sitter --list-languages`. Usar `--language` para forzar o instalar el parser faltante via npm/pip.

### Problema: Estilo de docstring incorrecto aplicado
- **Causa**: La detección automática falló o el proyecto carece de configuración.
- **Solución**: Establecer explícitamente el estilo con `--format=google|numpy|...` o crear archivo de configuración: `.docstyle` con `style = "jsdoc"`.

### Problema: "Permission denied" al escribir archivos
- **Causa**: Archivos de salida son de solo lectura o pertenecen a otro usuario.
- **Solución**: Ajustar permisos de archivo: `chmod u+w <file>`. Asegurar que eres dueño de los archivos o usar `sudo` si es apropiado (no recomendado).

### Problema: Docstrings duplicados generados
- **Causa**: Existía un docstring previo pero no se detectó debido a formato no estándar.
- **Solución**: Usar `--force` para sobrescribir existentes, o mejorar detección agregando un patrón regex personalizado en config: `code-ghostwriter config set detection.patterns='["^/\\*\\*", "^\"\"\""]'`.

### Problema: Descripciones generadas por IA son inexactas
- **Causa**: Alucinación del LLM o contexto insuficiente.
- **Solución**: Deshabilitar asistencia de IA: `code-ghostwriter config set ai.enabled=false`. O proveer mejor contexto via `--context-file=README.md`.

### Problema: Rendimiento demasiado lento en base de código grande
- **Solución**: Limitar escaneo a directorios específicos: `openclaw code-ghostwriter scan src/core/`. Aumentar paralelismo con `--jobs=N` (si está soportado).

## Soporte

Para problemas, solicitudes de features o contribuciones, visitar: https://github.com/openclaw/code-ghostwriter

Reportar logs con `openclaw code-ghostwriter scan --debug --output=log > debug.log`.
```