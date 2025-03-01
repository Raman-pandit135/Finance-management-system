// Backend Setup with Express and MongoDB const express = require('express'); const mongoose = require('mongoose'); const cors = require('cors'); require('dotenv').config();

const app = express(); app.use(express.json()); app.use(cors());

mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true, }) .then(() => console.log('Connected to MongoDB successfully!')) .catch(err => console.log('MongoDB Connection Error:', err));

const expenseRoutes = require('./routes/expenseRoutes'); app.use('/api/expenses', expenseRoutes);

app.listen(5000, () => console.log('Server is running on port 5000'));

// Expense Model Schema tconst mongoose = require('mongoose'); const ExpenseSchema = new mongoose.Schema({ title: { type: String, required: true }, amount: { type: Number, required: true }, category: { type: String, required: true }, date: { type: Date, default: Date.now }, }); module.exports = mongoose.model('Expense', ExpenseSchema);

// Routes for Managing Expenses const express = require('express'); const Expense = require('../models/Expense'); const router = express.Router();

router.post('/', async (req, res) => { try { const { title, amount, category } = req.body; const newExpense = new Expense({ title, amount, category }); await newExpense.save(); res.status(201).json(newExpense); } catch (error) { res.status(500).json({ error: error.message }); } });

router.get('/', async (req, res) => { try { const expenses = await Expense.find(); res.json(expenses); } catch (error) { res.status(500).json({ error: error.message }); } });

module.exports = router;

// Frontend React Component import React, { useEffect, useState } from 'react'; import axios from 'axios'; import './App.css';

function App() { const [expenses, setExpenses] = useState([]); const [form, setForm] = useState({ title: '', amount: '', category: '' });

useEffect(() => {
    axios.get('http://localhost:5000/api/expenses')
        .then(response => setExpenses(response.data))
        .catch(error => console.error('Error fetching expenses:', error));
}, []);

const handleChange = (e) => {
    setForm({ ...form, [e.target.name]: e.target.value });
};

const handleSubmit = async (e) => {
    e.preventDefault();
    try {
        const response = await axios.post('http://localhost:5000/api/expenses', form);
        setExpenses([...expenses, response.data]);
        setForm({ title: '', amount: '', category: '' });
    } catch (error) {
        console.error('Error adding expense:', error);
    }
};

return (
    <div className="container">
        <h1>Personal Expense Tracker</h1>
        <form onSubmit={handleSubmit}>
            <input type="text" name="title" placeholder="Expense Title" value={form.title} onChange={handleChange} required />
            <input type="number" name="amount" placeholder="Amount" value={form.amount} onChange={handleChange} required />
            <input type="text" name="category" placeholder="Category" value={form.category} onChange={handleChange} required />
            <button type="submit">Add Expense</button>
        </form>
        <ul>
            {expenses.map(expense => (
                <li key={expense._id}>💰 {expense.title} - ${expense.amount} ({expense.category})</li>
            ))}
        </ul>
    </div>
);

}

export default App;



const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const dotenv = require("dotenv");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcryptjs");
const { Server } = require("socket.io");
const { createServer } = require("http");

dotenv.config();
const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, { cors: { origin: "*" } });

app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true });

// User Schema
const UserSchema = new mongoose.Schema({
  username: String,
  email: String,
  password: String,
});
const User = mongoose.model("User", UserSchema);

// Expense Schema
const ExpenseSchema = new mongoose.Schema({
  userId: String,
  title: String,
  amount: Number,
  category: String,
  date: Date,
});
const Expense = mongoose.model("Expense", ExpenseSchema);

// Signup API
app.post("/signup", async (req, res) => {
  const { username, email, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  const newUser = new User({ username, email, password: hashedPassword });
  await newUser.save();
  res.status(201).json({ message: "User registered!" });
});

// Login API
app.post("/login", async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (user && (await bcrypt.compare(password, user.password))) {
    const token = jwt.sign({ userId: user._id }, "secretkey");
    res.json({ token });
  } else {
    res.status(400).json({ error: "Invalid credentials" });
  }
});

// Get Expenses
app.get("/expenses", async (req, res) => {
  const expenses = await Expense.find();
  res.json(expenses);
});

// Create Expense
app.post("/expenses", async (req, res) => {
  const newExpense = new Expense(req.body);
  await newExpense.save();
  io.emit("expenseUpdated");
  res.status(201).json(newExpense);
});

// Update Expense
app.put("/expenses/:id", async (req, res) => {
  await Expense.findByIdAndUpdate(req.params.id, req.body);
  io.emit("expenseUpdated");
  res.json({ message: "Expense updated!" });
});

// Delete Expense
app.delete("/expenses/:id", async (req, res) => {
  await Expense.findByIdAndDelete(req.params.id);
  io.emit("expenseUpdated");
  res.json({ message: "Expense deleted!" });
});

// Start Server
const PORT = process.env.PORT || 5000;
httpServer.listen(PORT, () => console.log(`Server running on port ${PORT}`));


npx create-react-app client  
cd client  
npm install axios socket.io-client react-router-dom @mui/material @emotion/react @emotion/styled


import React, { useEffect, useState } from "react";
import axios from "axios";
import io from "socket.io-client";

const socket = io("http://localhost:5000");

function App() {
  const [expenses, setExpenses] = useState([]);

  useEffect(() => {
    fetchExpenses();
    socket.on("expenseUpdated", fetchExpenses);
  }, []);

  const fetchExpenses = async () => {
    const { data } = await axios.get("http://localhost:5000/expenses");
    setExpenses(data);
  };

  return (
    <div>
      <h2>Expense Tracker</h2>
      <ul>
        {expenses.map((exp) => (
          <li key={exp._id}>{exp.title} - ${exp.amount}</li>
        ))}
      </ul>
    </div>
  );
}

export default App;


FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "server.js"]
EXPOSE 5000


FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
EXPOSE 3000


version: "3"
services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"


      <meta name="description" content="Track your expenses with real-time updates" />
<meta name="keywords" content="expense tracker, budget management" />


<meta property="og:title" content="Expense Manager" />
<meta property="og:description" content="Track your expenses easily with real-time updates" />
<meta property="og:image" content="https://example.com/preview.png" />
