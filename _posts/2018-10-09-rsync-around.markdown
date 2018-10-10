---
layout: post
title: "Rsync Around"
date: 2018-10-09
---

### Using rsync for transfers, backups and more

I have become very fond of rsync recently. Nominally rsync is for 'syncing' or mirroring directories, ensuring that files get kept up to date. One of the key features of rsync is that it only transfers updated files (in fact, it only transfers the differences between them), unlike simply copying a directory full of files where all files would be overwritten at the destination. This makes it much more efficient than copying, particularly to remote servers where the connection speed might be slow and large files take a long time to transfer.

There are lots of other useful features though. For example, when transferring a large, critical file via FTP (as I recently had to do for some sequencing data from BGI), you would be wise to verify that the transfer succeeded before deleting the original copy or going ahead with any anyalysis of it. At a minimum you can check that the file size is the same, but a better way is to calculate a checksum that uses an algorithm to reduce the entire contents of the file to a string of characters that can be compared relatively easily. The utility md5sum is one way of doing this. However, rsync automatically calculates a checksum after it finishes a transfer and will retry if the checksum indicates a problem with the transfer. Similarly, if your transfer fails halfway through due to a dropped connection, rsync can restart the aborted transfer, whereas a regular copy will start again at the beginning. Both of these features can avoid a lot of frustration when trying to transfer big files (e.g. sequencing data) over slow or unreliable connections.

Here are a couple of ways I use rsync. 

## Daily backups

To avoid me having to remember to make backups from my laptop, I use the following entry in my crontab:

```{cron}
29 12 * * * ssh servername
30 12 * * * rsync -rltgD ~/work/ servername:/path/to/backup/directory
```

This opens an ssh tunnel to the remote server (needs to be configured with ssh keys separately) at 12:29 every day and then runs the `rsync` command at 12:30 to update the copy of my ~/work folder on the backup server. You can use the `-a` option in most cases, which is short for -rlpgtoD. I previously had issues setting permissions and ownership on a samba server I was copying to, so had to omit the `-o` and `-p` options.

Importantly, file deletion on the source (~/work above) does not get mirrored on the destination by default. So even if I delete files on my laptop, they won't be deleted on the backup server. I used to worry a lot about this happening when I thought of rsync as a mirroring utility; now I consider it more of an advanced file copy. If I delete a file from the destination, it will be recopied the next day if it still exists on my laptop. The server I am copying to also runs daily backups, so there is an extra layer of protection against a file accidentally being deleted on both sides of this command.

## Mirroring when working from home

I use similar commands to pull files from the server to my computer when I am working from home---this means I have an up to date, accessible copy of everything I am working on. To speed this up I often run just on the subdirectory that I need, and use the `--max-size=50m` option to avoid transferring any files over 50 Mb by default. Another good way to make transfers like this more lightweight is to use a `.cvsignore` file, where you can specify file extensions or other patterns to exclude from transfers---see the man page for examples. I sometimes use this to avoid copying bam, fastq etc. files which I usually have stored elsewhere.

## Auto backup and delete

Recently I have been using the Oxford Nanopore MinION sequencer a lot. One of the very many cool things about this DNA sequencer is that it plugs into your laptop via USB. Unfortunately, as throughput has improved, this does tend to fill up your hard drive rather rapidly with directories full of very long sequencing read files. The software creates subdirectories with 4000 reads (files) each by default. Usually I leave the sequencer running overnight, so I wanted a solution to transfer the read files to a server as they completed.

I decided I didn't trust the connection to the server to be maintained overnight, and in there may have been I/O speed issues that I didn't want to get into, so saving them there directly from the sequencer  wasn't an option. I thought I might write a script that would check periodically and copy the files, but after a bit of digging found that rsync could do this for me with the following options: `--remove-source-files -avm --include="*.fast5" -f 'hide,! */'`

Useful options when setting something like this up are `--verbose` and `--dry-run` to show the file list that would be transferred without actually transferring them (or deleting the source files!). I actually use verbose mode in the cronjob transfers to allow a detailed log to be created, just in case anything did go wrong. The slightly weird part here is specifying that we only want fast5 files to be copied. The filter option here hides all non-directories, effectively listing the directories to copy. Specifying the `*.fast5` pattern alone in the `--include` option then limits the transferred files to those matching the pattern (At least that's how I understand it...this filter set is taken straight from the rsync man page). 

I was worried about what would happen if this command ran while a file was still being written so I tested this. It turns out that the file copy checksum that rsync does as standard takes care of this---if the checksums are different after the transfer has finished, rsync doesn't delete the source (very important!) and will retry the transfer.

Unrelated to rsync, I also thought that once I had emptied a directory of Nanopore fast5 files, the Nanopore software might start filling the "empty" directories back up again. This would have resulted in subdirectories with more than 4000 fast5 files. It turns out this doesn't happen, the MinKnow software keeps track and continues to make new directories 2/, 3/ etc even if 1/ is empty.

Running this command in an hourly cron job (with a preceding ssh command to ensure the connection is still open) while the sequencing run is underway ensures my laptop stays nice and clear. It's good to know that even if the server becomes inaccessible the worst that can happen is my disk will fill up and I won't lose data. I was really pleased to find I could do this with essentially a one-liner; rsync is a typical mature unix tool in that you get all this functionality, flexibility and error-checking thrown in, which would otherwise have to be incorporated into whatever clumsy script I might have written.

