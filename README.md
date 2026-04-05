# 🌐 Secure Cloud-Native Recruitment Portal (GCP)

A robust, production-grade web application built on **Google Cloud Platform (GCP)**. This project demonstrates advanced cloud engineering practices, including network isolation, managed database services, and IAM-driven security.

---

## 🚀 Architecture Overview
The system is designed following the **Principle of Least Privilege** and secure data flow:

*   **Custom VPC Network:** Resources are isolated within a private network, eliminating unauthorized external exposure.
*   **Compute Engine (VM):** A Linux-based web server (Apache/PHP) hosted in a public subnet, configured with a dedicated Service Account.
*   **Cloud SQL (Private IP):** A managed MySQL instance accessible *only* via the internal network. No public IP is assigned to the database.
*   **Cloud Storage (GCS):** A secure, private bucket for storing uploaded resumes, managed via IAM roles.
*   **IAM & Service Accounts:** The VM uses a specific Service Account with granular permissions to interact with GCS and SQL securely.

---

## 🛠️ Implementation Steps

### 1. Networking (VPC Setup)
Creating a secure, custom-built environment from scratch.

```bash
# Create the custom VPC
gcloud compute networks create my-custom-vpc --subnet-mode=custom

# Create a secure subnet
gcloud compute networks subnets create my-subnet \
    --network=my-custom-vpc \
    --range=10.0.1.0/24 \
    --region=us-central1

# Configure Firewall for Web Traffic (Port 80)
gcloud compute firewall-rules create allow-http-custom \
    --network=my-custom-vpc --allow=tcp:80 --target-tags=http-server

2. Identity & Access (IAM)
Provisioning a dedicated identity for the application.
# Create the Service Account
gcloud iam service-accounts create web-app-sa --display-name="Web App SA"

# Grant Storage Admin permissions (Scoped to the project)
gcloud projects add-iam-policy-binding [YOUR_PROJECT_ID] \
    --member="serviceAccount:web-app-sa@[YOUR_PROJECT_ID].iam.gserviceaccount.com" \
    --role="roles/storage.admin"

3. Web Server Deployment (Compute Engine)
Launching the VM attached to the custom VPC and the dedicated Service Account.
gcloud compute instances create cv-web-server \
    --zone=us-central1-a \
    --network=my-custom-vpc \
    --subnet=my-subnet \
    --service-account="web-app-sa@[YOUR_PROJECT_ID].iam.gserviceaccount.com" \
    --scopes=https://googleapis.com \
    --tags=http-server

💻 Application Stack (PHP/Apache)
⚙️ Installation Guide
Run these commands inside the VM to set up the environment:
# 1. Update system and install Apache, PHP, and MySQL drivers
sudo apt-get update
sudo apt-get install -y apache2 php libapache2-mod-php php-mysql

# 2. Set directory permissions for Apache
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html

# 3. Grant Apache permission to use 'gcloud' for storage uploads
echo "www-data ALL=(ALL) NOPASSWD: /usr/bin/gcloud" | sudo tee /etc/sudoers.d/www-data

# 4. Restart server
sudo systemctl restart apache2

📄 The Website Script (index.php)
This script bridges the Private Cloud SQL and Cloud Storage using the VM's internal identity.
<?php
// --- Configuration (Environment Variables) ---
$db_host = "[YOUR_DATABASE_PRIVATE_IP]"; 
$db_user = "root";
$db_pass = "[YOUR_DB_PASSWORD]";
$db_name = "mysql"; 
$bucket  = "[YOUR_GCS_BUCKET_NAME]";

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    $name = $_POST['name'];
    $job  = $_POST['job'];
    $file_tmp = $_FILES['cv_file']['tmp_name'];
    $file_name = $_FILES['cv_file']['name'];

    // 1. Connection to Cloud SQL via Private IP
    $conn = new mysqli($db_host, $db_user, $db_pass, $db_name);
    
    if (!$conn->connect_error) {
        $conn->query("CREATE TABLE IF NOT EXISTS job_applications (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100), job VARCHAR(100), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)");
        $stmt = $conn->prepare("INSERT INTO job_applications (name, job) VALUES (?, ?)");
        $stmt->bind_param("ss", $name, $job);
        $stmt->execute();
        $conn->close();
    }

    // 2. Upload to Cloud Storage using VM Service Account
    $destination = "gs://$bucket/cvs/" . time() . "_" . $file_name;
    $cmd = "sudo gcloud storage cp $file_tmp $destination 2>&1";
    exec($cmd, $output, $return_val);

    if ($return_val === 0) {
        echo "<h1 style='text-align:center; color:green;'>Application Submitted! ✅</h1>";
    }
    exit;
}
?>

html
<!-- Simple UI Form -->
<form method="post" enctype="multipart/form-data" style="text-align:center; padding:50px;">
    <h2>Submit Your Application</h2>
    <input type="text" name="name" placeholder="Full Name" required><br><br>
    <input type="text" name="job" placeholder="Target Job" required><br><br>
    <input type="file" name="cv_file" required><br><br>
    <button type="submit">Submit Application</button>
</form>

markdown
---
**Developed by Ghadah as part of a GCP Cloud Engineering project.**
