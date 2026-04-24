# Day 19 — Git Deploy, RDS Skeleton, Rolling Strategy, Ping Check

## 🔹 Ansible — Deploy App Using Git

---
- name: Deploy app from Git for Pathnex
  hosts: all
  become: yes

  tasks:
    - name: Install git
      yum:
        name: git
        state: present

    - name: Clone repository
      git:
        repo: "https://github.com/pathnex/app.git"
        dest: "/opt/pathnex-app"


## 🔹 Terraform — RDS Skeleton (MySQL)

resource "aws_db_instance" "PathnexRDS" {
  allocated_storage      = 20
  engine                 = "mysql"
  instance_class         = "db.t3.micro"
  username               = "admin"
  password               = "Pathnex123"
  skip_final_snapshot    = true
}


## 🔹 Kubernetes — Deployment Rolling Update (Detailed)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: pathnex-rollout
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: rollout
  template:
    metadata:
      labels:
        app: rollout
    spec:
      containers:
        - name: app
          image: nginx


## 🔹 Shell Script — Check Internet Connectivity

#!/bin/bash

ping -c 2 google.com &> /dev/null

if [ $? -eq 0 ]; then
  echo "Internet is available"
else
  echo "No internet connection"
fi


# Approvals

## 🔹 Jenkins Pipeline — Manual Approval
You will learn how to **pause pipeline until manual approval**.

pipeline {
    agent any
    environment {
        INSTITUTE_NAME = "Pathnex"
    }
    stages {
        stage('Deploy') {
            steps {
                input message: "Approve deployment for $INSTITUTE_NAME?"
                sh 'echo "Deployment approved and executed!"'
            }
        }
    }
}

## 🔹 GitLab CI — Manual Approval Job
You will learn how to **use manual jobs in GitLab CI for approvals**.

stages:
  - deploy

variables:
  INSTITUTE_NAME: "Pathnex"

deploy:
  stage: deploy
  image: alpine:latest
  script:
    - echo "Ready to deploy $INSTITUTE_NAME"
  when: manual


Day 19 — Advanced Monitoring, Logging, and Security
🔹 Ansible — Setup Prometheus for Monitoring
- name: Install Prometheus for Monitoring
  hosts: all
  become: yes
  tasks:
    - name: Install Prometheus dependencies
      yum:
        name:
          - prometheus
          - prometheus-node-exporter
        state: present
    - name: Start Prometheus service
      service:
        name: prometheus
        state: started
        enabled: yes
🔹 Terraform — Setup AWS CloudWatch for Monitoring
resource "aws_cloudwatch_log_group" "pathnex_log_group" {
  name = "pathnex-log-group"
}

resource "aws_cloudwatch_log_stream" "pathnex_log_stream" {
  log_group_name = aws_cloudwatch_log_group.pathnex_log_group.name
  name           = "pathnex-log-stream"
}
🔹 Kubernetes — Deploy Fluentd for Logging
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fluentd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluent/fluentd
          ports:
            - containerPort: 24224
---
apiVersion: v1
kind: Service
metadata:
  name: fluentd
spec:
  selector:
    app: fluentd
  ports:
    - protocol: TCP
      port: 24224
🔹 Jenkinsfile — Integrating Prometheus Monitoring in Pipeline
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building Docker image...'
                docker.build('pathnex-web-app')
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
            }
        }
        stage('Monitor Deployment') {
            steps {
                script {
                    sh 'kubectl apply -f kubernetes/deployment.yaml'
                    sh 'kubectl get pods --watch'
                }
            }
        }
    }
}
🔹 GitLab CI/CD — Prometheus and Logging Integration
stages:
  - build
  - push
  - deploy

build:
  stage: build
  script:
    - docker build -t pathnex-web-app .

push:
  stage: push
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD"
    - docker push pathnex-web-app

deploy:
  stage: deploy
  script:
    - kubectl apply -f kubernetes/deployment.yaml
    - kubectl get pods --watch