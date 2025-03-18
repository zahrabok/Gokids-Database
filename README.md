# Gokids-Database
Repository for SQL database and website for gokids
# Gokids-Database
Repository for SQL database and website for gokids
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_bcrypt import Bcrypt
from flask_jwt_extended import JWTManager, create_access_token, jwt_required

app = Flask(__name__)

# 🔹 Azure SQL Database Configuration
app.config['SQLALCHEMY_DATABASE_URI'] = "mssql+pyodbc://gokids:12345678/a@gokids.database.windows.net/gokids?driver=ODBC+Driver+17+for+SQL+Server"
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['JWT_SECRET_KEY'] = 'your_secret_key'  # Change this for security

db = SQLAlchemy(app)
bcrypt = Bcrypt(app)
jwt = JWTManager(app)

# ✅ FIXED: Home Route Moved to the Right Position
@app.route('/')
def home():
    return jsonify({"message": "Welcome to the GoKids API!"})

# 🔹 Employee Model (For Login & Authentication)
class Employee(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    email = db.Column(db.String(100), unique=True, nullable=False)
    password = db.Column(db.String(200), nullable=False)

    def __init__(self, name, email, password):
        self.name = name
        self.email = email
        self.password = password

# 🔹 Client Model (For Storing Client Information)
class Client(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    parent_name = db.Column(db.String(100), nullable=False)
    contact_email = db.Column(db.String(100), unique=True, nullable=False)
    contact_phone = db.Column(db.String(20), nullable=False)
    referred_from = db.Column(db.String(100), nullable=False)
    children_ages = db.Column(db.String(100))
    working_with = db.Column(db.String(100))

# 🔹 Register Employee (POST /register)
@app.route('/register', methods=['POST'])
def register():
    data = request.get_json()
    hashed_password = bcrypt.generate_password_hash(data['password']).decode('utf-8')
    new_employee = Employee(name=data['name'], email=data['email'], password=hashed_password)
    db.session.add(new_employee)
    db.session.commit()
    return jsonify({"message": "Employee registered successfully!"})

# 🔹 Employee Login (POST /login)
@app.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    employee = Employee.query.filter_by(email=data['email']).first()
    
    if employee and bcrypt.check_password_hash(employee.password, data['password']):
        access_token = create_access_token(identity=employee.id)
        return jsonify({"access_token": access_token})

    return jsonify({"message": "Invalid credentials"}), 401

# 🔹 Get All Clients (GET /clients)
@app.route('/clients', methods=['GET'])
@jwt_required()
def get_clients():
    clients = Client.query.all()
    return jsonify([
        {
            "id": c.id, "parent_name": c.parent_name, "email": c.contact_email,
            "phone": c.contact_phone, "referred_from": c.referred_from,
            "children_ages": c.children_ages, "working_with": c.working_with
        } for c in clients
    ])

# 🔹 Add a New Client (POST /clients)
@app.route('/clients', methods=['POST'])
@jwt_required()
def add_client():
    data = request.get_json()
    new_client = Client(
        parent_name=data['parent_name'],
        contact_email=data['contact_email'],
        contact_phone=data['contact_phone'],
        referred_from=data['referred_from'],
        children_ages=data['children_ages'],
        working_with=data['working_with']
    )
    db.session.add(new_client)
    db.session.commit()
    return jsonify({"message": "Client added successfully!"})

# 🔹 Update Client Info (PUT /clients/<id>)
@app.route('/clients/<int:id>', methods=['PUT'])
@jwt_required()
def update_client(id):
    client = Client.query.get(id)
    if not client:
        return jsonify({"message": "Client not found"}), 404

    data = request.get_json()
    client.parent_name = data.get('parent_name', client.parent_name)
    client.contact_email = data.get('contact_email', client.contact_email)
    client.contact_phone = data.get('contact_phone', client.contact_phone)
    client.referred_from = data.get('referred_from', client.referred_from)
    client.children_ages = data.get('children_ages', client.children_ages)
    client.working_with = data.get('working_with', client.working_with)

    db.session.commit()
    return jsonify({"message": "Client updated successfully!"})

# 🔹 Delete Client (DELETE /clients/<id>)
@app.route('/clients/<int:id>', methods=['DELETE'])
@jwt_required()
def delete_client(id):
    client = Client.query.get(id)
    if not client:
        return jsonify({"message": "Client not found"}), 404

    db.session.delete(client)
    db.session.commit()
    return jsonify({"message": "Client deleted successfully!"})

# 🔹 Run the Flask App (Final Fix)
if __name__ == '__main__':
    app.run(debug=True)
