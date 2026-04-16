# LAB 09: CI/CD with GitHub Actions

## Understanding CI/CD

### 1.1 The Problem We Are Solving

Imagine you have built a web application and want to deploy it. Without automation, your workflow looks like this:

| Step | Manual Process (Without CI/CD) |
|------|--------------------------------|
| 1. Write code | Write code on your machine |
| 2. Test | Manually run tests and hope nothing breaks |
| 3. Build Docker image | Run `docker build` on your laptop |
| 4. Push to Docker Hub | Run `docker push` manually |
| 5. Deploy | SSH into server, pull image, restart container |
| 6. Discover bug | Repeat all 5 steps again |

This manual process is:
- **Slow** - every deployment takes hours of human effort
- **Error prone** - you might forget a step or run the wrong command
- **Inconsistent** - it works on your machine but may fail on the server
- **Unscalable** - impossible to maintain when working in a team

CI/CD solves this by automating the entire process. Every time you push code to GitHub, the pipeline runs automatically.

### 1.2 What is CI/CD?

| Term | Full Name | What It Means |
|------|-----------|---------------|
| CI | Continuous Integration | Automatically test code every time a developer pushes changes |
| CD | Continuous Delivery | Automatically build and deliver the tested application |
| Pipeline | CI/CD Pipeline | The sequence of automated steps from code push to deployment |

### 1.3 The CI/CD Workflow

The typical CI/CD workflow follows these stages:
1. Developer writes code and pushes to GitHub
2. GitHub detects the push and triggers the pipeline
3. **CI stage**: Automated tests run on the code
4. If tests pass: **CD stage** begins
5. Docker image is built and pushed to Docker Hub
6. Application is deployed automatically
7. Developer gets notified of success or failure

### 1.4 Where GitHub Actions Fits In

GitHub Actions is GitHub's built-in CI/CD tool. It lets you define automated workflows using YAML files stored in your repository. When specific events happen (like a push), GitHub runs your workflow on a fresh Linux server called a **runner**.

| GitHub Actions Concept | What You Already Know |
|------------------------|-----------------------|
| Workflow (.yml file) | A shell script but triggered automatically |
| Runner | A Linux server - GitHub manages it for you |
| Step | A single shell command like `echo`, `docker build` |
| Job | A group of steps that run together on one runner |
| Event (push, PR) | The trigger like running `./script.sh` manually |
| Secret | Like an environment variable, but encrypted and hidden |

## GitHub Actions Core Concepts

### 2.1 Workflow File Structure

Every GitHub Actions workflow is a YAML file stored inside `.github/workflows/` in your repository. Here is the anatomy of a workflow file:

```yaml
# .github/workflows/my-pipeline.yml
name: My First Pipeline  # Display name on GitHub

on:
  push:
    branches: [ main ]  # TRIGGER — when to run
    # Run when code is pushed
    # Only on the main branch

jobs:
  build:
    runs-on: ubuntu-latest  # What to do
    # Job name (you choose this)
    # Use a fresh Ubuntu Linux server

    steps:
      # Individual commands
      - name: Checkout code
        uses: actions/checkout@v3  # Download your repo onto the runner

      - name: Say Hello
        run: echo 'Hello World!'  # A normal shell command (Lab 01!)
```

### 2.2 Understanding YAML Syntax

**Important**: YAML is indentation-sensitive. Incorrect spacing will break your workflow. Always use spaces (not tabs). 2 spaces per level is standard.

Key YAML rules you need to know:
- `key: value` - basic mapping (like a variable)
- `- ` Lines starting with `-` are list items
- Indentation defines nesting (`parent: child`)
- `#` is a comment (same as `#` in shell scripting)
- Strings with special characters should be quoted

```yaml
# YAML structure example
name: My Workflow      # String value
on: push              # Simple trigger
on:
  # Complex trigger with options
  push:
    branches: [ main, dev ]  # List of branches

jobs:
  my-job:
    runs-on: ubuntu-latest
    steps:
      - name: Step 1
        run: echo 'step 1'
      - name: Step 2
        run: echo 'step 2'  # note same indent as Step 1
```

### 2.3 Common Triggers (Events)

| Trigger | When It Fires |
|---------|---------------|
| `on: push` | Every time anyone pushes code to the repository |
| `on: pull_request` | When a pull request is opened or updated |
| `on: push`<br>`branches: [main]` | Only when pushing to the main branch |

### 2.4 GitHub Secrets

Secrets are encrypted variables stored on GitHub, not in your code. They are used to store sensitive information like passwords and tokens that your workflow needs.

You will use secrets in Section 5 to securely log into Docker Hub. To add a secret:
1. Go to your GitHub repository
2. Click **Settings** > **Secrets and variables** > **Actions**
3. Click **New repository secret**
4. Add your secret name and value

## Your First Workflow

### Step 1: Create a GitHub Repository
1. Go to [github.com](https://github.com) and sign in
2. Click the `+` icon and select **New repository**
3. Name it `devops-lab09` and set it to **Public**
4. Check **Add a README file**
5. Click **Create repository**

### Step 2: Clone the Repository
```bash
git clone https://github.com/YOUR_USERNAME/devops-lab09.git
cd devops-lab09
```

### Step 3: Create the Workflow Directory
GitHub Actions looks for workflows inside `.github/workflows/`:
```bash
mkdir -p .github/workflows
```

### Step 4: Write Your First Workflow
Create the workflow file:
```bash
nano .github/workflows/hello.yml
```

Type the following content exactly:

```yaml
name: Hello World Pipeline
on:
  push:
    branches: [ main ]
jobs:
  hello-job:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
      - name: Print Hello World
        run: echo 'Hello from GitHub Actions!'
      - name: Show Current User
        run: whoami
      - name: Show Current Directory
        run: pwd
      - name: List Files
        run: ls -la
      - name: Show System Info
        run: |
          echo '--- System Information ---'
          uname -a
          df -h
          date
```

### Step 5: Push and Watch the Pipeline Run
```bash
git add .
git commit -m "Add Hello World workflow"
git push origin main
```

Now go to GitHub and watch the pipeline:
1. Go to your repository on github.com
2. Click the **Actions** tab at the top
3. You will see your workflow running (🟡 = running, ✅ = passed, ❌ = failed)
4. Click on the workflow run to see details
5. Click on `hello-job` to expand and see each step's output

## Building a CI Pipeline

Now we build a real Continuous Integration pipeline. The goal: every time code is pushed, the pipeline automatically runs tests and reports pass or fail.

### Step 1: Create a Simple Python Application

Create a small calculator application to test:
```bash
nano calculator.py
```

```python
# calculator.py
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

def divide(a, b):
    if b == 0:
        raise ValueError('Cannot divide by zero')
    return a / b
```

### Step 2: Write Tests
```bash
nano test_calculator.py
```

```python
# test_calculator.py
import calculator

def test_add():
    assert calculator.add(2, 3) == 5
    assert calculator.add(-1, 1) == 0
    assert calculator.add(0, 0) == 0
    print('test_add PASSED')

def test_subtract():
    assert calculator.subtract(10, 3) == 7
    assert calculator.subtract(0, 5) == -5
    print('test_subtract PASSED')

def test_multiply():
    assert calculator.multiply(3, 4) == 12
    assert calculator.multiply(-2, 5) == -10
    print('test_multiply PASSED')

def test_divide():
    assert calculator.divide(10, 2) == 5.0
    assert calculator.divide(7, 2) == 3.5
    print('test_divide PASSED')

if __name__ == '__main__':
    test_add()
    test_subtract()
    test_multiply()
    test_divide()
    print('')
    print('All tests PASSED!')
```

### Step 3: Run Tests Locally First
Always verify your tests work locally before adding them to the pipeline:
```bash
python3 test_calculator.py
```

Expected output:
```
test_add PASSED
test_subtract PASSED
test_multiply PASSED
test_divide PASSED

All tests PASSED!
```

### Step 4: Create the CI Workflow
```bash
nano .github/workflows/ci.yml
```

```yaml
name: CI Pipeline
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      - name: Show Python version
        run: python3 --version
      - name: Run Tests
        run: python3 test_calculator.py
      - name: Show system info on success
        run: |
          echo 'Tests completed successfully!'
          echo 'Runner info:'
          uname -a
```

### Step 5: Push and Observe
```bash
git add .
git commit -m "Add calculator app and CI pipeline"
git push origin main
```

Go to the **Actions** tab on GitHub and watch the CI pipeline run. Confirm all test steps pass.

### Step 6: Intentionally Break the Pipeline
Open `calculator.py` and introduce a bug:
```python
def add(a, b):
    return a - b  # BUG: should be a + b
```

```bash
git add .
git commit -m "Introduce bug in add function"
git push origin main
```

Go to **Actions** on GitHub. You will see a red ❌. Click on it to read the `AssertionError`. This is CI doing its job - catching bugs automatically before they affect others.

Now fix the bug and push again:
```python
def add(a, b):
    return a + b  # Fixed
```

```bash
git add .
git commit -m "Fix bug in add function"
git push origin main
```

## CD Pipeline: Automate Docker Build & Push

Now we connect everything together. We will create a Continuous Delivery pipeline that automatically builds a Docker image and pushes it to Docker Hub every time code is pushed to `main` and tests pass.

### Step 1: Create the Application Files

Create a simple Flask app:
```bash
nano app.py
```

```python
# app.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Hello from CI/CD Pipeline!',
        'status': 'running'
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```bash
nano requirements.txt
```
```
flask
```

### Step 2: Create the Dockerfile
```bash
nano Dockerfile
```

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

### Step 3: Test Docker Build Locally
```bash
docker build -t devops-lab09-app:test .
docker run -d -p 5000:5000 --name lab09-test devops-lab09-app:test
curl http://localhost:5000
docker stop lab09-test && docker rm lab09-test
```

### Step 4: Add Docker Hub Secrets to GitHub
1. Log into [Docker Hub](https://hub.docker.com)
2. Go to **Account Settings** > **Security** > **New Access Token**
3. Name it `github-actions` and copy the token
4. Go to your GitHub repo > **Settings** > **Secrets and variables** > **Actions**
5. Add `DOCKERHUB_USERNAME` with your Docker Hub username
6. Add `DOCKERHUB_TOKEN` with the access token

### Step 5: Create the Full CI/CD Workflow
```bash
nano .github/workflows/cicd.yml
```

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
jobs:
  # ── JOB 1: Run Tests (CI) ──────────────────────────────────────
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      - name: Run tests
        run: python3 test_calculator.py

  # ── JOB 2: Build and Push Docker Image (CD) ────────────────────
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: test  # Only runs if the test job PASSES
    # Only push on main branch push (not on pull requests)
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build Docker Image
        run: |
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/devops-lab09:latest .
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/devops-lab09:${{ github.sha }} .
      - name: Push Docker Image
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/devops-lab09:latest
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/devops-lab09:${{ github.sha }}
      - name: Confirm Push
        run: echo 'Image pushed to Docker Hub successfully!'
```

### Step 6: Push and Watch the Full Pipeline
```bash
git add .
git commit -m "Add Flask app, Dockerfile, and full CI/CD pipeline"
git push origin main
```

Go to the **Actions** tab on GitHub. You should see:
- The `test` job runs first
- Once tests pass, the `build-and-push` job starts
- After completion, go to [hub.docker.com](https://hub.docker.com) — your image is there!

Verify by pulling the image on your local machine:
```bash
docker pull YOUR_DOCKERHUB_USERNAME/devops-lab09:latest
docker run -d -p 5000:5000 YOUR_DOCKERHUB_USERNAME/devops-lab09:latest
curl http://localhost:5000
```

## Exercises

### Exercise 1: System Info
Add a new step to your CI/CD workflow that prints detailed system information about the GitHub Actions runner.

## Key Commands Reference

| Command / Concept | What It Does |
|-------------------|--------------|
| `git push origin main` | Triggers the CI/CD pipeline automatically |
| `actions/checkout@v3` | Downloads your repo onto the runner |
| `actions/setup-python@v4` | Installs Python on the runner |
| `docker/login-action@v2` | Logs into Docker Hub securely using secrets |
| `needs: test` | Makes a job wait for another job to pass first |
| `${{ secrets.NAME }}` | Reads an encrypted GitHub secret |
| `${{ github.sha }}` | The unique commit hash used as image tag |
| `if: github.event_name == 'push'` | Conditional step - only run on push, not PR |

