# System Documentation

## Linux Documentation

### SSH Configuration
It is best to use an SSH key for authentication rather than a password. To configure an SSH key for a user, log in as that user through SSH or console/terminal of the system.

Once logged in issue the command `ssh-keygen -t rsa -b 4096`. This sets type as RSA and the size to 4096. Default I believe is 2048. Then press enter through all of the options. You can set a password on the file if you'd like as well for an extra level of security. Let it create the ssh key in the default directory of $HOME/.ssh.

After completing these steps navigate to the SSH directory by typing `cd .ssh`. Once in this directory output the contents of the id_rsa file by typing `cat id_rsa`. Copy the information from this file into a local text file on your PC. Save it with no extension and set appropriate permissions. For Linux it's best to set permissions to 600 and on Windows as long as you put it in your Users\'Username' directory, permissions should be set correctly. If using PuTTy in Windows for connection you can use the PuTTy-Gen to create a PPK file from the original key.

Once you've copied the information out of this file you should delete it using the command `rm id_rsa`. Next remove the authorized_keys file if one is already present. Lastly, rename the id_rsa.pub file to authorized_keys using the command `mv id_rsa.pub authorized_keys`. Then set the permissons on this file by running the command `chmod 600 authorized_keys`.

You should now be able to SSH into this machine using the the generated key rather than the user's password.

The final step is to disable password authentication through SSH. To disable this open the SSH configuration file for editing `sudo nano /etc/ssh/sshd_config`. Update the following lines to the following:

```
{
    UsePAM no
    PasswordAuthentication no
}
```

After restarting the SSH service these new settings should be applied. To restart run `sudo service ssh restart`. If you try to connect to the system now remotely without the SSH key it should fail immediately.

### Creating a new user with super user privileges
This is a short walkthrough for how to create a new user in Linux with super user privileges.

1. First run this command to create the user `sudo adduser -Username-`. Finish this out by setting your password, setting your name, and pressing enter throughout the rest of the prompts.

2. Next add the new user to the super user group for sudo access by running the command `sudo usermod -aG sudo -Username-`.

## Windows Documentation

More coming