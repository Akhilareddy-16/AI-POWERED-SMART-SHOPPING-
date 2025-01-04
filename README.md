# AI-POWERED-SMART-SHOPPING-
from flask import Flask, render_template, request, redirect, url_for, session, flash
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime
import mysql.connector
from sqlalchemy.sql.functions import current_user
from jinja2 import Environment,FileSystemLoader
import os
app = Flask(__name__, static_folder='static')

# Configuration
app.secret_key = os.urandom(24)  # Set a unique secret key
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:9848@localhost/budgetbuddy'  # Replace with your DB details
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

# Models
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password = db.Column(db.String(80), nullable=False)

class Expense(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    amount = db.Column(db.Float, nullable=False)
    date = db.Column(db.Date, nullable=False)
    user_id = db.Column(db.Integer, nullable=False)
    category = db.Column(db.String(255), nullable=True)
# Initialize Database
with app.app_context():
    db.create_all()
    with app.app_context():
        db.session.commit()
    

# Routes
@app.route('/')
def home():
    if 'user_id' in session:
        return redirect(url_for('dashboard'))
    return render_template('home.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')

        # Check if username exists
        existing_user = User.query.filter_by(username=username).first()
        if existing_user:
            flash("Username already exists.", "danger")
            return redirect(url_for('register'))

        # Register new user
        new_user = User(username=username, password=password)
        db.session.add(new_user)
        db.session.commit()
        flash("Registration successful! Please log in.", "success")
        return redirect(url_for('login'))

    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')

        # Authenticate user
        user = User.query.filter_by(username=username, password=password).first()
        if user:
            session['user_id'] = user.id
            flash("Login successful!", "success")
            return redirect(url_for('dashboard'))
        else:
            flash("Invalid username or password.", "danger")

    return render_template('login.html')

@app.route('/dashboard')
def dashboard():
    if 'user_id' in session:
        
        return render_template('dashboard.html')
    flash("Please log in to access the dashboard.", "danger")
    return redirect(url_for('login'))

@app.route('/add_expense', methods=['GET', 'POST'])
def add_expense():
    if 'user_id' in session:
        if request.method == 'POST':
            title = request.form.get('title')
            amount = request.form.get('amount')
            category = request.form['category']
            date = request.form.get('date')

            if not title or not amount or not date:
                flash("All fields are required.", "danger")
                return redirect(url_for('add_expense'))

            new_expense = Expense(
                title=title,
                category=category,
                amount=float(amount),
                date=datetime.strptime(date, '%Y-%m-%d'),
                user_id=session['user_id']
            )
            db.session.add(new_expense)
            db.session.commit()
            flash("Expense added successfully!", "success")
            return redirect(url_for('view_expenses'))

        return render_template('add_expense.html')
    flash("Please log in to add expenses.", "danger")
    return redirect(url_for('login'))

import mysql.connector
from sklearn.linear_model import LinearRegression
from datetime import datetime
import pandas as pd

# Database connection details
db_config = {
    "host": "localhost",
    "user": "root",
    "password": "9848",
    "database": "budgetbuddy"
}

# Fetch data for the current month
def fetch_data():
    db = mysql.connector.connect(**db_config)
    cursor = db.cursor()

    current_month = datetime.now().month
    current_year = datetime.now().year
    query = f"""
    SELECT id, title, amount, DAY(date) as day
    FROM expense
    WHERE MONTH(date) = {current_month} AND YEAR(date) = {current_year}
    """
    cursor.execute(query)
    data = cursor.fetchall()
    db.close()

    if not data:
        return pd.DataFrame()  # Return an empty DataFrame if no data
    return pd.DataFrame(data, columns=["id", "title", "amount", "day"])

# Generate predictions for future expenses
def generate_predictions():
    df = fetch_data()
    if df.empty:  # No data to process
        print("No data available for prediction.")
        return

    # Features and target for training
    X = df[["day"]]  # Using 'day' of the month as a feature
    y = df["amount"]  # Target is the amount

    # Train the Linear Regression model
    model = LinearRegression()
    model.fit(X, y)

    # Predict future prices
    df["future_prediction"] = model.predict(X)

    # Constrain predictions to ±10% of the original amount
    df["future_prediction"] = df.apply(
        lambda row: max(row["amount"] * 0.9, min(row["future_prediction"], row["amount"] * 1.1)),
        axis=1
    )

    # Update predictions in the database
    update_predictions(df)

# Update predictions in the database
def update_predictions(df):
    db = mysql.connector.connect(**db_config)
    cursor = db.cursor()

    for index, row in df.iterrows():
        query = f"""
        UPDATE expense
        SET future_prediction = {row['future_prediction']}
        WHERE id = {row['id']}
        """
        cursor.execute(query)

    db.commit()
    db.close()
    print("Future predictions updated successfully in the database.")
@app.route('/view_expenses')
def view_expenses():
    # Fetch data for the current month and predictions
    db = mysql.connector.connect(**db_config)
    cursor = db.cursor()

    current_month = datetime.now().month
    current_year = datetime.now().year
    query = f"""
    SELECT title, amount, date, future_prediction
    FROM expense
    WHERE MONTH(date) = {current_month} AND YEAR(date) = {current_year};
    """
    cursor.execute(query)
    data = cursor.fetchall()

    db.close()

    # Render the data in the template
    return render_template('view_expenses.html', expenses=data)
import pymysql
from collections import defaultdict
@app.route('/monthly_report')
def monthly_report():
    monthly_expense = {}
    # Connect to the database
    connection = pymysql.connect(
        host="localhost",
        user="root",
        password="9848",
        database="budgetbuddy"
    )
    cursor = connection.cursor()

    # Query to fetch expenses grouped by month and category
    query = """
        SELECT DATE_FORMAT(date, '%Y-%m') AS month, category, SUM(amount) AS total
        FROM expense
        GROUP BY month, category
        ORDER BY month;
    """
    cursor.execute(query)
    expenses_list = cursor.fetchall()  # Fetch results

    # Process results into a dictionary
    monthly_expense = defaultdict(lambda: {"total": 0, "categories": {}})
    for month, category, total in expenses_list:
        monthly_expense[month]["categories"][category] = total
        monthly_expense[month]["total"] += total

    # Convert defaultdict to a regular dictionary
    monthly_expense = dict(monthly_expense)

    # Debugging: Print the data structure
    print(monthly_expense)

    # Pass the data to the template
    return render_template('monthly_report.html', monthly_expense=monthly_expense)
@app.route('/logout')
def logout():
    session.pop('user_id', None)
    flash("You have been logged out.", "success")
    return redirect(url_for('login'))

# Run the App
if __name__ == "__main__":
     generate_predictions()
     app.run(debug=True)
