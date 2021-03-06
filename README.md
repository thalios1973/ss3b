# ss3b - Simple S3 Backup

Author: Mike Gauthier &lt;thalios1973 at 3cx dot org&gt;

## VERSION HISTORY

* 0.1 First working version. Had bugs.
* 0.2 Some bug fixes and feature changes, released but never deployed.
* 0.3 First "stable" release. It does what it's supposed to do, but more to come when time permits.
* 0.4 Added --warning=no-file-changed to tarcmd so it will not error out when backing up a file that was changed while reading. Better to back up a changed file than no backup at all. Placed the software under the MIT License.

ss3b is a very simple backup scheme making use of tar, openssl passphrase-based encryption, and Amazon's S3. The use of encryption and compression (gzip) is optional and off by default (configuration file and command-line options can enable this). The use of encryption is HIGHLY recommended (considering you'll likely be backing up your system's password/shadow files -- as one example). ss3b is intended to run as root, so protect your configuration files and this script appropriately.

## WHY BASH?

Why did I write this in bash when I could have likely made things easier on myself by using Perl/PAWS or Python or something. Well... it seems a lot of Linux administrators know bash, but not so many know more than that. These are usually more junior admins, but they're also the ones that often get "stuck" with the backup system in smaller shops (and sometimes not-so-small shops). So, I figured I'd write it in something that **everyone** should be able to read, take apart, change, updated, etc.

## INSTALLATION

* Copy ss3b.conf-example to your desired config directory (recommend /etc/ss3b as that is the default location ss3b will look). Please protect this file appropriately as it can contain your AWS credentials and optional encryption password/passphrase.

* Create a file containing the absolute paths you wish to be backed up. This file can contain commented lines and blank lines. Keep it simple as the sanity checking in the script isn't very robust (recommnd this file be placed in the same directory as the config file). This file is quite simple. An example file might look like the following.

 /etc  
 /home/someuser  
 /datastore/www/webpages  
 /var/log/http  

* Decide on a cache-dir location and ensure it exists and is writeable by root. (recommend /var/local/ss3b)

## S3 SET UP

Pretty simple. Ensure you have a bucket available for this script to back up to. As a minimum, the AWS user used by this script will need the following permission (use the Bucket Policy Editor).

	{
		"Version": "2012-10-17",
		"Id": "POLICYID",
		"Statement": [
			{
				"Sid": "Stmtxxxxxxxxxxxxx",
				"Effect": "Allow",
				"Principal": {
					"AWS": "arn:aws:iam::XXXXXXXXXXXX:user/USERNAME"
				},
				"Action": [
					"s3:PutObject",
					"s3:PutObjectAcl",
					"s3:PutObjectVersionAcl"
				],
				"Resource": "arn:aws:s3:::BUCKET_NAME/*"
			}
		]
	}

All systems that use this script can write to the same bucket. Each system backs up to a unique folder in this bucket.

Recommendation: Configure your bucket's lifcycle. It is recommended that after a certain age, back up files should be moved to glacier or deleted. If you do not configure this, you could find yourself over time with an unexpected AWS bill.

## CONFIG FILE

Reviewing the example config file should get you going pretty quickly. `ss3b -h` has some help as well.

## SYSTEM INITIALIZATION

Once you have your config file set, you'll need to initialize the system (this applies to each new system you deploy ss3b to). This can be done using the `-I` option.

Initialization simply sets up the cache directory and makes a unique hostname. The hostname (and the folder you'll find in the bucket for this host) is simply the existing hostname with 8 random alphanumeric characters tacked onto the end. This is done JUST IN CASE there are two systems with the same name in your environment (e.g. `webhost01.subdomainX.domain.com` and `webhost01.subdomainY.domain.com`). It ensure truly unique references for each system. The FQDN should ensure this isn't needed, but this doesn't seem to always be the case.

## CONFIGURING DAILY RUNS

This script has no way to run or schedule itself on its own. This is left up to the administrator. Typically running ss3b on a scheduled basis is accomplished via cron (or anacron). ss3b is designed to run daily. It will perform a full backup on the day defined in the config file (fullday variable) and a differential all other days. If ss3b for some reason is not run on the day for a full back up, it will continue to perform diffs until a full is run. A full can beforce with the `-F` flag. The first time ss3b is run after initialization, a full back up is run by default regardless of the day.

If everything is set up accordingly, one should simply be able to run ss3b with no flags out of cron (assuming your config file is `/etc/ss3b/ss3b.conf`).

## THE SETLIST

After the first backup is run, you'll see a `&lt;hostname&gt;.SETLIST` file has been created in the cachedir. This file simply contains a list of the backup sets, full or diff type run, and the timestamp of the run. This file is used by the system to know what it's doing but can also be used by you to restore old setsThis file simply contains a list of the backup sets, full or diff type run, and the timestamp of the run. This file is used by the system to know what it's doing but can also be used by you to restore old sets.

## HELP

	$ ./ss3b -h
	
	Usage: ss3b [options]

	All options (except debug) needed should be set from the config file. These
	options are provided so overriding the config is possible (mostly for
	testing). Debug output can only be enable with the -d option.
	
	AWS id, AWS secret key, and the encryption passphrase can only be set in
	the config file. This means you must ensure the config file has safe
	permissions.
	
	Options:
	 -h                  Show usage.
	 -c <config file>    Defaults to /etc/ss3b/ss3b.conf if not provided. This
	                     file must exist if config_file is not provided
	                     as an argument.
	 -p <path file>      File containing list of paths to back up.
	                      config setting: pathsfile=<list file>
	 -z                  Do compression (via gzip). Default = no
                      config setting: compress=yes
	 -e                  Do encryption (via gnupg). Default = no
	                      config setting: encrypt=yes
	 -b <bucket name>    S3 bucket name
	                      config setting: bucket=<bucket name>
	 -r <region>         AWS region
	                      config setting: region=<region>
	 -d                  Enable debug output
	                      config setting: _DEBUG=yes
	 -I                  Initiate this host. Usually only run the first
	                     time a host is set up.
	 -F                  Force a full backup, regardless of what is set
	                     in the config file. ss3b will pick up from
	                     there and continue to run the full/diff cylce
	                     as configured.

## RESTORING FILES

The backup files are simply incremental tar files. Restoring from these should be done per the GNU tar instructions. Downloading and staging of the backup files is up to the administrator. However, if the files are encrypted, they will need to be decrypted first. The following command will accomplish this.

	$ openssl enc -aes-256-cbc -d -in BACKUPFILE.tgz.enc -out BACKUPFILE.tgz

It will prompt you for the password/passphrase you used during encryption (passphrase variable in config file). One could easily stream the file through openssl using pipes and redirects (like most other Unix CLI tools). In fact, one could do a full restore by streaming the download from the aws cli tool, through openssl to decrypt, and through tar to expand the archive on disk.

