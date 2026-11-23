# ESTADO · Snack_Check

Actualizado: 23 nov 2026, hora de CDMX (curaduría de ramas). Archivo corto de portabilidad: con esto se retoma el repo en cualquier herramienta.

## Qué es
SnackCheck is an n8n automation that analyzes packaged-food barcodes using Open Food Facts, applies deterministic nutrition rules and Nutri-Score logic, and uses Groq to generate a clear, shopper-friendly health verdict.

## Cómo arrancar
Ver `README.md`. Más contexto: `README.md`.

## Rama principal
`main` en `e59e353060d5096815ae7035e3385ae1cc243774` antes de este PR.

## PRs vivos
- Ninguno además de este.

## Ramas vivas o protegidas
- Solo la rama principal.
- Ramas archivadas: tags `archivo/<rama>-23nov` (ver `git tag -l "archivo/*"`).

## Secretos que no van en el repo
Llaves de API, tokens, `.env`, credenciales de nube y datos personales: nunca en Git. Se guardan fuera (gestor de secretos o variables de entorno); en el repo solo pueden ir nombres de variables o un `.env.example` sin valores.

## Dónde se retoma
Este PR (`curaduria/estado-vigente-23nov`). Integra: solo este archivo. Índice general: `/workspace/INDICE_ESTADO.md` (caja de Grok Bot).
