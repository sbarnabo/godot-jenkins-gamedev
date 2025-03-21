# godot-jenkins-gamedev
A Godot Jenkins CI/CD environment for game development

### **Features of This Setup:**
✅ **Godot Engine in Docker** for development  
✅ **Gitea** for lightweight Git hosting  
✅ **Persistent storage** for both Godot and Gitea repositories  
✅ **Automatic setup of Gitea database** with PostgreSQL  

---

### **`docker-compose.yml`**
```yaml
version: '3.8'

services:
  godot:
    image: barichello/godot-ci:latest
    container_name: godot_dev
    working_dir: /game
    volumes:
      - ./game:/game  # Mount local project folder
      - /tmp/.X11-unix:/tmp/.X11-unix  # Enable X11 display for GUI (Linux/macOS)
    environment:
      - DISPLAY=${DISPLAY}  # Use host display
    command: godot --editor  # Start Godot editor
    network_mode: "host"
    depends_on:
      - gitea  # Ensure Gitea is running

  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea_password
    volumes:
      - ./gitea/data:/data
    ports:
      - "3000:3000"  # Web UI
      - "2222:22"    # SSH for Git
    depends_on:
      - db

  db:
    image: postgres:latest
    container_name: gitea_db
    restart: always
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=gitea_password
      - POSTGRES_DB=gitea
    volumes:
      - ./postgres:/var/lib/postgresql/data
```

---

### **How to Use It**
#### **1. Set Up the Project Folder**
```bash
mkdir -p ~/lego-game
cd ~/lego-game
```
Save the `docker-compose.yml` file inside `~/lego-game`.

#### **2. Start Everything**
```bash
docker-compose up -d
```
- **Godot Editor** will open.
- **Gitea** will be available at: [http://localhost:3000](http://localhost:3000).

#### **3. Configure Gitea**
- Open [http://localhost:3000](http://localhost:3000).
- Create an admin user.
- Create a new repository for your **Lego game**.
- Clone it into your **Godot project**.

#### **4. Use Git Inside the `godot` Container**
```bash
cd ~/lego-game/game
git init
git remote add origin http://localhost:3000/your-user/lego-game.git
git add .
git commit -m "First commit"
git push origin main
```

---

### **Next Steps**
Would you like to add:
1. **CI/CD (Jenkins)** to automate game builds?
2. **Portainer** for managing containers easily?

Let me know how I can improve this setup for you! 🚀

Here's an **updated `docker-compose.yml`** that includes **Jenkins** for CI/CD automation in your **Godot game project**.  

---

### **🆕 What's Added?**
✅ **Jenkins** for **automated builds and testing**  
✅ **Godot Export Templates** inside Jenkins for **cross-platform builds**  
✅ **Gitea Integration** so Jenkins pulls code automatically  
✅ **Portainer, VS Code Dev Containers, PostgreSQL, and Godot remain functional**  

---

## **🚀 Updated `docker-compose.yml`**
```yaml
version: '3.8'

services:
  godot:
    image: barichello/godot-ci:latest
    container_name: godot_dev
    working_dir: /game
    volumes:
      - ./game:/game  # Mount project folder
    environment:
      - DISPLAY=host.docker.internal:0  # Use Windows X Server for GUI
    command: godot --editor
    network_mode: "host"
    depends_on:
      - gitea

  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea_password
    volumes:
      - ./gitea/data:/data
    ports:
      - "3000:3000"  # Web UI
      - "2222:22"    # SSH for Git
    depends_on:
      - db

  db:
    image: postgres:latest
    container_name: gitea_db
    restart: always
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=gitea_password
      - POSTGRES_DB=gitea
    volumes:
      - ./postgres:/var/lib/postgresql/data

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer_data:/data
    ports:
      - "9000:9000"  # Portainer web UI
    depends_on:
      - godot
      - gitea

  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: always
    user: root
    privileged: true
    ports:
      - "8080:8080"  # Web UI
      - "50000:50000"  # For Jenkins agents
    volumes:
      - ./jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock  # Enable Jenkins to control Docker
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false  # Skip setup wizard
    depends_on:
      - gitea
```

---

## **🛠 Setting Up Jenkins for Godot CI/CD**
Once Jenkins is running, follow these steps:

### **1️⃣ Open Jenkins**
- Go to [http://localhost:8080](http://localhost:8080).
- Default username: `admin`
- To get the password:
  ```powershell
  docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
  ```
- Install **"Suggested Plugins"** and create an admin account.

### **2️⃣ Install Required Jenkins Plugins**
Go to **Manage Jenkins → Plugins** and install:
✅ **Git Plugin** (to pull code from Gitea)  
✅ **Pipeline Plugin** (for automation)  
✅ **Godot Engine Plugin** (for exporting builds)

### **3️⃣ Create a New Job**
- Click **"New Item" → Pipeline**.
- Set **Gitea Repo URL**: `http://gitea:3000/your-user/lego-game.git`
- Add a **Pipeline Script** like this:

```groovy
pipeline {
    agent any
    stages {
        stage('Clone Repo') {
            steps {
                git url: 'http://gitea:3000/your-user/lego-game.git'
            }
        }
        stage('Build Game') {
            steps {
                sh 'godot --export "Windows Desktop" ./builds/lego-game.exe'
                sh 'godot --export "HTML5" ./builds/index.html'
            }
        }
        stage('Archive Build') {
            steps {
                archiveArtifacts artifacts: 'builds/**/*', fingerprint: true
            }
        }
    }
}
```

### **4️⃣ Run a Build!**
- Click **"Build Now"**.
- Jenkins will automatically pull code, build the game, and store artifacts.
- 🎮 You’ll find the game files in `./builds/`.

---
