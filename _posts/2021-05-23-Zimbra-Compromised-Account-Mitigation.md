---
layout: post
title: Compromised Zimbra account mitigation
---

[Introduction](#introduction)\\
[Zimbra System Architecture](#zimbra-system-architecture)\\
[The Project Structure](#the-project-structure)\\
[The Main Script Functions](#the-main-script-functions)\\
[The Usage example](#the-usage-example)\\
[References](#references)

## Introduction

Securing email accounts is an important part of effective email security strategy beside hardening a mail server security.
Using compromised email accounts, cyber-criminals can send malicious and spam email messages to third users which could possibly lead to putting company's public IP address or even company's mail domain name on external blacklists which could further lead to problems in business communication.

In the following text I will cover scenario when a Zimbra account is
compromised and prompt response is required to reset user's password and optionally disable Active Directory authentication and failed login lockout policy.

## Zimbra System Architecture

The code is tested in single server Zimbra deployment environment with external Microsoft Active Directory server for authentication.
In single server Zimbra deployment services like web proxy, mailbox and MTA are installed on a single server:

<figure>
  <img class="picture" src="/assets/images/system_architecture.png" />
  <figcaption class="picture-caption">Picture 1. Single server deployment</figcaption>
</figure>
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

## The Main Script Functions

The [main](https://github.com/ranzer/compromised_zimbra_account_mitigation/blob/main/src/hacked_zimbra_account_mitigation.bash){:target="_blank"} script consists of the following functions:

1. **usage** - prints info how to invoke the script.
2. **check_args** - obtains options and their values from the defined list of parameters, if not all required options are provided invokes the usage function.
3. **get_email_credentials_formatted** - puts an array of email addresses and passwords into the HTML format.
4. **get_email_body_formatted** - replaces placholders for email addresses, passwords and company's logo in the message body template.
5. **generate_password** - generates random password, allows specified the password length, default 32 chars.
6. **get_new_passwords** - for each email address generates new password and returns and array of email address/password pairs.
7. **update_passwords** - updates mail account password foreach email address in an array of email address/password pairs.
8. **disable_ad_auth** - disables Active Directory authentication for each mail address provided.
9. **enable_lockout_policy** - enables or disables failed login policy for each mail address provided.
10. **send_email** - sends an email with updated credentials.
11. **main** - the function containing the program workflow.

## The Usage Example

Let us suppose that we want to disable Active Directory authentication and failed login policy for user1,
user2 and user3 mail accounts belonging to example.com mail domain.
The [main](https://github.com/ranzer/compromised_zimbra_account_mitigation/blob/main/src/hacked_zimbra_account_mitigation.bash){:target="_blank"} script is invoked in the following way:

> ./src/hacked_zimbra_account_mitigation.bash -c ./config/mail.config -d 1 -l 0 -a user1@example.com user2@example.com user3@example.com

## References

1. [Bats-core](https://github.com/bats-core/bats-core)
