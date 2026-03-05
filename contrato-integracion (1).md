# Contrato de Integración — Sistema de Gestión Logística

**Versión**: 1.1  
**Fecha**: 2026-02-27  
**Elaborado por**: Equipo Módulo 2  
**Estado**: Propuesta para revisión y aprobación de los tres equipos

---

## Propósito

Este documento establece formalmente los eventos y estructuras de datos que se intercambian entre los tres módulos del sistema. Ningún equipo debe asumir información que no esté definida aquí. Cualquier cambio debe ser acordado y versionado por los tres equipos antes de implementarse.

---

## Principios Generales

- Cada módulo es independiente. Ninguno accede directamente a la base de datos de otro.
- La comunicación entre módulos ocurre únicamente a través de los eventos definidos en este documento.
- El **Módulo 2 es la única fuente de verdad** sobre el estado de un paquete desde el momento del despacho en adelante.
- El **Módulo 1 es la única fuente de verdad** sobre los datos del paquete (peso, dirección, tipo de mercancía, etc.).
- El **Módulo** 3 la información que necesita la recibe del Módulo 2 al cierre de ruta y recibe la información de los estados del paquete sobre el Módulo 1.

---

## Ciclo de Vida del Paquete y Responsabilidad por Estado

| Estado | Responsable de generarlo | Descripción |
|:---|:---|:---|
| Recibido en Sede | Módulo 1 | El paquete entra al sistema |
| En Clasificación | Módulo 1 | Se agrupa por zona de destino |
| Listo para Despacho | Módulo 1 | Paquete disponible físicamente, solicita ruta |
| En Tránsito | **Módulo 2** | Despachador confirmó despacho |
| En Parada de Entrega | **Módulo 2** | Conductor llegó al sitio de entrega |
| Entregado | **Módulo 2** | Conductor registró entrega con evidencia |
| Novedad en Bodega | **Módulo 2** | Conductor registró novedad en campo |

---

## SECCIÓN 1: Módulo 1 → Módulo 2

### Evento: `solicitar_ruta`

**Cuándo se emite**: Cada vez que un paquete cambia a estado "Listo para Despacho" en el Módulo 1.  
**Frecuencia**: Por cada paquete individual, no en lote.  
**Propósito**: Que el Módulo 2 asigne el paquete a una ruta existente o cree una nueva según la zona geográfica.

```json
{
  "evento": "solicitar_ruta",
  "fecha_hora": "2026-02-22T08:30:00",
  "paquete": {
    "id": "uuid-del-paquete",

    "peso_kg": 12.5,
    "volumen_m3": 0.08,
    "tipo_mercancia": "frágil | peligroso | estándar",

    "direccion_destino": "Calle 123 #45-67, Santa Marta",
    "latitud": 11.2408,
    "longitud": -74.1990,

    "metodo_pago": "prepago | contra_entrega",
    "valor_declarado": 350000
  }
}
```

**Campos obligatorios**: todos.  
**Nota**: Si falta algún campo, el Módulo 2 rechaza la solicitud y notifica al Módulo 1 con el detalle del error.


---

### Evento: `actualizar_estado_paquete`

**Cuándo se emite**: Cada vez que ocurre una acción en campo que cambia el estado del paquete.  
**Propósito**: Mantener al Módulo 1 actualizado en tiempo real para trazabilidad y rastreo del cliente.

Este evento se reutiliza en cuatro momentos distintos cambiando el campo `nuevo_estado` y los datos de detalle.

#### Caso 1 — En Tránsito
Se emite cuando el despachador confirma el despacho formal de la ruta.

```json
{
  "evento": "actualizar_estado_paquete",
  "fecha_hora": "2026-02-23T06:00:00",
  "paquete_id": "uuid-del-paquete",
  "nuevo_estado": "en_transito",
  "detalle": {
    "ruta_id": "uuid-de-la-ruta",
    "vehiculo_placa": "ABC123",
    "vehiculo_tipo": "Van",
    "conductor_id": "uuid-del-conductor",
    "conductor_nombre": "Juan Pérez"
  }
}
```

#### Caso 2 — En Parada de Entrega
Se emite cuando el conductor registra su llegada al sitio de entrega.

```json
{
  "evento": "actualizar_estado_paquete",
  "fecha_hora": "2026-02-23T09:15:00",
  "paquete_id": "uuid-del-paquete",
  "nuevo_estado": "en_parada_de_entrega",
  "detalle": {
    "ruta_id": "uuid-de-la-ruta",
    "conductor_id": "uuid-del-conductor",
    "latitud_actual": 11.2350,
    "longitud_actual": -74.1920
  }
}
```

#### Caso 3 — Entregado
Se emite cuando el conductor registra la entrega exitosa con evidencia POD.

```json
{
  "evento": "actualizar_estado_paquete",
  "fecha_hora": "2026-02-23T09:22:00",
  "paquete_id": "uuid-del-paquete",
  "nuevo_estado": "entregado",
  "detalle": {
    "ruta_id": "uuid-de-la-ruta",
    "conductor_id": "uuid-del-conductor",
    "firma_receptor": "base64-de-la-firma",
    "foto_evidencia_url": "url-de-la-foto",
    "nombre_receptor": "María García"
  }
}
```

#### Caso 4 — Novedad de entrega Fallida
Se emite cuando el conductor registra que no pudo completar la entrega.

```json
{
  "evento": "actualizar_estado_paquete",
  "fecha_hora": "2026-02-23T10:05:00",
  "paquete_id": "uuid-del-paquete",
  "nuevo_estado": "novedad_en_bodega",
  "detalle": {
    "ruta_id": "uuid-de-la-ruta",
    "conductor_id": "uuid-del-conductor",
    "subtipo_novedad": "dañado | extraviado | devolución",
    "motivo": "cliente_ausente | dirección_incorrecta | zona_difícil_acceso | rechazado_por_cliente | dañado_en_ruta",
    "foto_evidencia_url": "url-de-la-foto",
    "origen_novedad": "conductor | sistema"
  }
}
```


> **Nota sobre `origen_novedad`**: El valor `sistema` se usa cuando la novedad fue generada automáticamente al cierre de ruta por paradas sin gestionar, no por acción del conductor.

---

## SECCIÓN 3: Módulo 2 → Módulo 3

### Evento: `ruta_cerrada`

**Cuándo se emite**: Al momento del cierre de ruta, ya sea manual por el conductor, forzado por el despachador, o automático por tiempo.  
**Propósito**: Entregarle al Módulo 3 toda la información necesaria para calcular la liquidación del conductor sin necesidad de consultar nada más.

```json
{
  "evento": "ruta_cerrada",
  "fecha_hora_cierre": "2026-02-23T15:00:00",

  "ruta": {
    "id": "uuid-de-la-ruta",
    "fecha": "2026-02-23",
    "fecha_hora_inicio": "2026-02-23T06:00:00",
    "tipo_cierre": "manual | automático "
  },

  "vehiculo": {
    "id": "uuid-del-vehiculo",
    "placa": "ABC123",
    "tipo": "Moto | Van | NHR | Turbo"
  },

  "conductor": {
    "id": "uuid-del-conductor",
    "nombre": "Juan Pérez"
  },

  "paradas": [
    {
      "paquete_id": "uuid-del-paquete",
      "estado_final": "entregado | fallido_cliente | fallido_conductor | dañado",
      "motivo_novedad": "cliente_ausente | dirección_incorrecta | zona_difícil_acceso | rechazado_por_cliente | dañado_en_ruta | sin_gestión_conductor | cierre_automático",
      "fecha_hora_gestion": "2026-02-23T09:22:00",
      "origen": "conductor | sistema"
    }
  ]
}
```

---


## SECCIÓN 5: Parámetros Configurables

Estos valores deben ser definidos por el negocio antes del inicio del desarrollo.

| Parámetro | Descripción | Valor por definir |
|:---|:---|:---|
| `radio_zona_km` | Radio en kilómetros para agrupar paquetes en la misma ruta | 5 km |
| `hora_corte` | Hora del día en que las rutas abiertas pasan a "Lista para Despacho" | 6:00 am |
| `porcentaje_capacidad_maxima` | Porcentaje máximo de carga del vehículo | 90% ✓ |
| `tiempo_maximo_ruta_horas` | Horas máximas de una ruta activa antes del cierre automático | 12 horas |

---

## Flujo Completo Resumido

```
MÓDULO 1                        MÓDULO 2                        MÓDULO 3
────────                        ────────                        ────────
Paquete → Listo para
Despacho
    │
    └── solicitar_ruta ────────►
                                Busca ruta abierta en zona
                                Si no existe → crea ruta nueva
                                Si existe → agrega paquete
                                Acumula peso
                                    │
                                ◄── ruta_asignada ──────────────
    │
    │       [A la hora de corte]
    │                           Cierra rutas abiertas
    │                           Asigna vehículo según peso total
    │                           Ruta → "Lista para Despacho"
    │
    │       [Despachador confirma]
    │                           Ruta → "Despachada"
                                    │
                                ◄── actualizar_estado_paquete ──
                                    nuevo_estado: en_transito
    │
    │       [Conductor en sitio]
                                ◄── actualizar_estado_paquete ──
                                    nuevo_estado: en_parada_de_entrega
    │
    │       [Conductor entrega]
                                ◄── actualizar_estado_paquete ──
                                    nuevo_estado: entregado
    │
    │       [Conductor registra novedad]
                                ◄── actualizar_estado_paquete ──
                                    nuevo_estado: novedad_en_bodega
    │
    │       [Conductor cierra ruta]
                                                │
                                                └── ruta_cerrada ────────►
                                                                        Calcula
                                                                        liquidación
```

---

## Control de Cambios

| Versión | Fecha | Cambio | Autor |
|:---|:---|:---|:---|
| 1.1 | 2026-02-27 | Equipo Módulo 2 |

---

> **Importante**: Este documento debe ser revisado y aprobado por el líder de cada equipo antes de iniciar el desarrollo. Cualquier modificación posterior debe incrementar la versión y ser acordada por los tres equipos.
