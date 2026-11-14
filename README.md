# 📊 Aviso de Ventas Totales – Minoristas Adidas (Power Automate)

Flujo automatizado en **Power Automate** que consulta las ventas totales de un modelo publicado en **Power BI Service**, las compara contra una **meta de ventas** y, si la meta se supera, envía una notificación por **correo (Outlook)** y por **Microsoft Teams**.

## Autor

**Sebastian Valentin** — Estudiante de Ingeniería Industrial, UPC
GitHub: [sebasvalma540](https://github.com/sebasvalma540)

## 📩 Si quieres contactarme aqui te dejo mi linkedin =)
<p align="center">
  <a href="https://www.linkedin.com/in/sebastian-valentin-malpartida-11664428a/">
    <img src="https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white" />
  </a>
</p>


## 🖼️ Vista general

> 📁 Guarda las capturas en una carpeta `imagenes/` junto a este README, con los nombres indicados.

<p align="center">
  <img src="Imagenes/Flujo%20Alerta%20Ventas.png" alt="Flujo Alerta Ventas" width="700">
  <br><em>📷 Flujo Alerta Ventas</em>
</p>

### ✅ Evidencia de funcionamiento

| Correo recibido | Mensaje en Teams |
|:---:|:---:|
| <img src="Imagenes/Mensaje%20Gmail.png" alt="Correo recibido" width="380"> | <img src="Imagenes/Mensaje%20Teams.png" alt="Mensaje en Teams" width="380"> |
| 📷 *Correo con el resumen de ventas* | 📷 *Mensaje del Bot de Flow* |

## 🎯 Objetivo

Automatizar el seguimiento de ventas: en lugar de abrir el dashboard cada día, el flujo revisa las cifras por su cuenta y avisa cuando se cumple la meta.

## 🧰 Herramientas

- Power Automate (flujo programado en la nube)
- Power BI Service (conjunto de datos `Datos_Minoristas_Excel`)
- DAX (consulta a la tabla de hechos)
- Outlook 365 y Microsoft Teams (canales de notificación)

## 🔄 Diagrama del flujo

```
Recurrence (diario, 8:00 y 18:00)
        │
        ▼
Ejecutar una consulta con un conjunto de datos (DAX)
        │
        ▼
VentasTotales - Redactar  →  ValorVentaTotal
                             ValorMetaReferencial
                             ValorDiferencia
        │
        ▼
Formatos de texto (VarVenta, VarMeta, ValorDiferencia)
        │
        ▼
¿Se superó la meta?  (ValorVentaTotal > ValorMetaReferencial)
   ├── True  → Enviar correo (V2) + Publicar mensaje en Teams
   └── False → Redactar: "Meta no superada"
```

## ⚙️ Paso a paso

### 1. Recurrence (programación)

<p align="center">
  <img src="imagenes/02-recurrence.png" alt="Configuración de Recurrence" width="700">
  <br><em>📸 Configuración de Recurrence</em>
</p>

Dispara el flujo automáticamente.

| Parámetro | Valor |
|---|---|
| Intervalo / Frecuencia | 1 / Día |
| Zona horaria | (UTC-05:00) Bogotá, Lima, Quito |
| Horas | 8 y 18 |
| Minutos | 0 |

Resultado: el flujo se ejecuta **todos los días a las 8:00 y 18:00**.

### 2. Ejecutar una consulta con un conjunto de datos

<p align="center">
  <img src="imagenes/03-consulta-powerbi.png" alt="Acción de consulta a Power BI con código DAX" width="700">
  <br><em>📸 Acción de consulta a Power BI con código DAX</em>
</p>

Acción de Power BI que se conecta a un conjunto de datos que subí previamente a **Power BI Service**.

- **Área de trabajo:** My Workspace
- **Conjunto de datos:** `Datos_Minoristas_Excel`
- **Consulta:** código DAX que suma `Fact_Ventas6[Ventas_Totales]` por minorista (`Dim_Minorista4[Minorista]`) y agrega una fila de **total general** con `ROLLUPADDISSUBTOTAL`. Se ordena con el total general primero (`IsGrandTotalRowTotal DESC`), por lo que la **primera fila** de resultados es la venta total.

```dax
DEFINE
  VAR __DS0Core =
    SUMMARIZECOLUMNS(
      ROLLUPADDISSUBTOTAL('Dim_Minorista4'[Minorista], "IsGrandTotalRowTotal"),
      "SumVentas_Totales", CALCULATE(SUM('Fact_Ventas6'[Ventas_Totales]))
    )
  VAR __DS0PrimaryWindowed =
    TOPN(502, __DS0Core, [IsGrandTotalRowTotal], 0, 'Dim_Minorista4'[Minorista], 1)

EVALUATE
  __DS0PrimaryWindowed
ORDER BY
  [IsGrandTotalRowTotal] DESC, 'Dim_Minorista4'[Minorista]
```

### 3. VentasTotales – Redactar

<p align="center">
  <img src="imagenes/04-redactar-ventas.png" alt="Redactar con la expresión que extrae la venta total" width="700">
  <br><em>📸 Redactar con la expresión que extrae la venta total</em>
</p>

Acción **Redactar** que extrae de la primera fila el valor de la venta total:

```
outputs('Ejecutar_una_consulta_con_un_conjunto_de_datos')?['body/firstTableRows'][0]?['[SumVentas_Totales]']
```

### 4. Inicializar variables

<p align="center">
  <img src="imagenes/05-variables.png" alt="Variables ValorVentaTotal, ValorMetaReferencial y ValorDiferencia" width="700">
  <br><em>📸 Variables ValorVentaTotal, ValorMetaReferencial y ValorDiferencia</em>
</p>

Se guardan los valores en variables de tipo **Flotante** para poder operarlos.

| Variable | Valor |
|---|---|
| `ValorVentaTotal` | Salida del paso anterior (venta total) |
| `ValorMetaReferencial` | `800000000` (meta fija) |
| `ValorDiferencia` | `sub(variables('ValorVentaTotal'), variables('ValorMetaReferencial'))` |

### 5. Formato de los valores

<p align="center">
  <img src="imagenes/06-formatos.png" alt="Acciones de formato de moneda" width="700">
  <br><em>📸 Acciones de formato de moneda</em>
</p>

Tres acciones **Redactar** convierten los números a texto con formato de moneda (ej. `S/ 899,902,125`):

```
concat('S/ ', formatNumber(variables('<NombreVariable>'), '#,##0', 'en-US'))
```

- `VarVenta - Formato` → ventas actuales
- `VarMeta - Formato` → meta
- `ValorDiferencia - Formato` → diferencia

### 6. Condición: ¿Se superó la meta?

<p align="center">
  <img src="imagenes/07-condicion.png" alt="Condición y ramas True / False" width="700">
  <br><em>📸 Condición y ramas True / False</em>
</p>

Compara:

```
ValorVentaTotal  es mayor que  ValorMetaReferencial
```

**Si es True:**
1. **Enviar correo electrónico (V2)** (Outlook): asunto *"SE CUMPLIO LA META"*, con cuerpo en HTML que muestra una tabla con *Ventas actuales*, *Meta* y *Diferencia* (valores dinámicos tomados de las variables formateadas) y el mensaje *"Estamos por encima de la Meta"*.
2. **Publicar mensaje en un chat o canal** (Teams): el **Bot de Flow** envía un chat con el resumen de ventas y el mensaje *"¡Meta superada!"*.

<p align="center">
  <img src="imagenes/08-correo-teams-config.png" alt="Configuración del correo y del mensaje de Teams" width="700">
  <br><em>📸 Configuración del correo y del mensaje de Teams</em>
</p>

**Si es False:**
- Una acción **Redactar** con el texto *"Meta no superada"* (el flujo no envía correo ni mensaje).

## 📬 Resultado esperado

Cuando las ventas superan los S/ 800,000,000, se recibe un correo y un mensaje de Teams con una tabla como esta:

| Indicador | Valor |
|---|---|
| Ventas actuales | S/ 899,902,125 |
| Meta | S/ 800,000,000 |
| **Diferencia** | **S/ 99,902,125** |

**Estamos por encima de la Meta 🥳**

## 📨 Destino de las notificaciones

El correo se envía desde la cuenta de **Outlook** conectada al flujo, y puede llegar a cualquier destinatario (Gmail, Outlook u otro).

- **Gmail:** el primer correo puede llegar a **Spam**, porque Gmail no reconoce al remitente automático. Solución: abrir el correo y marcar **"No es spam"**.
- **Outlook / correo institucional:** suele llegar directo a la bandeja de entrada.
- **Teams:** el mensaje llega por chat del **Bot de Flow**, sin pasar por filtros de spam.

## 🚀 Cómo reutilizarlo

1. Sube tu archivo a Power BI Service y crea el conjunto de datos.
2. Importa o recrea el flujo en Power Automate.
3. Ajusta en la acción de Power BI el **área de trabajo**, el **conjunto de datos** y la **consulta DAX** (tablas y columnas de tu modelo).
4. Cambia el valor de `ValorMetaReferencial` por tu meta.
5. Configura los destinatarios del correo y del mensaje de Teams.
6. Prueba el flujo con **Probar → Manualmente** antes de dejarlo activo.

## 🔧 Posibles mejoras

- Usar también las variables formateadas como contenido dinámico en el mensaje de Teams (incluir la diferencia).
- Enviar también un aviso cuando la meta **no** se supera.
- Leer la meta desde una tabla del modelo en lugar de dejarla fija.