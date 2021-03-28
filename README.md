# `wayback-machine-spn-scripts` (full title pending)

`spn.sh`, a Bash script that asks the Internet Archive Wayback Machine's [Save Page Now (SPN)](https://web.archive.org/save/) to save live web pages

## Introduction

### Features

* Cross-platform compatible (Windows 10 WSL, macOS, Linux, BSD)
* Run capture jobs in parallel
* Automatic retrying and error handling
* Recursively and selectively save outlinks
* Resume an aborted session
* Optional API authentication
* Extensible script structure

### Motivation

There exist several alternatives to this script, including the Python program [savepagenow by pastpages](https://github.com/pastpages/savepagenow) and the [wayback-gsheets](https://archive.org/services/wayback-gsheets/) interface on archive.org. However, in terms of functionality, this script has some advantages over other methods of sending captures to the Wayback Machine.

* As of March 2021, Save Page Now often has an error rate of around 20% to 30% regardless of the content being saved and the rate of operation. [This is a known issue](https://old.reddit.com/r/WaybackMachine/comments/m139pt/ive_got_an_amazing_response_from_the_wayback/) but has not been fixed for some time. The script will automatically retry those captures.
* In comparison to the outlinks function native to Save Page Now, the script allows outlinks to be recursively captured, and allows for inclusion/exclusion of groups of URLs. For example, the user can prevent a website's login form from being repeatedly captured. Additionally, the website's outlinks function can sometimes overwhelm smaller websites, since all of the outlink captures are queued at the same time, whereas the script allows the user to prevent this by limiting the number of parallel capture jobs.
* The script's structure is relatively extensible. For example, with minor modifications, any data returned by the Save Page Now JSON responses can be processed and output to a new log file and/or reused as input in another function. (The script uses POST requests rather than non-authenticated GET requests to submit URLs to Save Page Now; older scripts that use the latter method return much less data for completed captures and cannot set [capture request options](https://docs.google.com/document/d/1Nsv52MvSjbLb2PCpHlat0gkzw0EvtSgpKHu4mk0MnrA/edit#heading=h.uu61fictja6r).)

## spn.sh

### Installation

Download the script and make it executable with the command `chmod u+x spn.sh`.

### Dependencies

This script is written in Bash and has been tested using the shell environment preinstalled binaries on macOS and Windows 10 WSL Ubuntu. As far as possible, utilities have been used in ways in which behavior is consistent for both their GNU and BSD implementations. (The use of the `sed -E` flag in particular may be a problem for older versions of GNU `sed`, but otherwise there should be no major compatibility issues.)

### Operation

The only required input (unless resuming a previous session) is the first argument, which can be a file name or URL. If the file doesn't exist or if there's more than one argument, then the input is treated as a set of URLs. A sub-subfolder of `~/spn-data` is created when the script starts, and logs and data are stored in the new folder. Some information is also sent to the standard output. All dates and times are in UTC.

The main list of URLs is stored in memory. Every time the script reaches the end of the main list, and approximately once every hour (if the list has more than 50 URLs), URLs for failed capture jobs and outlinks are added to the list. When there are no more URLs, the script terminates.

The script may sometimes not output to the console or to the log files for an extended period. This can occur if Save Page Now introduces a delay for captures of a specific domain, though typically the delay is only around a few minutes at most. [If you're on Windows, make sure it isn't just PowerShell.](https://serverfault.com/a/205898)

#### Flags

```
Usage: spn.sh [-nqs] [-a auth] [-d data] [-o pattern] [-x pattern] [-p num] file
       spn.sh [-nqs] [-a auth] [-d data] [-o pattern] [-x pattern] [-p num] url [url]...
       spn.sh [-nqs] [-a auth] [-d data] [-o pattern] [-x pattern] [-p num] -r folder

 -a auth        S3 API keys, in the form accesskey:secret
                (get account keys at https://archive.org/account/s3.php)
 -d data        capture request options, or other arbitrary POST data
 -n             tell Save Page Now not to save errors into the Wayback Machine
 -o pattern     save detected capture outlinks matching regex (ERE) pattern
 -p N           run at most N capture jobs in parallel (off by default)
 -q             discard JSON for completed jobs instead of writing to log file
 -r folder      resume with the remaining URLs of an aborted session
                (aborted session's settings do not carry over)
 -s             use HTTPS for all captures and change HTTP input URLs to HTTPS
 -x pattern     save detected capture outlinks not matching regex (ERE) pattern
                (if -o is also used, outlinks are filtered using both regexes)
```

All flags should be placed before arguments.

* The `-a` flag allows the user to log in to an archive.org account with [S3 API keys](https://archive.org/account/s3.php). The keys should be provided in the form `accesskey:secret` (e.g. `-a YT2VJkcJV7ZuhA9h:7HeAKDvqN7ggrC3N`). If this flag is used, some login-only options can be enabled with the `-d` flag, in particular `capture_outlinks=1` and `capture_screenshot=1`. Additionally, much less data is downloaded when submitting URLs, and the captures will count towards the archive.org account's daily limit instead of that of the user's IP.
* The `-d` flag allows the use of Save Page Now's [capture request options](https://docs.google.com/document/d/1Nsv52MvSjbLb2PCpHlat0gkzw0EvtSgpKHu4mk0MnrA/edit#heading=h.uu61fictja6r), which should be formatted as POST data (e.g. `-d 'force_get=1&if_not_archived_within=86400'`). Documentation for the available options is available in the linked Google Drive document. Some options, in particular `capture_outlinks=1` and `capture_screenshot=1`, require authentication through the `-a` flag. The options are set for all submitted URLs. By default, the script sets the option `capture_all=on`; the `-n` flag disables it, but it can also be disabled by including `capture_all=0` in the `-d` flag's argument. Note that as of March 2021, the `outlinks_availability=1` option does not appear to work as described, and other parts of the documentation may be out of date.
* The `-o` and `-x` flags both enable recursive saving of outlinks. To save as many outlinks as possible, use `-o ''`. The argument for each flag should be a POSIX ERE regular expression pattern. If only the `-o` flag is used, then all links matching the provided pattern are saved; if only the `-x` flag is used, then all links _not_ matching the provided pattern are saved; and if both are used, then all links matching the `-o` pattern but not the `-x` pattern are saved. Around every hour, matching outlinks in the JSON received from Save Page Now are added to the list of URLs to be submitted to Save Page Now. If an outlink has already been captured in a previous job, it will not be added to the list. A maximum of 100 outlinks per capture can be sent by Save Page Now, and the maximum number of provided outlinks to certain sites (e.g. outlinks matching `example.com`) may be limited server-side. The `-o` and `-x` flags are separate to the server-side `capture_outlinks=1` option, and will not work if that option is enabled through use of the `-a` and `-d` flags.
* The `-p` flag sets the maximum number of parallel capture jobs, which can be between 2 and 60. If the flag is not used, capture jobs are not queued simultaneously. (Each submitted URL is queued for a period of time before the URL is actually crawled.) The Save Page Now rate limit will prevent a user from starting another capture job if the user has too many concurrently active jobs (not including queued jobs), so setting the value higher may not always be more efficient. Be careful with using this on websites that may be easily overloaded.
* The `-n` flag unsets the HTTP POST option `capture_all=on`. This tells Save Page Now not to save error pages to the Wayback Machine. Because the option is set by default when using the web form, the script does the same.
* The `-q` flag tells the script not to write JSON for successful captures to the disk. This can save disk space if you don't intend to use the JSON for anything.
* The `-s` flag forces the use of HTTPS for all captures. Input URLs and outlinks are automatically changed to HTTPS. At present, this flag does not work with FTP URLs.
* The `-r` flag allows the script to resume with the remaining URLs of a prior aborted session; the argument should be the location of a folder. If this flag is used, the script does not take any arguments. It is necessary for `index.txt` and `success.log` to exist in the folder in order for the session to be resumed; if `outlinks.txt` also exists, then the links in that file and the links in `captures.log` will also be accounted for.

#### Data files

The `.txt` files in the data folder of the running script may be modified manually to affect the script's operation, excluding old versions of those files which have been renamed to have a Unix timestamp in the title.

* `failed.txt` is used to compile the URLs that could not be saved, excluding files larger than 2 GB and URLs returning HTTP errors. They are periodically added back to the main list and retried. The user can add and remove URLs from this file to affect which URLs are added. Old versions of the file are renamed with a Unix timestamp.
* `outlinks.txt` is used to compile matching outlinks from captures if the `-o` flag is used. They are periodically deduplicated and added to the main list. The user can add and remove URLs from this file. When resuming an aborted session with the `-r` flag, the file is used to determine which URLs have yet to be captured. If the `-o` flag is not used, this file is not created and has no effect if created by the user. Old versions of the file are renamed with a Unix timestamp.
* `index.txt` is used to record which links are part of the list, and is created when the script starts. When outlinks are added to the main list, they are also appended to this file. When a capture job finishes successfully, if the submitted URL redirects to a different URL, the latter is added to the list. The user can add and remove URLs from this file.  When resuming an aborted session with the `-r` flag, the file is used to determine which URLs have yet to be captured. If the -o flag is not used, this file is not used unless the `-r` flag is used to resume the session.
* `max_parallel_jobs.txt` is initially set to the argument of the `-p` flag if the flag is used. The user can modify this file to change the maximum number of parallel capture jobs. If the `-p` flag is not used, this file is not created and has no effect if created by the user.
* `status_rate.txt` is the amount of time in seconds that each process sleeps before checking or re-checking the job status. It is initially set to 1 more than the maximum number of parallel processes (e.g. 3 for `-p 2`, 31 for `-p 30`), and is 2 if the `-p` flag is not used. The user can modify this file to change the rate at which job statuses are checked, although it is intentionally set low enough to avoid causing too many refused connections.
* `lock.txt` is created whenever a rate limit message is received upon submitting a URL. Submission of the URL is retried until it is successfully submitted and the file is removed, and while it exists it prevents more new capture jobs from being started. Removing the file manually will cause all jobs that are waiting to retry submission.
* `daily_limit.txt` is created if the user has reached the Save Page Now daily limit (presently 8,000 captures). The script will pause until the end of the current hour if the file is created, and the file will then be removed. Removing the file manually will not allow the script to continue.
* `quit.txt` lets the script exit if it exists. It is created when the list is empty and there are no more URLs to add to the list. It is also created after five instances (which may be non-consecutive) of the new list of URLs being exactly the same as the previous one. If the file is created manually, the script will run through the remainder of the current list without adding any URLs from `failed.txt` or `outlinks.txt`, and then exit. The user can also end the script abruptly by typing `^C` (`Ctrl`+`C`) in the terminal, which will prevent any new jobs from starting.

The `.log` files in the data folder of the running script do not affect the script's operation. The files will not be created until they receive data (which may include blank lines). They are updated continuously until the script finishes. If the script is aborted, the files may continue receiving data from capture jobs for a few minutes. Log files that contain JSON are not themselves valid JSON, but can be converted to valid JSON with the command `sed -e 's/}$/},/g' <<< "[$(<filename.log)]" > filename.json`.
* `success.log` contains the submitted URLs for successful capture jobs.  When resuming an aborted session with the `-r` flag, the file is used to determine which URLs have yet to be captured.
* `success-json.log` contains the final received status response JSON for successful capture jobs. If the `-q` flag is used, the file is not created and the JSON is discarded after being received.
* `captures.log` contains the Wayback Machine URLs for successful capture jobs (`https://web.archive.org` is omitted). If the submitted URL is a redirect, then the Wayback Machine URL may be different.  When resuming an aborted session with the `-r` flag, the file is used to determine which URLs have yet to be captured if `outlinks.txt` also exists.
* `error-json.log` contains the final received status response JSON for unsuccessful capture jobs that return errors, including those which are retried (i.e. Save Page Now internal/proxy errors).
* `failed.log` contains the submitted URLs for unsuccessful capture jobs that are not retried, along with the date and time and the error code provided in the status response JSON.
* `invalid.log` contains the submitted URLs for capture jobs that fail to start and are not retried, along with the date and time and the site's error message (if any).
* `unknown-json.log` contains the final received status response for capture jobs that end unsuccessfully after receiving an unparsable status response.

## Changelog

* March 18, 2021: Initial release of `spn.sh` 
* March 28, 2021: Addition of `-a`, `-d`, `-r`, `-s` and `-x` options; code cleanup and minor bug fixes (aborted sessions of the previous version of the script cannot be restarted unless the `-o` flag was used or `index.txt` is manually created)

### Future plans

* Add specialized scripts for specific purposes
* Improve code to prevent 429 errors in specific circumstances
* Add data logging/handling for server-side outlinks function
