# Oracle Database CI/CD Automation with Kubernetes

Automated Oracle Database provisioning and schema management for feature branch development using Oracle Database Kubernetes Operator, GitHub Actions, Liquibase, and Kustomize.

## 📖 Overview

This repository contains the complete implementation of an automated CI/CD pipeline that creates isolated Oracle Database instances for each feature branch, deploys schemas using Liquibase, and automatically cleans up resources when branches are deleted.

**Read the full story:** [Database CI/CD with Oracle Database Kubernetes Operator, GitHub Actions, and Liquibase](YOUR_BLOG_LINK_HERE)

### The Problem We Solved

Traditional database development workflows suffer from:
- Shared development databases causing conflicts between developers
- Manual database provisioning taking 2-3 days
- Inconsistent environments across development teams
- No automated cleanup of test databases
- Schema deployment inconsistencies

### Our Solution

Automatically provision an isolated Oracle Database instance for every feature branch with:
- ✅ Automated database creation (15-20 minutes from branch creation to ready)
- ✅ Consistent schema deployment via Liquibase
- ✅ External access via LoadBalancer for connectivity
- ✅ Automatic cleanup when branches are deleted
- ✅ Configuration management using Kustomize

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│              GitHub Repository                       │
│  ┌──────────────────────────────────────────┐      │
│  │  Feature Branches                         │      │
│  │  - feature/user-auth                      │      │
│  │  - feature/payment-system                 │      │
│  └──────────────────────────────────────────┘      │
└─────────────────┬───────────────────────────────────┘
                  │
                  │ Triggers on push/delete
                  ↓
┌─────────────────────────────────────────────────────┐
│           GitHub Actions Runner                      │
│  - Authenticates with OCI                           │
│  - Deploys database via Kustomize                   │
│  - Runs Liquibase migrations                        │
│  - Exposes via LoadBalancer                         │
└─────────────────┬───────────────────────────────────┘
                  │
                  │ kubectl apply
                  ↓
┌─────────────────────────────────────────────────────┐
│     Oracle Kubernetes Engine (OKE) Cluster          │
│  ┌─────────────────────────────────────────┐       │
│  │  Oracle Database Operator                │       │
│  └─────────────────────────────────────────┘       │
│  ┌─────────────────────────────────────────┐       │
│  │  Namespace: feature-user-auth            │       │
│  │    - SingleInstanceDatabase CR           │       │
│  │    - Oracle Database Pod                 │       │
│  │    - Services (ClusterIP + LoadBalancer)│       │
│  └─────────────────────────────────────────┘       │
│  ┌─────────────────────────────────────────┐       │
│  │  Namespace: feature-payment-system       │       │
│  │    - SingleInstanceDatabase CR           │       │
│  │    - Oracle Database Pod                 │       │
│  │    - Services (ClusterIP + LoadBalancer)│       │
│  └─────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────┘
```

## 🚀 Key Features

### 1. Automated Database Lifecycle
- **Create**: Triggered on feature branch creation
- **Deploy**: Automatic schema deployment via Liquibase
- **Delete**: Automatic cleanup on branch deletion

### 2. Isolation
Each feature branch gets:
- Dedicated Kubernetes namespace
- Isolated Oracle Database instance
- Independent schemas and data

### 3. External Connectivity
- OCI LoadBalancer for external access
- Enables Liquibase to run from GitHub Actions
- Provides connection strings for developer access

### 4. Configuration Management
- Base configurations in `base/sidb-free-lite/`
- Dynamic patching via Kustomize
- Environment-specific customization

## 📋 Prerequisites

### Infrastructure
- Oracle Cloud Infrastructure (OCI) account
- Oracle Kubernetes Engine (OKE) cluster
- Oracle Database Operator for Kubernetes installed

### Required Tools
- `kubectl` configured for your OKE cluster
- OCI CLI configured with proper credentials
- Git

### GitHub Secrets
Configure the following secrets in your GitHub repository:

| Secret Name | Description |
|------------|-------------|
| `OCI_USER_OCID` | OCI user OCID |
| `OCI_FINGERPRINT` | API key fingerprint |
| `OCI_TENANCY_OCID` | OCI tenancy OCID |
| `OCI_KEY_PRIVATE` | Private API key (PEM format) |
| `OCI_REGION` | OCI region (e.g., us-ashburn-1) |
| `OCI_CLUSTER_ID` | OKE cluster OCID |
| `FT_DEFAULT_ADMIN_PASSWORD` | Default admin password for databases |

## 🔧 Installation & Setup

### 1. Clone the Repository
```bash
git clone https://github.com/Kuassim/AiWorld.git
cd AiWorld
```

### 2. Configure OKE Cluster
Ensure Oracle Database Operator is installed:
```bash
kubectl get pods -n oracle-database-operator-system
```

### 3. Configure GitHub Secrets
Navigate to your repository's Settings → Secrets and variables → Actions, and add all required secrets.

### 4. Review Base Configuration
Check the base database configuration:
```bash
cat base/sidb-free-lite/sidb-free-lite.yaml
```

### 5. Customize Liquibase Changelogs
Update your database schemas in:
- `liquibase/admin/` - System-level changes
- `liquibase/user_service/` - Application schemas
- `liquibase/[other_services]/` - Additional services

## 📚 Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── deploy-to-oke.yml      # Connecting to OKE 
|       |── create-db.yml          # Database creation workflow
│       └── delete-db.yml          # Database deletion workflow
├── base/
│   └── sidb-free-lite/
│       └── sidb-free-lite.yaml    # Base database configuration
├── liquibase/
│   ├── admin/
│   │   └── changelog.xml          # System-level changes
│   ├── user_service/
│   │   └── changelog.xml          # User service schema
│   └── [other_services]/
│       └── changelog.xml          # Additional service schemas
└── README.md
```

## 🎯 Usage

### Creating a Feature Database

1. **Create a feature branch:**
```bash
git checkout -b feature/my-new-feature
```

2. **Push the branch:**
```bash
git push -u origin feature/my-new-feature
```

3. **Monitor the workflow:**
- Navigate to Actions tab in GitHub
- Watch the "Create Oracle Database" workflow
- Wait 15-20 minutes for completion

4. **Get connection details:**
- Check workflow output for external IP
- Connection string format: `jdbc:oracle:thin:@//<EXTERNAL_IP>:1521/FREEPDB1`

### Connecting to Your Database

```bash
# Using SQL*Plus
sqlplus system/<password>@//<EXTERNAL_IP>:1521/FREEPDB1

# Using SQLcl
sql system/<password>@//<EXTERNAL_IP>:1521/FREEPDB1
```

### Deleting a Feature Database

Simply delete the branch:
```bash
git push origin --delete feature/my-new-feature
```

The workflow will automatically clean up all resources.

## ⚙️ Workflow Details

### Create Workflow
Triggered on push to `feature/*` branches:
1. Configures OCI CLI and kubectl
2. Handles stuck namespace cleanup (if needed)
3. Creates namespace and secrets
4. Deploys database using Kustomize
5. Waits for database to be ready (8-12 minutes)
6. Exposes database via LoadBalancer (2-3 minutes)
7. Runs Liquibase migrations (system + schemas)

### Delete Workflow
Triggered on branch deletion:
1. Configures OCI CLI and kubectl
2. Deletes the namespace
3. Removes finalizers if namespace is stuck
4. Cleans up all resources

## 🔍 Technical Challenges Solved

### 1. Kubernetes Authentication from GitHub Actions
**Challenge:** Securely authenticate kubectl from CI/CD  
**Solution:** OCI CLI with dynamically generated fingerprints

### 2. Database Naming with Kustomize
**Challenge:** Dynamic resource naming per branch  
**Solution:** JSON patches in kustomization.yaml

### 3. Stuck Namespace Termination
**Challenge:** Namespaces stuck in "Terminating" state  
**Solution:** Remove finalizers from SingleInstanceDatabase CR

### 4. External Database Connectivity
**Challenge:** Liquibase can't reach internal ClusterIP  
**Solution:** Expose via OCI LoadBalancer with external IP

## 📊 Performance Metrics

| Phase | Duration | Notes |
|-------|----------|-------|
| Setup (OCI CLI & kubectl) | 45s | Downloads and configures tools |
| Force delete stuck namespace | 15s | Only if namespace exists |
| Create namespace & secrets | 5s | Fast Kubernetes operations |
| Deploy database with Kustomize | 30s | Manifest generation and apply |
| **Wait for database ready** | **8-12 min** | Oracle DB initialization |
| Expose via LoadBalancer | 2-3 min | OCI Load Balancer provisioning |
| Liquibase: System setup | 30s | Create users and grants |
| Liquibase: Schema deployment | 1-2 min | Deploy tables and data |
| **Total End-to-End** | **15-20 min** | From branch push to ready |

## 🛡️ Security Considerations

### Current Implementation
- Databases exposed via public LoadBalancer
- Suitable for development/test environments
- Credentials stored in GitHub Secrets

### Production Recommendations
- Run Liquibase as Kubernetes Job (internal cluster access)
- Implement network policies to restrict database access
- Use OCI Vault for secret management
- Add TLS/SSL for database connections
- Restrict LoadBalancer access to specific IP ranges

## 🔮 Future Improvements

### Security
- [ ] Run Liquibase as Kubernetes Job
- [ ] Add network policies for database access
- [ ] Migrate to OCI Vault for secrets
- [ ] Implement TLS/SSL connections

### Operations
- [ ] Automated backups before schema changes
- [ ] Prometheus metrics for database health
- [ ] Cost tracking per feature branch
- [ ] Slack notifications for deployment status

### Developer Experience
- [ ] Database seeding with realistic test data
- [ ] Schema diff tool (compare to main)
- [ ] Direct kubectl access for troubleshooting
- [ ] Rollback capability for schema changes

## 📖 Documentation

- **Blog Post**: [Full implementation story](YOUR_BLOG_LINK_HERE)
- **Oracle Database Operator**: [GitHub](https://github.com/oracle/oracle-database-operator)
- **Liquibase**: [Documentation](https://www.liquibase.org/)
- **Kustomize**: [Documentation](https://kustomize.io/)

## 🙏 Acknowledgments

Inspired by [Norman Aberin's blog post](https://medium.com/@npaberin/database-ci-cd-with-the-oracle-db-operator-for-kubernetes-github-actions-and-liquibase-6dd161940a68) on Oracle DB Operator CI/CD.

## 📝 License

[Add your license here - e.g., MIT, Apache 2.0]

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 💬 Support

- **Issues**: [GitHub Issues](https://github.com/Kuassim/AiWorld/issues)
- **Discussions**: [GitHub Discussions](https://github.com/Kuassim/AiWorld/discussions)
- **Blog**: [Read the full article](YOUR_BLOG_LINK_HERE)

## 📧 Contact

For questions or feedback, please open an issue or reach out through GitHub.

---

**Built with ❤️ using Oracle Database Kubernetes Operator, GitHub Actions, Liquibase, and Kustomize**
