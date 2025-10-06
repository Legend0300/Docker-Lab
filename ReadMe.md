
## 📋 Lab Objective

Create a multi-container Docker application with Flask web server and MySQL database.

---

## 🔧 Prerequisites

- Docker Desktop installed and running
- Text editor**Explanation:**

- **FROM:** Base image with Python pre-installed
- **WORKDIR**Web Service (`web`):**

- **build: .** - Builds image from Dockerfile in current directory
- **ports: "5000:5000"** - Maps Flask port 5000 to host port 5000
- **depends_on: db** - Ensures MySQL starts before Flask

---

### Step 8: Create Backup Directory `/app` as the working directory
- **COPY requirements.txt first:** Optimizes Docker build caching
- **RUN pip install:** Installs Flask and MySQL connector
- **COPY . .:** Copies all project files into the container
- **CMD:** Starts the Flask application

---

### Step 6: Create .dockerignore (Optional)ecommended)
- Terminal/PowerShell access

---

## 📚 Step-by-Step Guide

### Step 1: Create Project Directory

```powershell
mkdir docker-todo-app
cd docker-todo-app
```

---

### Step 2: Create Python Dependencies File

Create `requirements.txt`:

```txt
Flask==2.3.0
mysql-connector-python==8.0.33
```

---

### Step 3: Create Flask Application

Create `app.py`:

```python
from flask import Flask, render_template, request, redirect
import mysql.connector
import time

app = Flask(__name__)

def get_db_connection():
    # Retry connection for 30 seconds (MySQL takes time to initialize)
    for i in range(30):
        try:
            conn = mysql.connector.connect(
                host='db',          # Service name from docker-compose.yml
                user='root',
                password='root',
                database='todos_db'
            )
            return conn
        except mysql.connector.Error:
            time.sleep(1)
    raise Exception("Could not connect to database")

def init_db():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS todos (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            task TEXT NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    cursor.close()
    conn.close()

@app.route('/')
def index():
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute('SELECT * FROM todos ORDER BY id DESC')
    todos = cursor.fetchall()
    cursor.close()
    conn.close()
    return render_template('index.html', todos=todos)

@app.route('/add', methods=['POST'])
def add_todo():
    name = request.form.get('name')
    task = request.form.get('task')
    
    if name and task:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute('INSERT INTO todos (name, task) VALUES (%s, %s)', (name, task))
        conn.commit()
        cursor.close()
        conn.close()
    
    return redirect('/')

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000, debug=True)
```

---

### Step 4: Create HTML Template

Create `templates` folder:

```powershell
mkdir templates
```

Create `templates/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Todo App</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100 min-h-screen py-8">
    <div class="container mx-auto max-w-2xl px-4">
        <h1 class="text-4xl font-bold text-center mb-8 text-blue-600">Todo Application</h1>
        
        <!-- Add Todo Form -->
        <div class="bg-white rounded-lg shadow-md p-6 mb-8">
            <h2 class="text-2xl font-semibold mb-4">Add New Todo</h2>
            <form method="POST" action="/add" class="space-y-4">
                <div>
                    <label class="block text-gray-700 font-medium mb-2">Name:</label>
                    <input type="text" name="name" required 
                           class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                           placeholder="Enter your name">
                </div>
                <div>
                    <label class="block text-gray-700 font-medium mb-2">Task:</label>
                    <textarea name="task" required rows="3"
                              class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                              placeholder="Describe your task"></textarea>
                </div>
                <button type="submit" 
                        class="w-full bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded-lg transition duration-200">
                    Add Todo
                </button>
            </form>
        </div>

        <!-- Todo List -->
        <div class="bg-white rounded-lg shadow-md p-6">
            <h2 class="text-2xl font-semibold mb-4">Todo List</h2>
            {% if todos %}
                <div class="space-y-4">
                    {% for todo in todos %}
                    <div class="border-l-4 border-blue-500 bg-gray-50 p-4 rounded">
                        <div class="font-semibold text-lg text-gray-800">{{ todo.name }}</div>
                        <div class="text-gray-600 mt-2">{{ todo.task }}</div>
                        <div class="text-sm text-gray-400 mt-2">{{ todo.created_at }}</div>
                    </div>
                    {% endfor %}
                </div>
            {% else %}
                <p class="text-gray-500 text-center py-8">No todos yet. Add one above!</p>
            {% endif %}
        </div>
    </div>
</body>
</html>
```

---

### Step 5: Create Dockerfile

Create `Dockerfile`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

**Explanation:**

- **FROM:** Base image with Python pre-installed
- **WORKDIR:** Sets `/app` as the working directory
- **COPY requirements.txt first:** Optimizes Docker build caching
- **RUN pip install:** Installs Flask and MySQL connector
- **COPY: ** Copies all project files into the container
- **CMD:** Starts the Flask application
---

### Step 6: Create .dockerignore (Optional)

Create `.dockerignore`:

```
__pycache__
*.pyc
*.pyo
*.pyd
.Python
data
backup
*.log
.git
.gitignore
```

---

### Step 7: Create Docker Compose File

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    container_name: mysql_db
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: todos_db
    ports:
      - "3306:3306"
    volumes:
      - ./backup:/backup
    tmpfs:
      - /var/lib/mysql    # Temporary storage - data deleted on stop

  web:
    build: .
    container_name: flask_app
    ports:
      - "5000:5000"
    depends_on:
      - db
```

**Key Configuration Details:**

**Database Service (`db`):**

- **image: mysql:8.0** - Uses official MySQL 8.0 Docker image
- **MYSQL_ROOT_PASSWORD: root** - Sets root user password
- **MYSQL_DATABASE: todos_db** - Automatically creates database on startup
- **ports: "3306:3306"** - Maps MySQL port for external access (optional, for debugging)
- **volumes: ./backup:/backup** - Mounts local `backup` folder for database backups
- **tmpfs: /var/lib/mysql** - Stores database files in memory (deleted on container stop)

**Web Service (`web`):**

- **build: .** - Builds image from Dockerfile in current directory
- **ports: "5000:5000"** - Maps Flask port 5000 to host port 5000
- **depends_on: db** - Ensures MySQL starts before Flask
---

### Step 8: Create Backup Directory

```powershell
mkdir backup
```

---

## 📁 Final Project Structure

```
docker-todo-app/
├── app.py
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .dockerignore (optional)
├── templates/
│   └── index.html
└── backup/
    └── todos_backup.sql
```

---

### Step 9: Build and Run

```powershell
docker-compose up --build
```

Wait for both containers to start (30-60 seconds for MySQL initialization).

---

### Step 10: Access Application

Open browser:
```
http://localhost:5000
```

---

### Step 11: Test the Application

1. Enter your name: `Ahmad`
2. Enter task: `Complete Docker lab`
3. Click **Add Todo**
4. Verify todo appears in list

Add more to-dos to test:
- Name: `Sarah`, Task: `Learn Kubernetes`
- Name: `John`, Task: `Practice Docker Compose`

---

### Step 12: Verify Database

Open a new terminal in VS Code or open command prompt.

Check data in MySQL:

```powershell
docker exec -it mysql_db mysql -uroot -proot -e "SELECT * FROM todos_db.todos;"
```

---

### Step 13: Backup Database

Create backup file (Make sure you are in `docker-todo-app` folder and `backup` folder is created).

Open command prompt and execute this command:

```powershell
docker exec mysql_db mysqldump -uroot -proot todos_db > backup/todos_backup.sql
```

The backup is saved in `backup/todos_backup.sql` on your machine.

---

### Step 14: Stop Containers (Data Loss Demo)

```powershell
docker-compose down
```

**Or press `Ctrl + C` in terminal.**

Data is now deleted (tmpfs storage).

---

### Step 15: Restart and Restore

Restart containers:

```powershell
docker-compose up --build
```

Visit `http://localhost:5000` - Todo list is empty.

Restore from backup:

Open Command Prompt or type `cmd` in VS Code:

```powershell
docker exec -i mysql_db mysql -uroot -proot todos_db < backup/todos_backup.sql
```

Refresh browser - todos are back!

---

### Step 16: Stop Containers

```powershell
docker-compose down
```

---

## 🔧 Useful Commands

### View running containers

```powershell
docker ps
```

### View logs

```powershell
docker logs flask_app
docker logs mysql_db
```

### Access container shell

```powershell
docker exec -it mysql_db bash
docker exec -it flask_app bash
```

### Rebuild after code changes

```powershell
docker-compose down
docker-compose up --build
```

### List networks

```powershell
docker network ls
```

### Complete cleanup

```powershell
docker-compose down -v
```

---

## 🚨 Troubleshooting

### Flask can't connect to MySQL

```powershell
docker logs mysql_db
```

Wait for: `ready for connections`

### Port already in use

Change port in docker-compose.yml:

```yaml
ports:
  - "5001:5000"
```

### Changes not showing

```powershell
docker-compose down
docker-compose up --build
```
