---
layout: post
title:  "Session Riding OpenSSH (multiplexing) to bypass 2FA"
date:   2021-06-06 20:28:22 -0400
categories: web2
---

![image](/assets/img/paradise.jpeg)

##### Quick Links 
[What is SSH session injection](#what-is-ssh-session-injection)  
[Brief overview of SSH session creation code](#a-brief-view-of-connection--session-creation-in-openssh)  
[Good hackers read documentation](#reading-documentation-will-make-you-a-better-hacker-)  
[Abusing SSH multiplexing](#abusing-ssh-multiplexing-)  
[Detection](#detection)  
[EOF](#eof)  


# What is SSH session injection?

Mostly personal notes after revisiting the subject recently while reminiscing on past hacking techniques.  

In the past SSH session injection allowed an attacker to inject code into a target users ssh client process to open a new session to an already connected remote server.  

Unknown to the legitimate user the remote server was compromised silently while they were connected in another session without the attacker modifying the sshd binary on disk.  
 
Previous public work:

1) [Trust Transience: Post Intrustion SSH Hijacking](https://www.blackhat.com/presentations/bh-usa-05/bh-us-05-boileau.pdf).  

The technique was officially patched in OpenSSH 5.1 in 2008:  
[OpenSSH 5.1p1 Patch](https://github.com/openssh/openssh-portable/commit/8901fa9c88d52ac1f099e7a3ce5bd75089e7e731#diff-6e5958092d48b108bef3faadd24f2259a7e999ba8771cb64c986179c059fe130)


# A brief view of connection & session creation in OpenSSH:  

A high level overview of the functions and files involved with creating a connection/session/channel are provided.  

Starting with the server side files and functions:  

### packet.h  

The relevent part of the ssh struct we are interested in is the dispatch function table.  

```c
struct ssh {
        ...
	/* Dispatcher table */
	dispatch_fn *dispatch[DISPATCH_MAX];
        ...
};
```

### sshd.c

The main server loop handles the client connections initial authentication and creates the `ssh` and `authctxt` objects in memory that are passed between the different session related functions.  

```c
    int main(){
    ...
    // XXX - main server loop that handles individual connections 
    ...
    // XXX - called after all authentication checks have passed. 
    do_authenticated(ssh, authctxt);
    ...
}
```

### session.c  

After a connection passes all authentication checks in `sshd.c` its data is passed through the `do_authenticated(...)` function to the `do_authenticated2(...)` function in `session.c` before being passed to the `server_loop2(...)` function in the `serverloop.c` file.  

Nothing very important happens in the `session.c` file for our context.  

```c
do_authenticated(struct ssh *ssh, Authctxt *authctxt)
{
  ...
    do_authenticated2(ssh, authctxt);
  ...
}

do_authenticated2(struct ssh *ssh, Authctxt *authctxt)
{
    server_loop2(ssh, authctxt);
}
```

### serverloop.c  

The `server_loop2(...)` function calls a dispatch function `server_init_dispatch(...)` that sets certain handler functions depending on the `ssh` object dispatch command passed to it.    

If the `SSH2_MSG_GLOBAL_REQUEST` ssh dispatch command is received. This function calls `server_input_global_request(...)` This command is sent by the ssh client when initiating a connection/session with a server.  

The `server_input_global_request(...)` function checks if the client sent the `no-more-sessions@openssh.com` string and sets the `no_more_sessions` to `1` disabling additional sessions on the current connection channel.     

```c
// [2] XXX - global variable to disable further channels/sessions if set
/* Disallow further sessions. */
static int no_more_sessions = 0;

server_loop2(struct ssh *ssh, Authctxt *authctxt)
{
        ...
	debug("Entering interactive session for SSH2.");
        ...
	server_init_dispatch(ssh);
        ...
	for (;;) {
        ...
        }
}

server_init_dispatch(struct ssh *ssh)
{
        ...
	debug("server_init_dispatch");
	ssh_dispatch_set(ssh, SSH2_MSG_GLOBAL_REQUEST, &server_input_global_request);
        ssh_dispatch_set(ssh, SSH2_MSG_CHANNEL_OPEN, &server_input_channel_open);
        ...
}

server_input_global_request(int type, u_int32_t seq, struct ssh *ssh)
{
 ...
 else if (strcmp(rtype, "no-more-sessions@openssh.com") == 0) {
        // [3] XXX - if the client sends the "no-more-sessions@openssh.com" dispatch cmd to the server the no_more_sessions is set to "1".
        no_more_sessions = 1;
        success = 1;
 }
 ...
}
```
Likewise if the `SSH2_MSG_CHANNEL_OPEN` dispatch command is received. The `server_input_channel_open(...)` function is called.  

This function eventually calls `server_request_session(...)` that checks if `no_more_sessions` is `> 0` and exits if yes.   

```c
server_input_channel_open(int type, u_int32_t seq, struct ssh *ssh)
{
        Channel *c = NULL;
        char *ctype = NULL;
        ...
        if (strcmp(ctype, "session") == 0) {
	        // XXX - A "session" is requested from ssh_session2_open()
		// XXX - the following function will check no_more_sessions and fail/return
                c = server_request_session(ssh);
        }
        ...
}

server_request_session(struct ssh *ssh)
{
...
        Channel *c;
        int r;

        debug("input_session_request");
        if ((r = sshpkt_get_end(ssh)) != 0)
                sshpkt_fatal(ssh, r, "%s: parse packet", __func__);

	// [3] XXX - checks no_more_sessions is 1 and disconnects / returns 
        if (no_more_sessions) {
                ssh_packet_disconnect(ssh, "Possible attack: attempt to open a "
                    "session after additional sessions disabled");
        }

        // [4] XXX - open session if all checks passed
        c = channel_new(ssh, "session", SSH_CHANNEL_LARVAL,
            -1, -1, -1, /*window size*/0, CHAN_SES_PACKET_DEFAULT,
            0, "server-session", 1);
        if (session_open(the_authctxt, c->self) != 1) {
                debug("session open failed, free channel %d", c->self);
                channel_free(ssh, c);
                return NULL;
        }
        channel_register_cleanup(ssh, c->self, session_close_by_channel, 0);
        return c;
}
...
```

### ssh.c  
Having looked at the server related code above we are familiar with the specific `*ssh` dispatch commands.  

Next checking the client code that initiates a connection and what `*ssh` dispatch commands are sent & when:  

The `ssh_session2(...)` function is called from the `main(...)` function in `ssh.c` when initiating a connection with a server.  

```c
static int
ssh_session2(struct ssh *ssh, const struct ssh_conn_info *cinfo)
{
...
        /* If we don't expect to open a new session, then disallow it */
        if (options.control_master == SSHCTL_MASTER_NO &&
            (ssh->compat & SSH_NEW_OPENSSH)) {
		// [1] XXX - the CLIENT sends no-more-sessions@openssh.com string to the server after the call to ssh_session2_open() has finished
                debug("Requesting no-more-sessions@openssh.com");
                if ((r = sshpkt_start(ssh, SSH2_MSG_GLOBAL_REQUEST)) != 0 ||
                    (r = sshpkt_put_cstring(ssh,
                    "no-more-sessions@openssh.com")) != 0 ||
                    (r = sshpkt_put_u8(ssh, 0)) != 0 ||
                    (r = sshpkt_send(ssh)) != 0)
                        fatal_fr(r, "send packet");
        }
...
}
```  

At the very start of the clients connection if the `ControlMaster` option is not set then the ssh packet starts with the `SSH2_MSG_GLOBAL_REQUEST` dispatch command 
and the `no-more-sessions@openssh.com` string disabling further sessions from being opened on that connection/channel.  


# TLDR :

1) The SSH client checks if `ControlMaster` option is set via commandline or config. 

2) Upon initial connection to a remote server the SSH client sends `no-more-sessions@openssh.com` string to the server if no `ControlMaster` option is set.  

3) When receiving connections the SSHD server checks the initial connecting clients per-connection options. If the server receives the `no-more-sessions@openssh.com` string at this time then it sets `no_more_session = 1` server side preventing further sessions from being opened on that connection/channel by remote clients.  

4) If a client tries to open a new session on an existing connection at this point the server will disconnect the connecting client as the `no_more_sessions` is set to `1` for this connection.  


The OpenSSH documentation is clear about all of this behavior (but who reads docs): 

```
2.2. connection: disallow additional sessions extension
     "no-more-sessions@openssh.com"

Most SSH connections will only ever request a single session, but a
attacker may abuse a running ssh client to surreptitiously open
additional sessions under their control. OpenSSH provides a global
request "no-more-sessions@openssh.com" to mitigate this attack.

When an OpenSSH client expects that it will never open another session
(i.e. it has been started with connection multiplexing disabled), it
will send the following global request:

        byte            SSH_MSG_GLOBAL_REQUEST
        string          "no-more-sessions@openssh.com"
        char            want-reply

On receipt of such a message, an OpenSSH server will refuse to open
future channels of type "session" and instead immediately abort the
connection.

Note that this is not a general defence against compromised clients
(that is impossible), but it thwarts a simple attack.

NB. due to certain broken SSH implementations aborting upon receipt
of this message, the no-more-sessions request is only sent to OpenSSH
servers (identified by banner). Other SSH implementations may be
listed to receive this message upon request.

...

Client implementations SHOULD reject any session channel open
requests to make it more difficult for a corrupt server to attack the
client.
```

# Reading documentation will make you a better hacker :  

A valuable source of information that generally far surpasses most tutorials found online.  

The SSH documentation tells about three options, that when provided will enable multiplexing for SSH connections.   

From the [ssh_config manpage](https://man7.org/linux/man-pages/man5/ssh_config.5.html) :  

`ControlMaster`:   
Enables the sharing of multiple sessions over a single network connection.  
When set to yes, ssh(1) will listen for connections on a control socket specified using the ControlPath argument.  

<Mark>Additional sessions can connect to this socket using the same ControlPath with ControlMaster set to no (the default).</Mark>  

These sessions will try to reuse the master instance's network connection rather than initiating new ones,
but will fall back to connecting normally if the control socket does not exist, or is not listening.  

Setting this to ask will cause ssh(1) to listen for control connections, but require confirmation using ssh-askpass(1).  

If the ControlPath cannot be opened, ssh(1) will continue without connecting to a master instance.

X11 and ssh-agent(1) forwarding is supported over these multiplexed connections, however the display and agent forwarded 
will be the one belonging to the master connection i.e. it is not possible to forward multiple displays or agents.  

Two additional options allow for opportunistic multiplexing: try to use a master connection but fall back to creating a new one if one does not already exist. 
These options are: auto and autoask. The latter requires confirmation like the ask option.  


`ControlPath`:  
<Mark>Specify the path to the control socket used for connection sharing as described in the ControlMaster section above or the string none to disable connection sharing.</Mark>  

Arguments to ControlPath may use the tilde syntax to refer to a user's home directory, the tokens described in the TOKENS section and environment variables as described in the ENVIRONMENT VARIABLES section.  

It is recommended that any ControlPath used for opportunistic connection sharing include at least %h, %p, and %r (or alternatively %C) and be placed in a directory that is not writable by other users.  
This ensures that shared connections are uniquely identified.  

`ControlPersist`:  
<Mark>When used in conjunction with ControlMaster, specifies that the master connection should remain open in the background (waiting for future client connections) after the initial client connection has been closed.</Mark>   

If set to no (the default), then the master connection will not be placed into the background, and will close as soon as the initial client connection is closed.  

If set to yes or 0, then the master connection will remain in the background indefinitely (until killed or closed via a mechanism such as the "ssh -O exit"). 
If set to a time in seconds, or a time in any of the formats documented in sshd_config(5), then the backgrounded master connection will automatically terminate after it has remained idle (with no client connections) 
for the specified time.  

Lastly multiplexing is not available in the Windows version of OpenSSH as described [here](https://github.com/PowerShell/Win32-OpenSSH/wiki/Project-Scope)  

# Abusing SSH multiplexing :  

Placing a config file in the target users `~/.ssh/` directory is enough to enable basic lateral movement and bypass popular two-factor auth setups (ex: Google authenticator or DUO) while the admins SSH sessions are active and connected:  

```bash
ControlMaster auto
ControlPath ~/.ssh/control:%h:%p:%r
```

### Terminal 1 (victim)  

We test authentication first and verify that key-auth is required. Next we login successfully.  

![image](/assets/img/ss2.png)  


If the victim user disconnects from the remote host while the attacker is still active the victims session will not exit cleanly :  

![image](/assets/img/ss3.png)  


If the victim user force closes the session (ctrl-c) at this point the attackers session will be ended :  

![image](/assets/img/ss4.png)  

### Terminal 2 (attacker)  

The attacker checks the .ssh/config file in use and locates the victims ssh session.  
The attacker then successfully authenticates to the remote SSH server without providing the ssh-key for authentication using the multiplexed connection.  

Note: `-T` to prevent OpenSSH from spawning a terminal which keeps logging to a minimal. `-v` adds verbosity to let the user know the connection information.  

![image](/assets/img/ss6.png)  

If the session is force closed by the victim user the session is closed and the user is dropped back to their own console:  

![image](/assets/img/ss5.png)  

The primitive can be much more powerful by adding the `ControlPersist` option and setting it to `yes`.  

As we read above this option will background the multiplexed connection so its still active after the initial session is disconnected by the admin. `!!!`  

This allows the admin to disconnect cleanly and leave the multiplexed connection open in the background for the attacker to connect with.  

Editing the config file to:  
```
ControlMaster auto
ControlPath ~/.ssh/control:%h:%p:%r
ControlPersist yes
```

Is enough to enable silently back-grounding connections for future sessions. 

Now if a victim user connects to a host and exits the session the attacker can act more leisurely to gain access to the same remote hosts.    

This includes hosts protected by common 2FA setups such as Duo Security.  

In the below tests I've setup `Google-Authenticator` as a MFA precaution to demonstrate bypassing it with multiplexing.  

### Victim 

You can see in the first victim screen shot below it prompts for the `Verification code` then the user exits the session completely.  

![image](/assets/img/ss7.png)  

### Attacker  

The attacker locates the multiplexed socket file in use then connects to the host without providing the authentication key or the authenticator verification code:  

![image](/assets/img/ss8.png)  

# Detection  

Not really my thing. 

If using the ControlPersist option the initial connection stays alive and is visible in active connections using `netstat` : 
```
tcp        0      0 x.x.x.x:50356     54.152.139.0:22         ESTABLISHED 12832/ssh: /home/x/
```

`ps aux` usually shows the socket file in use also : 
```
x        12832  0.0  0.0  12244  2688 ?        Ss   09:50   0:00 ssh: /home/x/.ssh/control:54.152.139.0:22:ubuntu [mux]
```

Check your `.ssh` directory once in awhile to verify the config hasn't been tampered with or any ssh keys added to authorized_keys file.  

# EOF

Metlstorm suggested that RDP may be vulnerable to similar in the past in his talk. 🤷‍♂️  

Audit tip: Check SSH implementations when auditing to see if they disallow further sessions after initial session creation.  

Exercise for readers: Other forms of session riding also exist.  

DUO Security MFA implements a web based panel that allows admin users to create bypass-codes for other users having trouble authenticating.  

This can be abused if an attacker gains access to an admins machine (RDP) similar to the above attack. The attacker installs a hidden browser plugin to keep the session alive on the admin panel using invisible clicks. Then when the admin is away the attacker is able to stealthily create bypass-codes using the admins session thats been kept alive.  

[Chrome Keep Session Alive Extension](https://chrome.google.com/webstore/detail/session-alive/aodfoiacnmepndhojccepdcpmehjbham)
