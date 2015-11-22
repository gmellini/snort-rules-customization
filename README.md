# Snort rules advanced parser for pulledpork
This script was developed to be used in conjunction with pulledpork script (https://github.com/shirkdog/pulledpork) to make some adavanced modification, based on regular expression, to the SNORT downloaded.rules file (the file rules created by pulledpork after downloading and processing rules).

I use the script in Security Onion (https://security-onion-solutions.github.io/security-onion/) executing it inside /usr/bin/rule-update script after the pulledpork rules download/processing and before the restart of Security Onion IDS sensors.

I wrote the script mainly because I have a lot of false positives in SNORT alarms (mainly due to the Emergin Threats signatures and the fact that I'm monitoring a "large" network while I have few services exposed) and to avoid this I needed some more "precise" rule that triggers alarm only if a service is really alive.
I found that the pulledpork disablesid.conf, dropsid.conf etc files are not enough for my needs.

## Install
Create the taget dir and extract the tar.gz file inside 

```shell
$ mkdir /opt/snort-scripts/
$ cd /opt/snort-scripts/
$ tar zxvf build_advanced_rules.tgz
```

## Script help
The only mandatory argument is the pulledpork.conf file path (some needed configuration comes from here).

```shell
$ ./build_advanced_rules.pl -h
  Usage: ./build_advanced_rules.pl [-tVv?h] [-d <downloaded.rules file path>] -c <pulledpork.conf file path>
  
   Options:
   -? Print this help.
   -h Print this help.
   -c Where the pulledpork.conf config file lives. This is mandatory.
   -d Where the downloaded.rules file lives. If not set, read in pulledpork.conf file (rule_path option).
   -t TEST MODE. Write all the result files in /tmp/ directory without modify any file.
   -V Print version and exit.
   -v Verbose mode for troubleshooting.

   Note: The script saves the new local rules in a file named local.advanced.rules
         In this file a comment (#### at the beginning of the rule) is added to the original rules (the one in downloaded.rules file)
          if the rule match one of the regular expressions.
         The local.advanced.rules full path shall be set in pulledpork.conf (local_rules var) so new defined rules are loaded.
         If not set, local.advanced.rules file is saved in the same dir of downloaded.rules file.

   Note: In test mode (-t) the downloaded.rules.modified and local.advanced.rules temporary files can be found in /tmp/ directory.
         The original downloaded.rules and sid-msg.map files are not modified.

$
```

## Test it and how it works
To try the script without apply any modification to the real SNORT files use the test mode (-t flag).

I created a test/ folder and a shell script (test.it) that execute the program in test mode (all the needed files are inside the test dir). The test.it script execute following command: ./customize-advanced_rules.pl -c ./pulledpork.conf -d ./downloaded.rules -t

```shell
$ ./test.it 
Running command: [/opt/snort-scripts/test/../customize-advanced_rules.pl -c /opt/snort-scripts/test/pulledpork.conf -d /opt/snort-scripts/test/downloaded.rules -t]
----------------------------------------------------------------
=================
=== TEST MODE ===
=================

Please note that the generated downloaded.rules.modified and local.advanced.rules files are written to /tmp/ directory.
The original downloaded.rules and sid-msg.map files are not modified.

INFO: Pulled Pork file: /opt/snort-scripts/test/pulledpork.conf
INFO: SID-MSG map file version found: 1
INFO: Downloaded Rules file: /opt/snort-scripts/test/downloaded.rules
INFO: Advanced Rules file written: /tmp/local.advanced.rules
INFO: Full downloaded.rules file written: /tmp/downloaded.rules.modified
INFO: sid-msg.map file for _NEW_ Rules only written: /tmp/sid-msg.map
INFO: Script ended
----------------------------------------------------------------
Done
$
```

The script in this case do the following:
- read the test pulledpork.conf file (-c option) (find the sid-msg map version)
- get the dowloaded.rules path from -d option (in real execution read in pulledpork.conf file)
- write the local.advanced.rules in /tmp. Here you can find the new rules based on the regular expressions defined inside the script
- write the new downloaded.rules.modified in /tmp. This is the modified version of dowloaded.rules file generated by pulledpork, all the matching rules processed by the script are commented. You can check the differences with diff.
- write the sid-msg.map in /tmp. Here you can find the map only for the new created rules

I suggest to test the output before apply any new rule.

## Basic Configuration
After testing is time to review results and write some custom rule.

The file build_advanced_rules.pl comes with some already defined rule, defined in the %Rules_hash hash.
Each key of the hash define a rule, and each rule is composed as follow:
- 'regexp' key: the regexp to match against each rule inside the original dowloaded.rules file
- one or more 'transform' key that is composed by 2 other keys
-- 'from': the path to match inside the rule
-- 'to': the substitution to apply to the matched path

Follow an example:

```perl
$Rules_hash{0}{'regexp'} = 'tcp \[[0-9.,/]*\] any \-> \$HOME_NET any.* threshold:.* classtype:misc-attack;.* sid:(2[0-9]{6});';
$Rules_hash{0}{'transform'}{0}{'from'} = '\$HOME_NET any ';
$Rules_hash{0}{'transform'}{0}{'to'} = '[192.168.0.1] [443,5555] ';
$Rules_hash{0}{'transform'}{1}{'from'} = '\(msg:"';
$Rules_hash{0}{'transform'}{1}{'to'} = '(msg:"NEW RULE - ';
```

This rule match the SNORT rules against the 'regexp' field, and for the matching rules substitute:
- '$HOME_NET any ' with '[192.168.0.1] [443,5555] ' ==> instead HOME_NET we match specific destinatio IP and ports
- '(msg:"' with '(msg:"NEW RULE - ' ==> rename the original rule prepending "NEW RULE" word

As an example follow the new rule created for the matching "ET TOR Known Tor Relay/Router (Not Exit) Node TCP Traffic group 654" that match the 'regexp' key (find the new rule in /tmp/local.advanced.rules):

```text
alert tcp [98.242.102.183,98.245.213.97,98.26.112.238,98.28.166.24,99.226.196.67,99.229.118.205,99.229.138.198,99.234.43.160,99.239.184.79,99.52.176.162] any -> [192.168.0.1] [443,5555] (msg:"NEW RULE - ET TOR Known Tor Relay/Router (Not Exit) Node TCP Traffic group 654"; flags:S; reference:url,doc.emergingthreats.net/bin/view/Main/TorRules; threshold: type limit, track by_src, seconds 60, count 1; classtype:misc-attack; flowbits:set,ET.TorIP; sid:902523306; rev:2404;)
```

The new downloaded.rules file (/tmp/downloaded.rules.modified) has following line:

```text
####alert tcp [98.242.102.183,98.245.213.97,98.26.112.238,98.28.166.24,99.226.196.67,99.229.118.205,99.229.138.198,99.234.43.160,99.239.184.79,99.52.176.162] any -> $HOME_NET any (msg:"ET TOR Known Tor Relay/Router (Not Exit) Node TCP Traffic group 654"; flags:S; reference:url,doc.emergingthreats.net/bin/view/Main/TorRules; threshold: type limit, track by_src, seconds 60, count 1; classtype:misc-attack; flowbits:set,ET.TorIP; sid:2523306; rev:2404;)
```

As you can see the original rule is commented (with 4 dash, so we can easly find).
The new rule have a new sid, based on the original one.
- original: 2523306
- new one: 902523306

The new SID is generated as follow:
- a number is prepended to the rule ($Rule_prepend=9)
- the index of Rules_hash is added (in this case 0, $Rules_hash{0})
- the old SID is added, adding leading 0 up to 8 digits if needed

Last the script create the /tmp/sid-msg.map file with our new rule inside

```text
902523306 || NEW RULE - ET TOR Known Tor Relay/Router (Not Exit) Node TCP Traffic group 654 || url,doc.emergingthreats.net/bin/view/Main/TorRules
```

Now we are ready to write custom rules.

## Write Custom Rules
The file comes with 2 examples that rewrite the ET rules (sid=2xxxxxx) both for TCP and UDP.

```perl
# index 0
# outside TCP services
$Rules_hash{0}{'regexp'} = 'tcp \[[0-9.,/]*\] any \-> \$HOME_NET any.* threshold:.* classtype:misc-attack;.* sid:(2[0-9]{6});';
$Rules_hash{0}{'transform'}{0}{'from'} = '\$HOME_NET any ';
$Rules_hash{0}{'transform'}{0}{'to'} = '[192.168.0.1] [443,5555] ';
$Rules_hash{0}{'transform'}{1}{'from'} = '\(msg:"';
$Rules_hash{0}{'transform'}{1}{'to'} = '(msg:"NEW RULE - ';
# index 1
# outside UDP service
$Rules_hash{1}{'regexp'} = 'udp \[[0-9.,/]*\] any \-> \$HOME_NET any.* threshold:.* classtype:misc-attack;.* sid:(2[0-9]{6});';
$Rules_hash{1}{'transform'}{0}{'from'} = '\$HOME_NET any ';
$Rules_hash{1}{'transform'}{0}{'to'} = '[192.168.0.2] [12345] ';
$Rules_hash{1}{'transform'}{1}{'from'} = '\(msg:"';
$Rules_hash{1}{'transform'}{1}{'to'} = '(msg:"NEW RULE - ';
```

You can modify, add, delete rules, just remember to increment the index of the hash while adding new rules and test it before.

## Integrate with Secuity Onion
To integrate the script in Security Onion 

1. modify /etc/nsm/pulledpork/pulledpork.conf file and add the new generated file to the local_rules

```text
local_rules=/etc/nsm/rules/local.rules,/opt/snort-scripts/local.advanced.rules
```

2. modify the /usr/bin/rule-update script and add at the end of the file, just before the row "# Restart NIDS processes", the following lines

```shell
#########################################
# Generate Customized Advanced Rules
#########################################
su - $PULLEDPORK_USER -c "/opt/snort-scripts/customize-advanced_rules.pl -c /etc/nsm/pulledpork/pulledpork.conf"
```

## Contacts
You can contact me at giovanni [dot] mellini [at] gmail [dot] com
My web site: https://scubarda.wordpress.com