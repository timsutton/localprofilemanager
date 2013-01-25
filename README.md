# Local Profile Manager

This is a yet-to-be-completed tool designed manage the installation state of OS X Configuration Profiles. The goal is for it to run as a LaunchDaemon, watch a 'profiles' directory for changes, and maintain that all these profiles are installed.

This tool would function independently of any software management system or other installed profiles (performed manually, at deployment time, etc.).

There would also be helper functionality to easily generate and import profile 'items' for use with Munki as simple copy_from_dmg, which can be used as `managed_installs` and `managed_uninstalls` to perform profile management by simply managing files on disk, without the need for templated install/uninstall scripts.

## Why?

There are many reasons to not use Profile Manager for OS X client management, and the installation/removal of Profiles is possible with the built-in 'profiles' command-line tool. We can currently package a profile and include a postinstall script to install it, and provide an uninstall script to a system like Munki to remove it later. These extra steps 

## How it works

When run, it will go through every .mobileconfig file found at `/Library/Local Profile Manager/profiles/computer` to build a list of candidate profiles. It also dumps the current profile 'store' by invoking `profiles -P -o /a/tmp/plist`.

Profiles are as unique as their top-level `PayloadIdentifier`. OS X seems to use this mechanism internally to distinguish profiles. For example, double-clicking to install a profile will consider it an "upgrade" if a profile with the same identifier already exists.

Profiles that exist in the 'profiles' directory that haven't yet been installed are installed, profiles that were installed by LPM that are no longer there are removed.

For a profile that exists both in the store and in the 'profiles' directory, it copies its data from the store profile into a new custom `ConfigProfile` object, containing one or more custom `ConfigPayload` objects with data populated from the individual payloads. It does the same for each candidate .mobileconfig file. The data structure of a profile from the 'store' is slightly different from that of a .mobileconfig file, as well as the key names used, so this is why these custom classes exist, and have have two methods for importing all their data attributes: one from the store by an identifier name, one from a .mobileconfig file.

These custom classes implement comparison operators, which actually iterate through and compare every significant key value (uuid, names, and the payloads themselves) in order to determine if two profiles are equal or not. If it determines that they aren't, the candidate is installed.

Every profile installed with LPM first adds a custom suffix to the profile identifier, `.LocalProfileManager`, so that it will know which profiles it has installed, and so that it will not remove any profiles which may have been installed by other means. This has implications for signed profiles, see below.

## Design issues

### WatchPaths

A LaunchDaemon with defined WatchPaths is a simple implementation. WatchPaths seems to be triggered according to a change of a file's inode in this directory. This means that if you are testing by moving files on your local disk, simply overwriting a file will not trigger a job load (at least in my testing). Adding or removing a file _will_ trigger a change, but _not_ an overwrite when it comes from the same volume. The target deployment environment for managing profiles with LPM would usually involve copying profiles (which in Munki's case would live in a .dmg) from a Munki/Casper/Puppet/whatever repository pulled down via HTTP or network server mount, which _should_ always trigger a job load because it will create a new file inode on disk. There are fancier ways to watch a folder for changes ([watchdog](http://pypi.python.org/pypi/watchdog) implements [FSEvents](https://developer.apple.com/library/mac/#documentation/Darwin/Conceptual/FSEvents_ProgGuide/Introduction/Introduction.html), for example), but this is by far the simplest. FSEvents is possible with PyObjC, but implementing CFRunLoops introduces a great deal of complexity, and this is supposed to be a simple tool.

### Signed profiles

LPM needs to somehow know which profiles it has already installed, so that it will know whether it can remove one that is no longer in the 'profiles' directory. The current implementation involves LPM appending a '.LocalProfileManager' suffix to all profile identifiers that it is installing. Because this also requires modifying a .mobileconfig file on disk before one is to be installed, this means that a signed profile cannot be installed if it does not contain this '.LocalProfileManager' suffix, because manipulating the binary file would break the signature. There is currently some basic code to extract the XML portion of a signed binary profile, so it should in theory still be possible to use this information to compare signed profiles.

### User-level profiles

With locally-installed profiles, one can either install a profile at the level of the system or the user. The only way I've found to install a user-level profile is to actually invoke the 'profiles' command _as the user_, and therefore this user must already exist. There are some ways that LPM could have the added functionality of installing a user-level profile, but to handle the above case perhaps something like a LaunchAgent would be required that will invoke LPM in a different mode, that goes through user-named folders inside the `/Library/Local Profile Manager/profiles/users` folder, similar to how the structure of `/Library/Managed Client` looks when MCX management is being applied to a host. Invoking profiles at login time solves the problem that `WatchPaths` can't watch a directory recursively. However, it also means that certain profile management settings (dock, login items, network mounts, etc.) may well not apply for a user the first time that user logs in.

## Not even 'alpha' yet

This code was largely written in August/September, 2012 and it has barely been revisited since at the time of writing (January 2013). There is much testing, bug-fixing and design improvements to still be done.

I'd also like to prepare a list of test cases with sample profiles so that it's possible to test the matrix of different combinations, installations/removals/updates and profile "types" available in Profile Manager, to ensure that changes are actually taking effect when they are supposed to.
