---
layout: post
title: Compromised Zimbra account mitigation
---

[Introduction](#introduction)\\
[Dataset](#dataset)\\
[Development environment setup](#development-environment-setup)\\
[Data gathering and preprocessing](#data-gathering-and-preprocessing)\\
[Create Amazon Lookout for Vision project](#create-amazon-lookout-for-vision-project)\\
[Create train and test datasets](#create-train-and-test-datasets)\\
[Create the model](#create-the-model)\\
[Evaluate the model](#evaluate-the-model)\\
[Conclusion](#conclusion)\\
[References](#references)

## Introduction

Securing email accounts is an important part of effective email security strategy beside hardening a mail server security.
Using compromised email accounts, cyber-criminals can send malicious and spam email messages to third users which could possibly lead to putting company's public IP address or even company's mail domain name on external blacklists which could further lead to problems in business communication.

In the following text I will cover scenario when a Zimbra account is
compromised and prompt response is required to reset user's password and optionally disable Active Directory authentication and failed login lockout policy.
