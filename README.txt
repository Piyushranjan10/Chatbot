# 🍔 Food Delivery Chatbot (Dialogflow + NLP + FastAPI + MySQL)  

## 📌 Project Overview  
This project demonstrates an end-to-end **Food Delivery Chatbot** built using **Dialogflow** for Natural Language Processing (NLP), with a backend powered by **Python (FastAPI)** and a **MySQL database**.  

The chatbot can interact with users in natural language, handle food ordering, track orders, and manage customer details.  

---

## 🚀 Features  
- **Conversational AI** using Dialogflow NLP (intents, entities, contexts).  
- **Order placement & tracking** via chatbot interface.  
- **FastAPI backend** for handling API requests and business logic.  
- **MySQL integration** for storing menu data, customer details, and order history.  
- Scalable architecture for real-world deployment.  

---

## 🛠️ Tech Stack  
- **NLP & Conversational AI**: Dialogflow  
- **Backend**: Python, FastAPI  
- **Database**: MySQL  
- **APIs**: REST APIs for chatbot-backend communication  

---

## 📂 Project Structure  
```
📦 Food-Delivery-Chatbot
 ┣ 📂 dialogflow_agent       # Dialogflow agent configuration files
 ┣ 📂 backend                # FastAPI backend source code
 ┃ ┣ 📂 routers              # API route handlers
 ┃ ┣ 📂 models               # Database models
 ┃ ┗ main.py                 # Entry point of FastAPI app
 ┣ 📂 database               # MySQL scripts and schema
 ┣ 📄 requirements.txt       # Python dependencies
 ┗ 📄 README.md              # Project documentation
```

---

## ⚙️ Installation & Setup  

### 1️⃣ Clone the repository  
```bash
git clone https://github.com/Piyushranjan10/Chatbot.git
cd Chatbot
```

### 2️⃣ Setup Python environment  
```bash
pip install -r requirements.txt
```

### 3️⃣ Setup MySQL database  
- Import the SQL script from `database/` to create required tables.  
- Update database credentials in the FastAPI config file.  

### 4️⃣ Run FastAPI server  
```bash
uvicorn main:app --reload
```

### 5️⃣ Connect Dialogflow  
- Import the provided Dialogflow agent.  
- Set webhook URL in Dialogflow to your FastAPI endpoint.  

---

## 🎯 Example Use Cases  
- User: *“I want to order a pizza”*  
- Chatbot: *“Sure! Which type of pizza would you like?”*  
- User: *“A Margherita”*  
- Chatbot: *“Great! Your order has been placed. Would you like to track it?”*  

---

## 📊 Future Enhancements  
- Payment gateway integration.  
- Multi-language support.  
- Recommendation system for food items.  

---

## 🤝 Contributing  
Contributions, issues, and feature requests are welcome!  
Feel free to fork this repo and submit a pull request.  

---

## 📧 Contact  
👤 **Piyush Ranjan**  
- [LinkedIn](https://www.linkedin.com/in/piyush-ranjan-850638253)  
- [GitHub](https://github.com/Piyushranjan10)  
