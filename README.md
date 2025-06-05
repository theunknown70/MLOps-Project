# Project Title: End-to-End MLOps Project: Deploying a Scalable ML Application

## 🚀 Overview

This project demonstrates a comprehensive MLOps workflow, starting from foundational project setup and experiment tracking, and progressing through data/model versioning, automated CI/CD pipelines, containerization with Docker, deployment onto a Kubernetes cluster (AWS EKS), and finally, setting up monitoring with Prometheus and Grafana. It's designed to showcase best practices for building, deploying, and maintaining machine learning models in a production-like environment, making it a robust example for an MLOps portfolio.

**Goal:** To construct a resilient and scalable machine learning application by emphasizing automation, rigorous version control, and continuous observability.

## ✨ Key MLOps Features Demonstrated

* **Version Control:** Git for source code management; **DVC (Data Version Control)** for managing large data files, ML models, and intermediate artifacts.
* **Experiment Tracking:** **MLflow** integrated with **DagsHub** for logging, comparing, and visualizing ML experiments (parameters, metrics, artifacts).
* **Reproducibility:** DVC pipelines (`dvc.yaml`, `params.yaml`) to ensure experiments and results are fully reproducible.
* **Containerization:** **Docker** for packaging the ML application and its dependencies into portable images.
* **CI/CD Automation:** **GitHub Actions** for automating the build, test, DVC pipeline execution, Docker image creation, and deployment processes.
* **Cloud Deployment & Orchestration:** Deployment to **AWS EKS (Elastic Kubernetes Service)** for scalable and managed Kubernetes.
* **Infrastructure as Code (IaC) Principles:** Leveraging `eksctl` which utilizes **AWS CloudFormation** under the hood for EKS cluster provisioning.
* **Monitoring & Observability:** **Prometheus** for collecting application and system metrics, and **Grafana** for visualizing these metrics through dashboards.
* **Remote Storage Solutions:** **AWS S3** for DVC remote storage and **AWS ECR (Elastic Container Registry)** for storing Docker images.

## 🛠️ Tech Stack

* **Languages & Frameworks:** Python 3.10, Flask (for API)
* **ML Libraries:** *[User to specify, e.g., Scikit-learn, TensorFlow, PyTorch, XGBoost]*
* **MLOps Tools:** MLflow, DVC, Cookiecutter (`drivendata/cookiecutter-data-science`)
* **CI/CD:** GitHub Actions
* **Containerization:** Docker
* **Cloud Platform:** AWS (EKS, S3, ECR, EC2, IAM, CloudFormation)
* **Orchestration:** Kubernetes (managed via `eksctl`)
* **Monitoring:** Prometheus, Grafana
* **Version Control Hosting:** GitHub, DagsHub

---

## 🌊 Project Workflow

The project follows a structured, multi-stage workflow:

### 1. Initial Project & Environment Setup
* Repository created and structured using the `cookiecutter-data-science` template.
* A Conda virtual environment (`atlas`) with Python 3.10 was established for dependency management.

### 2. MLflow for Experiment Tracking with DagsHub
* DagsHub was connected as a remote MLflow tracking server.
* Experiments, including parameters, metrics, and artifacts, were logged from notebooks/scripts, enabling comparison and versioning of experimental runs.

### 3. Data and Model Versioning with DVC
* DVC was initialized to manage datasets and ML model files that are too large for Git.
* Initially configured with a local DVC remote (`local_s3`), then transitioned to **AWS S3** for robust, centralized artifact storage:
    ```bash
    dvc remote add -d myremote s3://<your-s3-bucket-name>
    ```
* A DVC pipeline was defined in `dvc.yaml` (stages: data ingestion, preprocessing, feature engineering, model building, evaluation) and parameters in `params.yaml`.
* Executed the pipeline using `dvc repro` to ensure reproducibility. Changes were versioned with Git and DVC (`dvc push` to S3).

### 4. Core ML Pipeline Application Code
The `src/` directory was structured to house the ML pipeline scripts:
* `logger/`: Custom logging setup.
* `data_ingestion.py`: Scripts for fetching raw data.
* `data_preprocessing.py`: Scripts for data cleaning and transformation.
* `feature_engineering.py`: Scripts for creating model features.
* `model_building.py`: Script for training the machine learning model.
* `model_evaluation.py`: Script for evaluating model performance against specified metrics.
* `register_model.py`: Script for registering the finalized model (e.g., in MLflow's model registry).

### 5. Flask Application for Model Serving
* A Flask web application (`flask_app/`) was developed to expose the trained ML model via a REST API endpoint, allowing predictions to be served.

### 6. Dockerization for Portability
* Application dependencies for the Flask app were captured in `requirements.txt` (using `pipreqs ./ --force` within `flask_app/`).
* A `Dockerfile` was created to package the Flask application, ML model, and all necessary dependencies into a container image.
    * MLflow server URIs and other configurations were parameterized for flexibility.
* Docker images were built locally for testing:
    ```bash
    docker build -t your-username/capstone-app:latest .
    ```
* Local execution with environment variables (e.g., API keys, DagsHub tokens):
    ```bash
    docker run -p 8888:5000 -e CAPSTONE_TEST=<your_dagshub_token> your-username/capstone-app:latest
    ```

### 7. Continuous Integration & Continuous Deployment (CI/CD) with GitHub Actions
* A GitHub Actions workflow (`.github/workflows/ci.yaml`) was set up to automate:
    1.  Code checkout and environment setup.
    2.  Installation of dependencies.
    3.  Execution of tests (scripts in `tests/`).
    4.  Running the DVC pipeline (`dvc repro`) to regenerate artifacts if inputs change.
    5.  Building the Docker image.
    6.  Pushing the Docker image to **AWS ECR**.
    7.  Deploying the new image to the **AWS EKS** cluster.
* **Secrets Management:** AWS credentials (Access Key ID, Secret Key, Region), DagsHub tokens, and ECR repository details were stored as GitHub Secrets and Variables.

### 8. Deployment to AWS EKS (Elastic Kubernetes Service)
* **Prerequisites Installation:** Ensured `aws-cli`, `kubectl`, and `eksctl` command-line tools were installed and configured.
* **EKS Cluster Creation:**
    * An EKS cluster was provisioned using `eksctl`. This tool simplifies cluster creation and management.
      ```bash
      eksctl create cluster --name flask-app-cluster \
                            --region us-east-1 \
                            --nodegroup-name flask-app-nodes \
                            --node-type t3.small \
                            --nodes 1 --nodes-min 1 --nodes-max 1 --managed
      ```
    * `eksctl` leverages **AWS CloudFormation** to create and manage the necessary AWS resources (EKS control plane, node groups) as stacks.
* **Kubernetes Configuration (`kubectl`):**
    * The local `kubeconfig` file was updated to point `kubectl` to the newly created EKS cluster:
      ```bash
      aws eks --region us-east-1 update-kubeconfig --name flask-app-cluster
      ```
* **Application Deployment to EKS:**
    * Kubernetes manifest files (`deployment.yaml`, `service.yaml`) were created:
        * `deployment.yaml`: Defined the desired state for the application pods (Docker image from ECR, number of replicas, environment variables, port exposure). Secrets (like `capstone-secret` for DagsHub token) were mounted into pods.
        * `service.yaml`: Defined a Kubernetes Service of type `LoadBalancer` to expose the application to external traffic.
    * The CI/CD pipeline automatically applied these manifests to deploy/update the application on EKS.
    * **Security Group Configuration:** Inbound rules for the EKS node group's security group were modified to allow traffic on the application's port (e.g., 5000) from the LoadBalancer.
* **Accessing the Deployed Application:**
    * The external IP address of the LoadBalancer was retrieved:
      ```bash
      kubectl get svc flask-app-service
      ```
    * The application was accessed via `http://<external-ip-from-svc>:5000`.

### 9. Monitoring with Prometheus and Grafana
* **Prometheus Server Setup:**
    1.  Launched an Ubuntu EC2 instance (e.g., `t3.medium`, 20GB SSD).
    2.  Allowed inbound traffic on port `9090` (Prometheus UI) and `22` (SSH) in its security group.
    3.  Downloaded and installed Prometheus.
    4.  Configured `prometheus.yml` to scrape metrics from the Flask application (exposed via the EKS LoadBalancer's external IP and application port, typically on a `/metrics` endpoint if implemented in Flask).
        ```yaml
        global:
          scrape_interval: 15s
        scrape_configs:
          - job_name: "flask-app"
            static_configs:
              - targets: ["<your-eks-loadbalancer-external-ip>:<app-port>"] # e.g., a1b2c3d4.us-east-1.elb.amazonaws.com:5000
        ```
    5.  Ran Prometheus: `/usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml`.
    6.  Accessed Prometheus Web UI via `http://<prometheus-ec2-public-ip>:9090`.
* **Grafana Server Setup:**
    1.  Launched a separate Ubuntu EC2 instance (e.g., `t3.medium`, 20GB SSD).
    2.  Allowed inbound traffic on port `3000` (Grafana UI) and `22` (SSH).
    3.  Downloaded and installed Grafana.
    4.  Started and enabled the `grafana-server` service.
    5.  Accessed Grafana Web UI via `http://<grafana-ec2-public-ip>:3000` (default login: `admin`/`admin`).
    6.  Added the Prometheus server as a data source in Grafana (URL: `http://<prometheus-ec2-public-ip>:9090`).
    7.  Created dashboards to visualize metrics scraped by Prometheus.

---

## 💡 Key AWS & Kubernetes Concepts Utilized

* **AWS CloudFormation & `eksctl`:** When `eksctl create cluster` is executed, it generates CloudFormation templates to provision and manage the EKS control plane and worker node groups as distinct stacks (e.g., `eksctl-flask-app-cluster-cluster` for the control plane, `eksctl-flask-app-cluster-nodegroup-flask-app-nodes` for nodes). This exemplifies Infrastructure as Code.
* **EC2 Fleet Requests:** EKS NodeGroups use Auto Scaling Groups (ASGs) to request EC2 instances. These requests are processed as Fleet Requests. AWS accounts have quotas for Fleet Requests, which can impact NodeGroup creation if the limit is reached.
* **Kubernetes PersistentVolumeClaim (PVC):** While this specific project flow might not deeply detail stateful storage, PVCs are fundamental in Kubernetes for applications requiring persistent data. A PVC is a request for storage by a user, which is then fulfilled by a PersistentVolume (PV), often provisioned dynamically by a StorageClass (e.g., using AWS EBS).

---

## 🔧 High-Level Setup & Replication Notes

1.  **Prerequisites:**
    * AWS account with appropriate IAM permissions (EKS, EC2, S3, ECR, CloudFormation).
    * GitHub account.
    * DagsHub account (for MLflow).
    * Installed tools: `git`, `conda`/`python 3.10`, `docker`, `aws-cli`, `kubectl`, `eksctl`.
2.  **Clone Repository:** `git clone <your-repository-url>`
3.  **Configuration:**
    * Set up AWS credentials locally (`aws configure`).
    * Populate GitHub Secrets (AWS keys, DagsHub token, etc.) for CI/CD.
    * Update placeholders in configuration files (e.g., S3 bucket names, ECR repo names, Docker Hub username if applicable).
4.  **Execution:**
    * Follow the workflow steps, paying attention to `dvc.yaml`, `params.yaml`, `.github/workflows/ci.yaml`, `Dockerfile`, Kubernetes manifests (`deployment.yaml`, `service.yaml`), and `prometheus.yml`.

---

## 🧹 AWS Resource Cleanup

To prevent ongoing charges, ensure all created AWS resources are deleted:

1.  **Kubernetes Application Resources:**
    * `kubectl delete deployment flask-app`
    * `kubectl delete service flask-app-service`
    * `kubectl delete secret capstone-secret` (or other secrets created)
2.  **EKS Cluster:**
    * `eksctl delete cluster --name flask-app-cluster --region us-east-1`
    * Verify deletion: `eksctl get cluster --region us-east-1`
3.  **ECR Repository:** Delete Docker images and the repository itself from AWS ECR.
4.  **S3 Bucket:** Delete all objects and versions from the DVC remote S3 bucket, then delete the bucket.
5.  **EC2 Instances:** Terminate the EC2 instances used for Prometheus and Grafana.
6.  **CloudFormation Stacks:** Verify in the AWS CloudFormation console that stacks created by `eksctl` (e.g., `eksctl-flask-app-cluster-cluster`, `eksctl-flask-app-cluster-nodegroup-flask-app-nodes`) are deleted.
7.  **IAM Roles/Policies:** Delete any specific IAM roles or policies created if they are no longer needed.

---

## 🌟 Potential Future Enhancements

* Implement a dedicated `/metrics` endpoint in the Flask app using a library like `prometheus-flask-exporter` for richer application-specific metrics.
* Integrate advanced model monitoring for data drift, concept drift, and prediction quality.
* Set up automated model retraining triggers based on performance degradation or new data.
* Utilize AWS managed services for monitoring: Amazon Managed Service for Prometheus (AMP) and Amazon Managed Grafana (AMG).
* Implement more sophisticated deployment strategies in EKS like Canary or Blue/Green deployments.
* Expand test coverage with more comprehensive unit, integration, and end-to-end tests.
* Enhance EKS security using Network Policies, Pod Security Policies, and IAM Roles for Service Accounts (IRSA).
* Introduce a feature store for managing and serving ML features.

---

This project serves as a practical demonstration of orchestrating a full MLOps lifecycle, highlighting key skills in automation, cloud-native deployment, and operational best practices for machine learning systems.
