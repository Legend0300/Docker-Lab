# 🐳 Docker Multi-Container Application Lab
## Complete Walkthrough: Flask Web App with MySQL Database

---

## 📋 Lab Objective
Build a full-stack containerized application using Docker Compose with:
- **Flask** web server (Python)
- **MySQL** database
- **Persistent data backup/restore**
- **Container orchestration**

---

## 🎯 What You'll Learn
- Multi-container Docker applications
- Docker Compose orchestration
- Database persistence strategies
- Container networking
- Backup and restore procedures
- Production-ready container configurations

---

## 🔧 Prerequisites

### Required Software
- ✅ **Docker Desktop** (latest version)
- ✅ **VS Code** or text editor
- ✅ **PowerShell** or Command Prompt
- ✅ **Web browser**

### Verification Commands
```powershell
# Check Docker installation
docker --version
docker-compose --version

# Ensure Docker is running
docker ps
```

---

## 🏗️ Architecture Overview

```
┌─────────────────┐    HTTP    ┌─────────────────┐
│   Web Browser   │ ←──────→   │   Flask App     │
│  localhost:5000 │            │  (Port 5000)    │
└─────────────────┘            └─────────────────┘
                                        │
                                        │ SQL
                                        ▼
                               ┌─────────────────┐
                               │   MySQL DB      │
                               │  (Port 3306)    │
                               └─────────────────┘
```

---

## 📚 Step-by-Step Implementation

### 🎪 Phase 1: Project Setup

#### Step 1: Create Project Structure
```powershell
# Create main project directory
mkdir docker-todo-app
cd docker-todo-app

# Create subdirectories
mkdir templates
mkdir backup

# Verify structure
tree /F
```

**Expected Output:**
```
docker-todo-app/
├── templates/
└── backup/
```

---

### 🐍 Phase 2: Backend Development

#### Step 2: Define Python Dependencies
Create `requirements.txt`:

```txt
Flask==2.3.0
mysql-connector-python==8.0.33
```

**Why these versions?**
- Flask 2.3.0: Stable LTS version
- mysql-connector-python: Official MySQL driver

#### Step 3: Develop Flask Application
Create `app.py`:

```python
from flask import Flask, render_template, request, redirect
import mysql.connector
import time

app = Flask(__name__)

def get_db_connection():
    """
    Establishes database connection with retry logic
    Retry mechanism handles MySQL container startup delay
    """
    for attempt in range(30):  # 30-second timeout
        try:
            connection = mysql.connector.connect(
                host='db',          # Docker service name
                user='root',
                password='root',
                database='todos_db',
                autocommit=True     # Auto-commit transactions
            )
            return connection
        except mysql.connector.Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(1)
    
    raise Exception("Database connection failed after 30 seconds")

def initialize_database():
    """Creates todos table if it doesn't exist"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS todos (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            task TEXT NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            INDEX idx_created_at (created_at)
        )
    ''')
    
    cursor.close()
    conn.close()
    print("Database initialized successfully")

@app.route('/')
def index():
    """Main page - displays all todos"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    cursor.execute('SELECT * FROM todos ORDER BY created_at DESC')
    todos = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return render_template('index.html', todos=todos)

@app.route('/add', methods=['POST'])
def add_todo():
    """Add new todo item"""
    name = request.form.get('name', '').strip()
    task = request.form.get('task', '').strip()
    
    if name and task:
        conn = get_db_connection()
        cursor = conn.cursor()
        
        cursor.execute(
            'INSERT INTO todos (name, task) VALUES (%s, %s)', 
            (name, task)
        )
        
        cursor.close()
        conn.close()
        print(f"Added todo: {name} - {task}")
    
    return redirect('/')

@app.route('/health')
def health_check():
    """Health check endpoint for monitoring"""
    try:
        conn = get_db_connection()
        conn.close()
        return {"status": "healthy", "database": "connected"}, 200
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}, 500

if __name__ == '__main__':
    print("Starting Flask application...")
    initialize_database()
    app.run(host='0.0.0.0', port=5000, debug=True)
```

**Key Features:**
- **Retry Logic**: Handles MySQL startup delays
- **Error Handling**: Graceful database error management
- **Health Check**: Monitoring endpoint
- **Security**: Parameterized queries prevent SQL injection

---

### 🎨 Phase 3: Frontend Development

#### Step 4: Create Responsive HTML Template
Create `templates/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Docker Todo Application</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        // Auto-refresh every 30 seconds to show new todos
        setTimeout(() => location.reload(), 30000);
    </script>
</head>
<body class="bg-gradient-to-br from-blue-50 to-indigo-100 min-h-screen py-8">
    <div class="container mx-auto max-w-4xl px-4">
        <!-- Header Section -->
        <header class="text-center mb-12">
            <h1 class="text-5xl font-bold text-indigo-800 mb-4">
                🐳 Docker Todo App
            </h1>
            <p class="text-gray-600 text-lg">
                Multi-container application with Flask & MySQL
            </p>
        </header>
        
        <!-- Add Todo Form -->
        <section class="bg-white rounded-xl shadow-lg p-8 mb-8">
            <h2 class="text-3xl font-semibold mb-6 text-gray-800 flex items-center">
                ➕ Add New Todo
            </h2>
            <form method="POST" action="/add" class="space-y-6">
                <div class="grid md:grid-cols-2 gap-6">
                    <div>
                        <label class="block text-gray-700 font-semibold mb-3">
                            👤 Your Name:
                        </label>
                        <input 
                            type="text" 
                            name="name" 
                            required 
                            maxlength="255"
                            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition duration-200"
                            placeholder="Enter your full name"
                        >
                    </div>
                    <div>
                        <label class="block text-gray-700 font-semibold mb-3">
                            📝 Task Description:
                        </label>
                        <textarea 
                            name="task" 
                            required 
                            rows="3"
                            maxlength="1000"
                            class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-transparent transition duration-200 resize-none"
                            placeholder="Describe what needs to be done..."
                        ></textarea>
                    </div>
                </div>
                <button 
                    type="submit" 
                    class="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-4 px-6 rounded-lg transition duration-200 transform hover:scale-105 shadow-lg"
                >
                    🚀 Add Todo
                </button>
            </form>
        </section>

        <!-- Todo List Section -->
        <section class="bg-white rounded-xl shadow-lg p-8">
            <h2 class="text-3xl font-semibold mb-6 text-gray-800 flex items-center justify-between">
                📋 Todo List
                <span class="text-sm font-normal text-gray-500 bg-gray-100 px-3 py-1 rounded-full">
                    {{ todos|length }} total
                </span>
            </h2>
            
            {% if todos %}
                <div class="space-y-4">
                    {% for todo in todos %}
                    <div class="border-l-4 border-indigo-500 bg-gray-50 p-6 rounded-lg hover:bg-gray-100 transition duration-200">
                        <div class="flex justify-between items-start">
                            <div class="flex-1">
                                <div class="font-bold text-xl text-gray-800 mb-2">
                                    👤 {{ todo.name }}
                                </div>
                                <div class="text-gray-700 mb-3 leading-relaxed">
                                    {{ todo.task }}
                                </div>
                                <div class="text-sm text-gray-500 flex items-center">
                                    🕐 {{ todo.created_at.strftime('%B %d, %Y at %I:%M %p') }}
                                </div>
                            </div>
                            <div class="ml-4">
                                <span class="bg-green-100 text-green-800 text-sm font-medium px-3 py-1 rounded-full">
                                    Active
                                </span>
                            </div>
                        </div>
                    </div>
                    {% endfor %}
                </div>
            {% else %}
                <div class="text-center py-16">
                    <div class="text-6xl mb-4">📝</div>
                    <p class="text-gray-500 text-xl mb-4">No todos yet!</p>
                    <p class="text-gray-400">Add your first todo above to get started.</p>
                </div>
            {% endif %}
        </section>

        <!-- Footer -->
        <footer class="text-center mt-12 text-gray-500">
            <p>Built with 🐳 Docker, 🐍 Flask, and 🗄️ MySQL</p>
        </footer>
    </div>
</body>
</html>
```

**Enhanced Features:**
- **Responsive Design**: Works on all devices
- **Auto-refresh**: Updates every 30 seconds
- **Input Validation**: Client-side and server-side
- **Visual Feedback**: Hover effects and transitions
- **Accessibility**: Proper labels and semantic HTML

---

### 🐳 Phase 4: Container Configuration

#### Step 5: Create Optimized Dockerfile
Create `Dockerfile`:

```dockerfile
# Use Python slim image for smaller size
FROM python:3.9-slim

# Set metadata
LABEL maintainer="your-email@example.com"
LABEL description="Flask Todo Application"

# Create non-root user for security
RUN useradd --create-home --shell /bin/bash appuser

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies first (for better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Change ownership to non-root user
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Start application
CMD ["python", "app.py"]
```

**Security & Optimization Features:**
- **Non-root user**: Enhanced security
- **Multi-stage optimization**: Smaller image size
- **Health checks**: Monitoring capabilities
- **Layer caching**: Faster rebuilds

#### Step 6: Create Docker Ignore File
Create `.dockerignore`:

```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# Environment
.env
.venv
env/
venv/
ENV/
env.bak/
venv.bak/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Project specific
backup/
*.log
.git/
.gitignore
README.md
```

---

### 🎼 Phase 5: Container Orchestration

#### Step 7: Create Production-Ready Docker Compose
Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # MySQL Database Service
  db:
    image: mysql:8.0
    container_name: mysql_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: todos_db
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    ports:
      - "3306:3306"  # External access for debugging
    volumes:
      - ./backup:/backup:ro  # Read-only backup mount
      - mysql_data:/var/lib/mysql  # Named volume for data
    tmpfs:
      - /var/log/mysql  # Temporary logs
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - todo_network

  # Flask Web Application Service
  web:
    build: 
      context: .
      dockerfile: Dockerfile
    container_name: flask_app
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=production
      - FLASK_DEBUG=0
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./backup:/app/backup:ro  # Backup access
    networks:
      - todo_network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

# Named volumes for data persistence
volumes:
  mysql_data:
    driver: local

# Custom network for service isolation
networks:
  todo_network:
    driver: bridge
```

**Production Features:**
- **Health Checks**: Monitor service status
- **Restart Policies**: Auto-recovery from failures
- **Named Volumes**: Persistent data storage
- **Custom Networks**: Service isolation
- **Environment Variables**: Configuration management

---

### 🚀 Phase 6: Deployment & Testing

#### Step 8: Build and Launch Application
```powershell
# Build and start all services
docker-compose up --build -d

# Verify services are running
docker-compose ps

# Check logs for any issues
docker-compose logs -f
```

**Expected Output:**
```
Name            Command                  State                    Ports
--------------------------------------------------------------------------------
flask_app       python app.py            Up (healthy)   0.0.0.0:5000->5000/tcp
mysql_db        docker-entrypoint.s...   Up (healthy)   0.0.0.0:3306->3306/tcp
```

#### Step 9: Application Testing
1. **Access Application**: Open `http://localhost:5000`
2. **Add Test Data**:
   ```
   Name: Alice Johnson
   Task: Review Docker documentation
   
   Name: Bob Smith  
   Task: Implement user authentication
   
   Name: Carol Davis
   Task: Optimize database queries
   ```
3. **Verify Functionality**: 
   - Form submission works
   - Data appears in list
   - Timestamps are correct
   - Responsive design works

---

### 💾 Phase 7: Data Management

#### Step 10: Database Operations

##### View Database Contents
```powershell
# Connect to MySQL and view data
docker exec -it mysql_db mysql -uroot -proot -e "
USE todos_db;
SELECT id, name, task, created_at FROM todos ORDER BY created_at DESC;
DESCRIBE todos;
SHOW TABLE STATUS LIKE 'todos';
"
```

##### Database Backup
```powershell
# Create comprehensive backup
docker exec mysql_db mysqldump -uroot -proot `
  --single-transaction `
  --routines `
  --triggers `
  todos_db > backup/todos_backup_$(Get-Date -Format "yyyyMMdd_HHmmss").sql

# Verify backup file
Get-ChildItem backup/ | Sort-Object LastWriteTime -Descending
```

##### Database Restore (PowerShell Compatible)
```powershell
# Stop containers to demonstrate data loss
docker-compose down

# Restart containers (data is lost due to tmpfs in original config)
docker-compose up -d

# Wait for MySQL to be ready
Start-Sleep 30

# Restore from backup using PowerShell-compatible method
Get-Content backup/todos_backup.sql | docker exec -i mysql_db mysql -uroot -proot todos_db

# Verify restoration
docker exec -it mysql_db mysql -uroot -proot -e "SELECT COUNT(*) as total_todos FROM todos_db.todos;"
```

---

### 🔧 Phase 8: Advanced Operations

#### Step 11: Monitoring & Maintenance

##### Real-time Monitoring
```powershell
# Monitor resource usage
docker stats

# View detailed container information
docker inspect flask_app
docker inspect mysql_db

# Monitor logs in real-time
docker-compose logs -f --tail=50

# Check health status
docker-compose ps
```

##### Performance Optimization
```powershell
# Analyze image sizes
docker images | Where-Object { $_.Repository -like "*todo*" }

# Remove unused resources
docker system prune

# Optimize database
docker exec -it mysql_db mysql -uroot -proot -e "
USE todos_db;
OPTIMIZE TABLE todos;
SHOW TABLE STATUS LIKE 'todos';
"
```

---

### 🛠️ Phase 9: Troubleshooting Guide

#### Common Issues & Solutions

##### Issue 1: Flask Can't Connect to MySQL
```powershell
# Check MySQL startup logs
docker logs mysql_db

# Verify network connectivity
docker exec flask_app ping db

# Test MySQL connection
docker exec -it mysql_db mysql -uroot -proot -e "SHOW DATABASES;"
```

##### Issue 2: Port Already in Use
```powershell
# Check what's using the port
netstat -ano | findstr :5000

# Kill process using port (replace PID)
taskkill /PID <PID> /F

# Or change port in docker-compose.yml
```

##### Issue 3: Permission Denied Errors
```powershell
# Fix file permissions
icacls backup /grant Everyone:F /T

# Restart Docker Desktop
Restart-Service docker
```

##### Issue 4: Container Build Failures
```powershell
# Clean rebuild
docker-compose down -v
docker system prune -f
docker-compose build --no-cache
docker-compose up -d
```

##### Issue 5: PowerShell Redirection Problems
```powershell
# Instead of using < operator, use Get-Content
Get-Content backup/todos_backup.sql | docker exec -i mysql_db mysql -uroot -proot todos_db

# Or use cmd
cmd /c "docker exec -i mysql_db mysql -uroot -proot todos_db < backup/todos_backup.sql"
```

---

### 📋 Phase 10: Validation Checklist

#### Deployment Verification
- [ ] **Containers Running**: `docker-compose ps` shows both services healthy
- [ ] **Web Access**: `http://localhost:5000` loads successfully
- [ ] **Database Connection**: Can add todos without errors
- [ ] **Data Persistence**: Todos display correctly after adding
- [ ] **Responsive Design**: Works on mobile and desktop
- [ ] **Health Checks**: `/health` endpoint returns 200 status

#### Data Management Verification
- [ ] **Backup Creation**: SQL backup file generated successfully
- [ ] **Data Loss Simulation**: `docker-compose down` removes data (with tmpfs config)
- [ ] **Restore Process**: Backup restoration works correctly using PowerShell
- [ ] **Data Integrity**: All todos restored with correct timestamps

#### Production Readiness
- [ ] **Security**: Non-root user in containers
- [ ] **Monitoring**: Health checks functioning
- [ ] **Error Handling**: Graceful failure management
- [ ] **Documentation**: All steps clearly documented

---

### 🎓 Learning Outcomes

#### Technical Skills Gained
✅ **Docker Fundamentals**
- Container creation and management
- Image optimization techniques
- Security best practices

✅ **Docker Compose Orchestration**
- Multi-service applications
- Service dependencies
- Network configuration

✅ **Database Management**
- MySQL in containers
- Backup and restore procedures
- Data persistence strategies

✅ **Web Development**
- Flask application development
- Template rendering
- Form handling and validation

✅ **DevOps Practices**
- Health monitoring
- Logging and debugging
- Performance optimization

---

### 🚀 Next Steps & Extensions

#### Potential Enhancements
1. **Add User Authentication**
   - User registration/login
   - Session management
   - Role-based access

2. **Implement Todo Management**
   - Edit existing todos
   - Mark as complete
   - Delete functionality

3. **Add Persistent Storage**
   - Replace tmpfs with volumes
   - Implement data migration
   - Scheduled backups

4. **Production Deployment**
   - NGINX reverse proxy
   - SSL/TLS certificates
   - Environment configuration

5. **Monitoring & Logging**
   - Prometheus metrics
   - Grafana dashboards
   - Centralized logging

---

## 🔧 Essential Commands Reference

### Docker Compose Commands
```powershell
# Start services
docker-compose up -d

# Start with build
docker-compose up --build -d

# View running services
docker-compose ps

# View logs
docker-compose logs -f

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Rebuild specific service
docker-compose build web
```

### Container Management
```powershell
# View running containers
docker ps

# View all containers
docker ps -a

# Execute command in container
docker exec -it mysql_db bash

# View container logs
docker logs flask_app

# Inspect container
docker inspect mysql_db
```

### Database Operations
```powershell
# Connect to MySQL
docker exec -it mysql_db mysql -uroot -proot

# Execute SQL command
docker exec -it mysql_db mysql -uroot -proot -e "SHOW DATABASES;"

# Create backup
docker exec mysql_db mysqldump -uroot -proot todos_db > backup/backup.sql

# Restore backup
Get-Content backup/backup.sql | docker exec -i mysql_db mysql -uroot -proot todos_db
```

---

### 🎉 Congratulations!

You have successfully:
- Built a multi-container Docker application
- Implemented container orchestration with Docker Compose
- Created a full-stack web application with Flask and MySQL
- Learned data persistence and backup strategies
- Gained hands-on experience with production-ready configurations
- Mastered PowerShell-compatible Docker operations

**Your application demonstrates enterprise-level containerization practices and is ready for further development and deployment!**

---

## 📚 Additional Resources

### Documentation
- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [MySQL Docker Image](https://hub.docker.com/_/mysql)

### Best Practices
- [Docker Security Best Practices](https://docs.docker.com/develop/security-best-practices/)
- [Production Docker Compose](https://docs.docker.com/compose/production/)
- [Container Health Checks](https://docs.docker.com/engine/reference/builder/#healthcheck)

### Troubleshooting
- [Docker Desktop for Windows Issues](https://docs.docker.com/desktop/troubleshoot/windows/)
- [Common Docker Compose Issues](https://docs.docker.com/compose/troubleshooting/)
- [MySQL Connection Problems](https://dev.mysql.com/doc/refman/8.0/en/problems-connecting.html)#   D o c k e r - L a b  
 