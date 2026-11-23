# CSC418 Lab 11 — Step-by-step execution guide

This project follows the uploaded Lab 11 handout. The initial `Jenkinsfile` has the five stages used in Task 4. The `Jenkinsfile.with-deploy` has the six-stage pipeline for Task 5 and later.

> IMPORTANT: Replace `YOUR NAME` inside `src/index.html` **before** you build the site. The student-name Selenium test deliberately fails if you leave that placeholder.

## Before you start

- Existing EC2 Jenkins instance from Labs 9 and 10, with Git, Docker, Jenkins, Docker Pipeline/JUnit plugins and a Jenkins admin login.
- `.pem` file on your laptop; public EC2 IP from AWS console.
- EC2 security group should allow inbound TCP 22 (from your IP), 8080 (Jenkins access and GitHub webhook), and 80 (live website). The staging port 8090 only needs local access.
- A GitHub account. For HTTPS Git push, authenticate with your GitHub PAT instead of your password. Never put the PAT in the repository or screenshots.

## Upload project — on Windows PowerShell

After downloading `Lab11_Starter_Files.zip` to your Downloads folder:

```powershell
cd $HOME\Downloads
Expand-Archive .\Lab11_Starter_Files.zip -DestinationPath .\Lab11_Starter_Files -Force
scp -i "C:\path\to\csc418-lab2-key.pem" -r .\Lab11_Starter_Files\jenkins-lab11 ec2-user@YOUR_EC2_IP:~/
ssh -i "C:\path\to\csc418-lab2-key.pem" ec2-user@YOUR_EC2_IP
```

All following commands are run **on the EC2 terminal**, except when a step explicitly says Windows, Jenkins or GitHub.

## Task 1 — Prepare server and Selenium test agent

```bash
sudo systemctl status jenkins
sudo systemctl status docker
sudo systemctl disable --now nginx
sudo ss -tlnp | grep ':80 ' || echo 'port 80 is free'
sudo -u jenkins docker ps
free -h
df -h /

cd ~/jenkins-lab11
sed -i 's/YOUR NAME/Your Full Name/' src/index.html

docker build -t selenium-test-agent tests/
docker images selenium-test-agent
docker run --rm selenium-test-agent chromium --version
docker run --rm selenium-test-agent chromedriver --version
docker run --rm selenium-test-agent python -c 'import selenium; print(selenium.__version__)'
```

Replace `Your Full Name` in the `sed` command with your real name. Check Chromium and ChromeDriver have the same **major** version. Check Jenkins > Manage Jenkins > Plugins includes Docker Pipeline and JUnit. Save screenshots of `docker images` and all three version checks.

## Task 3 — Manual test (do before GitHub if convenient)

```bash
cd ~/jenkins-lab11
docker run --rm -u $(id -u):$(id -g) -e HOME=/tmp -v "$PWD":/app -w /app node:22-alpine sh -c 'npm install && npm run build'
ls dist
docker build -t lab11-site:manual .
docker run -d --name lab11-staging -p 8090:80 lab11-site:manual
curl -s http://localhost:8090 | head -n 8

docker run --rm --network host -u $(id -u):$(id -g) -e HOME=/tmp -e BASE_URL=http://localhost:8090 -v "$PWD":/work -w /work selenium-test-agent pytest tests/test_site.py -v
```

Expected: `4 passed`. Screenshot the `pytest -v` output. To copy the browser screenshot to your Windows laptop, open a **different PowerShell window** on Windows:

```powershell
scp -i "C:\path\to\csc418-lab2-key.pem" ec2-user@YOUR_EC2_IP:~/jenkins-lab11/screenshot.png .
```

Open `screenshot.png` on Windows and capture it for evidence. Back on EC2:

```bash
docker rm -f lab11-staging
```

## Task 2 — Push GitHub repository

Create a **public empty** repo named `jenkins-lab11` on GitHub, without an automatically generated README. On EC2:

```bash
cd ~/jenkins-lab11
git init
git config user.name 'Your Full Name'
git config user.email 'your-github-email@example.com'
git add .
git commit -m 'Add website, Dockerfiles, Selenium tests and initial Jenkins pipeline'
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/jenkins-lab11.git
git push -u origin main
```

Use your GitHub username + Personal Access Token when prompted for HTTPS credentials. Screenshot the GitHub repository file list and save its URL.

## Task 4 — Jenkins five-stage pipeline

The included `Jenkinsfile` has the 5 initial stages: Checkout, Build, Package, Start Staging, Test.

In Jenkins browser: **New Item** > `lab11-pipeline` > **Pipeline** > OK. In configuration: **Pipeline script from SCM** > **Git**, repo URL `https://github.com/YOUR_USERNAME/jenkins-lab11.git`, branch `*/main`, script path `Jenkinsfile`. Save > **Build Now**.

Check stage view all green, Test Result shows four passes, and console contains the pytest command and success. Open the archived `screenshot.png`. Screenshots required.

## Task 5 — Six-stage production pipeline

On EC2:

```bash
cd ~/jenkins-lab11
cp Jenkinsfile.with-deploy Jenkinsfile
git add Jenkinsfile
git commit -m 'Add Deploy stage'
git push
```

Jenkins > `lab11-pipeline` > **Build Now**. Check all **6 stages** green. On EC2:

```bash
docker ps
curl -I http://localhost:80
```

In your Windows browser open `http://YOUR_EC2_IP` (without `:8080`). Screenshot the live page and Docker `lab11-prod` running with port 80.

## Task 6 — Analyze timings, results, artifacts

Click **Build Now** two more times (at least three successful runs in total). Capture stage view with times, Test Result / Test Result Trend, and archived screenshot. Check console for image build and test commands.

On EC2:

```bash
docker images lab11-site
docker ps
sudo ls /var/lib/jenkins/workspace/lab11-pipeline
```

Example analysis (replace with what you actually observe): "On the first run, the Docker image build or test-agent setup was slowest because dependencies had to be downloaded. Later runs were faster because Docker reused cached image layers. All four Selenium tests passed and Jenkins saved a screenshot artifact."

## Task 7 — Intentionally fail a Selenium check, then fix it

On EC2:

```bash
cd ~/jenkins-lab11
sed -i 's|Deployed by a CI/CD Pipeline|Deployed by Hand|' src/index.html
git add src/index.html
git commit -m 'Temporarily change heading to demonstrate quality gate'
git push
```

Jenkins > **Build Now**. Screenshot Test = failed/red, Deploy = skipped, the `test_main_heading` assertion error, and the **unchanged** live website at port 80. Then restore:

```bash
sed -i 's|Deployed by Hand|Deployed by a CI/CD Pipeline|' src/index.html
git add src/index.html
git commit -m 'Restore passing heading'
git push
```

Jenkins > **Build Now**. Screenshot six green stages again.

The additional Part 7.2 exercise (not one of the nine specific graded tasks) can be demonstrated by changing `fs.mkdirSync` to `fs.mkdirSyncX` in `build.js`, pushing, showing the Build failure, then restoring it. Avoid leaving that typo in the final submission.

## Task 8 — Automatic GitHub webhook

GitHub repository > **Settings** > **Webhooks** > **Add webhook**:
- Payload URL: `http://YOUR_EC2_IP:8080/github-webhook/`
- Content type: `application/json`
- Event: **Just the push event**
- Active: checked

Jenkins > `lab11-pipeline` > **Configure** > **Build Triggers** > check **GitHub hook trigger for GITScm polling** > Save. If this trigger is absent, check for the GitHub plugin.

On EC2 make another legitimate text change (outside the tested heading), e.g. add `Version 2` to the student line. Commit and push. **Do not click Build Now**. Watch for a new Jenkins build. Screenshot the green delivery on GitHub, Jenkins trigger checkbox, automatically started build, and updated production page.

## Task 9 — Reflection (five sentences)

A CI/CD pipeline runs tests before deployment to catch problems before users see them. Selenium checks whether the actual website loads and behaves correctly in a browser. If any test fails, Jenkins stops the pipeline and leaves the existing live version unchanged. Running Selenium inside Docker keeps Chromium, ChromeDriver and the Python testing tools together in a consistent environment. This avoids installing browser dependencies directly on the Jenkins server and makes tests easier to repeat.

## Final submission

Put your real screenshots and the reflection into one PDF named `LastName_RollNumber_Lab11.pdf`. Put your actual GitHub repository URL on the first page. The lab requires evidence for all nine tasks. Never include PATs, passwords or private keys in screenshots.

## Important troubleshooting

- `docker permission denied` when used by Jenkins: verify `sudo -u jenkins docker ps` works, and revisit your Lab 10 Docker permissions.
- `port is already allocated`: inspect `sudo ss -tlnp` and `docker ps`; stop Nginx on 80 or remove only the old lab11 container.
- `pytest` cannot connect: verify staging container exists and test agent has host networking.
- Browser crashes: check `free -h`, swap, and Chromium/ChromeDriver version match.
- No results in Jenkins: confirm JUnit plugin and `--junitxml=test-results.xml` output path.
- GitHub webhook not firing: verify Jenkins is reachable from GitHub, its 8080 security group rule, webhook delivery response, the GitHub plugin and job trigger.

This bundle has been checked for source syntax and a Node build, but the Docker/Jenkins operations must run on your EC2 system; this local environment has no Docker daemon available for full verification.
