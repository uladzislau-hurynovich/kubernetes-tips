# 🧩 Kubernetes Infrastructure

Unified repository for managing and deploying **Kubernetes environments** across different platforms — both **on-premises** and in the **cloud**.

This project provides reusable infrastructure blueprints, scripts, and Helm configurations for building consistent Kubernetes clusters, regardless of the environment.

---

## 🚀 Environments

| Environment | Description | Link |
|--------------|-------------|------|
| 🏠 **On-prem K8S** | Local Kubernetes cluster running on virtual machines. Includes NFS server, 1 master, and 2 worker nodes for development and testing. | [View setup →](./on_premise_k8s/README.md) |
| ☁️ **AWS EKS** | Cloud-based Kubernetes cluster deployed via Terraform and Helm. Includes monitoring stack and best practices for production-grade workloads. | [View setup →](./eks/README.md) |

---

## 📄 License
This project is provided under the [MIT License](LICENSE).

---

## 👨‍💻 Author
Maintained by *Uladzislau Hurynovich*  
Feel free to contribute or open issues for improvement.

---