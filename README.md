# Royal Anchor Shipping — Database Design (ERD + SQL)

A relational database designed for **Royal Anchor Shipping**, a fictional shipping & logistics company.  
The schema supports **customers, shipments, vessels, routes, ports, containers, tracking, billing/invoices, and payments**, with referential integrity, indexes, and sample analytics queries.

---

## 🎯 Objectives
- Model core shipping operations with a clean, normalized schema (up to **3NF**).
- Enable **operational queries** (e.g., active shipments, tracking) and **analytics** (on-time % KPIs, revenue).
- Provide **DDL scripts**, seed data, and **reproducible** setup steps.

---

## 📂 Repository Structure
```
royal-anchor-shipping-db/
├── README.md
├── /docs
│   ├── ERD_Royal_Anchor_Shipping.png
│   ├── Data_Dictionary.xlsx
│   └── Design_Rationale.pdf
├── /sql
│   ├── 01_schema.sql          # DDL (tables, PK/FK, indexes, constraints)
│   ├── 02_seed_data.sql       # sample data
│   ├── 03_views.sql           # reporting views
│   ├── 04_procedures.sql      # optional SPs/functions
│   └── 05_queries_examples.sql# common queries
└── /samples
    ├── ports.csv
    ├── routes.csv
    └── shipments_sample.csv
```

---

## 🧱 Core Entities (High Level)
- **CUSTOMER** – Shipper/consignee information
- **PORT** – Ports with UN/LOCODE, country, timezone
- **VESSEL** – Fleet registry (IMO/MMSI, capacity)
- **ROUTE** – From/To ports, planned schedule
- **CONTAINER** – Container registry (size/type)
- **SHIPMENT** – Booking, status, linked to route and customer
- **SHIPMENT_CONTAINER** – Many-to-many between shipments and containers
- **TRACKING_EVENT** – Time-stamped events (Loaded, Departed, Arrived, Customs, Delivered)
- **INVOICE / INVOICE_LINE / PAYMENT** – Billing and collections

---

## 🗺️ ERD
See: `docs/ERD_Royal_Anchor_Shipping.png`  

---<img width="940" height="945" alt="image" src="https://github.com/user-attachments/assets/4b24c913-fb19-48c5-b7ac-3e8865427aca" />


## 🧩 DDL (Schema Excerpt)

> Full DDL in **/sql/01_schema.sql**. Below is a concise preview.

```sql
-- PORT
CREATE TABLE port (
  port_id        SERIAL PRIMARY KEY,
  port_code      VARCHAR(10) UNIQUE NOT NULL,
  port_name      VARCHAR(100) NOT NULL,
  country        VARCHAR(60) NOT NULL,
  timezone       VARCHAR(40) NOT NULL
);

-- CUSTOMER
CREATE TABLE customer (
  customer_id    SERIAL PRIMARY KEY,
  customer_name  VARCHAR(120) NOT NULL,
  contact_email  VARCHAR(120),
  contact_phone  VARCHAR(40),
  billing_address TEXT
);

-- VESSEL
CREATE TABLE vessel (
  vessel_id      SERIAL PRIMARY KEY,
  vessel_name    VARCHAR(120) NOT NULL,
  imo_number     VARCHAR(20) UNIQUE,
  capacity_teu   INT CHECK (capacity_teu >= 0)
);

-- ROUTE
CREATE TABLE route (
  route_id       SERIAL PRIMARY KEY,
  origin_port_id INT NOT NULL REFERENCES port(port_id),
  dest_port_id   INT NOT NULL REFERENCES port(port_id),
  planned_departure_date DATE,
  planned_arrival_date   DATE,
  CONSTRAINT route_ports_chk CHECK (origin_port_id <> dest_port_id)
);

-- CONTAINER
CREATE TABLE container (
  container_id   SERIAL PRIMARY KEY,
  container_no   VARCHAR(20) UNIQUE NOT NULL,
  size_ft        INT CHECK (size_ft IN (20, 40, 45)),
  type_code      VARCHAR(10)
);

-- SHIPMENT
CREATE TABLE shipment (
  shipment_id    SERIAL PRIMARY KEY,
  booking_no     VARCHAR(30) UNIQUE NOT NULL,
  customer_id    INT NOT NULL REFERENCES customer(customer_id),
  route_id       INT NOT NULL REFERENCES route(route_id),
  vessel_id      INT REFERENCES vessel(vessel_id),
  status         VARCHAR(20) NOT NULL DEFAULT 'BOOKED',
  created_at     TIMESTAMP NOT NULL DEFAULT NOW(),
  delivered_at   TIMESTAMP
);

-- SHIPMENT_CONTAINER
CREATE TABLE shipment_container (
  shipment_id    INT NOT NULL REFERENCES shipment(shipment_id) ON DELETE CASCADE,
  container_id   INT NOT NULL REFERENCES container(container_id),
  PRIMARY KEY (shipment_id, container_id)
);

-- TRACKING_EVENT
CREATE TABLE tracking_event (
  event_id       SERIAL PRIMARY KEY,
  shipment_id    INT NOT NULL REFERENCES shipment(shipment_id) ON DELETE CASCADE,
  event_type     VARCHAR(30) NOT NULL,
  event_time     TIMESTAMP NOT NULL,
  location_port_id INT REFERENCES port(port_id),
  notes          TEXT
);

-- INVOICE
CREATE TABLE invoice (
  invoice_id     SERIAL PRIMARY KEY,
  invoice_no     VARCHAR(30) UNIQUE NOT NULL,
  customer_id    INT NOT NULL REFERENCES customer(customer_id),
  shipment_id    INT REFERENCES shipment(shipment_id),
  invoice_date   DATE NOT NULL,
  due_date       DATE NOT NULL,
  total_amount   NUMERIC(12,2) NOT NULL CHECK (total_amount >= 0),
  status         VARCHAR(20) NOT NULL DEFAULT 'OPEN'
);

CREATE TABLE invoice_line (
  invoice_line_id SERIAL PRIMARY KEY,
  invoice_id     INT NOT NULL REFERENCES invoice(invoice_id) ON DELETE CASCADE,
  line_desc      VARCHAR(200) NOT NULL,
  qty            NUMERIC(10,2) NOT NULL CHECK (qty > 0),
  unit_price     NUMERIC(10,2) NOT NULL CHECK (unit_price >= 0),
  line_total     NUMERIC(12,2) GENERATED ALWAYS AS (qty * unit_price) STORED
);

CREATE TABLE payment (
  payment_id     SERIAL PRIMARY KEY,
  invoice_id     INT NOT NULL REFERENCES invoice(invoice_id) ON DELETE CASCADE,
  paid_amount    NUMERIC(12,2) NOT NULL CHECK (paid_amount > 0),
  paid_date      DATE NOT NULL,
  method         VARCHAR(20)
);
```

---

## 🧪 Sample Queries

```sql
-- On-time delivery rate
SELECT
  DATE_TRUNC('month', r.planned_arrival_date) AS month,
  ROUND(100.0 * AVG(CASE WHEN s.delivered_at::date <= r.planned_arrival_date THEN 1 ELSE 0 END), 2) AS on_time_pct
FROM shipment s
JOIN route r ON r.route_id = s.route_id
WHERE s.status = 'DELIVERED'
GROUP BY 1
ORDER BY 1;

-- Revenue by customer
SELECT c.customer_name, SUM(i.total_amount) AS total_revenue
FROM invoice i
JOIN customer c ON c.customer_id = i.customer_id
WHERE i.status IN ('PAID','PARTIAL')
GROUP BY c.customer_name
ORDER BY total_revenue DESC;
```

---

## 🧭 How to Run
1. Create a database:
   ```bash
   createdb royal_anchor_db
   ```
2. Execute schema & seed files:
   ```bash
   psql -d royal_anchor_db -f sql/01_schema.sql
   psql -d royal_anchor_db -f sql/02_seed_data.sql
   ```
3. Run sample queries:
   ```bash
   psql -d royal_anchor_db -f sql/05_queries_examples.sql
   ```

---

## 📒 Data Dictionary
See `docs/Data_Dictionary.xlsx` for:
- Column definitions & data types
- PK/FK relationships
- Value lists (status, sizes)

---

## ✅ Test Scenarios
- Create shipment → assign route → add containers → add tracking events.  
- Generate invoice → register payment.  
- Run analytics queries: on-time %, revenue by customer.  

---

**Author:** *Madhavi Kunte*  
MSc Information Systems | Salesforce QA & PM | ISTQB | 2× Salesforce Certified
