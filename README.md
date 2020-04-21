The Smol-Bang
============

A fork of [Sovereign](https://github.com/sovereign/sovereign) Ansible playbooks used to maintain our personal infrastructure.

Requirements
------------

An Arch Linux box somewhere with ssh access enabled.

Installation
------------

## On the remote server

The following steps are done on the remote server by `ssh`ing into it and running these commands.

### 1. Install required packages

    sudo pacman -Syu

### 3. Prep the server

For goodness sake, change the root password:

    passwd

Create a user account for Ansible to do its thing through:

    useradd deploy
    passwd deploy
    mkdir /home/deploy

Authorize your ssh key if you want passwordless ssh login (optional):

    mkdir /home/deploy/.ssh
    chmod 700 /home/deploy/.ssh
    nano /home/deploy/.ssh/authorized_keys
    chmod 400 /home/deploy/.ssh/authorized_keys
    chown deploy:deploy /home/deploy -R
    echo 'deploy ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/deploy

Your new account will be automatically set up for passwordless `sudo`. Or you can just add your `deploy` user to the sudo group.

## On your local machine

Ansible (the tool setting up your server) runs locally on your computer and sends commands to the remote server.

    git clone https://github.com/anthony2025/smolbang.git

### 4. Configure your installation

Modify the settings in the `group_vars/smolbang` folder to your liking. If you want to see how they’re used in context, just search for the corresponding string.

For Git hosting, copy your public key into place:

  cp ~/.ssh/id_rsa.pub roles/git/files/gitolite.pub

Finally, replace the `host.example.net` in the file `hosts`. If your SSH daemon listens on a non-standard port, add a colon and the port number after the IP address. In that case you also need to add your custom port to the task `Set firewall rules for web traffic and SSH` in the file `roles/common/tasks/ufw.yml`.

### 5. Set up DNS

Create `A` or `CNAME` records which point to your server's IP address:

* `www.example.com` (for web hosting)
* `git.example.com` (for git hosting)

### 6. Run the Ansible Playbooks

To run the whole dang thing:

    ansible-playbook -i ./hosts --ask-sudo-pass site.yml

If you chose to make a passwordless sudo deploy user, you can omit the `--ask-sudo-pass` argument.

To run just one or more piece, use tags. I try to tag all my includes for easy isolated development. For example, to focus in on your firewall setup:

    ansible-playbook -i ./hosts --tags=ufw site.yml

You might find that it fails at one point or another. This is probably because something needs to be done manually, usually because there’s no good way of automating it. Fortunately, all the tasks are clearly named so you should be able to find out where it stopped. I’ve tried to add comments where manual intervention is necessary.

The `dependencies` tag just installs dependencies, performing no other operations. The tasks associated with the `dependencies` tag do not rely on the user-provided settings that live in `variables`. Running the playbook with the `dependencies` tag is particularly convenient for working with Docker images.

### 8. Miscellaneous Configuration

Sign in to the ZNC web interface and set things up to your liking. It isn’t exposed through the firewall, so you must first set up an SSH tunnel:

  ssh deploy@example.com -L 6643:localhost:6643

Then proceed to http://localhost:6643 in your web browser.

Similarly, to access the server monitoring page, use another SSH tunnel:

    ssh deploy@example.com -L 2812:localhost:2812

Again proceeding to http://localhost:2812 in your web browser.

### Reboots

You will need to manually enter the password for any encrypted volumes on reboot. This is not Sovereign-specific, but rather a function of how EncFS works. This will necessitate SSHing into your machine after reboot, or accessing it via a console interface if one is available to you. Once you're in, run this:

    encfs /encrypted /decrypted --public

It is possible that some daemons may need to be restarted after you enter your password for the encrypted volume(s). Some services may stall out while looking for resources that will only be available once the `/decrypted` volume is available and visible to daemon user accounts.
