# Contratos de Integración — Módulo 2: Planificación de Rutas y Flota

**Created:** 2026-03-01  
**By:** Esteban Puello, Jose Rodriguez, Laura Perez, Robert Gonzalez

---

## Contexto

Este documento define los contratos de integración entre el Módulo 2 (Planificación de Rutas y Flota) y los sistemas con los que se comunica: el Módulo 1 (Gestión de Paquetes) y el Módulo 3 (Facturación y Liquidación).

Cada contrato especifica el sentido de la comunicación, el tipo de interacción, la estructura del payload y las condiciones que lo disparan.

---

## 1. Módulo 1 → Módulo 2: Solicitud de Ruta

**Tipo:** Llamada directa con respuesta (síncrona)  
**Disparador:** Un paquete alcanza el estado `'Listo para Despacho'` en el Módulo 1  
**Descripción:** El Módulo 1 solicita al Módulo 2 que asigne el paquete a una ruta existente o cree una nueva.

### Request

```json
{
  "paquete_id": "UUID",
  "peso_kg": 12.5,
  "volumen_m3": 0.04,
  "tipo_mercancia": "FRAGIL | PELIGROSA | NORMAL",
  "latitud_destino": 11.0041,
  "longitud_destino": -74.8070,
  "direccion_destino": "Calle 45 # 32-10, Barranquilla",
  "fecha_limite_entrega": "2026-03-08T08:00:00"
}
```

### Response — Éxito

```json
{
  "resultado": "OK",
  "ruta_id": "UUID",
  "zona_geografica": "Norte",
  "estado_ruta": "EN_ESPERA | LISTA_PARA_DESPACHO",
  "fecha_limite_despacho": "2026-03-06T08:00:00"
}
```

### Response — Error

```json
{
  "resultado": "ERROR",
  "codigo": "CAMPO_FALTANTE | FECHA_VENCIDA | PESO_EXCEDE_CAPACIDAD_MAXIMA",
  "detalle": "Descripción del error"
}
```

### Notas
- El Módulo 2 retorna `fecha_limite_despacho` para que el Módulo 1 pueda tener trazabilidad del plazo de la ruta.
- Si `fecha_limite_entrega` ya venció al recibir la solicitud, el Módulo 2 rechaza con código `FECHA_VENCIDA`.
- Si el peso excede la capacidad del Camión Turbo (4,500 kg), el Módulo 2 rechaza con código `PESO_EXCEDE_CAPACIDAD_MAXIMA` y notifica al Despachador Logístico.

---

## 2. Módulo 2 → Módulo 1: Eventos de Estado de Paquete

**Tipo:** Evento asíncrono (sin respuesta esperada)  
**Descripción:** El Módulo 2 notifica al Módulo 1 cada vez que el estado de un paquete cambia como resultado de la operación de campo o del despacho.

Todos los eventos comparten la siguiente estructura base y se diferencian por el campo `tipo_evento`.

### Estructura base

```json
{
  "tipo_evento": "string",
  "paquete_id": "UUID",
  "ruta_id": "UUID",
  "fecha_hora_evento": "2026-03-06T14:32:00"
}
```

---

### Evento 1 — Paquete en Tránsito

**Disparador:** El conductor confirma el inicio del tránsito desde su dispositivo  
**Acción esperada en M1:** Cambiar estado del paquete a `'En Tránsito'`

```json
{
  "tipo_evento": "PAQUETE_EN_TRANSITO",
  "paquete_id": "UUID",
  "ruta_id": "UUID",
  "fecha_hora_evento": "2026-03-06T07:45:00"
}
```

---

### Evento 2 — Paquete Entregado

**Disparador:** El conductor registra entrega exitosa con POD (foto + firma)  
**Acción esperada en M1:** Cambiar estado del paquete a `'Entregado'`

```json
{
  "tipo_evento": "PAQUETE_ENTREGADO",
  "paquete_id": "UUID",
  "ruta_id": "UUID",
  "fecha_hora_evento": "2026-03-06T10:15:00",
  "evidencia": {
    "url_foto": "https://storage/.../foto.jpg",
    "url_firma": "https://storage/.../firma.png"
  }
}
```

---

### Evento 3 — Parada Fallida

**Disparador:** El conductor registra una parada fallida con motivo  
**Acción esperada en M1:** Actualizar estado del paquete y gestionar según motivo

```json
{
  "tipo_evento": "PARADA_FALLIDA",
  "paquete_id": "UUID",
  "ruta_id": "UUID",
  "fecha_hora_evento": "2026-03-06T11:00:00",
  "motivo": "CLIENTE_AUSENTE | DIRECCION_INCORRECTA | RECHAZADO_CLIENTE | ZONA_DIFICIL_ACCESO",
  "numero_intento": 1
}
```

---

### Evento 4 — Novedad Grave

**Disparador:** El conductor registra un paquete dañado, extraviado o que requiere devolución  
**Acción esperada en M1:** Activar protocolo correspondiente (seguro o devolución al origen)

```json
{
  "tipo_evento": "NOVEDAD_GRAVE",
  "paquete_id": "UUID",
  "ruta_id": "UUID",
  "fecha_hora_evento": "2026-03-06T09:30:00",
  "tipo_novedad": "DAÑADO | EXTRAVIADO | DEVOLUCION"
}
```

---

### Evento 5 — Estado Final al Cierre de Ruta

**Disparador:** Cierre de ruta (manual, automático o forzado por despachador)  
**Acción esperada en M1:** Registrar estado final de cada paquete que no haya sido notificado previamente

```json
{
  "tipo_evento": "CIERRE_RUTA_ESTADO_FINAL",
  "ruta_id": "UUID",
  "tipo_cierre": "MANUAL | AUTOMATICO | FORZADO_DESPACHADOR",
  "fecha_hora_evento": "2026-03-06T18:00:00",
  "paquetes": [
    {
      "paquete_id": "UUID",
      "estado_final": "ENTREGADO | FALLIDO | NOVEDAD | SIN_GESTION_CONDUCTOR",
      "motivo": "string | null"
    }
  ]
}
```

---

## 3. Módulo 2 → Módulo 3: Evento de Cierre de Ruta

**Tipo:** Evento asíncrono (sin respuesta esperada)  
**Disparador:** Cierre de ruta (manual, automático o forzado por despachador)  
**Descripción:** El Módulo 2 envía al Módulo 3 el resumen completo de la ruta para que calcule la liquidación del conductor.

```json
{
  "tipo_evento": "RUTA_CERRADA",
  "ruta_id": "UUID",
  "tipo_cierre": "MANUAL | AUTOMATICO | FORZADO_DESPACHADOR",
  "fecha_hora_inicio_transito": "2026-03-06T07:45:00",
  "fecha_hora_cierre": "2026-03-06T18:00:00",
  "conductor": {
    "conductor_id": "UUID",
    "nombre": "Juan Pérez"
  },
  "vehiculo": {
    "vehiculo_id": "UUID",
    "placa": "ABC-123",
    "tipo": "MOTO | VAN | NHR | TURBO"
  },
  "resumen": {
    "total_paradas": 15,
    "exitosas": 11,
    "fallidas_culpa_cliente": 2,
    "fallidas_culpa_conductor": 1,
    "novedades_graves": 1
  },
  "paradas": [
    {
      "paquete_id": "UUID",
      "estado_final": "ENTREGADO | FALLIDO | NOVEDAD | SIN_GESTION_CONDUCTOR",
      "motivo_novedad": "string | null",
      "fecha_hora_gestion": "2026-03-06T10:15:00 | null"
    }
  ]
}
```

### Notas
- Si el Módulo 3 no está disponible al momento del cierre, el Módulo 2 guarda el evento en cola y reintenta cada 5 minutos hasta un máximo de 3 intentos. Si los 3 fallan, se emite alerta al Despachador Logístico con el payload completo para gestión manual.
- El conductor no tiene interacción directa con el Módulo 3. El envío de este evento es responsabilidad exclusiva del Módulo 2.

---

## 4. Resumen de Eventos

| # | Sentido | Tipo | Disparador |
|---|---|---|---|
| 1 | M1 → M2 | Síncrono | Paquete llega a `'Listo para Despacho'` |
| 2 | M2 → M1 | Asíncrono | Conductor confirma inicio de tránsito |
| 3 | M2 → M1 | Asíncrono | Conductor registra entrega exitosa |
| 4 | M2 → M1 | Asíncrono | Conductor registra parada fallida |
| 5 | M2 → M1 | Asíncrono | Conductor registra novedad grave |
| 6 | M2 → M1 | Asíncrono | Cierre de ruta (estado final de cada paquete) |
| 7 | M2 → M3 | Asíncrono | Cierre de ruta (resumen para liquidación) |
