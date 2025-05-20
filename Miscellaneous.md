# Configuring Environment Variables for a Frappe-Based Application

## Table of Contents

* [Overview](#overview)
* [Environment Variables](#environment-variables)
* [Setup Instructions](#setup-instructions)
* [Additional Notes](#additional-notes)
* [Troubleshooting](#troubleshooting)

## Overview

Frappe-based applications often require environment variables to configure settings such as database connections, email services, or other integrations. This document provides a generic guide for setting up environment variables on a localhost environment to ensure consistent behavior, similar to configurations managed on Frappe Cloud.

## Environment Variables

Environment variables are key-value pairs used to configure your Frappe application. Examples include settings for email accounts, database credentials, or API keys. Below is an example set of variables for configuring an email account, but you can adapt this for any required variables:

* `EMAIL_ID`: Email address for the account (e.g., `your_email@gmail.com`).
* `EMAIL_PASSWORD`: Password or App Password for the email account (use an App Password if 2FA is enabled for Gmail).
* `EMAIL_SERVICE`: Email service provider (default: `GMail`).
* `EMAIL_ACCOUNT_NAME`: Display name for the email account (default: `Clapgrow Support`).
* `SMTP_SERVER`: SMTP server address (default: `smtp.gmail.com` for Gmail).
* `SMTP_PORT`: SMTP port number (default: `587` for Gmail).
* `USE_TLS`: Enable TLS for secure connection (default: `1` for Gmail).
* `USE_SSL`: Enable SSL for secure connection (default: `0`).

Replace these with the specific variables required by your application, as defined in your code or configuration.

## Setup Instructions

To set environment variables persistently across terminal sessions on your localhost, follow these steps:

1. **Open your shell configuration file**:
   Depending on your shell, this could be `~/.bashrc`, `~/.zshrc`, or `~/.profile`. Open the file in a text editor:
   ```bash
   nano ~/.bashrc
   ```

2. **Add the environment variables**:
   Append the required environment variables to the configuration file. For example, for email settings:
   ```bash
   export EMAIL_ID="your_email@gmail.com"
   export EMAIL_PASSWORD="your_email_password"
   export EMAIL_SERVICE="GMail"
   export EMAIL_ACCOUNT_NAME="Clapgrow Support"
   export SMTP_SERVER="smtp.gmail.com"
   export SMTP_PORT="587"
   export USE_TLS="1"
   export USE_SSL="0"
   ```
   Replace these with the specific variables your application needs.

3. **Save and reload the configuration**:
   Save the file and reload the shell configuration to apply the changes:
   ```bash
   source ~/.bashrc
   ```

4. **Verify the variables**:
   Confirm that the variables are set correctly by checking one or more of them:
   ```bash
   echo $EMAIL_ID
   ```
   This should display the value you set (e.g., `your_email@gmail.com`).

## Additional Notes

* **Security**: Avoid storing sensitive information (e.g., passwords, API keys) in version-controlled files. For production environments, use a secure vault or secret management system.
* **Frappe Configuration Alternative**: Instead of environment variables, you can add settings to `common_site_config.json` in your Frappe bench directory (`sites/common_site_config.json`). Example for email settings:
  ```json
  {
    "email_id": "your_email@gmail.com",
    "email_password": "your_email_password",
    "email_service": "GMail",
    "email_account_name": "Clapgrow Support",
    "smtp_server": "smtp.gmail.com",
    "smtp_port": 587,
    "use_tls": 1,
    "use_ssl": 0
  }
  ```
  After updating, restart the Frappe server:
  ```bash
  bench restart
  ```

* **Frappe Cloud**: On Frappe Cloud, environment variables are typically managed via the platform’s environment settings. The steps above replicate this setup for localhost development.
* **Custom Variables**: Identify the specific environment variables required by your application by reviewing its code or documentation. Variables are often accessed in Python using `os.getenv()` or `frappe.conf.get()`.

## Troubleshooting

* If your application fails to recognize environment variables, check the Frappe error logs (e.g., via `frappe.log_error`) for details.
* Ensure all required variables are set, as missing variables may cause configuration errors.
* Verify that the values match the expected format or requirements of the service (e.g., correct SMTP server/port for email providers).
* For Gmail with 2FA, ensure an App Password is used for `EMAIL_PASSWORD`.
* If using `common_site_config.json`, confirm the file syntax is valid JSON and restart the server after changes.

