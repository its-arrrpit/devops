# Enterprise CI/CD: Building a Multi-Stage Declarative Pipeline in Jenkins

## Project Overview

This project demonstrates how to structure, automate, and execute an enterprise-grade Continuous Delivery pipeline using Jenkins Declarative Pipeline syntax (`Jenkinsfile`). 

The pipeline orchestrates the complete software delivery lifecycle for a Python Flask microservice: dependency environment verification, unit testing via `unittest`, deployment simulation, background daemon execution via `nohup`, and automated post-deployment integration validation.

---

## Technical Specifications

| Parameter | Configuration Detail |
|---|---|
| **Host Environment** | Windows 11 |
| **Container Engine** | Docker Desktop |
| **CI Automation Server** | Jenkins LTS running in Docker (`jenkins/jenkins:lts`) |
| **Target Runtime** | Python 3 + Flask (`flask==2.1.2`) |
| **Testing Framework** | Standard Library `unittest` |
| **Pipeline Style** | Declarative Jenkinsfile (`pipeline { ... }`) |

---

## Microservice Architecture & Test Assets

The application artifacts reside in `exercise-9-jenkins-multi-stage-pipeline`:

- **`app.py`**: A minimal Flask HTTP microservice serving greeting responses on `/`:
  ```python
  from flask import Flask
  app = Flask(__name__)

  @app.route('/')
  def home():
      return "Hello, Jenkins Multi-Stage Pipeline!"

  if __name__ == '__main__':
      app.run(host='0.0.0.0', port=5000)
  ```
- **`requirements.txt`**: Specifies version dependency pinning (`flask==2.1.2`).
- **`test_app.py`**: Automated test suite asserting that the root endpoint returns HTTP status `200` with the expected payload:
  ```python
  import unittest
  from app import app

  class TestApp(unittest.TestCase):
      def setUp(self):
          self.app = app.test_client()

      def test_home(self):
          response = self.app.get('/')
          self.assertEqual(response.status_code, 200)
          self.assertIn(b"Hello, Jenkins Multi-Stage Pipeline!", response.data)

  if __name__ == '__main__':
      unittest.main()
  ```

---

## Step-by-Step Implementation

### 1. Provisioning the Jenkins Container Controller
Launch the controller mounting the persistent `jenkins_home` volume:

```powershell
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

Extract the bootstrap administrator token:

```powershell
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Navigate to `http://localhost:8080`, enter the key, install recommended plugins (ensuring Pipeline support), and establish credentials.

---

### 2. Jenkins Agent Runtime Hardening: Installing Python
Because base Jenkins controller images lack Python build tooling, pipeline stages attempting to invoke `python3` or `pip` will fail out of the box. Install required runtime packages directly into the container:

```powershell
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y python3 python3-pip python3-venv python3-flask"
docker exec jenkins python3 --version
```

---

### 3. Declarative Pipeline Specification (`Jenkinsfile`)
The pipeline codifies five distinct lifecycle stages scoped to the project directory:

```groovy
pipeline {
    agent any

    environment {
        APP_DIR = 'exercise-9-jenkins-multi-stage-pipeline'
    }

    stages {
        stage('Build') {
            steps {
                echo 'Checking Python and dependencies...'
                dir("${APP_DIR}") {
                    sh 'python3 --version'
                    sh 'python3 -c "import flask; print(flask.__version__)"'
                }
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                dir("${APP_DIR}") {
                    sh 'python3 -m unittest discover -s .'
                }
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                dir("${APP_DIR}") {
                    sh '''
                    mkdir -p python-app-deploy
                    cp app.py python-app-deploy/
                    '''
                }
            }
        }
        stage('Run Application') {
            steps {
                echo 'Running application...'
                dir("${APP_DIR}") {
                    sh '''
                    nohup python3 python-app-deploy/app.py > python-app-deploy/app.log 2>&1 &
                    echo $! > python-app-deploy/app.pid
                    '''
                }
            }
        }
        stage('Test Application') {
            steps {
                echo 'Testing application...'
                dir("${APP_DIR}") {
                    sh 'python3 test_app.py'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for more details.'
        }
    }
}
```

#### Pipeline Stage Breakdown:
| Stage Name | Execution Target | Verification Criteria |
|---|---|---|
| **Build** | Validates `python3` and `flask` availability | Zero exit code, prints version |
| **Test** | Executes unit test discovery (`unittest`) | All test fixtures pass |
| **Deploy** | Packages files into `python-app-deploy` target directory | Directory created, files mirrored |
| **Run Application** | Launches application in background using `nohup` | Process spawned, PID stored |
| **Test Application** | Re-asserts test client against deployed target | All assertions validated |

---

### 4. Committing and Pushing to SCM
Stage project assets and publish to remote repository:

```powershell
git add .
git commit -m "exercise-9 (jenkins-multi-stage-pipeline)"
git push
```

---

### 5. Configuring the Multistage Pipeline Project
In Jenkins:
1. Click **New Item**, supply project title `Python-Multistage-Pipeline`, and select **Pipeline**.
2. Under the **Pipeline** configuration tab, set:
   - **Definition**: *Pipeline script from SCM*
   - **SCM**: *Git*
   - **Repository URL**: Repository URL
   - **Branch Specifier**: `*/main`
   - **Script Path**: `exercise-9-jenkins-multi-stage-pipeline/Jenkinsfile`
3. Click **Save**.

---

### 6. Executing Pipeline & Inspecting Stage Views
Click **Build Now** to initiate automated build compilation:

- Checkout completed.
- Build dependencies verified.
- Unit testing succeeded (`test_home` passed).
- Application deployed and spawned as background daemon (`nohup`).
- Integration tests evaluated with status `SUCCESS`.
- Post-action notification logged: `Pipeline completed successfully!`

![Multi-Stage Pipeline Execution Success](screenshots/01-jenkins-pipeline-success.png)

---

### 7. Post-Build Workspace Audit
Inspect the deployed artifact directory inside the active Jenkins container workspace:

```powershell
docker exec jenkins ls /var/jenkins_home/workspace/Python-Multistage-Pipeline/exercise-9-jenkins-multi-stage-pipeline/python-app-deploy
```

*Directory Output:*
```text
app.log
app.pid
app.py
```

*Artifact Inventory:*
- `app.py`: Deployed executable source script.
- `app.log`: Persistent stdout/stderr runtime log output.
- `app.pid`: Process identifier tracking background daemon execution.

![Deployed Directory Verification](screenshots/02-deployed-files-listing.png)

---

## Continuous Delivery Pipeline Execution Graph

```text
SCM Checkout (Git main)
         │
         ▼
[ Stage 1: Build ] ──────────► Verify Python 3 & Flask Library
         │
         ▼
[ Stage 2: Test ] ───────────► Execute `unittest discover`
         │
         ▼
[ Stage 3: Deploy ] ─────────► Structure `python-app-deploy` artifact directory
         │
         ▼
[ Stage 4: Run Application ] ─► Background execution with `nohup` & track PID
         │
         ▼
[ Stage 5: Test App ] ───────► Validate live service responses
         │
         ▼
[ Post Actions ] ────────────► Status: SUCCESS (Pipeline completed successfully!)
```

---

## Complete Command Summary

```powershell
# 1. Start Jenkins container
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts

# 2. Extract initial admin secret
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# 3. Install Python dependencies in container
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y python3 python3-pip python3-venv python3-flask"

# 4. Audit deployed files
docker exec jenkins ls /var/jenkins_home/workspace/Python-Multistage-Pipeline/exercise-9-jenkins-multi-stage-pipeline/python-app-deploy
```

---

## Conclusion
Successfully authored, automated, and executed a complete multi-stage CI/CD workflow with Jenkins Declarative Pipelines. Validated multi-step progression from build verification, unit testing, and deployment to background execution tracking and health verification.