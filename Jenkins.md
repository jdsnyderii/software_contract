Below is a step-by-step guide to adding a build to a Jenkins open-source system and integrating a GitHub webhook to
trigger a build on a "merge to main" via a pull request (PR). This assumes you have administrative access to a Jenkins
instance and a GitHub repository.

---

### **Part 1: Adding a Build to Jenkins**

#### **Step 1: Install Required Plugins**

1. **Log in to Jenkins** with an admin account.
2. Navigate to **Manage Jenkins** > **Manage Plugins**.
3. Go to the **Available** tab and search for the following plugins:
    - **Git Plugin**: For Git integration.
    - **GitHub Plugin**: For GitHub-specific features.
    - **Pipeline Plugin** (optional, if using Jenkins Pipeline for builds).
4. Install these plugins and restart Jenkins if prompted.

#### **Step 2: Configure Git in Jenkins**

1. Go to **Manage Jenkins** > **Global Tool Configuration**.
2. Scroll to the **Git** section.
3. Ensure Git is installed or configure the path to the Git executable if needed.

#### **Step 3: Create a New Jenkins Job**

1. From the Jenkins dashboard, click **New Item**.
2. Enter a name for your job (e.g., `MyProjectBuild`).
3. Select **Freestyle project** or **Pipeline** (depending on your preference) and click **OK**.
4. Configure the job:
    - **General**:
        - Add a description (optional).
        - Check **GitHub project** and provide the GitHub repository URL (e.g., `https://github.com/username/repo`).
    - **Source Code Management**:
        - Select **Git**.
        - Enter the repository URL (e.g., `https://github.com/username/repo.git`).
        - Add credentials if required (click **Add** > **Jenkins** to create credentials for GitHub).
        - Specify the branch to build (e.g., `main` or `*/main`).
    - **Build Triggers**:
        - Check **GitHub hook trigger for GITScm polling** (this enables webhook triggering).
    - **Build Steps**:
        - For a Freestyle project, add a build step (e.g., **Execute shell** for Linux or **Windows Batch Command** for
          Windows).
        - Example for a simple Node.js project:
          ```bash
          npm install
          npm test
          ```
        - For a Pipeline project, define a `Jenkinsfile` in your repository (see Step 4).
    - **Post-build Actions** (optional):
        - Add actions like archiving artifacts or sending notifications.
5. Click **Save**.

#### **Step 4: (Optional) Create a Jenkinsfile for Pipeline**

If using a Pipeline project:

1. In your GitHub repository, create a file named `Jenkinsfile` in the root directory.
2. Define your pipeline script. Example for a Node.js project:
   ```groovy
   pipeline {
       agent any
       stages {
           stage('Checkout') {
               steps {
                   git url: 'https://github.com/username/repo.git', branch: 'main'
               }
           }
           stage('Install') {
               steps {
                   sh 'npm install'
               }
           }
           stage('Test') {
               steps {
                   sh 'npm test'
               }
           }
       }
   }
   ```
3. Commit and push the `Jenkinsfile` to your repository.
4. In the Jenkins Pipeline job configuration:
    - Under **Pipeline**, select **Pipeline script from SCM**.
    - Set **SCM** to **Git**, provide the repository URL, and ensure the branch is `main`.
    - Set the **Script Path** to `Jenkinsfile`.

#### **Step 5: Test the Build**

1. Go to the Jenkins job page and click **Build Now**.
2. Verify that the build runs successfully and check the console output for errors.

---

### **Part 2: Integrating a GitHub Webhook to Trigger Builds on Merge to Main**

#### **Step 1: Configure GitHub Repository**

1. Go to your GitHub repository.
2. Navigate to **Settings** > **Webhooks** > **Add webhook**.
3. Configure the webhook:
    - **Payload URL**: Enter the Jenkins GitHub webhook URL:
      ```
      http://<jenkins-url>/github-webhook/
      ```
      Replace `<jenkins-url>` with your Jenkins instance URL (e.g., `http://jenkins.example.com:8080`).
      Ensure the trailing `/` is included.
    - **Content type**: Select `application/json`.
    - **Secret**: (Optional) Set a secret token for security. If used, note it for Jenkins configuration.
    - **Events**: Select **Let me select individual events** and check:
        - **Pull requests** (to trigger on PR actions).
        - **Pushes** (to trigger on merges to main).
    - Click **Add webhook**.

#### **Step 2: Configure Jenkins for Webhook**

```plaintext
#### **Step 2: Configure Jenkins for Webhook**
1. If you used a secret token in the GitHub webhook:
   - Go to **Manage Jenkins** > **Configure System**.
   - Scroll to the **GitHub Server** section.
   - Add a GitHub server configuration and include the secret token.
2. Ensure the job configuration (from Part 1) has **GitHub hook trigger for GITScm polling** enabled.
3. If using a Pipeline, ensure the `Jenkinsfile` is correctly set up to handle the branch (e.g., `main`).

#### **Step 3: Test the Webhook**
1. Create a pull request in your GitHub repository to merge a branch into `main`.
2. Merge the pull request.
3. Check Jenkins to confirm that a build is triggered automatically.
4. Go to the GitHub webhook settings (**Settings** > **Webhooks** > click the webhook > **Recent Deliveries**) to verify successful delivery.

#### **Step 4: Troubleshoot (if needed)**
- If the webhook doesn’t trigger:
  - Ensure the Jenkins URL is publicly accessible (or use a tunneling service like `ngrok` for local testing).
  - Check the GitHub webhook delivery logs for errors.
  - Verify the branch name in Jenkins matches `main`.
- If the build fails, check the console output in Jenkins for errors in the build steps.

---

### **Additional Notes**
- **Security**: Use HTTPS for Jenkins and GitHub to secure webhook communication. Consider using a secret token for webhooks.
- **Advanced**: For complex workflows, use GitHub Actions alongside Jenkins or explore multibranch pipelines in Jenkins.
- **Jenkinsfile**: For production, add error handling, notifications, and artifact storage in the `Jenkinsfile`.
- **Network**: If Jenkins is behind a firewall, ensure GitHub can reach it (e.g., via a reverse proxy or public URL).

This setup ensures that every merge to the `main` branch via a pull request triggers an automated build in Jenkins. Let me know if you need further clarification or help with specific configurations!