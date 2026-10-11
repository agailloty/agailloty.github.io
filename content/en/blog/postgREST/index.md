---
title: "Discovering PostgREST"
description: "Expose your Postgres tables through a REST endpoint"
date: 2025-05-02T10:00:07+01:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [postgres, python, docker]
---

In this article we discover together how to expose tables from your Postgres database through a REST API.

Postgres is an open-source database management system that I love to use in my personal projects.

## Starting the Postgres and PostgREST containers

For the demonstration I will create a Postgres container, but if you are following this demo you probably already have a Postgres database.

I create a Docker compose file in which I declare two services: `db` for Postgres and `postgrest` for PostgREST.

```docker
services:
  db:
    image: postgres:16
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: postgrest_user
      POSTGRES_PASSWORD: secretpassword
      POSTGRES_DB: ordersDB
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  postgrest:
    image: postgrest/postgrest
    container_name: postgrest
    restart: always
    depends_on:
      - db
    ports:
      - "3000:3000"
    environment:
      PGRST_DB_URI: postgres://postgrest_user:secretpassword@db:5432/ordersDB
      PGRST_DB_ANON_ROLE: web_anon
      PGRST_SERVER_PORT: 3000
    volumes:
    - ./postgrest.conf:/etc/postgrest.conf

volumes:
  pgdata:
```

Start the containers with the docker compose up command.

```bash
docker compose up -d
```

## Project setup

This project sets up a simple yet functional REST API for an e-commerce company specialized in selling computer hardware. The database relies on three main tables: `users`, `products` and `orders`.

### Table creation script

```sql
-- Create the anonymous role (used by PostgREST)
CREATE ROLE web_anon NOLOGIN;

-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    full_name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL,
    stock INT DEFAULT 0
);

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    product_id INT REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    order_date TIMESTAMP DEFAULT NOW()
);
```

### Inserting the data

```sql
-- Users
INSERT INTO users (full_name, email) VALUES
('Alice Dupont', 'alice@example.com'),
('Bob Martin', 'bob@example.com'),
('Claire Morel', 'claire@example.com'),
('David Lefevre', 'david@example.com'),
('Eva Bernard', 'eva@example.com'),
('Florian Dubois', 'florian@example.com'),
('Gisèle Laurent', 'gisele@example.com'),
('Henri Petit', 'henri@example.com'),
('Inès Renault', 'ines@example.com'),
('Julien Mercier', 'julien@example.com');
```

```sql
-- Products
INSERT INTO products (name, description, price, stock) VALUES
('Dell XPS 13 laptop', 'Powerful and lightweight ultrabook', 1299.99, 15),
('Logitech mechanical keyboard', 'RGB keyboard with Cherry MX switches', 149.99, 30),
('Logitech MX Master 3 mouse', 'Ergonomic wireless mouse', 99.99, 40),
('27" LG 4K monitor', 'UHD monitor for designers and developers', 399.99, 20),
('Samsung 1TB SSD', 'High performance NVMe SSD drive', 159.99, 25),
('USB-C docking station', 'To connect multiple peripherals', 89.99, 35),
('NVIDIA RTX 4070 graphics card', 'GPU for gaming and creation', 649.99, 10),
('Bose QC45 headphones', 'Bluetooth noise-cancelling headphones', 329.99, 18),
('Logitech C920 webcam', 'HD webcam for video conferences', 89.99, 50),
('TP-Link Wi-Fi 6 router', 'Powerful router for the connected home', 129.99, 22);
```

```sql
-- Orders
INSERT INTO orders (user_id, product_id, quantity) VALUES
(1, 1, 1),
(2, 3, 2),
(3, 2, 1),
(4, 5, 1),
(5, 4, 1),
(6, 1, 1),
(7, 7, 1),
(8, 6, 1),
(9, 8, 1),
(10, 9, 2);
```

## Creating a view to display the orders

```sql
CREATE VIEW public_orders AS
SELECT
    o.id AS order_id,
    u.full_name AS customer,
    p.name AS product,
    o.quantity,
    p.price,
    o.quantity * p.price AS total_price,
    o.order_date
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id;
```

## Accessing the data through the PostgREST endpoint

PostgREST can only access our database objects if we have granted it the rights.

```sql
GRANT USAGE ON SCHEMA public TO web_anon;
GRANT SELECT ON public_orders TO web_anon;
```

### Returning all the data from the `public_orders` view

```sh
curl GET http://localhost:3000/public_orders
```

```json
[
  {
    "order_id": 1,
    "customer": "Alice Dupont",
    "product": "Dell XPS 13 laptop",
    "quantity": 1,
    "price": 1299.99,
    "total_price": 1299.99,
    "order_date": "2025-05-02T12:51:32.03124"
  },
  {
    "order_id": 2,
    "customer": "Bob Martin",
    "product": "Logitech MX Master 3 mouse",
    "quantity": 2,
    "price": 99.99,
    "total_price": 199.98,
    "order_date": "2025-05-02T12:51:32.03124"
  }
]
```

### Filtering the data from the `public_orders` view

```sh
http://localhost:3000/public_orders?customer=eq.Alice%20Dupont
```

```json
[
  {
    "order_id": 1,
    "customer": "Alice Dupont",
    "product": "Dell XPS 13 laptop",
    "quantity": 1,
    "price": 1299.99,
    "total_price": 1299.99,
    "order_date": "2025-05-02T12:51:32.03124"
  }
]
```

## Conclusion and next steps

To keep this article manageable, I will stop at the GET request to the PostgREST API. In future articles we will see together how to send POST, DELETE, PUT ... requests with PostgREST.

I created a GitHub repository to follow the project's progress: https://github.com/agailloty/postgrest-demo
