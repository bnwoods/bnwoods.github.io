---
layout: post
title:  'Chef Automate 2.0 SAML Setup with Azure AD'
tags: [ Chef Automate ]
excerpt: 'A Tutorial for Setting Up SAML in A2 with Azure AD'
---

#### Creating the App in Azure

- Navigate to your Azure Portal and create a new enterprise application for Chef Automate

- Within the new app configuration, under the Single Sign-On option, fill out the following information:
    - ##### Basic SAML Configuration
        - **Identifier (Entity ID)**: https://your-automate-server.com/dex/callback
        - **Reply URL (Assertion Consumer Service URL)**: https://your-automate-server.com/dex/callback
        - **Sign On URL**: *leave blank*
        - **Relay State**: *leave blank*
        - **Logout URL**: *leave blank*
    - ##### User Attributes & Claims
        - **surname**: *optional*
        - **name**: *optional*
        - **username**: user.mail
        - **emailaddress**: user.mail
        - **givenname**: *optional*
        - **Unique User Identifier**: *optional*
    - ##### SAML Signing Certificate
        - **Status**: *This should be set to active*
        - Download the base64 certificate for use in the Chef Automate Server config

#### Configuring SAML Within the Chef Automate Server

- ssh to your Chef Automate server

- Create a file called saml-config.toml with the following content:

<pre>
 [dex.v1.sys.connectors.saml]
  ca_contents="""-----BEGIN CERTIFICATE-----
  THIS
  IS
  THE
  BASE64
  CERT
  FROM
  AZURE AD
  -----END CERTIFICATE-----
  """
  sso_url = "{LOGON URL FROM AZURE AD}"
  email_attr = "emailaddress"
  username_attr = "username"
  entity_issuer = "https://your-automate-server.com/dex/callback"
</pre>

- Run `sudo chef-automate config patch saml-config.toml` to apply the settings