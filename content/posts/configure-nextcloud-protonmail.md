---
title: "Configure Nextcloud with Protonmail SMTP"
date: 2023-08-19T00:00:00-04:00
author: Ze Xuan Ong
draft: false
feature_image: /configure-nextcloud-protonmail/nextcloud.png
tags: ["dev"]
summary: "Configure Nextcloud to use Protonmail SMTP"
---

It is quite straightforward to configure any SMTP mail server with Nextcloud. To do so with Protonmail requires a bit more work.

For a start, we have to use the official Protonmail Bridge service which acts as an SMTP server we can use for Protonmail.

We will use this unofficial [shenxn/protonmail-bridge](https://hub.docker.com/r/shenxn/protonmail-bridge) docker image, it seems to be the only one available for Unraid.

## Point docker ports

The docker container setup is just the default options in the Unraid menu, however I forwarded the container port 1025 to something on the host e.g. 10555. This is because Protonmail Bridge only listens on 127.0.0.1:1025 (for SMTP traffic), so any traffic being passed to the container from external sources will need to look like it is coming internally.

## Setting up the account

After installing the container on Unraid, you will want to first setup your login credentials. 

1. Enter a console session for the protonmail-bridge container
2. Run `top`, and you will see a `bridge` process, a `protonmail...` process, and two `socat` processes. Kill all of them with `k`.
3. Run `protonmail-bridge --cli` to start the bridge prompt. You should be able to enter things into the prompt.
4. Enter `login` to start the login process to your account. If you have multiple domain names on your account it does not matter, it will all be associated with the same account.
5. Once that is done, you can enter `info` to verify that the login credentials are correct. Copy the password displayed, you will need it for the configuration on Nextcloud.
6. Exit the console and restart the container.

{{< figure src="/configure-nextcloud-protonmail/top.png" caption="Default `top` processes in `protonmail-bridge`` container, some to kill." >}}

If you run into errors such as "port in use", it is because you didn't kill the `socat` processes.

There are other guides that suggest `chmod u+x entrypoint.sh`, that also works, but it is less clean. This is because the `entrypoint.sh` script also sets up the two `socat` commands.

## Configure Nextcloud

Add the following section to yout `config.php` in your Nextcloud config. This is because the certificate for the bridge is self-signed, and Nextcloud will reject it by default.

```
'mail_smtpstreamoptions' => 
  array (
    'ssl' => 
    array (
      'allow_self_signed' => true,
      'verify_peer' => false,
      'verify_peer_name' => false,
    ),
  ),
```

Credits here for this suggestion: https://github.com/nextcloud/server/issues/37329#issuecomment-1488147847

Restart your Nextcloud container for good measure, and then setup the SMTP configuration. You should use the username and password you saw in the `info` command print out in the protonmail-bridge container. If you have named domains, you can set up the appropriate send-as email in this section.

## Final notes

That should work now! You can send yourself a test email, and it will send it to the email configured for the account.

If that doesn't work, you can set the following to log in debug mode, so you might get a better handle on what is not working.

```
'loglevel' => 0,
'mail_smtpdebug' => true,
```

In this case, you can also do poor man's debugging using telnet to make sure that your host and port (SMTP server) in the protonmail-bridge container is reachable.

