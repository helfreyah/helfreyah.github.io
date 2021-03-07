# 1-Windows System Architecture

### USER MODE

### User Processes :

Windows 32-bit ve 64-bit uygulamalarıdır. Bunun yanında POSIX uygulamarıda bulunur fakat POSIX uygulamaları Windows 8’den itibaren desteklenmiyor.

### Service Processes :

Windows servis prosesleri “Task Scheduler” , “Print Spooler” gibi servislerdir. Bu servisler kullanıcı oturumlarından bağımsız çalışır.

### System Processes :

Sistem prosesleri “Session Manager(SMSS), WINLOGON.exe” gibi proseslerdir ve bunlar windows servisleri değildir sistem prosesleridir.

### Environment Subsystems :

Windows NT , Windows,POSIX,OS/2 subsystem ile gelir. Fakat OS/2 Windows 2000, POSIX ise en son Win XP ile yayınlandı. Windows 7 Ultimate ve Enterprise sürümlerinde ve Windows 2008 R2 sürümlerinde UNIX tabanlı uygulamalar için SUA adı verilen bir POSIX subsystem desteği bulunur. Fakat artık Windows tarafından sunulmuyor.
KERNEL MODE

### Executive :

Memory Management, Process & Threat Management, I/O, Networking, IPC(Inter-process communication) gibi OS servislerini içerir.

### Kernel :

Thread Scheduling, Interrupt & Exception Dispatching, Multiprocessor Synchronization gibi low-level OS fonksiyonlarını içerir.

### Device Drivers :

Donanım sürücülerini ve donanım aygıtı olmayan sürücüleri içerir mesela dosya sistemi ve ağ sürücüsü gibi.

### Hardware Abstraction Layer :

Donanım ve yazılımların iletişimini sağlar. Yüksek seviyede yazılan sürücüleri , donanım gibi alt seviye bileşenleri ile iletimi sağlar.

### Windows GUI :

Kullanıcı grafik arabiriminin işlevlerini gerçekleştirir.
