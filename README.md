from flask import Flask, render_template, request, redirect, url_for, session, flash
import sqlite3
import hashlib
from functools import wraps

app = Flask(__name__)
app.secret_key = 'your-secret-key-change-this'  # Change this in production!

# ---------------- DATABASE ----------------
def get_db():
    conn = sqlite3.connect("ecommerce.db")
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_db()
    cursor = conn.cursor()
    
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE,
        password TEXT
    )""")

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS products (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        price REAL,
        stock INTEGER DEFAULT 100
    )""")

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS cart (
        user_id INTEGER,
        product_id INTEGER,
        quantity INTEGER,
        UNIQUE(user_id, product_id)
    )""")

    cursor.execute("""
    CREATE TABLE IF NOT EXISTS orders (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        total REAL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )""")
    
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS order_items (
        order_id INTEGER,
        product_name TEXT,
        price REAL,
        quantity INTEGER
    )""")
    
    # Add default products
    cursor.execute("SELECT COUNT(*) FROM products")
    if cursor.fetchone()[0] == 0:
        products = [
            ("Laptop", 50000, 10),
            ("Phone", 20000, 25),
            ("Headphones", 2000, 50),
            ("Mouse", 500, 100),
            ("Keyboard", 1500, 75)
        ]
        cursor.executemany("INSERT INTO products (name, price, stock) VALUES (?, ?, ?)", products)
    
    conn.commit()
    conn.close()

# ---------------- HELPER FUNCTIONS ----------------
def hash_password(password):
    return hashlib.sha256(password.encode()).hexdigest()

def login_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            flash('Please login first', 'warning')
            return redirect(url_for('login'))
        return f(*args, **kwargs)
    return decorated_function

# ---------------- ROUTES ----------------
@app.route('/')
def index():
    return render_template('index.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        
        conn = get_db()
        cursor = conn.cursor()
        
        try:
            cursor.execute(
                "INSERT INTO users (username, password) VALUES (?, ?)",
                (username, hash_password(password))
            )
            conn.commit()
            flash('Registration successful! Please login.', 'success')
            return redirect(url_for('login'))
        except sqlite3.IntegrityError:
            flash('Username already exists', 'danger')
        finally:
            conn.close()
    
    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        
        conn = get_db()
        cursor = conn.cursor()
        cursor.execute(
            "SELECT id, username FROM users WHERE username=? AND password=?",
            (username, hash_password(password))
        )
        user = cursor.fetchone()
        conn.close()
        
        if user:
            session['user_id'] = user['id']
            session['username'] = user['username']
            flash('Login successful!', 'success')
            return redirect(url_for('products'))
        else:
            flash('Invalid credentials', 'danger')
    
    return render_template('login.html')

@app.route('/logout')
def logout():
    session.clear()
    flash('Logged out successfully', 'info')
    return redirect(url_for('index'))

@app.route('/products')
@login_required
def products():
    conn = get_db()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM products WHERE stock > 0")
    products = cursor.fetchall()
    conn.close()
    return render_template('products.html', products=products)

@app.route('/add_to_cart/<int:product_id>', methods=['POST'])
@login_required
def add_to_cart(product_id):
    quantity = int(request.form.get('quantity', 1))
    
    conn = get_db()
    cursor = conn.cursor()
    
    # Check if product exists and has stock
    cursor.execute("SELECT stock FROM products WHERE id=?", (product_id,))
    product = cursor.fetchone()
    
    if not product or product['stock'] < quantity:
        flash('Product not available or insufficient stock', 'danger')
        conn.close()
        return redirect(url_for('products'))
    
    # Add or update cart
    cursor.execute("""
        INSERT INTO cart (user_id, product_id, quantity) 
        VALUES (?, ?, ?)
        ON CONFLICT(user_id, product_id) 
        DO UPDATE SET quantity = quantity + ?
    """, (session['user_id'], product_id, quantity, quantity))
    
    conn.commit()
    conn.close()
    flash('Added to cart!', 'success')
    return redirect(url_for('products'))

@app.route('/cart')
@login_required
def view_cart():
    conn = get_db()
    cursor = conn.cursor()
    cursor.execute("""
        SELECT products.id, products.name, products.price, cart.quantity,
               (products.price * cart.quantity) as subtotal
        FROM cart 
        JOIN products ON cart.product_id = products.id
        WHERE cart.user_id=?
    """, (session['user_id'],))
    
    items = cursor.fetchall()
    total = sum(item['subtotal'] for item in items)
    conn.close()
    
    return render_template('cart.html', items=items, total=total)

@app.route('/remove_from_cart/<int:product_id>')
@login_required
def remove_from_cart(product_id):
    conn = get_db()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM cart WHERE user_id=? AND product_id=?", 
                   (session['user_id'], product_id))
    conn.commit()
    conn.close()
    flash('Item removed from cart', 'info')
    return redirect(url_for('view_cart'))

@app.route('/checkout', methods=['POST'])
@login_required
def checkout():
    conn = get_db()
    cursor = conn.cursor()
    
    # Get cart items
    cursor.execute("""
        SELECT products.id, products.name, products.price, cart.quantity, products.stock
        FROM cart 
        JOIN products ON cart.product_id = products.id
        WHERE cart.user_id=?
    """, (session['user_id'],))
    
    items = cursor.fetchall()
    
    if not items:
        flash('Cart is empty', 'warning')
        return redirect(url_for('view_cart'))
    
    # Check stock availability
    for item in items:
        if item['stock'] < item['quantity']:
            flash(f'Insufficient stock for {item["name"]}', 'danger')
            conn.close()
            return redirect(url_for('view_cart'))
    
    # Calculate total
    total = sum(item['price'] * item['quantity'] for item in items)
    
    # Create order
    cursor.execute("INSERT INTO orders (user_id, total) VALUES (?, ?)",
                   (session['user_id'], total))
    order_id = cursor.lastrowid
    
    # Add order items and update stock
    for item in items:
        cursor.execute("""
            INSERT INTO order_items (order_id, product_name, price, quantity)
            VALUES (?, ?, ?, ?)
        """, (order_id, item['name'], item['price'], item['quantity']))
        
        cursor.execute("""
            UPDATE products SET stock = stock - ? WHERE id = ?
        """, (item['quantity'], item['id']))
    
    # Clear cart
    cursor.execute("DELETE FROM cart WHERE user_id=?", (session['user_id'],))
    
    conn.commit()
    conn.close()
    
    flash(f'Order placed successfully! Order ID: {order_id}', 'success')
    return redirect(url_for('orders'))

@app.route('/orders')
@login_required
def orders():
    conn = get_db()
    cursor = conn.cursor()
    cursor.execute("""
        SELECT id, total, created_at FROM orders 
        WHERE user_id=? 
        ORDER BY created_at DESC
    """, (session['user_id'],))
    orders = cursor.fetchall()
    conn.close()
    return render_template('orders.html', orders=orders)

@app.route('/order/<int:order_id>')
@login_required
def order_detail(order_id):
    conn = get_db()
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT * FROM orders WHERE id=? AND user_id=?
    """, (order_id, session['user_id']))
    order = cursor.fetchone()
    
    if not order:
        flash('Order not found', 'danger')
        return redirect(url_for('orders'))
    
    cursor.execute("""
        SELECT * FROM order_items WHERE order_id=?
    """, (order_id,))
    items = cursor.fetchall()
    conn.close()
    
    return render_template('order_detail.html', order=order, items=items)

# ---------------- RUN ----------------
if __name__ == '__main__':
    init_db()
    app.run(debug=True, host='localhost', port=5000)
