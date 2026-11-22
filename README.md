# Inventario Tienda

[![CI](https://github.com/wulflo/StoreInventory/actions/workflows/ci.yml/badge.svg)](https://github.com/wulflo/StoreInventory/actions/workflows/ci.yml)
[![Release](https://github.com/wulflo/StoreInventory/actions/workflows/release.yml/badge.svg)](https://github.com/wulflo/StoreInventory/actions/workflows/release.yml)
[![Versión](https://img.shields.io/badge/version-0.4.0-green.svg)](package.json)
[![Licencia: MIT](https://img.shields.io/badge/licencia-MIT-blue.svg)](LICENSE)

> Gestión de inventario **local-first** para emprendimientos de indumentaria (remeras y buzos). Sin base de datos SQL, sin cuentas y sin servidor: los datos viven en una carpeta que eliges tú.

**Stack:** Tauri 2 · TypeScript vanilla · Vite · Rust · Event Sourcing (JSONL append-only)

## Contenido

- [Por qué local-first](#-por-qué-local-first)
- [Funcionalidades](#-funcionalidades)
- [Inicio rápido](#-inicio-rápido)
- [Cómo viven tus datos](#-cómo-viven-tus-datos)
- [Descargas y actualizaciones](#-descargas-y-actualizaciones)
- [Desarrollo](#-desarrollo)
- [Limitaciones conocidas](#-limitaciones-conocidas)
- [Documentación](#-documentación)
- [Licencia](#-licencia)

## 🎯 Por qué local-first

| | |
|---|---|
| **Sin dependencias externas** | Funciona 100 % offline. No hay backend, login ni suscripción. |
| **Historial inmutable** | Cada cambio es un evento; el estado se deriva, nunca se sobrescribe. |
| **Cifrado por diseño** | Cada `payload` se cifra con XChaCha20-Poly1305; la clave la desbloqueas con contraseña y código de recuperación. |
| **Tú eliges dónde** | La bóveda puede estar en disco local, en una unidad o en una carpeta sincronizada por otro agente. |

## ✨ Funcionalidades

### Catálogo

- Colecciones con temporada/año y resumen de ventas, unidades producidas y margen.
- Productos con SKU, tipo (remera/buzo), precio y costos (tela, impresión, mano de obra, otros).
- Colores con código hexadecimal.
- Variantes generadas automáticamente en la grilla **talle × color** (XS, S, M, L, XL, XXL, 3XL).
- Stock mínimo por variante con alertas.

### Pedidos

- Alta con cliente, líneas (variante + cantidad), fecha de entrega y notas.
- Estados: pendiente, en producción, listo, entregado, cancelado.
- Pagos parciales y estados de pago: sin pago, seña, parcial, pagado.
- Entrega total o parcial por línea (genera movimiento de stock de "venta").
- Validación de stock disponible con advertencia: se permite vender sin stock y se marca *a producir*.
- Envío por WhatsApp y impresión del detalle del pedido.

### Producción

- Órdenes de producción con líneas (variante + cantidad planificada).
- Estados: planificada, en curso, completada, cancelada.
- Al completar, se genera el movimiento de stock de "producción".
- Vinculación automática de lo producido con las líneas de pedidos abiertos.
- Sugerencias de producción según pedidos pendientes vs. stock disponible.

### Stock

- Stock derivado de movimientos inmutables (*append-only*): total, reservado (pedidos no cancelados) y disponible.
- Tipos de movimiento: producción, venta, ajuste positivo/negativo, devolución e inicial.
- Historial filtrable por tipo, fecha y búsqueda, con alertas de stock mínimo.

### Clientes y gastos

- **Clientes:** listado y detalle con contacto (teléfono, email, dirección, notas), historial de pedidos, saldo y acceso directo a WhatsApp. Borrado lógico (archivado) si tiene pedidos.
- **Gastos:** registro con fecha, categoría, descripción, monto y medio de pago; resumen por categoría y período; edición y eliminación.

### Reportes

- Ingresos, utilidad bruta, gastos y utilidad neta por período (mes, trimestre, año o rango personalizado).
- Desglose por colección y productos más vendidos.
- Gastos por categoría y producción del período.

### Datos y seguridad

- **Cifrado obligatorio:** DEK aleatoria de 32 bytes por evento con XChaCha20-Poly1305; la DEK se envuelve (Argon2id) con una contraseña maestra **y** un código de recuperación de un solo uso. Los metadatos quedan en texto plano para poder ordenarlos. La keyring vive en `<bóveda>/omega/keys/keyring.json`; la clave desbloqueada solo reside en RAM. La misma contraseña/código abre la bóveda en otro equipo. Cambiar la contraseña reenvuelve la DEK; rotar la clave vuelve a cifrar todo.
- **Bóveda elegida por el usuario:** datos en `<bóveda>/omega/`; copias diarias en `<bóveda>/longlife/<dispositivo>/<fecha>/`, con 7 días de retención por dispositivo. La copia del día se actualiza minutos después de cada cambio y al cerrar o pasar a segundo plano.
- **Respaldos pre-cifrado:** al activar o rotar el cifrado se escriben copias en texto plano en `omega/backups/pre-encryption-*`. Quedan fuera de *longlife* y se pueden revisar/borrar desde Ajustes > Cifrado.
- **Borrado y restauración:** antes de vaciar los eventos se refresca la snapshot del día; el refresco automático se niega a reemplazar una snapshot no vacía por un `omega` vacío. Restaurar valida todos los hashes, crea una copia `pre-restore-*`, limpia el journal local y reemplaza `omega` con bloqueo de escritura. La app recarga bloqueada para desbloquear la keyring restaurada.
- **Identidad de dispositivo:** `device_id` pertenece a la instalación local (fuera de la bóveda); un *heartbeat* en `omega/writer.json` avisa si otro equipo abrió la bóveda recientemente. La regla de escritor único es manual.
- **Exportar/importar:** respaldo JSON de todos los eventos (en la app nativa los `payload` se exportan tal cual están cifrados; en el store de desarrollo del navegador se exportan descifrados) e importación inicial de CSV de productos, clientes y pedidos con mapeo de columnas.
- **Sincronización externa opcional:** la bóveda se puede compartir con un agente de sincronización o nube. La app tolera copias de conflicto, pero no resuelve conflictos semánticos entre ediciones.

### Semántica de datos

- Identificadores **UUID v7** (ordenables por tiempo, sin autoincremento).
- `created_at`, `updated_at`, `deleted_at` y `device_id` en cada entidad; los eventos conservan el dispositivo que los escribió.
- Dinero siempre en **centavos de ARS** (enteros). La moneda de visualización (ARS/USD/EUR con cotización manual) solo cambia el formato.
- Cambios como eventos inmutables; el estado se deriva.
- Conflictos: movimientos siempre se fusionan; campos editables con *last-write-wins* por campo e HLC.

## 🚀 Inicio rápido

### Requisitos

- Node 20+ y npm
- Rust (para Tauri)
- Android Studio + NDK (solo para compilar Android)

### Ejecutar

```bash
npm ci                 # instalar dependencias
npm run dev            # frontend en http://127.0.0.1:1420
npm run app:dev        # app nativa con Tauri
npm test               # pruebas unitarias (Vitest)
npm run typecheck      # verificación de tipos
```

La interfaz es **solo en español** por ahora (el selector de inglés se retiró hasta que existan traducciones).

## 🗄️ Cómo viven tus datos

```
<bóveda>/
├── omega/                     # estado canónico (eventos JSONL)
│   ├── keys/keyring.json      # DEK envuelta
│   └── writer.json            # heartbeat del dispositivo
└── longlife/                  # snapshots diarios
    └── <dispositivo>/<fecha>/
```

- **Desktop:** el usuario elige la carpeta bóveda.
- **Android:** el usuario elige la carpeta mediante SAF.
- No hay sincronización P2P/LAN integrada. Para compartir entre equipos, usa un agente externo sobre la carpeta y **no edites en dos equipos a la vez**.

## 📦 Descargas y actualizaciones

Los binarios se publican en [Releases](https://github.com/wulflo/StoreInventory/releases/latest):

| Plataforma | Formatos |
|---|---|
| Linux | `.deb`, `.AppImage` |
| Windows | `.msi`, `.nsis` |
| Android | `.apk`, `.aab` |

La búsqueda e instalación de actualizaciones están firmadas con **minisign**. En escritorio se usa el actualizador de Tauri; en Android se descarga el APK, se verifica la firma y se guarda en **Descargas** para instalarlo manualmente.

## 🛠️ Desarrollo

| Comando | Descripción |
|---|---|
| `npm run dev` | Servidor Vite en `127.0.0.1:1420` |
| `npm run build` | Typecheck + build de producción |
| `npm run app:dev` / `npm run app:build` | Desarrollo / build de Tauri |
| `npm test` / `npm run test:watch` | Pruebas (una vez / en watch) |
| `npm run release` | Flujo de versión |
| `npm run version:map` | Verificar el mapa de versiones |

Los workflows de CI y release se ejecutan en GitHub Actions; los artefactos, notas y `latest.json` se generan en el pipeline de release.

## ⚠️ Limitaciones conocidas

- La UI es solo en español; las traducciones están pendientes.
- Multimoneda es **solo visual**, con cotización manual ARS por unidad; todo se almacena en centavos de ARS.
- La regla de escritor único es manual: la app avisa si otro equipo tocó la bóveda, pero no fusiona conflictos.
- Falta la prueba física en Android con un agente de sincronización externo.

El detalle completo y verificable está en [`docs/LIMITACIONES_CHECKLIST.md`](docs/LIMITACIONES_CHECKLIST.md) y [`docs/CONTINUIDAD_CHECKLIST.md`](docs/CONTINUIDAD_CHECKLIST.md).

## 📚 Documentación

| Documento | Contenido |
|---|---|
| [Manual de implementación](docs/IMPLEMENTATION_MANUAL.md) | Detalles completos de implementación |
| [Checklist de limitaciones](docs/LIMITACIONES_CHECKLIST.md) | Problemas conocidos y verificación offline |
| [Checklist de continuidad](docs/CONTINUIDAD_CHECKLIST.md) | Estado de continuidad del proyecto |
| [Progreso](docs/PROGRESO.md) | Historial de avances |
| [Refactor de bóveda](docs/REFACTOR_VAULT_CHECKLIST.md) | Checklist del refactor de almacenamiento |
| [Robustez de sincronización](docs/SYNC_ROBUSTNESS_CHECKLIST.md) | Checklist de sincronización |

## 📄 Licencia

[MIT](LICENSE)
