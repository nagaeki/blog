---
title: "Signing Git Commits with SSH"
date: 2026-06-29T14:00:44+09:00
description: "Proving who you are with contributions"
tags: [git, ssh]
featured_image: "featured_image.webp"
draft: false
hidden: false
---

Apparently Git has had the ability to sign commits with SSH keys for a while now. I did known this was possible with GPG keys, but this is a nice quality of life change, as SSH keys are much more common now.

> Git can sign with SSH keys after version 2.34. Older systems might not come with a version of Git that supports this.

# Why you would sign commits

Just like any other form of signing, you sign the commits to should you are who you claim to be.

Probably the form of digital signing Linux users are most familiar with is GPG signing used on package repositories. For example, the [nginx documents for installing on Debian](https://nginx.org/en/linux_packages.html#Debian) gives an example like this.

```
Import an official nginx signing key so apt could verify the packages authenticity. Fetch the key:

curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

Verify that the downloaded file contains the proper key:

gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg

The output should contain the full fingerprint 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 as follows:

pub   rsa2048 2011-08-19 [SC] [expires: 2027-05-24]
      573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
uid                      nginx signing key <signing-key@nginx.com>

Note that the output can contain other keys used to sign the packages.
```

In case where it is not obvious how this is important, this provides another layer of security to prove that the packages are actually released by the nginx team, on top of HTTPS security provided with the repository mirrors, as I do believe that it is still not possible to impersonate a GPG key.

This level of authentication is not exactly needed for every kind of Git repository, especially for those with smaller teams. However, it could come in handy with larger or more complex teams.

Signing Git commits provides another kind of authentication apart from the HTTP(S) authentication used when pushing. **Git allows you to set your user name and email to anything you like.** With HTTP(S) authentication, by default Git and the server only checks whether you have the ability to push, but it does not check whether you are pushing as who you claim you are.

However, with Git signing, apart from the HTTP(S) token, you still need to provide the private key needed for signing, and possibly the password to the key in the case the key is locked with a passphrase. (It really should be!)

This will be helpful with the following cases.

- Shared accounts. I don't really like them, but they are a reality. This makes sure you are commiting with the correct identity.
- Stolen HTTP token. Which is exactly why your key should be locked even on disk. This is the default for GPG but not for SSH.
- Higher levels of security, where signing is mandatory.

The first time that I was actually required to use Git signing was with [DN42](https://dn42.dev), which uses Git as its internet registry. You can see how it may be important to sign any commits, when it could result in another user's resources being taken in mistake. The same could be said for important repositories, where changing the code could break many more things.

# SSH keys over GPG

Same with Linux repositories, traditionally GPG keys has been utilized as the signing key with Git. GPG keys has many advantages over SSH keys, such as more detailed permissions, multiple sub-keys, as well as general encryption of data. It is also the older of the two.

However, as a single user or developer, GPG keys does not provide that much more when it comes to Git signing, as much of its advanced functions are not utilized. Also, most users probably already has got an SSH key for servers, and even they don't it is very simple to set up one.

Organizations and repositories where the functions of GPG are required will still use GPG keys. For me SSH keys are much easier to manage.

> Note that if you access Git over SSH, the signing key could be different from the access key, or it could be the same. The two are not necessarily connected. For example, when adding SSH keys on GitHub, it is required to specify whether the key is a 'signing key' or an 'authentication key'.

# How to set it up

Depending on the Git server being used, the steps could be a bit different, but the essentials should be the same.

## Generating an SSH key

This should be simple, as there are many guides online. Make sure to use the key type `ed25519`, as it is more modern and provides higher levels of protection at lower calculation costs (which makes SSH much faster! Try it!), if your system supports it.

> In theory, you should not be carrying one single SSH for all your machines. This makes revoking one particular key and auditing difficult, and presents security risks. What you should instead do it to make one key for each machine you will access from. If you're a system admin, then it should be one key per user per machine for all of your employees.

> Support for syncing SSH keys securely and even taking over the SSH agent, such as 1Password and Bitwarden offers, has made reusing keys a less worse offender. Discussion of this topic is way beyond this post, so will leave it there.

## Setting up the server

Setting up keys on GitHub simple as adding said key to your personal settigns page, as mentioned [here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account). Note that this should be the public key.

> You are not allowed to reuse SSH keys for multiple accounts on a Git server, as the server determines the account to verify as from the SSH key. The server can figure out what the public key is from nothing more than the signature itself, however having two accounts with the same key registered breaks this dependency, thus it is not allowed. The same can be said with the authentication keys.

> From the GitHub help pages, "Regardless of the signature choice - GPG, SSH, or S/MIME - once a commit signature is verified, it remains verified within its repository's network. ...When a commit signature is verified upon being pushed to GitHub, a verification record is stored alongside the commit. This record can't be edited and will persist so that signatures remain verified over time, even if signing keys are rotated, revoked, or if contributors leave the organization. The verification record includes a timestamp marking when the verification was completed. This persistent record ensures a consistent verified state, providing a stable history of contributions within the repository." This means that commits will still be considered signed, even after the user changes their email or registered keys.

## Configuring Git to use signing

Configure Git to use SSH for signing commits.

```
git config gpg.format ssh
```

To set the key used for signing, use this command. Although this commands sets up the public key, the actual used key is the private key. Therefore, you should have the private key together with the public key with the same name. (So`my_key` with `my_key.pub`.)

```
git config user.signingkey /PATH/TO/.SSH/KEY.PUB
```

At last, tell Git to sign commits, and tags if required.

```
git config commit.gpgsign true
git config tag.gpgsign true
```

> You can add the `--global` parameter to make changed global on your system.

# How to sign commits

When using the CLI version of Git, simply append the `-S` flag to git.

```
$ git commit -S -m "YOUR_COMMIT_MESSAGE"
# Creates a signed commit
```

With GUI IDEs such as VSCode, which I am using, simply committing as usual will automatically sign the commits. With both options, if your key has a passphrase to it, the application will ask for it.

> If you have committed already but has forgot to sign it, you can simply make a amending commit without any changes. The resulting commit will be signed. Don't do this on a remote repository if it is shared (unless you are very commited to doing it).

# References
- [GitHub Docs: About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)
- [GitHub Docs: Telling Git about your signing key](https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key)
- [GitHub Docs: Signing commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits)
- [GitHub Docs: Signing tags](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-tags)