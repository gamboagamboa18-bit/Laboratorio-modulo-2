# 🍕 Mi Pizzería en la Nube: así automaticé los pedidos con API Gateway, EventBridge y Step Functions

Soy **John Luis Gamboa Gamboa**, estudiante del Bootcamp Cloud Computing de BeTek (Cohorte 3). En este laboratorio construí un sistema **serverless** en la región `us-east-1` (Norte de Virginia) que recibe pedidos por una API, los publica como eventos y los procesa con una máquina de estados. Esa máquina valida el pedido, revisa el inventario, cobra, confirma y deja un registro de auditoría.

No tengo servidores encendidos: cada pieza se ejecuta solo cuando llega un pedido.

---

## 🗺️ Mi arquitectura

![Arquitectura de mi Pizzería en la Nube](arquitectura.png)

Para explicarlo de forma sencilla, imaginé la solución como una **pizzería con robots**: cada servicio de AWS que usé cumple el papel de una persona de la pizzería.

| # | Servicio de AWS que usé | Papel en mi pizzería | Para qué lo usé |
|---|---|---|---|
| 1 | **Amazon API Gateway** (API HTTP) | La ventanilla | Creé la ruta `POST /pedido` para recibir el JSON del cliente y entregárselo a mi Lambda `receive-order`. |
| 2 | **AWS Lambda** `receive-order` | El mesero | Lee el `body`, genera un `orderId` único (UUID), arma el detalle del pedido y lo publica en EventBridge con `put_events`. Le responde al cliente con el `orderId`. |
| 3 | **Amazon EventBridge** | El altavoz | Recibe el evento en el bus `default`. Configuré una regla con el patrón `source = demo.pedidos` y `detail-type = NuevoPedido` para que inicie mi máquina de estados. |
| 4 | **AWS Step Functions** (Standard) | La cocina con su lista de pasos | Orquesta los 5 pasos del pedido en orden. Con estados `Choice` decido si el pedido sigue o termina en un `Fail`. |
| 5 | **AWS Lambda** `validate-order` | El inspector | Verifica que `cliente` y `producto` no estén vacíos y que `cantidad` y `total` sean mayores que 0. |
| 6 | **AWS Lambda** `check-inventory` | Los ingredientes | Marca `AVAILABLE` si la cantidad es 5 o menos; si es mayor, `OUT_OF_STOCK`. |
| 7 | **AWS Lambda** `process-payment` | El cajero | Aprueba el pago si el total es 1000 o menos; si es mayor, lo rechaza. |
| 8 | **Wait** (estado de Step Functions) | La espera | Espera 5 segundos antes de confirmar, para simular una confirmación externa. |
| 9 | **AWS Lambda** `confirm-order` | El que confirma | Marca el pedido como `CONFIRMED`. |
| 10 | **AWS Lambda** `register-audit` | El cuaderno | Devuelve el resumen final del pedido para auditoría. |
| 11 | **AWS IAM** | Los permisos | Le di a cada pieza solo el permiso que necesita (mínimo privilegio). |
| 12 | **Amazon CloudWatch Logs** | El diario | Aquí leí los `print()` de cada Lambda. Fue mi herramienta clave para encontrar errores. |

### Los permisos que configuré (IAM)

| Quién | Necesita permiso para | Cómo lo configuré |
|---|---|---|
| Lambda `receive-order` | `events:PutEvents` | Le agregué una política insertada a su rol de ejecución |
| API Gateway | `lambda:InvokeFunction` sobre `receive-order` | Política basada en recursos de la Lambda, creada por la consola al adjuntar la integración |
| Regla de EventBridge | `states:StartExecution` | Dejé que EventBridge creara su rol automáticamente |
| Step Functions | `lambda:InvokeFunction` sobre las 5 Lambdas | Dejé que Step Functions creara su rol automáticamente |

---

## 🔄 Cómo viaja un pedido

1. Envío un `POST /pedido` con el JSON del pedido.
2. API Gateway invoca a `receive-order`.
3. `receive-order` genera el `orderId`, publica el evento `NuevoPedido` y me responde.
4. La regla de EventBridge detecta el evento e inicia una ejecución de Step Functions.
5. Step Functions ejecuta: **ValidateOrder → CheckInventory → ProcessPayment → Wait 5 s → ConfirmOrder → RegisterAudit → Éxito**.
6. Si el pedido no cumple una regla, la ejecución termina en `InvalidOrder`, `OutOfStock` o `PaymentRejected`.

### Mis reglas de negocio

| Paso | Regla | Si no se cumple |
|---|---|---|
| ValidateOrder | `cliente` y `producto` con valor; `cantidad` > 0; `total` > 0 | `InvalidOrder` |
| CheckInventory | `cantidad` ≤ 5 | `OutOfStock` |
| ProcessPayment | `total` ≤ 1000 | `PaymentRejected` |

---

## 🚀 Cómo se usa mi solución

### Con curl (Windows, Símbolo del sistema)

Yo lo probé así:

```bat
curl -X POST https://<API_ID>.execute-api.us-east-1.amazonaws.com/pedido -H "Content-Type: application/json" -d "{\"cliente\":\"John\",\"producto\":\"Laptop\",\"cantidad\":1,\"total\":900}"
```

### Con Postman

- **Método:** `POST`
- **URL:** `https://<API_ID>.execute-api.us-east-1.amazonaws.com/pedido`
- **Body → raw → JSON:**

```json
{
  "cliente": "John",
  "producto": "Laptop",
  "cantidad": 1,
  "total": 900
}
```

### La respuesta que recibí

```json
{
  "mensaje": "Pedido recibido y enviado al bus de eventos",
  "orderId": "490448ca-cb04-4164-bfa8-3707f5a16264"
}
```

Después, en **Step Functions → procesar-pedidos-john-v2**, me apareció una ejecución nueva que terminó en **Ejecutada correctamente** en unos 7 segundos (5 son de la espera).

### Mis casos de prueba

| Pedido de prueba | Resultado esperado |
|---|---|
| `cantidad: 1`, `total: 900` | ✅ Ejecutada correctamente (`CONFIRMED`) |
| `cantidad: 10`, `total: 900` | ❌ `OutOfStock` |
| `cantidad: 1`, `total: 2000` | ❌ `PaymentRejected` |
| `cliente: ""` | ❌ `InvalidOrder` |

---

## 📸 Mis evidencias de funcionamiento

| Evidencia | Archivo |
|---|---|
| Respuesta de mi API con el `orderId` | `evidencias/01-respuesta-api.png` |
| Mi ejecución en Step Functions "Ejecutada correctamente" | `evidencias/02-ejecucion-correcta.png` |
| El diagrama de la ejecución con todos los pasos en verde | `evidencias/03-flujo-verde.png` |
| La salida de RegisterAudit con `orderStatus: CONFIRMED` | `evidencias/04-auditoria.png` |

![Respuesta de la API](evidencias/01-respuesta-api.png)
![Ejecución correcta](evidencias/02-ejecucion-correcta.png)
![Flujo en verde](evidencias/03-flujo-verde.png)
![Auditoría final](evidencias/04-auditoria.png)

---

## 🛠️ Problemas que encontré y cómo los resolví

| Problema | Por qué pasaba | Cómo lo resolví |
|---|---|---|
| Todos mis pedidos terminaban en `InvalidOrder` | Mi regla de EventBridge tenía el patrón `{"account": [...]}`, que captura **todos** los eventos de la cuenta, incluidos los avisos de mi propia Step Function | Corregí el patrón a `source = demo.pedidos` y `detail-type = NuevoPedido` |
| Se crearon miles de ejecuciones en bucle | Ese mismo patrón: cada cambio de estado de mi Step Function la volvía a iniciar | Desactivé la regla, creé una regla nueva con el patrón correcto y una máquina de estados nueva (v2) |
| Mi API respondía `Internal Server Error` | El código de `receive-order` me quedó incompleto y Lambda no encontraba `lambda_handler` (`Runtime.HandlerNotFound`) | Pegué el código completo y lo desplegué con **Deploy** |
| `Lambda.ResourceNotFoundException` en CheckInventory | Había nombrado la función `check-inventary-john` (con "a") | Creé `check-inventory-john` con el nombre exacto |

**Lo que aprendí:** un patrón de eventos demasiado amplio puede crear bucles; en AWS los nombres de los recursos distinguen cada letra y las mayúsculas; y probar cada pieza por separado (el Test de Lambda y la entrada y salida de cada paso de Step Functions) es la forma más rápida de encontrar dónde falla un flujo.

---

## 🧹 Limpieza de recursos

Al terminar el laboratorio eliminé mis recursos para no generar costos:

- API Gateway: `api-pedidos-john`
- Reglas de EventBridge: `rule-nuevo-pedido-john` y `reglas-pedido-john`
- Step Functions: `procesar-pedidos-john-v2`
- Lambdas: `receive-order-john`, `validate-order-john`, `check-inventory-john`, `check-inventary-john`, `process-payment-john`, `confirm-order-john` y `register-audit-john`

---

## 📁 Estructura de mi repositorio

```
.
├── README.md
├── arquitectura.png
├── arquitectura.svg
└── evidencias/
    ├── 01-respuesta-api.png
    ├── 02-ejecucion-correcta.png
    ├── 03-flujo-verde.png
    └── 04-auditoria.png
```

---

Diseñé y documenté este laboratorio: **John Luis Gamboa Gamboa** · 2026
