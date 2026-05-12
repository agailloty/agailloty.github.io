---
title: "Découvrir PostgREST"
description: "Exposer vos tables Postgres via un endpoint REST"
date: 2025-05-02T10:00:07+01:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [postgres, python, docker]
---

Dans cet article nous découvrons ensemble comment exposer des tables de votre base de données Postgres via une API REST.

Postgres est un système de gestion de base de données open-source que j'aime utiliser dans mes projets personnels. 

## Lancer les conteneurs Postgres et PostgREST

Pour la démonstration je vais créer un conteneur Postgres mais si vous suivez cette démo c'est probablement que vous avez déjà une base de données Postgres. 

Je crée un fichier Docker compose dans lequel je déclare deux services : `db` pour Postgres et `postgrest` pour PostgREST.

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

Lancer les conteneurs avec la commande docker compose up.

```bash
docker compose up -d
```

## Setup d'un projet

Ce projet met en place une API REST simple mais fonctionnelle pour une entreprise de e-commerce spécialisée dans la vente de matériel informatique. La base de données repose sur trois tables principales : `users`, `products` et `orders`.

### Script de création des tables 

```sql
-- Créer le rôle anonyme (utilisé par PostgREST)
CREATE ROLE web_anon NOLOGIN;

-- Table des utilisateurs
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    full_name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Table des produits
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL,
    stock INT DEFAULT 0
);

-- Table des commandes
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    product_id INT REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    order_date TIMESTAMP DEFAULT NOW()
);
```

### Insertion des données 

```sql
-- Utilisateurs
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
-- Produits
INSERT INTO products (name, description, price, stock) VALUES
('Ordinateur portable Dell XPS 13', 'Ultrabook performant et léger', 1299.99, 15),
('Clavier mécanique Logitech', 'Clavier RGB avec switchs Cherry MX', 149.99, 30),
('Souris Logitech MX Master 3', 'Souris ergonomique sans fil', 99.99, 40),
('Écran 27" LG 4K', 'Moniteur UHD pour graphistes et développeurs', 399.99, 20),
('SSD Samsung 1To', 'Disque dur SSD NVMe haute performance', 159.99, 25),
('Station d''accueil USB-C', 'Pour connecter plusieurs périphériques', 89.99, 35),
('Carte graphique NVIDIA RTX 4070', 'GPU pour gaming et création', 649.99, 10),
('Casque audio Bose QC45', 'Casque Bluetooth à réduction de bruit', 329.99, 18),
('Webcam Logitech C920', 'Webcam HD pour visio-conférences', 89.99, 50),
('Routeur Wi-Fi 6 TP-Link', 'Routeur performant pour maison connectée', 129.99, 22);
```

```sql
-- Commandes
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

## Créer une vue pour afficher les commandes

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

## Accéder aux données via l'endpoint PostgREST

PostgREST ne peut accéder aux éléments de notre BDD que si nous lui avons donné le droit.

```sql
GRANT USAGE ON SCHEMA public TO web_anon;
GRANT SELECT ON public_orders TO web_anon;
```

### Retourner toutes les données de la vue `public_orders`

```sh
curl GET http://localhost:3000/public_orders
```

```json
[
  {
    "order_id": 1,
    "customer": "Alice Dupont",
    "product": "Ordinateur portable Dell XPS 13",
    "quantity": 1,
    "price": 1299.99,
    "total_price": 1299.99,
    "order_date": "2025-05-02T12:51:32.03124"
  },
  {
    "order_id": 2,
    "customer": "Bob Martin",
    "product": "Souris Logitech MX Master 3",
    "quantity": 2,
    "price": 99.99,
    "total_price": 199.98,
    "order_date": "2025-05-02T12:51:32.03124"
  }
]
```

### Filtrer les données de la vue `public_orders`

```sh
http://localhost:3000/public_orders?customer=eq.Alice%20Dupont
```

```json
[
  {
    "order_id": 1,
    "customer": "Alice Dupont",
    "product": "Ordinateur portable Dell XPS 13",
    "quantity": 1,
    "price": 1299.99,
    "total_price": 1299.99,
    "order_date": "2025-05-02T12:51:32.03124"
  }
]
```

## Conclusion et suite du projet

Pour ne pas surcharger cet article, je vais m'arrêter à la requête GET vers l'API PostgREST. Mais dans des articles ultérieurs nous verrons ensemble comment envoyer des requêtes de type POST, DELETE, PUT ... avec PostgREST. 

J'ai créé un répertoire Github pour suivre l'évolution du projet : https://github.com/agailloty/postgrest-demo
