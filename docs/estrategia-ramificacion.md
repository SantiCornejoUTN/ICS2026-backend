# Estrategia de ramificación

## Ramas permanentes
- `main`: código estable, lo que está en producción.
- `development`: integración de las funcionalidades terminadas.

## Ramas de trabajo
- `feature/nombre-funcionalidad`: una funcionalidad nueva. Sale de `development` y vuelve a `development`.
- `hotfix/descripcion`: arreglo urgente de producción. Sale de `main` y vuelve a `main` y a `development`.

## Reglas
1. No se permiten commits directos en `main` ni en `development`.
2. Todo cambio se integra mediante Pull Request.
3. Cada Pull Request requiere la aprobación de al menos una persona distinta del autor.
4. Las ramas `feature/` y `hotfix/` se eliminan después del merge.
5. Los nombres van en minúsculas, con guiones y sin espacios.

## Flujo
feature/... → development → main