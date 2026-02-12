# **GCE VM Protection Migration Tool: Management Console to Backup Vault**

## **Overview**

This tool facilitates migrating GCE VM protection from Management Console SLTs to Backup Vault Backup Plans. It does **not** migrate existing backups. The process uses an **overlap and cutover** strategy:

1. **Protect:** Use `MigrateToBackupVault` to protect VMs via a Backup Plan.  
2. **Verify:** Confirm backups are being created successfully in the Backup Vault.  
3. **Unprotect:** Use `UnprotectVMsFromManagementConsole` to safely remove protection from the Management Console.

The tool includes checks to ensure a safe transition. For example, unprotection from the Management Console will only occur if a successful backup exists in the Backup Vault for instance.

## **Limitations**

Migration is **not supported** if your current SLT or VM setup includes any of the following:

**SLT Features:**

* **High-Frequency Backups:** Intervals less than 4 hours.  
* **Cross-Region Backups:** Storing backups in a different region than the source VM.  
* **App-Consistent / VSS Snapshots:** Backups requiring agent-based quiescing. Backup Vault currently provides **Crash-Consistent** backups only.  
* **Archive Snapshots:** Long-term retention using archive snapshots.

**VM Features:**

* **Hyperdisk Storage:** VMs using Google Cloud Hyperdisk.  
* **Partial / Selective Disk Backup:** VMs where specific disks are excluded from backups. Backup Vault protects the entire VM.

The tools will check for these limitations at both the SLT and individual VM levels.

## **Prerequisites**

### **Required IAM Roles**

The user or service account running the tool needs:

* `Backup and DR Admin` (Legacy role for Management Console operations)  
* `roles/backupdr.admin` (To manage Backup Vault, Backup Plans, and Associations)

### **Required Resources**

* A pre-existing **Backup Vault**.  
* A pre-existing **Backup Plan** configured to use the Backup Vault.  
  * See Google Cloud documentation for creating [Backup Vaults](https://cloud.google.com/backup-disaster-recovery/docs/concepts/backup-vault) and [Backup Plans](https://cloud.google.com/backup-disaster-recovery/docs/create-plan/create-backup-plan).

## **Installation**

The tool is provided as a single executable: `backupdr_gcevm_migration_tool`.

### **Method 1: Google Cloud Shell**

1. **Upload:**  
   * Open Cloud Shell in the Google Cloud Console.  
   * Click the **three vertical dots** (More) \> **Upload** and select the `backupdr_gcevm_migration_tool` file.  
2. **Make Executable:**

```shell
chmod +x ~/backupdr_gcevm_migration_tool
```

   You can now run the tool using `~/backupdr_gcevm_migration_tool`.

### **Method 2: Local Environment**

1. **Download:** Place `backupdr_gcevm_migration_tool` in a directory (e.g., `~/tools`).  
2. **Install Google Cloud SDK:** Ensure `gcloud` CLI is installed and authenticated. [Install Guide](https://cloud.google.com/sdk/docs/install).  
3. **Make Executable:**

```shell
mkdir -p ~/tools
# Assuming download is in ~/Downloads
mv ~/Downloads/backupdr_gcevm_migration_tool ~/tools/
chmod +x ~/tools/backupdr_gcevm_migration_tool
```

4. **Update PATH (Optional):** To run the tool from any directory, add it to your PATH. Add to `~/.bashrc`, `~/.zshrc`, etc.:

```shell
export PATH="$PATH:$HOME/tools"
```

   Reload your shell: `source ~/.bashrc` (or equivalent).

## **Running the Tool**

Replace placeholder values in examples with your actual resource names and URLs.

### **1\. Protect GCE VMs via Backup Vault (`MigrateToBackupVault`)**

This command identifies eligible VMs in an SLT and protects them using the specified Backup Plan.

**Command:**

```shell
backupdr_gcevm_migration_tool MigrateToBackupVault \
  --slt_name="your-slt-name" \
  --management_server_url="https://your-ms-url.backupdr.googleusercontent.com/" \
  --backup_plan="projects/YOUR_PROJECT/locations/YOUR_LOCATION/backupPlans/your-bp-name" \
  --workload_projects="project1,project2" \
  --apply \
  --trigger_backup
```

**Arguments:**

* `--slt_name` (Required): Name of the source SLT in the Management Console.  
* `--management_server_url` (Required): URL of the Backup and DR Management Console.  
* `--backup_plan` (Required): Full resource name of the Backup Plan to apply.  
* `--workload_projects` (Required): Comma-separated list of project IDs containing GCE VMs, or "`all`" for all projects associated with the SLT.  
* `--apply` (Optional): Executes the protection. Without this flag, the command runs in dry-run mode, reporting on eligible VMs and potential issues.  
* `--trigger_backup` (Optional): Initiates an on-demand backup immediately after protection is applied. Useful for quick verification.

**Output:** `MigrateToBackupVault_{slt_name}_output.txt` summarizes actions taken and VM status.

**Verify:** Check the **Backup and DR \> Vaulted Backups** page in the Cloud Console to confirm VMs are protected by the Backup Plan.

### **2\. Unprotect GCE VMs from Management Console (`UnprotectVMsFromManagementConsole`)**

This command removes SLT-based protection from VMs in the Management Console, **only if** they are protected by the Backup Plan AND have at least one successful Backup Vault backup.

**Command:**

```shell
backupdr_gcevm_migration_tool UnprotectVMsFromManagementConsole \
  --slt_name="your-slt-name" \
  --management_server_url="https://your-ms-url.backupdr.googleusercontent.com/" \
  --workload_projects="project1,project2" \
  --apply
```

**Arguments:**

* `--slt_name` (Required): Name of the source SLT.  
* `--management_server_url` (Required): URL of the Backup and DR Management Console.  
* `--workload_projects` (Required): Comma-separated list of project IDs or "`all`".  
* `--apply` (Optional): Executes the unprotection. Dry-run mode (default) lists VMs eligible for unprotection based on Backup Vault status.

**Output:** `UnprotectVMsFromManagementConsole_{slt_name}_output.txt` summarizes actions.

**Verify:** Check the Management Console to confirm eligible instances are no longer protected by the SLT.  
