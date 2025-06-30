# Kubernetes Deployment and Secret Management Guide

This document provides a step-by-step guide to deploying applications on Google Kubernetes Engine (GKE) with secure secret management using Google Secret Manager integrated via the Secrets Store CSI Driver.

---

## Overview

This guide covers:

- Building and pushing Docker images to Google Container Registry (GCR)
- Configuring local access to a GKE cluster with `kubectl`
- Creating Kubernetes Deployment and Service manifests
- Integrating Google Secret Manager secrets into Kubernetes Pods securely
- Applying manifests in the correct order for smooth deployment
- Debugging running containers
- Safely removing secrets from deployments and the cluster

---

## Prerequisites

- Google Cloud SDK (`gcloud`) installed and authenticated
- Docker installed locally
- Access to a GKE cluster with appropriate permissions
- `kubectl` installed and configured
- Google Secret Manager enabled in your GCP project
- Secrets Store CSI Driver installed on your GKE cluster (see [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/))
