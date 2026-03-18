Journalctl Niceties

Part of the systemd solution is the log manager journalctl -- which takes
a while to get used to. I does provide some nifty features though.

You can list the times the machine has booted, with unique identifiers per boot:

    journalctl --list-boots

Add an `-n X` to only list the last X boots; the last two boots on my machine right now
is thus:

    IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY                 
     -1 5fdf88e99fc443368952110898c387d8 Tue 2026-03-17 07:47:16 CET Wed 2026-03-18 08:10:19 CET
      0 4a374ebd43fc4260a02fcdb38f9d0b67 Wed 2026-03-18 08:11:31 CET Wed 2026-03-18 08:19:18 CET

Then, using the index you can get the system logs from just before the last boot by doing:

    journalctl -b -1 -e

You may add the `-p err` flag to only get error level, and perhaps `-k` to only get kernel logs.

For ordinary applications (units) the `-u` parameter selects the unit -- specifying `--follow` mimics
the use of `tail` on ordinary log files -- and finally `--since='10 min ago'` lets you _not_ get logs
since the beginning of time :-)

Example:

    journalctl -u nginx --since='10 min ago' --follow

Will give you the last 10 minutes of the nginx log and tail for new log lines. Nify.

Tags: computer, debian, linux
