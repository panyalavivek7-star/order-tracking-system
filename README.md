{
  "name": "order-tracking-system",
  "version": "1.0.0",
  "description": "A simple order tracking system using Node.js, Express, and SQLite",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.19.2",
    "sqlite3": "^5.1.7"
  }
}const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const path = require('path');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// Initialize SQLite Database
const db = new sqlite3.Database('./orders.db', (err) => {
    if (err) {
        console.error('Error opening database', err.message);
    } else {
        console.log('Connected to the SQLite database.');
        // Create orders table if it doesn't exist
        db.run(`CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            tracking_id TEXT UNIQUE,
            customer_name TEXT,
            item_name TEXT,
            status TEXT,
            location TEXT,
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )`);
    }
});

// API: Create a new order (useful for testing/admin)
app.post('/api/orders', (req, res) => {
    const { customer_name, item_name, status, location } = req.body;
    const tracking_id = 'TRK-' + Math.random().toString(36).substring(2, 9).toUpperCase();

    const query = `INSERT INTO orders (tracking_id, customer_name, item_name, status, location) VALUES (?, ?, ?, ?, ?)`;
    db.run(query, [tracking_id, customer_name, item_name, status || 'Order Placed', location || 'Warehouse'], function(err) {
        if (err) {
            return res.status(500).json({ error: err.message });
        }
        res.json({
            message: 'Order created successfully',
            tracking_id: tracking_id,
            orderId: this.lastID
        });
    });
});

// API: Track an order by tracking ID
app.get('/api/orders/:trackingId', (req, res) => {
    const trackingId = req.params.trackingId.toUpperCase();
    const query = `SELECT * FROM orders WHERE tracking_id = ?`;
    
    db.get(query, [trackingId], (err, row) => {
        if (err) {
            return res.status(500).json({ error: err.message });
        }
        if (!row) {
            return res.status(404).json({ error: 'Order not found with this tracking ID.' });
        }
        res.json(row);
    });
});

// Start Server
app.listen(PORT, () => {
    console.log(`Server is running on http://localhost:${PORT}`);
});<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Order Tracking System</title>
    <style>
        :root {
            --primary-color: #2ea44f;
            --bg-color: #0d1117;
            --card-bg: #161b22;
            --text-color: #c9d1d9;
            --border-color: #30363d;
        }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 40px 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
        }
        .container {
            width: 100%;
            max-width: 600px;
            background: var(--card-bg);
            border: 1px solid var(--border-color);
            border-radius: 6px;
            padding: 30px;
            box-shadow: 0 8px 24px rgba(0,0,0,0.5);
        }
        h1 {
            margin-top: 0;
            font-size: 24px;
            border-bottom: 1px solid var(--border-color);
            padding-bottom: 15px;
        }
        .search-box {
            display: flex;
            gap: 10px;
            margin-top: 20px;
        }
        input[type="text"] {
            flex: 1;
            padding: 10px 12px;
            background-color: #0d1117;
            border: 1px solid var(--border-color);
            border-radius: 6px;
            color: var(--text-color);
            font-size: 16px;
        }
        button {
            padding: 10px 20px;
            background-color: var(--primary-color);
            color: white;
            border: none;
            border-radius: 6px;
            font-size: 16px;
            font-weight: bold;
            cursor: pointer;
        }
        button:hover {
            opacity: 0.9;
        }
        .result-card {
            margin-top: 25px;
            background: #0d1117;
            border: 1px solid var(--border-color);
            border-radius: 6px;
            padding: 20px;
            display: none;
        }
        .result-item {
            margin-bottom: 12px;
        }
        .result-item label {
            font-weight: bold;
            color: #8b949e;
            display: block;
            font-size: 13px;
            margin-bottom: 3px;
        }
        .error-msg {
            color: #f85149;
            margin-top: 15px;
            display: none;
        }
    </style>
</head>
<body>

    <div class="container">
        <h1>📦 Order Tracking System</h1>
        <p>Enter your tracking reference ID below to check live shipping status.</p>
        
        <div class="search-box">
            <input type="text" id="trackingInput" placeholder="e.g. TRK-ABC1234">
            <button onclick="trackOrder()">Track</button>
        </div>

        <div id="errorMsg" class="error-msg"></div>

        <div id="resultCard" class="result-card">
            <h3>Shipment Details</h3>
            <div class="result-item">
                <label>Tracking ID</label>
                <span id="resTrackingId"></span>
            </div>
            <div class="result-item">
                <label>Item Name</label>
                <span id="resItem"></span>
            </div>
            <div class="result-item">
                <label>Customer Name</label>
                <span id="resCustomer"></span>
            </div>
            <div class="result-item">
                <label>Current Status</label>
                <span id="resStatus" style="color: #2ea44f; font-weight: bold;"></span>
            </div>
            <div class="result-item">
                <label>Current Location</label>
                <span id="resLocation"></span>
            </div>
        </div>
    </div>

    <script>
        async function trackOrder() {
            const trackingId = document.getElementById('trackingInput').value.trim();
            const resultCard = document.getElementById('resultCard');
            const errorMsg = document.getElementById('errorMsg');

            resultCard.style.display = 'none';
            errorMsg.style.display = 'none';

            if (!trackingId) {
                errorMsg.innerText = 'Please enter a valid tracking ID.';
                errorMsg.style.display = 'block';
                return;
            }

            try {
                const response = await fetch(`/api/orders/${trackingId}`);
                const data = await response.json();

                if (!response.ok) {
                    throw new Error(data.error || 'Order not found.');
                }

                document.getElementById('resTrackingId').innerText = data.tracking_id;
                document.getElementById('resItem').innerText = data.item_name;
                document.getElementById('resCustomer').innerText = data.customer_name;
                document.getElementById('resStatus').innerText = data.status;
                document.getElementById('resLocation').innerText = data.location;

                resultCard.style.display = 'block';
            } catch (err) {
                errorMsg.innerText = err.message;
                errorMsg.style.display = 'block';
            }
        }
    </script>
</body>
</html>
