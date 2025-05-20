# Frappe Email Account Setup

This README provides instructions for configuring environment variables to set up an email account in a Frappe-based application on a localhost environment. These variables are used by the `setup_email_account` function to enable sending and receiving emails.

## Table of Contents

* [Overview](#overview)
* [Environment Variables](#environment-variables)
* [Setup Instructions](#setup-instructions)
* [Additional Notes](#additional-notes)
* [Troubleshooting](#troubleshooting)

## Overview

The email account setup in the Frappe application requires specific environment variables to configure email services, such as Gmail, for sending and receiving emails. This document explains how to set these variables persistently on a localhost environment using your shell configuration file.

## Environment Variables

The following environment variables are required:

* `EMAIL_ID`: The email address for the account (e.g., `your_email@gmail.com`).
* `EMAIL_PASSWORD`: The password or App Password for the email account (use an App Password if 2FA is enabled for Gmail).
* `EMAIL_SERVICE`: The email service provider (default: `GMail`).
* `EMAIL_ACCOUNT_NAME`: The display name for the email account (default: `Clapgrow Support`).
* `SMTP_SERVER`: The SMTP server address (default: `smtp.gmail.com` for Gmail).
* `SMTP_PORT`: The SMTP port number (default: `587` for Gmail).
* `USE_TLS`: Enable TLS for secure connection (default: `1` for Gmail).
* `USE_SSL`: Enable SSL for secure connection (default: `0`).

## Setup Instructions

To make these environment variables persistent across terminal sessions on your localhost, follow these steps:

1. **Open your shell configuration file**:
   Depending on your shell, this could be `~/.bashrc`, `~/.zshrc`, or `~/.profile`. Open the file in a text editor:
   ```bash
   nano ~/.bashrc
   ```

2. **Add the environment variables**:
   Append the following lines to the configuration file:
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

3. **Save and reload the configuration**:
   Save the file and reload the shell configuration to apply the changes:
   ```bash
   source ~/.bashrc
   ```

4. **Verify the variables**:
   Confirm that the variables are set correctly:
   ```bash
   echo $EMAIL_ID
   ```
   This should display the email address (e.g., `your_email@gmail.com`).

## Additional Notes

* **Security**: Do not store sensitive information like `EMAIL_PASSWORD` in version-controlled files. For production, use a secure vault or secret management system.
* **Gmail App Password**: If using Gmail with two-factor authentication (2FA), generate an App Password from your Google Account settings and use it as `EMAIL_PASSWORD`.
* **Frappe Configuration Alternative**: You can configure these settings in `common_site_config.json` in your Frappe bench directory (`sites/common_site_config.json`):
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

* **Frappe Cloud**: On Frappe Cloud, these variables are managed via the platform's environment settings. The above steps replicate that setup for localhost development.

## Troubleshooting

* If the email account setup fails, check the Frappe error logs for details (logged via `frappe.log_error`).
* Ensure `EMAIL_ID` and `EMAIL_PASSWORD` are set, as they are mandatory for the `setup_email_account` function.
* Verify that the SMTP server and port match your email provider's requirements.
* For Gmail, ensure the correct App Password is used if 2FA is enabled.

For additional support, refer to the [Frappe Documentation](https://frappeframework.com/docs) or contact the development team.
