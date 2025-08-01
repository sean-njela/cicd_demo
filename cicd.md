To create the **best repeatable multi-stage Docker build and test solution** for your app — which includes a mix of Java, API, and browser (Behave) tests — here’s a **battle-tested structure** you can rely on.

---

## ✅ Goals

1. **Multi-stage Dockerfile** to build and package your Java web app.
2. **Split test strategy**:

   * **Inside Docker**: Unit & integration tests (Java/Gradle)
   * **Outside Docker**: Browser/Behave & API tests (Python/Selenium), because they need live services or a browser driver.
3. **Fully automatable with GitHub Actions**.

---

## 📦 1. Multi-Stage Dockerfile (`Dockerfile`)

```Dockerfile
# === Stage 1: Build app ===
FROM gradle:7.6.1-jdk11 AS builder
COPY . /app
WORKDIR /app
RUN gradle clean build -x runApiTests -x runBehaveTests

# === Stage 2: Runtime image ===
FROM openjdk:11-jre-slim
EXPOSE 8080
COPY --from=builder /app/build/libs/demo.war /usr/local/tomcat/webapps/demo.war

# Use Tomcat
RUN apt-get update && apt-get install -y curl && \
    apt-get clean

ENV CATALINA_HOME=/usr/local/tomcat
ENV PATH=$CATALINA_HOME/bin:$PATH

# Download Tomcat
RUN curl -sSLo tomcat.tar.gz https://downloads.apache.org/tomcat/tomcat-9/v9.0.82/bin/apache-tomcat-9.0.82.tar.gz \
 && mkdir -p $CATALINA_HOME \
 && tar -xzf tomcat.tar.gz -C $CATALINA_HOME --strip-components=1 \
 && rm tomcat.tar.gz

CMD ["catalina.sh", "run"]
```

---

## ⚙️ 2. GitHub Actions Workflow (`.github/workflows/ci.yml`)

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    services:
      docker: # needed if you plan to use DinD
        image: docker:dind
        privileged: true

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '11'

      - name: Build Docker Image
        run: docker build -t myapp .

      - name: Run App in Background
        run: docker run -d --rm -p 8080:8080 --name myapp myapp

      - name: Wait for App to Start
        run: |
          for i in {1..10}; do
            curl -sSf http://localhost:8080/demo && break || sleep 3
          done

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install Pipenv
        run: pip install pipenv

      - name: Set up API/UI Test Env
        working-directory: ./src/api_tests
        run: |
          pipenv install --dev --skip-lock
          pipenv run pytest

      - name: Run Behave Tests
        working-directory: ./src/ui_tests/behave
        env:
          PIPENV_MAX_DEPTH: 10
        run: |
          pipenv install --dev --skip-lock
          pipenv run behave

      - name: Stop Container
        run: docker stop myapp
```

---

## 🧪 3. Test Strategy

| Test Type   | Tool   | Run Where     |
| ----------- | ------ | ------------- |
| Unit Tests  | JUnit  | Inside Docker |
| Integration | Gradle | Inside Docker |
| API Tests   | Pytest | GitHub runner |
| Browser/BDD | Behave | GitHub runner |

> UI tests need ChromeDriver — GitHub runners support headless Chrome out of the box.

---

## 🔁 Why This Is **Repeatable**

* Docker ensures builds run the same *everywhere*.
* External (non-Docker-friendly) tests run in GitHub-hosted runners that support GUI-less testing.
* You can re-run it **locally**, in CI, or any Linux environment without setup drift.

---

## ⏭️ Want More?

I can:

* Auto-build `.war` in the Dockerfile.
* Split out test reports and publish them.
* Add SonarQube/OWASP/PIT coverage steps.

Just say the word!
