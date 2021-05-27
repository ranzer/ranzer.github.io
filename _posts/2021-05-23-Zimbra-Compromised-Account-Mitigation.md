---
layout: post
title: Compromised Zimbra account mitigation
---

[Introduction](#introduction)\\
[Zimbra System Architecture](#zimbrasystemarchitecture)\\
[The Project Structure](#theprojectstructure)\\
[The Main Script](themainscript)\\
[References](#references)

## Introduction

Securing email accounts is an important part of effective email security strategy beside hardening a mail server security.
Using compromised email accounts, cyber-criminals can send malicious and spam email messages to third users which could possibly lead to putting company's public IP address or even company's mail domain name on external blacklists which could further lead to problems in business communication.

In the following text I will cover scenario when a Zimbra account is
compromised and prompt response is required to reset user's password and optionally disable Active Directory authentication and failed login lockout policy.

## Zimbra System Architecture

The code is tested in single server Zimbra deployment environment with external Microsoft Active Directory server for authentication.

## The Project Structure

The project has the following structure:

<pre>
├── config
│   └── mail.config
├── infrastructure
│   └── deploy.bash
├── src
│   └── hacked_zimbra_account_mitigation.bash
├── templates
│   ├── logo_data.txt
│   └── mail_credentials_changed.txt
└── test
    └── unit
        ├── config
        │   └── mail.config
        ├── hacked_zimbra_account_mitigation_unit_test.bats
        └── templates
            ├── logo_data.txt
            └── mail_credentials_changed.txt
</pre>

More details about what each file in the project structure represents can be found [here](https://github.com/ranzer/compromised_zimbra_account_mitigation/blob/main/README.md){:target="_blank"}.
In this text I will explain the code inside the [main](https://github.com/ranzer/compromised_zimbra_account_mitigation/blob/main/src/hacked_zimbra_account_mitigation.bash){:target="_blank"} script.
The workflow of the ./src/hacked_zimbra_account_mitigation.bash script is as following:
1. Check whether all required arguments are provided and make all provided arguments available for the script functions to use.
2. Check whether values of provided arguments are valid or not, if there are invalid values abort the script execution.
3. For each email address provided generate a new password.
4. Update passwords for each provided mail account (mail address).
5. Disable Active Directory authentication if required (this is done in order to prevent locking Active Directory accounts in situations when
   Zimbra allows more failed login attempts then Active Directory's account lockout threshold policy).
6. If required disable lockout policy on Zimbra for each provided mail account (if we set strong password then we can disable failed login policy on Zimbra in order to prevent mail account lockout).
7. Read mail template text and substitute placeholders for email addresses, passwords and company's logo with appropriate content.
8. Send formatted email body to appropriate recipients.

## The Main Script

The Main Script

## References
