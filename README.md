# RoboShop Shell Automation

Bash automation for provisioning and configuring the RoboShop microservices environment on AWS.

This repository represents the shell scripting stage of my RoboShop DevOps work. It automates both AWS infrastructure tasks and Linux service configuration using Bash, AWS CLI, systemd, Route 53, and service-specific installation scripts.

The project helped move repeated server setup away from manual commands and into scripts that can validate inputs, install dependencies, configure services, record logs, and verify whether the expected service is running.

## What This Repository Automates

The repository contains automation for the RoboShop application and its supporting services.

| Component | Script |
| --- | --- |
| MongoDB | `roboshop-mongodb.sh` |
| Redis | `roboshop-redis.sh` |
| MySQL | `roboshop-mysql.sh` |
| RabbitMQ | `roboshop-rabbitmq.sh` |
| Catalogue | `roboshop-catalogue.sh` |
| User | `roboshop-user.sh` |
| Cart | `roboshop-cart.sh` |
| Shipping | `roboshop-shipping.sh` |
| Payment | `roboshop-payment.sh` |
| Dispatch | `roboshop-dispatch.sh` |
| Frontend | `roboshop-frontend.sh` |

Supporting configuration includes systemd service definitions, package repository configuration, and the nginx configuration used by the frontend.

## Repository Structure

```text
.
├── roboshop-instance.sh
│
├── roboshop-mongodb.sh
├── roboshop-redis.sh
├── roboshop-mysql.sh
├── roboshop-rabbitmq.sh
│
├── roboshop-catalogue.sh
├── roboshop-user.sh
├── roboshop-cart.sh
├── roboshop-shipping.sh
├── roboshop-payment.sh
├── roboshop-dispatch.sh
├── roboshop-frontend.sh
│
├── catalogue.service
├── user.service
├── cart.service
├── shipping.service
├── payment.service
├── dispatch.service
│
├── mongodb.repo
├── rabbitmq.repo
└── nginx.conf
```

`roboshop-instance.sh` handles the AWS-side instance lifecycle.

The remaining scripts configure the operating system and individual RoboShop components.

## AWS Instance Automation

`roboshop-instance.sh` uses AWS CLI to create and remove EC2 instances used by the RoboShop environment.

The script accepts an action followed by one or more component names:

```bash
sh roboshop-instance.sh create mongodb
```

Multiple components can be supplied in one command:

```bash
sh roboshop-instance.sh create \
  mongodb \
  redis \
  mysql \
  rabbitmq \
  catalogue \
  user \
  cart \
  shipping \
  payment \
  dispatch \
  frontend
```

Instances can be removed with:

```bash
sh roboshop-instance.sh delete catalogue
```

The script performs tasks including:

- command-line argument validation
- checking whether a named instance is already running
- creating EC2 instances through AWS CLI
- applying common and component-specific security groups
- waiting for EC2 state transitions
- reading public and private IP addresses
- creating or updating Route 53 records
- terminating instances when requested

The script uses:

```bash
set -euo pipefail
```

so common Bash failures such as failed commands, unset variables, and pipeline errors do not silently continue.

## Provisioning Flow

The infrastructure automation follows this general path:

```text
Component name
      |
      v
roboshop-instance.sh
      |
      +--> Check existing EC2 instance
      |
      +--> Create instance if required
      |
      +--> Wait until running
      |
      +--> Read instance IP
      |
      +--> Update Route 53
      |
      v
Server ready for configuration
```

This separates the EC2 lifecycle from the scripts that configure the application.

## Service Configuration

Each component has its own shell script.

The exact installation steps depend on the service, but the application scripts follow a similar pattern:

```text
Validate execution
       |
       v
Install required runtime
       |
       v
Create application user
       |
       v
Download application artifact
       |
       v
Install dependencies
       |
       v
Configure systemd
       |
       v
Start service
       |
       v
Verify service state
```

Keeping one script per component made it possible to automate the environment incrementally before moving the same configuration into Ansible roles later.

## Error Handling and Logging

The service scripts use strict Bash execution:

```bash
set -euo pipefail
```

and an error trap so failures can be reported with the line where the command failed.

Application setup output is written under:

```text
/var/log/roboshop/
```

The scripts also check for root access before making system-level changes.

A shared validation pattern records whether an operation succeeded and stops the script when a required step fails.

## Repeatable Setup

Several scripts check the machine before installing software or creating operating-system resources.

For example, the catalogue setup checks whether Node.js is already installed and whether the `roboshop` system user already exists before creating it.

That allows selected setup operations to be skipped rather than blindly repeating them.

The catalogue script also checks whether its MongoDB data has already been loaded before importing it again.

## Catalogue Example

The catalogue automation demonstrates the general application-service pattern.

It:

1. Enables the required Node.js version
2. Installs Node.js if required
3. Configures the MongoDB client repository
4. Installs `mongosh`
5. Creates the `roboshop` system user if it does not exist
6. Downloads the catalogue application artifact
7. Installs Node.js dependencies
8. Loads the MongoDB catalogue data when needed
9. Installs the systemd service definition
10. Enables and starts the service
11. Checks that systemd reports the service as active

This turns a sequence of manual Linux commands into a repeatable service setup.

## Frontend Automation

The frontend script configures nginx.

It:

- enables the required nginx module
- installs nginx when required
- removes the default web content
- downloads and extracts the RoboShop frontend
- replaces the default nginx configuration
- enables and restarts nginx
- verifies that nginx is active

The frontend configuration provides the web entry point for the application.

## systemd Services

Backend applications are managed as Linux systemd services.

Service definitions are kept in the repository for components including:

```text
catalogue
user
cart
shipping
payment
dispatch
```

The setup scripts copy these definitions into:

```text
/etc/systemd/system/
```

and then reload systemd before enabling and starting the application.

This means application lifecycle operations can use normal Linux commands such as:

```bash
systemctl status catalogue
systemctl restart catalogue
journalctl -u catalogue
```

## Running a Service Script

The configuration scripts make system-level changes, so they require root privileges.

For example:

```bash
sudo bash roboshop-catalogue.sh
```

For the frontend:

```bash
sudo bash roboshop-frontend.sh
```

For MongoDB:

```bash
sudo bash roboshop-mongodb.sh
```

Before running the scripts in another environment, review values such as:

- AWS region
- AMI ID
- Route 53 hosted zone
- domain names
- security-group names
- application artifact locations

These values are tied to the environment in which the scripts were originally developed.

## Useful Verification Commands

Check an application service:

```bash
systemctl status catalogue
```

Check nginx:

```bash
systemctl status nginx
```

Follow service logs:

```bash
journalctl -u catalogue -f
```

Inspect the RoboShop setup logs:

```bash
ls -l /var/log/roboshop/
```

Check the EC2 instances created for RoboShop:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=roboshop-*"
```

Check Route 53 records:

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id <HOSTED_ZONE_ID>
```

## Technologies

- Bash
- Linux
- AWS CLI
- Amazon EC2
- Amazon Route 53
- systemd
- Nginx
- Node.js
- Java
- Python
- Go
- MongoDB
- Redis
- MySQL
- RabbitMQ

## RoboShop Automation Journey

This repository is the shell automation stage of my RoboShop DevOps work.

The same application was then used to practise progressively more reusable deployment approaches:

1. **shell-roboshop** — Bash and AWS CLI automation
2. [roboshop-ansible-v3](https://github.com/sai-pillalamarri/roboshop-ansible-v3) — configuration management using reusable Ansible roles
3. [roboshop-docker](https://github.com/sai-pillalamarri/roboshop-docker) — application containerisation with Docker and Docker Compose
4. [roboshop-dev-infra](https://github.com/sai-pillalamarri/roboshop-dev-infra) — AWS infrastructure managed with Terraform

The progression moved from scripting individual hosts toward configuration management, containers, reusable infrastructure, and eventually the platform engineering practices demonstrated in my newer projects.
