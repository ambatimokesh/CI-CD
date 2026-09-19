# Travel Eazy CI/CD bundle

Includes the original project plus starter Docker, Jenkins, SonarQube, and Kubernetes configuration.

## Before running
- Replace `REPLACE_DOCKERHUB_USERNAME` in Jenkinsfile and `DOCKERHUB_USERNAME` in `k8s/deployment.yaml`.
- Replace the placeholder allowed host with the real hostname.
- Create namespace and secret:
  `kubectl create namespace travel-easy`
  `kubectl create secret generic travel-easy-secret -n travel-easy --from-literal=django-secret-key='YOUR_STRONG_SECRET'`
- Add Jenkins credential `dockerhub-creds` as username/password (Docker Hub access token as password).
- Configure Jenkins SonarQube server named `SonarQube`, SMTP settings, and recipient email.
- Jenkins agent needs Python 3, Docker daemon access, AWS CLI, kubectl, Trivy, and SonarScanner.
- Jenkins AWS identity must be authorized for the EKS cluster and namespace.

## Local test
`docker build -t travel-easy:local .`
`docker run --rm -p 8000:8000 -e DJANGO_SECRET_KEY='local-test-secret' -e DJANGO_ALLOWED_HOSTS='localhost,127.0.0.1' travel-easy:local`

## Important
- requirements.txt is a minimal starter; reconcile it with all imports used by the app.
- Review Django production settings, secret handling, allowed hosts, HTTPS, and static files.
- SQLite is not suitable for multi-replica production. Use managed PostgreSQL and durable media storage.
- This is a deployment scaffold; validate it in your AWS environment before production use.
