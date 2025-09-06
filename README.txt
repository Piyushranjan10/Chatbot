Below is a complete, minimal-yet-production-ready project you can paste into your repo. It includes:

* FastAPI backend with routers for menu, customers, orders, and Dialogflow webhook.
* SQLAlchemy ORM models + Pydantic schemas.
* MySQL schema + seed data.
* Dialogflow agent stubs for intents/entities/webhook.
* `requirements.txt` and `.env.example`.

> **How to use**: Create the folders/files exactly as shown, paste the code blocks into the matching paths, update `.env`, run the SQL in MySQL, then start the API with `uvicorn backend.main:app --reload`.

---

## Repository Layout

```
📦 Chatbot
├─ 📂 backend
│  ├─ 📂 core
│  │  └─ config.py
│  ├─ 📂 db
│  │  └─ database.py
│  ├─ 📂 models
│  │  ├─ __init__.py
│  │  ├─ customer.py
│  │  ├─ menu_item.py
│  │  ├─ order.py
│  │  └─ order_item.py
│  ├─ 📂 routers
│  │  ├─ customers.py
│  │  ├─ menu.py
│  │  ├─ orders.py
│  │  └─ webhook.py
│  ├─ 📂 schemas
│  │  ├─ __init__.py
│  │  ├─ customer.py
│  │  ├─ menu_item.py
│  │  └─ order.py
│  ├─ 📂 services
│  │  └─ order_service.py
│  └─ main.py
├─ 📂 database
│  ├─ schema.sql
│  └─ seed.sql
├─ 📂 dialogflow_agent
│  ├─ agent.json
│  ├─ 📂 intents
│  │  ├─ Default Welcome Intent.json
│  │  ├─ PlaceOrder.json
│  │  ├─ TrackOrder.json
│  │  └─ Fallback.json
│  └─ 📂 entities
│     └─ food-item.json
├─ .env.example
├─ requirements.txt
└─ README.md
```

---

## `backend/core/config.py`

```python
import os
from pydantic import BaseModel

class Settings(BaseModel):
    APP_NAME: str = os.getenv("APP_NAME", "Food Delivery Chatbot API")
    ENV: str = os.getenv("ENV", "dev")
    DB_HOST: str = os.getenv("DB_HOST", "127.0.0.1")
    DB_PORT: str = os.getenv("DB_PORT", "3306")
    DB_NAME: str = os.getenv("DB_NAME", "foodbot")
    DB_USER: str = os.getenv("DB_USER", "root")
    DB_PASS: str = os.getenv("DB_PASS", "password")
    DB_DRIVER: str = os.getenv("DB_DRIVER", "mysql+pymysql")
    CORS_ORIGINS: str = os.getenv("CORS_ORIGINS", "*")

    @property
    def DATABASE_URL(self) -> str:
        return f"{self.DB_DRIVER}://{self.DB_USER}:{self.DB_PASS}@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}?charset=utf8mb4"

settings = Settings()
```

---

## `backend/db/database.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
from backend.core.config import settings

engine = create_engine(settings.DATABASE_URL, pool_pre_ping=True, pool_recycle=3600)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Dependency for FastAPI routes

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

---

## `backend/models/__init__.py`

```python
from .customer import Customer
from .menu_item import MenuItem
from .order import Order
from .order_item import OrderItem
```

---

## `backend/models/customer.py`

```python
from sqlalchemy import Column, Integer, String, DateTime, func
from backend.db.database import Base

class Customer(Base):
    __tablename__ = "customers"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    phone = Column(String(20), unique=True, index=True, nullable=False)
    email = Column(String(120), unique=True, index=True, nullable=True)
    address = Column(String(255), nullable=True)
    created_at = Column(DateTime, server_default=func.now())
```

---

## `backend/models/menu_item.py`

```python
from sqlalchemy import Column, Integer, String, Numeric, Boolean
from backend.db.database import Base

class MenuItem(Base):
    __tablename__ = "menu_items"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(120), unique=True, index=True, nullable=False)
    description = Column(String(255), nullable=True)
    price = Column(Numeric(10, 2), nullable=False)
    is_available = Column(Boolean, default=True, nullable=False)
    category = Column(String(50), nullable=True)
```

---

## `backend/models/order.py`

```python
from sqlalchemy import Column, Integer, ForeignKey, String, Numeric, DateTime, func
from sqlalchemy.orm import relationship
from backend.db.database import Base

class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True, index=True)
    customer_id = Column(Integer, ForeignKey("customers.id"), nullable=False)
    status = Column(String(30), default="PLACED", nullable=False)  # PLACED, CONFIRMED, PREPARING, OUT_FOR_DELIVERY, DELIVERED, CANCELLED
    total_amount = Column(Numeric(10, 2), default=0)
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())

    customer = relationship("Customer")
    items = relationship("OrderItem", back_populates="order", cascade="all, delete-orphan")
```

---

## `backend/models/order_item.py`

```python
from sqlalchemy import Column, Integer, ForeignKey, Numeric
from sqlalchemy.orm import relationship
from backend.db.database import Base

class OrderItem(Base):
    __tablename__ = "order_items"

    id = Column(Integer, primary_key=True)
    order_id = Column(Integer, ForeignKey("orders.id", ondelete="CASCADE"))
    menu_item_id = Column(Integer, ForeignKey("menu_items.id"))
    quantity = Column(Integer, nullable=False, default=1)
    unit_price = Column(Numeric(10, 2), nullable=False)

    order = relationship("Order", back_populates="items")
```

---

## `backend/schemas/__init__.py`

```python
from .customer import CustomerCreate, CustomerOut
from .menu_item import MenuItemCreate, MenuItemOut
from .order import OrderCreate, OrderItemCreate, OrderOut, OrderStatusUpdate
```

---

## `backend/schemas/customer.py`

```python
from pydantic import BaseModel, EmailStr
from typing import Optional

class CustomerCreate(BaseModel):
    name: str
    phone: str
    email: Optional[EmailStr] = None
    address: Optional[str] = None

class CustomerOut(BaseModel):
    id: int
    name: str
    phone: str
    email: Optional[EmailStr] = None
    address: Optional[str] = None

    class Config:
        from_attributes = True
```

---

## `backend/schemas/menu_item.py`

```python
from pydantic import BaseModel
from typing import Optional
from decimal import Decimal

class MenuItemCreate(BaseModel):
    name: str
    description: Optional[str] = None
    price: Decimal
    is_available: bool = True
    category: Optional[str] = None

class MenuItemOut(BaseModel):
    id: int
    name: str
    description: Optional[str] = None
    price: Decimal
    is_available: bool
    category: Optional[str] = None

    class Config:
        from_attributes = True
```

---

## `backend/schemas/order.py`

```python
from pydantic import BaseModel
from typing import List
from decimal import Decimal

class OrderItemCreate(BaseModel):
    menu_item_id: int
    quantity: int

class OrderCreate(BaseModel):
    customer_phone: str
    items: List[OrderItemCreate]
    delivery_address: str | None = None

class OrderOutItem(BaseModel):
    menu_item_id: int
    quantity: int
    unit_price: Decimal

class OrderOut(BaseModel):
    id: int
    customer_id: int
    status: str
    total_amount: Decimal
    items: List[OrderOutItem]

    class Config:
        from_attributes = True

class OrderStatusUpdate(BaseModel):
    status: str
```

---

## `backend/services/order_service.py`

```python
from decimal import Decimal
from sqlalchemy.orm import Session
from backend.models import Customer, MenuItem, Order, OrderItem

class OrderService:
    @staticmethod
    def ensure_customer(db: Session, name: str | None, phone: str, address: str | None = None) -> Customer:
        customer = db.query(Customer).filter(Customer.phone == phone).first()
        if customer:
            if address and customer.address != address:
                customer.address = address
            return customer
        customer = Customer(name=name or "Guest", phone=phone, address=address)
        db.add(customer)
        db.flush()
        return customer

    @staticmethod
    def create_order(db: Session, customer: Customer, items: list[dict]) -> Order:
        order = Order(customer_id=customer.id, status="PLACED", total_amount=Decimal("0.00"))
        db.add(order)
        total = Decimal("0.00")
        for item in items:
            mi = db.query(MenuItem).get(item["menu_item_id"])
            if not mi or not mi.is_available:
                raise ValueError(f"Menu item {item['menu_item_id']} unavailable")
            qty = int(item["quantity"])
            line = OrderItem(order=order, menu_item_id=mi.id, quantity=qty, unit_price=mi.price)
            db.add(line)
            total += Decimal(mi.price) * qty
        order.total_amount = total
        db.flush()
        return order
```

---

## `backend/routers/menu.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from backend.db.database import get_db
from backend.models import MenuItem
from backend.schemas import MenuItemCreate, MenuItemOut
from typing import List

router = APIRouter(prefix="/menu", tags=["menu"])

@router.get("/", response_model=List[MenuItemOut])
def list_menu(db: Session = Depends(get_db)):
    items = db.query(MenuItem).filter(MenuItem.is_available == True).order_by(MenuItem.name).all()
    return items

@router.post("/", response_model=MenuItemOut)
def create_menu_item(payload: MenuItemCreate, db: Session = Depends(get_db)):
    if db.query(MenuItem).filter(MenuItem.name == payload.name).first():
        raise HTTPException(status_code=400, detail="Item already exists")
    item = MenuItem(**payload.dict())
    db.add(item)
    db.commit()
    db.refresh(item)
    return item
```

---

## `backend/routers/customers.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from backend.db.database import get_db
from backend.models import Customer
from backend.schemas import CustomerCreate, CustomerOut
from typing import List

router = APIRouter(prefix="/customers", tags=["customers"])

@router.post("/", response_model=CustomerOut)
def create_customer(payload: CustomerCreate, db: Session = Depends(get_db)):
    if db.query(Customer).filter(Customer.phone == payload.phone).first():
        raise HTTPException(status_code=400, detail="Phone already registered")
    c = Customer(**payload.dict())
    db.add(c)
    db.commit()
    db.refresh(c)
    return c

@router.get("/", response_model=List[CustomerOut])
def list_customers(db: Session = Depends(get_db)):
    return db.query(Customer).order_by(Customer.created_at.desc()).all()
```

---

## `backend/routers/orders.py`

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from backend.db.database import get_db
from backend.schemas import OrderCreate, OrderOut, OrderStatusUpdate
from backend.models import Order, OrderItem
from backend.services.order_service import OrderService

router = APIRouter(prefix="/orders", tags=["orders"])

@router.post("/", response_model=OrderOut)
def create_order(payload: OrderCreate, db: Session = Depends(get_db)):
    customer = OrderService.ensure_customer(db, name="Guest", phone=payload.customer_phone, address=payload.delivery_address)
    order = OrderService.create_order(db, customer, [i.dict() for i in payload.items])
    db.commit()
    db.refresh(order)
    return _serialize_order(order)

@router.get("/{order_id}", response_model=OrderOut)
def get_order(order_id: int, db: Session = Depends(get_db)):
    order = db.query(Order).get(order_id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return _serialize_order(order)

@router.patch("/{order_id}", response_model=OrderOut)
def update_status(order_id: int, payload: OrderStatusUpdate, db: Session = Depends(get_db)):
    order = db.query(Order).get(order_id)
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    order.status = payload.status
    db.commit()
    db.refresh(order)
    return _serialize_order(order)

# Helper

def _serialize_order(order: Order) -> OrderOut:
    return OrderOut(
        id=order.id,
        customer_id=order.customer_id,
        status=order.status,
        total_amount=order.total_amount,
        items=[
            {
                "menu_item_id": oi.menu_item_id,
                "quantity": oi.quantity,
                "unit_price": oi.unit_price,
            } for oi in order.items
        ]
    )
```

---

## `backend/routers/webhook.py` (Dialogflow Fulfillment)

```python
from fastapi import APIRouter, Request, Depends
from sqlalchemy.orm import Session
from backend.db.database import get_db
from backend.models import MenuItem, Order
from backend.services.order_service import OrderService

router = APIRouter(prefix="/webhook", tags=["dialogflow"])

@router.post("")
async def dialogflow_webhook(request: Request, db: Session = Depends(get_db)):
    body = await request.json()
    query_result = body.get("queryResult", {})
    intent = query_result.get("intent", {}).get("displayName")
    params = query_result.get("parameters", {})

    if intent == "Default Welcome Intent":
        return _simple_response("Hi! I'm your food assistant. You can say 'Order a Margherita pizza'.")

    if intent == "PlaceOrder":
        # Expected parameters: phone, address, items (as list of objects {food: name, number: qty})
        phone = str(params.get("phone")) if params.get("phone") else ""
        address = params.get("address") or None
        items_param = params.get("items") or []
        items = []
        for it in items_param:
            name = it.get("food") or it.get("food-item") or it.get("item")
            qty = int(it.get("number") or it.get("quantity") or 1)
            menu = db.query(MenuItem).filter(MenuItem.name.ilike(f"%{name}%")).first()
            if not menu:
                return _simple_response(f"Sorry, I couldn't find '{name}'. Try another item or say 'menu'.")
            items.append({"menu_item_id": menu.id, "quantity": qty})
        customer = OrderService.ensure_customer(db, name="Guest", phone=phone, address=address)
        order = OrderService.create_order(db, customer, items)
        db.commit()
        return _simple_response(f"Order #{order.id} placed! Total ₹{order.total_amount}. Say 'track order {order.id}'.")

    if intent == "TrackOrder":
        order_id = int(params.get("order_id") or 0)
        if not order_id:
            return _simple_response("Please provide an order id, e.g., 'track order 101'.")
        order = db.query(Order).get(order_id)
        if not order:
            return _simple_response("I couldn't find that order id.")
        return _simple_response(f"Order #{order.id} is currently {order.status}.")

    # Fallback
    return _simple_response("Sorry, I didn't get that. You can say: 'show menu', 'order a pizza', or 'track order 101'.")

# Dialogflow webhook response format helper

def _simple_response(text: str):
    return {
        "fulfillmentMessages": [
            {
                "text": {"text": [text]}
            }
        ]
    }
```

---

## `backend/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from backend.core.config import settings
from backend.db.database import Base, engine
from backend.models import *  # import models so metadata is registered
from backend.routers import menu, customers, orders, webhook

# Create tables if they don't exist (for dev/demo). For prod, use migrations.
Base.metadata.create_all(bind=engine)

app = FastAPI(title=settings.APP_NAME)

app.add_middleware(
    CORSMiddleware,
    allow_origins=[settings.CORS_ORIGINS] if settings.CORS_ORIGINS != "*" else ["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(menu.router)
app.include_router(customers.router)
app.include_router(orders.router)
app.include_router(webhook.router)

@app.get("/")
async def root():
    return {"status": "ok", "app": settings.APP_NAME}
```

---

## `database/schema.sql`

```sql
CREATE DATABASE IF NOT EXISTS foodbot CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE foodbot;

CREATE TABLE IF NOT EXISTS customers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  phone VARCHAR(20) NOT NULL UNIQUE,
  email VARCHAR(120) UNIQUE,
  address VARCHAR(255),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS menu_items (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(120) NOT NULL UNIQUE,
  description VARCHAR(255),
  price DECIMAL(10,2) NOT NULL,
  is_available TINYINT(1) NOT NULL DEFAULT 1,
  category VARCHAR(50)
);

CREATE TABLE IF NOT EXISTS orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  status VARCHAR(30) NOT NULL DEFAULT 'PLACED',
  total_amount DECIMAL(10,2) DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE TABLE IF NOT EXISTS order_items (
  id INT AUTO_INCREMENT PRIMARY KEY,
  order_id INT NOT NULL,
  menu_item_id INT NOT NULL,
  quantity INT NOT NULL DEFAULT 1,
  unit_price DECIMAL(10,2) NOT NULL,
  CONSTRAINT fk_order_items_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  CONSTRAINT fk_order_items_menu FOREIGN KEY (menu_item_id) REFERENCES menu_items(id)
);
```

---

## `database/seed.sql`

```sql
USE foodbot;
INSERT INTO menu_items (name, description, price, is_available, category) VALUES
('Margherita Pizza','Classic cheese & tomato',299.00,1,'pizza'),
('Farmhouse Pizza','Veggies overload',399.00,1,'pizza'),
('Veg Burger','Crispy patty with veggies',149.00,1,'burger'),
('French Fries','Potato fries',99.00,1,'sides'),
('Coke 500ml','Soft drink',59.00,1,'beverage')
ON DUPLICATE KEY UPDATE name=VALUES(name);
```

---

## `dialogflow_agent/agent.json`

```json
{
  "displayName": "FoodDeliveryBot",
  "defaultLanguageCode": "en",
  "timeZone": "Asia/Kolkata",
  "webhook": {
    "enabled": true
  }
}
```

---

## `dialogflow_agent/intents/Default Welcome Intent.json`

```json
{
  "name": "Default Welcome Intent",
  "trainingPhrases": [
    {"parts": [{"text": "hi"}]},
    {"parts": [{"text": "hello"}]},
    {"parts": [{"text": "hey"}]}
  ],
  "messages": [
    {"text": {"text": ["Hi! I can help you order food. Try 'order a Margherita pizza'."]}}
  ]
}
```

---

## `dialogflow_agent/intents/PlaceOrder.json`

```json
{
  "name": "PlaceOrder",
  "parameters": [
    {"id": "phone", "entityType": "@sys.phone-number", "isList": false},
    {"id": "address", "entityType": "@sys.location", "isList": false},
    {"id": "items", "entityType": "@item", "isList": true}
  ],
  "trainingPhrases": [
    {"parts": [{"text": "I want to order Margherita"}]},
    {"parts": [{"text": "Order 2 veg burgers"}]},
    {"parts": [{"text": "Get me a farmhouse pizza"}]}
  ]
}
```

---

## `dialogflow_agent/intents/TrackOrder.json`

```json
{
  "name": "TrackOrder",
  "parameters": [
    {"id": "order_id", "entityType": "@sys.number", "isList": false}
  ],
  "trainingPhrases": [
    {"parts": [{"text": "track order 101"}]},
    {"parts": [{"text": "where is my order 5"}]}
  ]
}
```

---

## `dialogflow_agent/intents/Fallback.json`

```json
{
  "name": "Default Fallback Intent",
  "messages": [
    {"text": {"text": ["Sorry, I didn't get that. Try 'order a pizza' or 'track order 101'."]}}
  ]
}
```

---

## `dialogflow_agent/entities/food-item.json`

```json
{
  "name": "@item",
  "isRegexp": false,
  "automatedExpansion": true,
  "synonyms": [
    { "value": "Margherita Pizza", "synonyms": ["margherita", "cheese pizza"]},
    { "value": "Veg Burger", "synonyms": ["burger", "veg burger"]},
    { "value": "French Fries", "synonyms": ["fries", "french fries"]}
  ]
}
```

> Note: Dialogflow CX/ES export formats differ. The above is a simplified stub. Import these as guidance and build equivalent entities/intents inside Dialogflow if direct import fails.

---

## `.env.example`

```env
APP_NAME=Food Delivery Ch
```
