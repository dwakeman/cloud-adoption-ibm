# Attach hosts

In order for your hosts to be available in your location you have to attach them by running the script that you downloaded when you created your location.  Before you can do that you first need to update your host by running the commands documented [here](https://cloud.ibm.com/docs/satellite?topic=satellite-host-reqs#reqs-host-system).

1. SSH into your virtual server instance.  If you have a mac, the command would look like this:

    `ssh root@<your host ip address>`
    
    Without the `<>`, of course.

    For example:

    `ssh root@163.75.64.199`

    !!! note
        You might see a prompt like the one below.  It's telling you that the host has returned a fingerprint that your machine does not recognize.  Type in `yes` at the prompt and SSH will add that fingerprint to the `known_hosts` file on your machine.

    ```
    The authenticity of host '163.75.64.199 (163.75.64.199)' can't be established.
    ECDSA key fingerprint is SHA256:pTS6heGnq2hBa4XAQzP1vWx4qFQFD20TgliZ1vc4u9E.
    Are you sure you want to continue connecting (yes/no/[fingerprint])?
    ```

1. Run this command:

    `subscription-manager refresh`

    The output should look like this:

    ```
    [root@sat-control-plane-tor01-01 ~]# subscription-manager refresh
    All local data refreshed
    ```

1. Run this command:

    `subscription-manager repos --enable=*`

    The output should look like this:

    ```
    [root@sat-control-plane-tor01-01 ~]# subscription-manager repos --enable=*
    Repository 'rhel-ha-for-rhel-7-server-eus-rpms' is enabled for this system.
    Repository 'rhel-server-rhscl-7-eus-rpms' is enabled for this system.
    Repository 'rhel-server-rhscl-7-rpms' is enabled for this system.
    Repository 'rhel-7-server-optional-rpms' is enabled for this system.
    Repository 'rhel-7-server-eus-optional-rpms' is enabled for this system.
    Repository 'rhel-7-server-eus-rpms' is enabled for this system.
    Repository 'rhel-7-server-devtools-rpms' is enabled for this system.
    Repository 'rhel-ha-for-rhel-7-server-rpms' is enabled for this system.
    Repository 'rhel-rs-for-rhel-7-server-eus-rpms' is enabled for this system.
    Repository 'rhel-rs-for-rhel-7-server-rpms' is enabled for this system.
    Repository 'rhel-7-server-rpms' is enabled for this system.
    Repository 'rhel-7-server-supplementary-rpms' is enabled for this system.
    Repository 'rhel-7-server-extras-rpms' is enabled for this system.
    Repository 'rhel-7-server-eus-supplementary-rpms' is enabled for this system.
    ```

1. Exit out of the SSH command by typing `exit`.  You are now back in your local terminal.  You need to secure copy the registration script that you downloaded when you created your location over to your host.  The command will look like this:

    `scp file.txt username@to_host:/remote/directory/`

    for example:

    `scp attachHost-Toronto.sh root@<your ip address>:/tmp`

    !!! note
        The above example assumes that the script `attachHost-Toronto.sh` is in the current directory from which you are running the command.  If it is in a different location you can specify a fully qualified path.  Also, the name of the script will be different, as the location name is part of the script name.

    For example:

    ```
    Samaritan:tmp dwakeman$ scp attachHost-Toronto.sh root@169.55.188.70:/tmp
    attachHost-Toronto.sh                                               100% 5435   108.8KB/s   00:00    
    ```

1. Now you can SSH back into your host and run the script, as documented in Step 6 [here](https://cloud.ibm.com/docs/satellite?topic=satellite-hosts#attach-hosts-console).

    `nohup bash /tmp/<your-script-name>.sh &`

    !!! note
        After you type the command and hit `Enter` it will show you a line about ignoring input.  Hit `Enter` again when you see that.

1. Repeat the steps above for your other two hosts.

At this point you should have 3 hosts attached to your location that can assign to your control plane in the next step.

!!! tip
    IBM Cloud Satellite treats all hosts alike; when you create them you can use them for anything that is supported by IBM Cloud Satellite.  In these steps we are using them for the control plane, but the exact same steps can be followed to create other hosts that can be used for services like OpenShift on IBM Cloud.