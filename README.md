# Raspberry Pi OS Trixie: Fixing logout problem for xrdp Sessions to pishutdown 0.43

Date: 9 November 2026\
Platform: Raspberry Pi 4 Model B Rev 1.4\
Operating system: Raspberry Pi OS Trixie (Debian 13, aarch64)\
Remote desktop: xrdp, using an X11 session\
Note: I used AI to help troubleshooting, fixing and generating this document.

## 1. Background

After installing Raspberry Pi OS Trixie, the Windows Remote Desktop
connection worked, but the desktop's Logout, Reboot and Shutdown
functions did not work at all.

The installed version of pishutdown was 0.43. The original source
package was obtained using apt source pishutdown.

The original source was kept in \~/pishutdown-0.43, and all
modifications were made in a separate working copy:

``` bash
~/pishutdown-0.43-work
```

Two separate issues were identified:

-   Logout did not correctly terminate the xrdp X11 session.

-   Reboot and Shutdown were blocked by Polkit authorization.

## 2. Logout problem: incorrect session detection

The relevant code was in src/pishutdown.c, in the original pishutdown
0.43 source.

The original implementation attempted to detect whether the desktop was
running under Labwc:

``` bash
if (!system ("pgrep labwc > /dev/null"))
  system ("/usr/bin/labwc -e");
else
  system ("/usr/bin/pkill lxsession");
```

This detection was unreliable in the xrdp environment.

The LightDM greeter also runs a Labwc process. Consequently, pgrep labwc
could succeed even though the actual remote desktop session was an X11
session running LXsession and Openbox.

The program therefore attempted to terminate Labwc instead of correctly
closing the LXsession.

Running /usr/bin/lxsession-logout manually in the remote desktop session
worked correctly.

### Modification

The Logout handling was changed to use the session type reported by the
environment variable XDG_SESSION_TYPE.

``` bash
if (!strcmp (data, "exit"))
{
  system ("/usr/bin/pkill orca");
  if (g_strcmp0 (getenv ("XDG_SESSION_TYPE"), "x11") == 0)
    system ("/usr/bin/lxsession-logout");
  else
    system ("/usr/bin/labwc -e");
}
```

The resulting behaviour is:

-   X11 session: use lxsession-logout.

-   Other session types: use labwc -e.

### Essential source-code difference

The following illustrates the change in the Logout handling of pishutdown.c. This is the
essential logic change, not a complete diff of the source file.

``` bash
- if (!system ("pgrep labwc > /dev/null"))
-     system ("/usr/bin/labwc -e");
- else
-     system ("/usr/bin/pkill lxsession");
+ system ("/usr/bin/pkill orca");
+ if (g_strcmp0 (getenv ("XDG_SESSION_TYPE"), "x11") == 0)
+     system ("/usr/bin/lxsession-logout");
+ else
+     system ("/usr/bin/labwc -e");
```

## 3. Reboot and Shutdown problem: Polkit authorization

The Reboot and Shutdown commands in pishutdown.c are:

``` bash
if (!strcmp (data, "shutdown"))
  system ("/usr/bin/pkill orca;/sbin/shutdown -h now");
if (!strcmp (data, "reboot"))
  system ("/usr/bin/pkill orca;/sbin/reboot");
```

These commands were not modified. The problem was not in the commands
themselves, but in the authorization policy applied to remote sessions.

### 3.1 Inspecting the Polkit authorization settings

The Polkit action settings were inspected using:

``` bash
pkaction --verbose --action-id org.freedesktop.login1.reboot
```

The relevant output was:

``` bash
implicit any: auth_admin_keep
implicit inactive: auth_admin_keep
implicit active: yes
```

The same inspection (with same results) was also performed for:

``` bash
org.freedesktop.login1.reboot-multiple-sessions
```

### 3.2 What are the login1 actions?

login1 refers to the systemd-logind service, which manages login
sessions and system power operations. The relevant Polkit action IDs
are:

| **Action ID** | **Purpose** |
| --- | --- |
| org.freedesktop.login1.reboot | Reboot the system |
| org.freedesktop.login1.reboot-multiple-sessions | Reboot when  multiple sessions exist |
| org.freedesktop.login1.power-off | Power off the system |
| org.freedesktop.login1.power-off-multiple-sessions | Power off when multiple sessions exist |

These are separate authorization actions. The rule added later covers
all four.

### 3.3 Inspecting the xrdp session properties

The session properties were obtained from a terminal inside the xrdp
desktop session using:

``` bash
loginctl show-session "$XDG_SESSION_ID" \
-p Name -p Type -p Class -p Active -p Remote -p Seat
```

The relevant output was:

``` bash
Name=raspi
Seat=
Remote=yes
Type=x11
Class=user
Active=yes
```

The important property is:

``` bash
Remote=yes
```

The xrdp session was classified as a remote and active session.

### 3.4 Why the authorization failed

The important distinction is that the implicit active authorization
applies to active local sessions.

The xrdp session is remote, so it uses the **implicit any** policy rather
than the local-session implicit active policy.

-   implicit active: yes allows the operation without further
    authentication for an active local session.

-   implicit any: auth_admin_keep requires administrator authentication
    for other sessions, including this remote session.

Since pishutdown invokes the commands through non-interactive system()
calls, it cannot complete the Polkit authentication challenge.

### 3.5 Verifying the authorization challenge

The authorization check was performed from the xrdp terminal using:

``` bash
pkcheck --action-id org.freedesktop.login1.reboot --process $$
```

Before the Polkit rule was added, the result was:

``` bash
polkit\56retains_authorization_after_challenge=1
Authorization requires authentication and -u wasn't passed.
```

The command returned exit code 2.

This confirmed that Polkit required authentication and that the
non-interactive process could not provide it.

The resulting error when attempting Reboot through pishutdown was:

``` bash
Call to Reboot failed: Interactive authentication required.
```

Thus, Reboot and Shutdown could not work from the remote desktop before
the authorization policy was corrected.

### 3.6 Polkit rule added

A new rule was created:

``` bash
/etc/polkit-1/rules.d/50-raspi-power.rules
```

Contents:

``` bash
polkit.addRule(function(action, subject) {
  if (subject.user == "raspi" &&
  (
    action.id == "org.freedesktop.login1.reboot" ||
    action.id == "org.freedesktop.login1.reboot-multiple-sessions" ||
    action.id == "org.freedesktop.login1.power-off" ||
    action.id == "org.freedesktop.login1.power-off-multiple-sessions"
  )) 
  {
      return polkit.Result.YES;
  }
});
```

This grants the user raspi permission to reboot and power off the
system, including when multiple sessions exist.

The rule is deliberately restricted to this one user and these four
specific actions. It does not change the authorization policy for other
users or grant permission to ignore shutdown inhibitors.

After installing the rule, the same pkcheck test returned exit code 0 in
both SSH and xrdp sessions.

## 4. Building and installing the modified program

Build dependencies were installed with:

``` bash
sudo apt build-dep pishutdown
```

The program was built in the working directory:

``` bash
cd ~/pishutdown-0.43-work
meson setup build
meson configure build --prefix=/usr
meson compile -C build
```

The build completed successfully.

The original installed binary was backed up:

``` bash
sudo cp -a /usr/bin/pishutdown /usr/bin/pishutdown.orig
```

The modified binary was installed:

``` bash
sudo install -o root -g root -m 755 \
~/pishutdown-0.43-work/build/src/pishutdown \
/usr/bin/pishutdown
```

The modified program was then tested from the normal Raspberry Pi
desktop menu.

## 5. Test results

### pishutdown interface

The following functions were tested using the installed, modified
pishutdown program:

| Function | Result |
| --- | --- |
| Logout | Successful -- the xrdp session closes |
| Reboot | Successful -- the Pi restarts and the remote connection can be re-established |
| Shutdown | Successful -- the Pi shuts down and the remote connection becomes unavailable until the Pi is powered on again |

### Separate Raspberry Pi desktop Logout interface

There is another interface that should not be confused with pishutdown.

When selecting Shutdown/Logout from the Raspberry Pi desktop menu, a
separate window appears:

``` bash
Logout rpd-x 13 session
```

This window offers its own Shutdown, Reboot and Logout options.

Only its Logout function was tested. It worked correctly after the
Logout fix.

The Reboot and Shutdown options in this separate window were not tested.
They are separate desktop-session functions and do not use the modified
pishutdown program.

This is an intentional limitation of the testing scope. No conclusions
are drawn here about those two untested options.

## 6. Rollback instructions

To restore the original pishutdown executable:

``` bash
sudo cp -a /usr/bin/pishutdown.orig /usr/bin/pishutdown
```

To remove the additional Polkit rule:

``` bash
sudo rm /etc/polkit-1/rules.d/50-raspi-power.rules
```

Restoring the original executable also restores the original Logout
behaviour, including its problem with the xrdp X11 session.

## 7. Final notes

The solution consists of two independent changes:

1.  A source-code modification to make Logout work correctly in both X11
    and non-X11 sessions.

2.  A narrowly scoped Polkit rule allowing the user raspi to reboot and
    shut down the system from a remote session.

The Reboot and Shutdown commands in the original pishutdown source were
left unchanged. Their failure was caused by the authorization policy,
not by incorrect command syntax.

The modified executable was installed manually. A future package upgrade
of pishutdown may overwrite it, in which case the source modification
would need to be rebuilt and reinstalled.

Final result: The pishutdown 0.43 interface now supports Logout, Reboot
and Shutdown correctly from the Windows Remote Desktop connection to the
Raspberry Pi OS Trixie system.
