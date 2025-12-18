# 🚀 AWS EC2 + Provisioners Lab (Terraform)

In this lab, I learned how to **provision an EC2 instance with Terraform**, create and attach an **SSH key pair**, install **Nginx using user data**, work with **provisioners**, and create & associate an **Elastic IP**. I also learned how dependency chains work and how Terraform determines which resources depend on which.

---

## 📋 Lab Overview

**Goal:**

* Provision an EC2 instance with variables
* Create and use an AWS key pair
* Install software via user data
* Understand why user_data only works at first boot
* Implement provisioners (`local-exec`)
* Create an Elastic IP and attach it to the instance
* Manage dependencies and resource updates

**Learning Outcomes:**

* Use variables in Terraform (`ami`, `region`, `instance_type`)
* Understand resource replacement when immutable fields change
* Use `file()` and Heredoc syntax
* Use provisioners correctly and know their limitations
* Associate EIP resources with EC2
* Understand Terraform dependency inference

---

## 🛠 Step-by-Step Journey

### Step 1: Create EC2 Instance Resource

**Directory:**

```
/root/terraform-projects/project-servers
```

Created variables & resource:

```hcl
variable "ami" {
  default = "ami-061..."
}

variable "region" {
  default = "eu-west-2"
}

variable "instance_type" {
  default = "m5.large"
}

resource "aws_instance" "servers" {
  ami           = var.ami
  instance_type = var.instance_type
}
```

Commands:

```bash
terraform init
terraform plan
terraform apply
# yes
```

✔️ EC2 instance created.

---

### Step 2: Inspect Instance State

Command:

```bash
terraform show
```

This prints every attribute stored in state (AMI, instance_type, IDs, IPs, metadata, etc.)

---

### Step 3: Create SSH Key Pair

A key pair already exists in:

```
/root/terraform-projects/project-servers/.ssh
```

Files:

```
servers        # private key
servers.pub    # public key
```

Create resource:

```hcl
resource "aws_key_pair" "servers_key" {
  key_name   = "Cerberus"
  public_key = file(".ssh/servers.pub")
}
```

❌ Initial mistake: `key_pair` instead of `aws_key_pair`
✔️ Fixed → Terraform recognized AWS provider correctly.

---

### Step 4: Attach Key Pair to EC2 Instance

Updated EC2 resource:

```hcl
resource "aws_instance" "servers" {
  ami           = var.ami
  instance_type = var.instance_type
  key_name      = "Cerberus"
}
```

Running:

```bash
terraform plan
terraform apply
```

This triggers **instance replacement** (destroy + recreate) because `key_name` is an immutable field.

✔️ New instance now supports SSH login.

---

### Step 5: Add User Data Script to Install Nginx

```hcl
user_data = file("install-nginx.sh")
```

**Question: What happens now if we run `terraform apply`?**

Correct answer:

➡️ **The instance will be modified, but Nginx will NOT be installed.**

Why?

* `user_data` only runs at first boot
* Updating `user_data` does not recreate the instance unless forced

---

### Step 6: Apply Changes

Command:

```bash
terraform apply
# yes
```

✔️ State updated
❌ Nginx not installed — expected behavior

---

### Step 7: Understanding Provisioners

**Where should a provisioner block go?**
➡️ Inside the resource block (nested)

**Which provisioner does *not* need a connection block?**
➡️ `local-exec`

(Remote-exec & file provisioners *do* require SSH connection config)

---

### Step 8: Get the Public IPv4 Address

Using:

```bash
terraform show
```

Example:

```
public_ip = 54.214.185.180
```

---

### Step 9: Create an Elastic IP and Associate It

Resource:

```hcl
resource "aws_eip" "eip" {
  vpc      = true
  instance = aws_instance.servers.id

  provisioner "local-exec" {
    command = "echo ${aws_eip.eip.public_dns} >> /root/servers_public_dns.txt"
  }
}
```

Commands:

```bash
terraform apply
# yes
```

✔️ EIP created
✔️ EIP associated with instance
✔️ DNS saved to file

---

### Step 10: Verify Public DNS

Using:

```bash
cat /root/servers_public_dns.txt
```

Example:

```
ec2-127-186-120-xx.compute.amazonaws.com
```

---

### Step 11: Dependency Question

**Which dependency is NOT true?**

Options:

1. EIP depends on Cerberus
2. Cerberus depends on servers_key
3. Cerberus depends on EIP

Correct answer:

➡️ **Cerberus (EC2 instance) does NOT depend on the EIP resource.**

The instance is created first, then the EIP attaches to it.

---

## ✅ Key Commands Summary

| Task                | Command                              |
| ------------------- | ------------------------------------ |
| Create EC2 instance | `aws_instance`                       |
| Create key pair     | `aws_key_pair`                       |
| Attach key pair     | `key_name = "Cerberus"`              |
| Run user data       | `user_data = file(...)`              |
| Run provisioner     | `provisioner "local-exec" {}`        |
| Create EIP          | `aws_eip`                            |
| Associate EIP       | `instance = aws_instance.servers.id` |
| Show state          | `terraform show`                     |
| Apply & recreate    | `terraform apply`                    |

---

## 💡 Notes / Tips

* If a field is **immutable** (like `key_name`), Terraform must recreate the instance.
* User data scripts **only run on first boot** — updating them doesn’t trigger recreation.
* Provisioners should be avoided in production — but are useful for lab demos.
* Elastic IPs ensure a **persistent public IP** even if the instance reboots.
* `local-exec` runs on the **Terraform machine**, not the EC2 instance.

---

## 📌 Lab Summary

| Step                    | Status | Key Takeaways                          |
| ----------------------- | ------ | -------------------------------------- |
| Create EC2 instance     | ✅      | Variables + AMI + instance_type        |
| Create key pair         | ✅      | Resource naming fixed                  |
| Attach key to instance  | ✅      | Instance replaced                      |
| Add user data script    | ⚠️     | Script not executed after modification |
| Understand provisioners | ✅      | local-exec needs no connection         |
| Create EIP              | ✅      | Associated + public DNS saved          |
| Analyze dependencies    | ✅      | EC2 does NOT depend on EIP             |

---

## 📎 References

* [Terraform AWS Instance](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance)
* [Terraform AWS Key Pair](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/key_pair)
* [Terraform User Data](https://developer.hashicorp.com/terraform/language/functions/file)
* [Terraform Provisioners](https://developer.hashicorp.com/terraform/language/resources/provisioners)
* [Terraform AWS Elastic IP](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip)
