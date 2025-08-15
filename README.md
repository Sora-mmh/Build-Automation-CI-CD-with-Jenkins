# Build-Automation-CI-CD-with-Jenkins

The listed projects in this module cover:

### **1. Infrastructure & Jenkins Setup**
- Provisioned a **DigitalOcean Droplet** and opened **ports 22 & 8080**.
- Installed **Docker** on the host and launched Jenkins with a **named volume** for persistence.
- Retrieved the **initial admin password** and completed the Jenkins on-boarding wizard.

---

### **2. Tooling & First Job**
- Installed **Maven**, **Node.js 20**, and **npm** inside the Jenkins container.
- Created a **Freestyle job** that:
  - Pulls code from Git
  - Runs unit tests
  - Builds a Java application

---

### **3. Docker in Jenkins**
- Mounted the **Docker socket** into Jenkins so builds can create images.
- Built and pushed images to **Docker Hub** and a **private Nexus registry**.
- Added **registry credentials** in Jenkins and handled **insecure-registry** settings.

---

### **4. From Freestyle to Declarative Pipeline**
- Converted the freestyle job into a **declarative Pipeline**.
- Wrote a **Jenkinsfile** stored in SCM for versioned pipeline-as-code.
- Explored **parameters, environment variables, post-build actions, external Groovy scripts, and input steps**.

---

### **5. Full End-to-End Pipeline**
- **Builds** the Java artifact (jar).
- **Packages** it into a Docker image.
- **Tags & pushes** the image to **Docker Hub**.
- Uses **credentials, environment variables, and post-build hooks** for robustness.

---

### **6. Multibranch & Shared Library**
- Enabled **Multibranch Pipeline** to automatically discover feature/PR branches.
- Added **branch-based logic** inside the Jenkinsfile.
- Created a **Shared Library** repo:
  - Hosted reusable Groovy classes & steps.
  - Configured it **globally** (and optionally **project-scoped**) in Jenkins.

---

### **7. Automated Triggers & Versioning**
- Installed **GitHub webhooks** so every push triggers the pipeline.
- Used the **Maven Build-Helper plugin** to **increment versions** automatically.
- Committed the **new version back to Git** from within Jenkins while **skipping webhook loops**.
