---
layout: post
title: "Configuring the Date and Time on Modern Linux"
---

## Clarify concepts first: RTC, system clock, UTC

Modern operating systems distinguish between the following two types of clocks:  
- **RTC**  
A **real-time clock** (**RTC**), commonly referred to as a hardware clock, (typically an integrated circuit on the system board) 
that is completely independent of the current state of the operating system and runs even when the computer is shut down.

- **system clock**  
A **system clock**, also known as a software clock, that is maintained by the kernel and its initial value is based on the real-time 
clock. Once the system is booted and the system clock is initialized, the system clock is completely independent of the real-time clock.


The system time is always kept in **Coordinated Universal Time** (**UTC**) and converted in applications to local time as needed. Local time 
is the actual time in your current time zone, taking into account daylight saving time (DST). The real-time clock can use either UTC 
or local time. UTC is recommended.

## Configuring tools on Mordern Linux

Usually Linux offers two command line tools that can be used to configure and display information about the system date and time:  
- the traditional **date** command;
- the **hwclock** utility for accessing the hardware clock.

Now mordern Linux offers new **timedatectl** utility, **timedatectl** is distributed as part of the [systemd System and Service Manager](https://www.freedesktop.org/wiki/Software/systemd/)  
(about systemd, see also: [systemd.io](https://systemd.io/), [systemd - wikipedia](https://en.wikipedia.org/wiki/Systemd)). You can use this 
tool to change the current date and time, set the time zone, or enable automatic synchronization of the system clock with a remote server.

## Using the timedatectl Command
Here let's see the Usage of timedatectl

### Displaying the Current Date and Time
```
~ ]$ timedatectl 
               Local time: Fri 2022-01-01 13:53:32 CST
           Universal time: Fri 2022-01-01 05:53:32 UTC
                 RTC time: Fri 2022-01-01 05:53:32
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: no
              NTP service: inactive
          RTC in local TZ: no
```

### Changing the Current Time and Date
This command updates **both the system time and the hardware clock**. The result it is similar to using both the date --set and hwclock --systohc commands.  
```
~]# timedatectl set-time 13:58:00
~]# timedatectl set-time "2022-01-01 $(date +%T)"
```

### Changing the Time Zone
```
~]$ timedatectl list-timezones | grep -i asia | less
~]# timedatectl set-timezone Asia/Shanghai
```

### set RTC as UTC: **'sudo timedatectl set-local-rtc 0'**
```
[yjh@fedora ~]$ timedatectl 
               Local time: 二 2022-12-27 20:15:32 CST
           Universal time: 二 2022-12-27 12:15:32 UTC
                 RTC time: 二 2022-12-27 07:15:32
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: yes

Warning: The system is configured to read the RTC time in the local time zone.
         This mode cannot be fully supported. It will create various problems
         with time zone changes and daylight saving time adjustments. The RTC
         time is never updated, it relies on external facilities to maintain it.
         If at all possible, use RTC in UTC by calling
         'timedatectl set-local-rtc 0'.
[yjh@fedora ~]$ sudo timedatectl set-local-rtc 0
[yjh@fedora ~]$ timedatectl 
               Local time: 二 2022-12-27 20:16:46 CST
           Universal time: 二 2022-12-27 12:16:46 UTC
                 RTC time: 二 2022-12-27 12:16:46
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

---
## ref
[Fedora-35 - Configuring_the_Date_and_Time](https://docs.fedoraproject.org/en-US/fedora/f35/system-administrators-guide/basic-system-configuration/Configuring_the_Date_and_Time/)
