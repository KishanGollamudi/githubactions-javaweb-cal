# 🚀 End-to-End CI/CD Pipeline with GitHub Actions, SonarQube, Nexus & Tomcat

## 📘 1. Introduction
This project demonstrates an **end-to-end CI/CD pipeline** for a Java-based web application using **GitHub Actions** as the automation engine.  
The workflow integrates the following key tools:

- **SonarQube** → Code quality analysis  
- **Maven** → Build & package WAR files  
- **Nexus Repository** → Artifact management  
- **Apache Tomcat** → Application deployment  

Everything is automated — from **code commit** to **deployment**.

---

## 🧭 2. CI/CD Pipeline Flow
**Developer Commit → GitHub Actions → SonarQube → Maven Build → Nexus → Tomcat**

### 🔄 Pipeline Stages
| Stage | Description | Tool |
|--------|--------------|------|
| 1️⃣ Checkout Code | Pull latest code from GitHub | GitHub |
| 2️⃣ SonarQube Analysis | Run static code quality checks | SonarQube |
| 3️⃣ Build Artifact | Create WAR file using Maven | Maven |
| 4️⃣ Upload Artifact | Push WAR to Nexus repo | Nexus |
| 5️⃣ Deploy to Tomcat | Force-clean and deploy new WAR | Apache Tomcat |
| 6️⃣ Verify Deployment | Check if app is reachable | curl |

---

## ⚙️ 3. Infrastructure Requirements
| Component | Purpose | Example |
|------------|----------|----------|
| Source Repository | Application code | [GitHub Repo]() |
| SonarQube Server | Code analysis | http://3.139.87.2:9000 |
| Nexus Repository | Artifact storage | http://18.209.9.207:8081/repository/maven-releases/ |
| Tomcat Server | Application hosting | http://98.81.123.228:8080/manager/text |

---

## 🔐 4. Credentials Setup
Store all credentials securely as **GitHub Secrets** under  
**Repository → Settings → Secrets → Actions → New Repository Secret**

| Secret Name | Description | Example |
|--------------|-------------|----------|
| SONAR_HOST_URL | SonarQube URL | http://3.139.87.2:9000 |
| SONAR_TOKEN | SonarQube Access Token | squ_0cf181e458a690f26fdfdd1d8ac4a12abac9c866 |
| NEXUS_USER | Nexus username | admin |
| NEXUS_PASS | Nexus password | admin123 |
| TOMCAT_USER | Tomcat Manager username | admin |
| TOMCAT_PASS | Tomcat Manager password | admin123 |

---

## 🧩 5. Repository Folder Structure
```

Java-Web-Calculator-App/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── webapp/
│
├── pom.xml
└── .github/
└── workflows/
└── ci-cd.yml   ← GitHub Actions Pipeline

````

---

## 🧾 6. GitHub Actions Workflow File
📁 **Path:** `.github/workflows/ci-cd.yml`

```yaml
name: Java CI/CD - SonarQube, Nexus, Tomcat (Full Clean Deploy + Slack Alerts)

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build-deploy:
    runs-on: ubuntu-latest

    env:
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      NEXUS_URL: http://18.209.9.207:8081
      NEXUS_REPO: maven-releases
      NEXUS_GROUP: com/web/cal
      NEXUS_ARTIFACT: webapp-add
      TOMCAT_URL: http://98.81.123.228:8080/manager/text
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

    steps:
      - name: 📦 Checkout Code
        uses: actions/checkout@v4

      - name: ☕ Setup Java 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: 🧰 Cache Maven Dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: 🔍 Run SonarQube Analysis
        run: |
          mvn clean verify sonar:sonar \
            -DskipTests \
            -Dsonar.projectKey=JavaWebCalculator \
            -Dsonar.host.url=${{ secrets.SONAR_HOST_URL }} \
            -Dsonar.login=${{ secrets.SONAR_TOKEN }}

      - name: ⚙️ Build WAR File
        run: |
          mvn clean package -DskipTests
          echo "✅ Build completed!"
          ls -lh target/*.war

      - name: ⬆️ Upload WAR to Nexus
        env:
          NEXUS_USER: ${{ secrets.NEXUS_USER }}
          NEXUS_PASS: ${{ secrets.NEXUS_PASS }}
        run: |
          set -e
          WAR_FILE=$(ls target/*.war | head -1)
          VERSION="0.0.${{ github.run_number }}"
          echo "📦 Uploading WAR to Nexus version $VERSION ..."
          curl -u ${NEXUS_USER}:${NEXUS_PASS} --upload-file "$WAR_FILE" \
            "${{ env.NEXUS_URL }}/repository/${{ env.NEXUS_REPO }}/${{ env.NEXUS_GROUP }}/${{ env.NEXUS_ARTIFACT }}/${VERSION}/${{ env.NEXUS_ARTIFACT }}-${VERSION}.war"
          echo "✅ Uploaded successfully."

      - name: 🚀 Deploy WAR to Tomcat (Force Clean)
        env:
          TOMCAT_USER: ${{ secrets.TOMCAT_USER }}
          TOMCAT_PASS: ${{ secrets.TOMCAT_PASS }}
          NEXUS_USER: ${{ secrets.NEXUS_USER }}
          NEXUS_PASS: ${{ secrets.NEXUS_PASS }}
        run: |
          set -e
          APP_NAME="webapp-add"
          cd /tmp; rm -f *.war

          echo "🔍 Fetching latest WAR from Nexus..."
          DOWNLOAD_URL=$(curl -s -u ${NEXUS_USER}:${NEXUS_PASS} \
            "${{ env.NEXUS_URL }}/service/rest/v1/search/assets?repository=${{ env.NEXUS_REPO }}" \
            | grep -oP '"downloadUrl"\s*:\s*"[^"]*webapp-add-[0-9.]+\.war' | tail -1 | cut -d'"' -f4)

          if [ -z "$DOWNLOAD_URL" ]; then
            echo "❌ No WAR found in Nexus!"
            exit 1
          fi

          echo "⬇️ Downloading WAR: $DOWNLOAD_URL"
          curl -u ${NEXUS_USER}:${NEXUS_PASS} -O "$DOWNLOAD_URL"
          WAR_FILE=$(basename "$DOWNLOAD_URL")

          echo "🧹 Undeploying old application..."
          curl -u ${TOMCAT_USER}:${TOMCAT_PASS} "${{ env.TOMCAT_URL }}/undeploy?path=/${APP_NAME}" || true
          sleep 5

          echo "🧼 Cleaning Tomcat cache directories via manager text..."
          curl -u ${TOMCAT_USER}:${TOMCAT_PASS} "${{ env.TOMCAT_URL }}/expire?path=/${APP_NAME}" || true
          curl -u ${TOMCAT_USER}:${TOMCAT_PASS} "${{ env.TOMCAT_URL }}/reload?path=/${APP_NAME}" || true

          echo "🚀 Deploying fresh WAR to Tomcat..."
          curl -u ${TOMCAT_USER}:${TOMCAT_PASS} --upload-file "$WAR_FILE" \
            "${{ env.TOMCAT_URL }}/deploy?path=/${APP_NAME}&update=true"

          echo "✅ Deployment completed successfully!"

      - name: 🔎 Verify Deployment
        id: verify
        run: |
          sleep 15
          STATUS=$(curl -o /dev/null -s -w "%{http_code}" http://98.81.123.228:8080/webapp-add/)
          if [ "$STATUS" -eq 200 ]; then
            echo "✅ App is live with the latest code!"
          else
            echo "⚠️ App returned status $STATUS"
            exit 1
          fi
````

---

## 🧠 7. Triggering the Pipeline

You can trigger the workflow in two ways:

* **Automatic:** On every push to the `main` branch
* **Manual:** Go to **Actions → Java CI/CD - SonarQube, Nexus, Tomcat (Full Clean Deploy)** → Click **Run workflow**

---

## ✅ 8. Verification Checklist

| Check     | Expected Result               |
| --------- | ----------------------------- |
| Build     | WAR generated under `/target` |
| SonarQube | Analysis report visible       |
| Nexus     | New version stored            |
| Tomcat    | New WAR deployed              |
| App URL   | Returns HTTP 200 OK           |

---

## ⚠️ 9. Troubleshooting

| Issue                     | Cause                      | Solution                            |
| ------------------------- | -------------------------- | ----------------------------------- |
| Old app still loads       | Tomcat cache not cleared   | Added `/undeploy` and redeploy step |
| 401 Unauthorized (Tomcat) | Invalid credentials        | Update `tomcat-users.xml`           |
| SonarQube step fails      | Server not reachable       | Check host & token                  |
| Nexus upload 400          | Duplicate artifact version | Version auto-increment added        |

---

## 📊 10. Improvements & Add-Ons

| Feature                | Description                          |
| ---------------------- | ------------------------------------ |
| SonarQube Quality Gate | Fail pipeline if quality < 80%       |
| Rollback Feature       | Deploy previous version from Nexus   |
| Multiple Environments  | Add staging/production workflows     |
| Parallel Jobs          | Split build, test, and deploy stages |

---

## 🧾 11. Summary

| Component      | Role                |
| -------------- | ------------------- |
| GitHub Actions | CI/CD engine        |
| Maven          | Build automation    |
| SonarQube      | Code quality        |
| Nexus          | Artifact repository |
| Tomcat         | Deployment server   |

---

## 🧱 12. Expected Output

Once the workflow runs successfully:

* ✅ GitHub Actions shows a green build status
* 📦 New artifact appears in **Nexus Repository**
* 🚀 **Tomcat** hosts the latest build
* 🌐 Browser shows updated version of the app at
  👉 [http://98.81.123.228:8080/webapp-add/](http://98.81.123.228:8080/webapp-add/)

---

### 👨‍💻 Author

**Kishan Gollamudi**
DevOps Engineer | Cloud & CI/CD Enthusiast
🔗 [GitHub Repository](https://github.com/mrtechreddy/Java-Web-Calculator-App)

```

---

Would you like me to include a **minimal architecture diagram (text or image placeholder)** at the top of the README — showing the flow from GitHub → SonarQube → Nexus → Tomcat?  
It would make the README visually stronger and recruiter-friendly.
```
