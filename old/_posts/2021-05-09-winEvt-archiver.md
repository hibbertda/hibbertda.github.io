---
layout: single
title:  "Windows Event Log Archiver"
date:   2021-05-07 13:25:00 -0400
categories: Windows PowerShell
tags: Windows, PowerShell, 
toc: false
classes: wide
excerpt: "Throwback to a PowerShell script I wrote several years ago. The write up was originally on the Microsoft TechNet blogs / PowerShell Gallery. Shortly before it was shutdown I remembered to get in and snag the good bits. Including an awesome comment. Enjoy!"
---

Throwback to a PowerShell script I wrote several years ago. The write up was originally on the Microsoft TechNet blogs / PowerShell Gallery. Shortly before it was shutdown I remembered to get in and snag the good bits. Including an awesome comment. Enjoy!

![PowerShell Gallery Comment](/assets/images/2021-05-09-winevtarchive/psg-comment.jpeg)

## Background

I Received a request to archive all of the event logs on server, and maintain the archived logs on the server for up to six months. To meet these requirements the following script will create a schedule task that will run every 30 minutes. Each event log will be checked, and if the size exceeds 75% of the configured maximum log file size the log will be exported and compressed to the configured archive directory. The script will query for any event logs that were automatically archived by the system in the default event log location (C:\Windows\System32\Winevt\logs) and will compress and archive the files to the configured archive location. After the archived event logs have exceeded the configured retention period it will be automatically removed the next time the task is run.\

All of the actions of the script will be written to the application log on the local system. This will allow monitoring of the status of the script with SCOM or other monitoring tools.

## Description

This script will be used to automate the collection and archival of Windows event logs. When an Event Log exceeds 75% of the configured maximum size the log will be backed up, compressed, moved to the configured archive location and the log will be cleared. If no location is specified the script will default to the C:\ drive. It is recommended to set the archive path to another drive to move the logs from the default system drive. In order to run continuously the script will created a scheduled task on the computer to run every 30 minutes to to check the current status of event logs.
Status of the script will be written to the Application log **[MSP_LogMaintenance]**.

### Event log ID Reference

As the script is setup and durring subsequest runs. The actions adn status is written to the eventlog on the local Windows machine.

|---|---|
|775|Script setup operations|
|776|Script run start|
|777|Event Log|
|778|Remove expired Event Log|
|779|Script run complete|

## Example

Use this example PowerShell command to run and setup the script for the first time. The Event Log archives will be set as "**D:\Eventlog_Archive**".

```powershell
EventLog_Archive.ps1 -EventLogArchivePath D:\EventLog_Archive
```

## Check it out

The full script and instructions are over on GitHub - [hibbertda/ps-win-eventlogarchiver][git-archive]

[git-archive](https://github.com/hibbertda/ps-win-eventlogarchive)