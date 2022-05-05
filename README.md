# gpg-settings

Based on [this blogpost](https://withblue.ink/2020/05/17/how-and-why-to-sign-git-commits.html).

To [sign Git commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key), you need a gpg key. GPG stands for GNU Privacy Guard and is the de facto implementation of the [OpenPGP message format](https://www.ietf.org/rfc/rfc4880.txt). PGP stands for ‘Pretty Good Privacy’ and is a standard to sign and encrypt messages.

## Setting up

Install with Homebrew:

```bash
$ brew install gpg
```

Create config files for `gpg` and the `gpg-agent`. The agent will make sure you don’t have to type in your GPG passphrase for every commit.

```bash
$ mkdir ~/.gnupg
$ touch ~/.gnupg/gpg.conf ~/.gnupg/gpg-agent.conf
```

Open the `gpg.conf` file and add:

```bash
use-agent
```

In `gpg-agent.conf`, add the following lines to make sure your credentials are ‘kept alive’ ([source](https://superuser.com/questions/624343/keep-gnupg-credentials-cached-for-entire-user-session)):

```bash
default-cache-ttl 34560000
max-cache-ttl 34560000
```

Optionally, you can install a GUI for entering your passphrase. You don’t need to, but the default is a CLI program and might not provide a nice user experience. With `pinentry-mac` you can choose to save your passphrase in your MacOS keychain. That’s up to your personal preference.

```bash
$ brew install pinentry-mac
```

If you installed `pinentry-mac`, make sure to configure the agent. Open the `gpg-agent.conf` file and add this line:

```bash
pinentry-program /opt/homebrew/bin/pinentry-mac
```

**Note**: if you’re on Intel, `/opt/homebrew` should be `/usr/local`.


Add the following lines `~/.zshrc` (the `GPG_TTY` environment variable is [a requirement for GPG](https://www.gnupg.org/documentation/manuals/gnupg/Invoking-GPG_002dAGENT.html#Invoking-GPG_002dAGENT); the second line launches the `gpg-agent` when you open a new shell):

```bash
export GPG_TTY=$(tty)
gpgconf --launch gpg-agent
``` 

To effectuate the changes to `.zshrc`, type:

```bash
$ source ~/.zshrc
```

## Create GPG keypair

Now that your environment is properly set up, we need to generate a public/private GPG keypair.

```bash
$ gpg --full-gen-key
```

A wizard is printed to your terminal. You should configure as follows:

- Kind of key: `4` (RSA, sign only)
- Keysize: `4096`
- Expiration: `2y` (your key will expire after 2 years; you should set a reminder somewhere)
- Real name: `<your github username>`
- Email address: `<your email address>`

**Note**: I heartily recommend setting your email address to your 'noreply' GitHub address: `username@users.noreply.github.com`. You can find your email address on the [GitHub Email settings page](https://github.com/settings/emails). Note that if you created a GitHub account after July 2017, your address will also have an ID prefixed to your username; [read more here](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-github-user-account/managing-email-preferences/setting-your-commit-email-address).

The final step in setting up the GPG keypair is typing a passphrase. Make sure it is strong and you have it safely stored in your password vault (I recommend [Bitwarden](https://bitwarden.com/)). Whoever has your passphrase can sign your commits and there is no way to prove it wasn’t you.

After creating the keypair, output similar to the following is printed to your terminal:

```bash
pub   rsa4096 2021-11-12 [SC] [expires: 2023-11-12]
      AAABBBCCCDDDEEEFFF1112223334445556667778
uid                      username <username@users.noreply.github.com>
```

The string of characters is your key ID. To confirm you can sign messages with your newly created key, enter in your terminal:

```bash
$ echo 'it works' | gpg --clearsign
```

A message similar to this should appear:

```bash
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA256

it works
-----BEGIN PGP SIGNATURE-----
<many characters>
-----END PGP SIGNATURE-----
```

## Adding to Git

We need to add your key to your git config, and to GitHub. First, you need to find the key ID. The (short) ID uses the last 8 characters of the key that was printed to the terminal before. You can retrieve it:

```bash
$ gpg --list-secret-keys --keyid-format SHORT
```

Outputs:

```bash
/Users/username/.gnupg/pubring.kbx
----------------------------------
sec   rsa4096/56667778 2021-11-12 [SC] [expires: 2023-11-12]
      AAABBBCCCDDDEEEFFF1112223334445556667778
uid         [ultimate] username <username@users.noreply.github.com>
```

The `56667778` bit after `rsa4096/` is your short key ID. We need it to configure Git to sign commits and tags. Replace the `user.signingkey` value below with your own key ID:

```bash
$ git config --global user.signingkey 56667778
$ git config --global commit.gpgSign true
$ git config --global tag.gpgSign true
```

Git needs to know your email, and it needs to be the same as the one for your GPG key. This email address needs to be verified on GitHub as well. If you use your ‘private’ GitHub email, that’s already the case.

```bash
$ git config --global user.email username@users.noreply.github.com
```

Finally, you need to add your public GPG key to GitHub. Again, make sure to replace the ID with your own ID:

```bash
$ gpg --armor --export 56667778
```

Outputs: 

```bash
-----BEGIN PGP PUBLIC KEY BLOCK-----

<many characters>
-----END PGP PUBLIC KEY BLOCK-----
```

You need to copy the whole block and add it to GitHub. If you’re not sure what to copy, use this command:

```
$ gpg --armor --export 56667778 | pbcopy
```

The `| pbcopy` part will pipe the output of the first part directly to your copy-paste memory. 

Go to the [GitHub SSH and GPG keys section](https://github.com/settings/keys), click [New GPG key] and paste into the box. Click [Add GPG key], and you’re done!

After getting this done, and after having made your first signed commit, you can see the ‘Verified’ badge on GitHub for that commit ([see an example here](https://github.com/phortuin/gist-ssg/commit/5bd42616d2395f5511faa84cf02be82619d3c161)). Your GPG key ID will be shown when the badge is clicked. 

## Visual Studio Code

If you use Visual Studio Code, you can turn on signing by changing a setting.

Open VSCode, go to Preferences > Settings, and search for `git.enableCommitSigning`. Turn this setting on, and you’re good to go.

## Troubleshooting

### 1.
If for some reason you can’t sign, simply kill the agent. [It will restart when needed](https://superuser.com/a/1150399):

```bash
$ gpgconf --kill gpg-agent
```

### 2.
On older MacOS versions or certain (remote) shells, you might encounter the error `inappropriate ioctl for device`. (This error might also turn up if you haven’t configured the `GPG_TTY` environment variable correctly, see above for instructions.) [More context here](https://d.sb/2016/11/gpg-inappropriate-ioctl-for-device-errors). You can fix this by using the so called ‘loopback’ option to enter your passphrase directly on the CLI.

Edit `gpg.conf` and add:

```bash
pinentry-mode loopback
```

Edit `gpg-agent.conf` and add:

```bash
allow-loopback-pinentry
```

Now, when the agent wants your passphrase it will simply render a basic password input on the CLI:

```
$ echo 'it works' | gpg --clearsign
Enter passphrase:
```

### 3.
If you use SourceTree, you should point it to the right binary. A solution is posted on [Stack Overflow](https://stackoverflow.com/a/27069408/554821), make sure to also follow [this comment](https://stackoverflow.com/questions/26697343/why-is-the-gnupg-sign-checkbox-disabled-in-sourcetree#comment86720717_27069408).
