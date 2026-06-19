# Auth, Inventory & Order Management System (Microservices Architecture)

This project implements an E-Commerce backend system using a **Microservices Architecture** built with **.NET Core Web API**. The system is designed using the **Controller-Service-Repository** pattern and leverages **Entity Framework Core (EF Core) Database-First Approach** for data persistence. 

> **Note on Database Setup:** To simplify local development and integration testing while keeping logical separation, all service schemas/tables are hosted within a single consolidated database (`auth_db`).

---

## 🚀 Key Features

* **Role-Based Access Control (RBAC):** Secured via JWT Authentication supporting `ADMIN` and `USER` roles.
* **Inventory & Stock Control:** Strict validation to prevent negative stock and overselling.
* **Atomic Order Processing:** Database transactions ensure orders are safely processed or completely rolled back on failure.
* **Inter-Service Communication:** Services communicate via REST APIs (Order Service validates and updates stock through the Product/Inventory APIs without direct DB bypass).
* **Pagination:** Implemented across all listing endpoints for optimized performance.
* **Cross-Cutting Concerns:** Global Exception Handling, Standardized Error Responses, and File-based Log Management.

---

## 📐 Architecture & Design Pattern

Each microservice is structured following clean coding practices split into distinct layers:
* **Controllers:** Handles HTTP Requests, routes, and model validations.
* **Services:** Encapsulates core business logic (e.g., Auth generation, Stock deduction).
* **Repositories:** Directly interacts with the database via EF Core DB Context.
* **DTOs (Data Transfer Objects):** Safely moves data between layers without exposing internal DB Entities.
* **Entities:** Structured models mapped directly from the PostgreSQL tables via EF Core Database-First scaffolding.

---

## 🛠️ Tech Stack

* **Framework:** .NET Core Web API (Latest)
* **ORM:** Entity Framework Core (EF Core) - Database First Approach
* **Database:** PostgreSQL
* **Security:** JWT (JSON Web Tokens)
* **Logging:** File-Based Logging

---

## 🗄️ Database Schema & Architecture

All the tables are created inside the centralized database `auth_db`. Run the following script to set up the tables, constraints, and optimized indexes:

```sql
CREATE DATABASE auth_db;
\c auth_db;

-- ==========================================
-- 1. AUTHENTICATION SERVICE TABLES
-- ==========================================
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role VARCHAR(30) NOT NULL CHECK (role IN ('ADMIN', 'USER')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ==========================================
-- 2. INVENTORY / PRODUCT SERVICE TABLES
-- ==========================================
CREATE TABLE products (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_name VARCHAR(150) NOT NULL,
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    stock_qty INT NOT NULL CHECK (stock_qty >= 0),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);
CREATE INDEX idx_product_name ON products(product_name);

CREATE TABLE inventory_items (
    item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_name VARCHAR(150) NOT NULL,
    category VARCHAR(100),
    quantity INT NOT NULL CHECK (quantity >= 0),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP
);
CREATE INDEX idx_inventory_item_name ON inventory_items(item_name);
CREATE INDEX idx_inventory_is_active ON inventory_items(is_active);

-- ==========================================
-- 3. ORDER SERVICE TABLES
-- ==========================================
CREATE TABLE orders (
    order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL, -- or user_id mapping depending on client payload
    quantity INT NOT NULL CHECK (quantity > 0),
    order_status VARCHAR(30) NOT NULL DEFAULT 'CREATED' CHECK (order_status IN ('CREATED', 'PAID', 'CONFIRMED', 'CANCELLED')),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_orders_product_id ON orders(product_id);

CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    product_id UUID NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    CONSTRAINT fk_order_items_order FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
);