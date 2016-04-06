Post-Install Hack for Setting Up EcryptFS with user created by Debian-Installer on Single-User Laptop or Desktop
===============================================================================

This script exists because I think EcryptFS and DM-Crypt/LUKS probably both
have strengths and weaknesses, and because I'd like to explore them both
interacting at the same time on the same Debian-based Desktop system. But, it
turns out it's actually a little bit tricky to go all the way from a
non-encrypted $HOME directory to an encrypted $HOME directory, even with
ecryptfs-migrate-user. This is because none of the user being migrated's
files can be in use at the time of migration, so if you're running as yourself
and run "sudo ecryptfs-migrate-user -u $(whoami)" then you'll end up getting
an error and the process will be aborted. The only way to work around this
(Unless you have set up your system to allow login as root, I prefer not to
allow login as root directly) is to set up another user, add it to the sudoers
list, logout, log back in as that user, and then run "sudo ecryptfs-migrate-user -u $(\ls /home)",
log back out when it's done, log back in immediately as $(\ls /home), aka
your default username, now encrypted, delete the temporary user and associated
groups, encrypt the swap partition, delete the temporary folder generated by
ecryptfs-migrate-home, and be done. Which isn't so much difficult as sort
of tedious and when dealing with programs specifically designed to make data
unreadable, mistakes are costly. So this script essentially reduces it to just
the logout/login steps and takes care of the rest for you.

These are the actual commands that it runs in the default configuration. Options
may be added depending on how I feel.

        sudo useradd -k /etc/pefsh/skel -d /home/.ecryptor ecryptor
        PASSWORD=$(apg -n 1)
        echo "The password for the temporary user(ecryptor) will be $PASSWORD. Please write this
        down. This password will only be used for the required duration of encrypting
        your primary user's Home Directory, then both the temporary user and the
        password will be deleted."
        sudo usermod -p $PASSWORD ecryptor
        sudo usermod -a -G sudo ecryptor
        echo "if [ -d /home/.ecryptor]; then
                /usr/bin/pefsh_cleanup > $HOME/pefsh_cleanup_report.log
        fi
        ">>$HOME/.profile
        echo "The temporary user(ecryptor) has been created. Now you will be logged out. You
        must log back in as ecryptor to complete the procedure. You will be logged out in 60
        seconds, or you may log out manually."
        sleep 60
        sudo pkill -u $(\ls /home)

Then you log in as ecryptor with the password you generated, and ecryptor has

        x-terminal-emulator -c "sudo ecryptfs-migrate-user -u $(\ls /home) && sleep 60 && sudo pkill -u ecryptor" || sudo ecryptfs-migrate-user -u $(\ls /home) && sleep 60 && logout

at the end of it's .bashrc, automatically encrypting your files. If you have trouble
logging into the desktop as the new user, you can always CTRL+ALT+F1 from the login
screen to get a terminal-only session. Per the waring that the script gives the user
ample time to read before logging them out automatically, the user should immediately
log back in as the default user. If that user logs in successfully, pefsh_cleanup will
run the following commands

        
        if [ -d /home/.ecryptfs ]; then
                sudo deluser ecryptor
                sudo srm -rf ~/home/.ecryptor	
                sudo ecryptfs-setup-swap
                REMOVE_ME="if [ -d /home/.ecryptor]; then 
        /usr/bin/pefsh_cleanup
        fi"
        REMOVED_ME="
        "
                if [ ! -d /home/.ecryptor ]; then
                        sed -i s|$REMOVE_ME|$REMOVED_ME| $HOME/.profile
                fi
        fi
        

Lastly, you should ecryptfs-unwrap-passphrase(on your own) and save the password somewhere
very safe.

