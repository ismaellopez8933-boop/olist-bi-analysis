# Olist E-Commerce: Logistics Performance & Delivery SLA Analysis

Dashboard analítico end-to-end diseñado para evaluar el rendimiento logístico, cumplimiento de acuerdos de nivel de servicio (SLA) e impacto regional en los tiempos de entrega utilizando datos reales del marketplace brasileño Olist.

El proyecto abarca desde la ingesta y modelado relacional en **PostgreSQL** alojado en un servidor Linux virtualizado, hasta la construcción de métricas DAX y visualización ejecutiva multipágina en **Power BI Desktop**.

---

## 📌 Contexto de Negocio

Olist conecta pequeños negocios de todo Brasil con los mayores marketplaces del país. Al gestionar una red descentralizada de vendedores y múltiples transportistas a nivel nacional, los cuellos de botella logísticos impactan directamente la satisfacción del cliente.

**Preguntas clave que responde este proyecto:**
1. ¿Cuál es la tasa real de entregas con retraso frente a la fecha prometida al cliente?
2. ¿Cómo ha evolucionado el cumplimiento operativo mes a mes y qué factores estacionales lo golpean?
3. ¿Qué estados y ciudades concentran la mayor severidad de retrasos y qué patrones geográficos existen?

---

## 🛠️ Stack Tecnológico y Arquitectura

* **Base de Datos:** PostgreSQL 18
* **Entorno de Datos:** Ubuntu Server sobre KVM (`192.168.122.185`), configurado con acceso TCP/IP y roles dedicados para analítica (`bi_user`).
* **Modelado y ETL:** Vistas SQL para limpieza, tipado de fechas y cálculo de deltas de entrega.
* **Capa Semántica y Visualización:** Power BI Desktop conectado de forma remota vía Npgsql.
* **Lenguajes:** SQL, DAX.

```
[ Fuentes CSV Olist ] 
        │
        ▼ (scp / \copy)
[ PostgreSQL (Ubuntu Server KVM) ] 
        │
        ▼ (SQL Views: vw_entregas)
[ Power BI Desktop (Windows VM) ] ──▶ [ Dashboard Ejecutivo Multipágina ]
```

---

## 🗄️ Modelado y Preparación de Datos (SQL)

Se creó una vista analítica optimizada (`vw_entregas`) que consolida únicamente pedidos efectivamente entregados (`order_status = 'delivered'`), omitiendo registros incompletos y calculando el margen de cumplimiento operativo:

```sql
DROP VIEW IF EXISTS vw_entregas CASCADE;

create or replace view vw_entregas as (
    select c.customer_id,
           c.customer_city,
           c.customer_state,
           o.order_status,
           o.order_purchase_timestamp,
           o.order_delivered_carrier_date,
           o.order_estimated_delivery_date,
           o.order_delivered_customer_date,
           (o.order_delivered_customer_date::date - o.order_estimated_delivery_date::date) as real_delivery,
           case
               when o.order_delivered_customer_date::date > o.order_estimated_delivery_date::date then 'Retrasado'
    else 'A tiempo' end as status
        from customer c join orders o on c.customer_id = o.customer_id
            where o.order_status = 'delivered'
              and o.order_delivered_carrier_date is not null
              and o.order_delivered_customer_date is not null
    );
```

---

## 📐 Métricas y Lógica DAX

Se diseñaron medidas explícitas para evitar sesgos de columnas numéricas implícitas y asegurar consistencia analítica en todos los visuales:

* **Volumen Total de Entregas:**
  ```dax
  Total de entregas = COUNTROWS('public vw_entregas')
  ```

* **Porcentaje de Entregas Retrasadas (% SLA Fallido):**
  ```dax
  % de entregas retrasadas = 
  VAR totalEntregas = COUNTROWS('public vw_entregas')
  VAR totalRetrasadas = CALCULATE(
      COUNTROWS('public vw_entregas'), 
      'public vw_entregas'[status] = "Retrasado"
  )
  RETURN
      DIVIDE(totalRetrasadas, totalEntregas, 0)
  ```

* **Severidad Promedio del Retraso:**
  ```dax
  Días promedio de retraso = 
  CALCULATE(
      AVERAGE('public vw_entregas'[real_delivery]),
      'public vw_entregas'[status] = "Retrasado"
  )
  ```

---

## 📊 Estructura del Dashboard

### Página 1: Monitoreo Ejecutivo de Rendimiento y SLA
* **Tarjetas KPI:** Resumen general de 96,469 entregas totales, tasa global de retraso (6.77%) y retraso medio cuando ocurre la contingencia (10.62 días).
* **Distribución de Cumplimiento (Donut Chart):** Proporción directa entre envíos "A tiempo" (93.23%) y "Retrasados" (6.77%).
* **Evolución Temporal de la Tasa de Retraso (2017 - 2018):** Tendencia mensual que excluye anomalías de la fase beta (2016) para reflejar picos operativos reales.

### Página 2: Análisis Logístico Regional por Estado y Ciudad
* **Ranking de Severidad por Estado (Column Chart):** Muestra los días promedio de retraso por estado brasileño, identificando a los estados de la región Norte (AP, RR, AM) como los casos más críticos superando hasta 70 días de desviación.
* **Matriz de Desempeño por Estado:** Tabulación de volumen frente a tasa porcentual de retraso y días de impacto.
* **Focos Rojos a Nivel Ciudad:** Tabla con mapa de calor condicional filtrada por representatividad de muestra (volumen relevante), identificando municipios con alta concentración de retraso como Santarém (30.23%) o Maceió (27.12%).

---

## 🔍 Hallazgos Clave de Negocio (Insights)

1. **Estabilidad General vs. Picos Críticos:** A nivel global, el 93.23% de las órdenes se entregan antes o el día pactado. Sin embargo, en eventos de alta demanda como Black Friday 2017 y durante contingencias de transporte a inicios de 2018 (huelgas postales en Brasil), la tasa mensual de retraso se triplicó, rozando el 20%.
2. **Brecha Geográfica Centro-Sur vs. Norte-Nordeste:** Los estados con mayor volumen (São Paulo y Río de Janeiro) mantienen una operación relativamente ágil, pero los estados del extremo norte (Amapá, Roraima y Amazonas) sufren demoras medias que van de 30 a más de 70 días, evidenciando limitaciones en rutas terrestres fluviales y dependencia de enlaces aéreos.
3. **Control de Muestras Estadísticas:** Al auditar los datos por ciudad, se detectó que tasas del 100% solían pertenecer a municipios con 1 solo pedido histórico. Al aplicar filtros de significancia estadística, se focalizaron ciudades medianas que verdaderamente representan un problema de servicio al cliente.

---

## 🚀 Cómo Reproducir el Proyecto

1. Clonar el repositorio:
   ```bash
   git clone https://github.com/ismaellopez8933-boop/olist-bi-analysis.git
   ```
2. Cargar los datos CSV de Olist en PostgreSQL ejecutando los scripts de esquema y la vista `vw_entregas`.
3. Abrir el archivo `.pbix` en Power BI Desktop.
4. En caso de requerir recarga de datos, actualizar los parámetros de conexión de PostgreSQL con la IP y credenciales locales en el editor de Power Query.
