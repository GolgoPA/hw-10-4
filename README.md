# Домашнее задание к занятию 10.4 «Резервное копирование»

---

### Задание 1

В чём разница между:

- полным резервным копированием,
- дифференциальным резервным копированием,
- инкрементным резервным копированием.

*Приведите ответ в свободной форме.*

- полное резервное копирование: 
> Каждый раз полное копирование всего.
- дифференциальное резервное копирование:
>  Сначала делается полное копирование всего. При следующих запусках копируются только данные, изменённые с момента полного бэкапа.
- инкрементное резервное копирование:
>   Сначала делается полное копирование всего. При следующих запусках копируются только данные, изменённые с момента последнего проведённого резервного копирования.

---

### Задание 2

Установите программное обеспечении Bacula, настройте bacula-dir, bacula-sd,  bacula-fd. Протестируйте работу сервисов.

*Пришлите конфигурационные файлы для bacula-dir, bacula-sd,  bacula-fd.*
 bacula-dir.conf: 
```


Director {                            # define myself
  Name = hw-10-4-n1-dir
  DIRport = 9101                # where we listen for UA connections
  QueryFile = "/etc/bacula/scripts/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/run/bacula"
  Maximum Concurrent Jobs = 20
  Password = "qwerty12345"         # Console password
  Messages = Daemon
  DirAddress = 127.0.0.1
}


...skipping 1 line
  Name = "DefaultJob"
  Type = Backup
  Level = Incremental
  Client = hw-10-4-n1-fd
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File1
  Messages = Standard
  Pool = File
  SpoolAttributes = yes
  Priority = 10
  Write Bootstrap = "/var/lib/bacula/%c.bsr"
}


Job {
  Name = "BackupClient1"
  JobDefs = "DefaultJob"
}


Job {
  Name = "BackupCatalog"
  JobDefs = "DefaultJob"
  Level = Full
  FileSet="Catalog"
  Schedule = "WeeklyCycleAfterBackup"
  # This creates an ASCII copy of the catalog
  # Arguments to make_catalog_backup.pl are:
  #  make_catalog_backup.pl <catalog-name>
  RunBeforeJob = "/etc/bacula/scripts/make_catalog_backup.pl MyCatalog"
  # This deletes the copy of the catalog
  RunAfterJob  = "/etc/bacula/scripts/delete_catalog_backup"
  Write Bootstrap = "/var/lib/bacula/%n.bsr"
  Priority = 11                   # run after main backup
}


Job {
  Name = "RestoreFiles"
  Type = Restore
  Client=hw-10-4-n1-fd
  Storage = File1
  FileSet="Full Set"
  Pool = File
  Messages = Standard
  Where = /nonexistant/path/to/file/archive/dir/bacula-restores
}


FileSet {
  Name = "Full Set"
  Include {
    Options {
      signature = MD5
    }

    File = /usr/sbin
  }


  Exclude {
    File = /var/lib/bacula
    File = /nonexistant/path/to/file/archive/dir
    File = /proc
    File = /tmp
    File = /sys
    File = /.journal
    File = /.fsck
  }
}


Schedule {
  Name = "WeeklyCycle"

...skipping 1 line
  Run = Differential 2nd-5th sun at 23:05
  Run = Incremental mon-sat at 23:05
}


Schedule {
  Name = "WeeklyCycleAfterBackup"
  Run = Full sun-sat at 23:10
}


FileSet {
  Name = "Catalog"
  Include {
    Options {
      signature = MD5
    }
    File = "/var/lib/bacula/bacula.sql"
  }
}


Client {
  Name = hw-10-4-n1-fd
  Address = localhost
  FDPort = 9102
  Catalog = MyCatalog
  Password = "qwerty12345"          # password for FileDaemon
  File Retention = 60 days            # 60 days

...skipping 1 line
  AutoPrune = yes                     # Prune expired Jobs/Files
}




Autochanger {
  Name = File1
  Address = localhost                # N.B. Use a fully qualified name here
  SDPort = 9103
  Password = "qwerty12345"
  Device = FileChgr1
  Media Type = File1
  Maximum Concurrent Jobs = 10        # run up to 10 jobs a the same time

...skipping 1 line
}


Autochanger {
  Name = File2
  Address = localhost                # N.B. Use a fully qualified name here
  SDPort = 9103
  Password = "qwerty12345"
  Device = FileChgr2
  Media Type = File2
  Autochanger = File2                 # point to ourself
  Maximum Concurrent Jobs = 10        # run up to 10 jobs a the same time
}

...skipping 1 line
Catalog {
  Name = MyCatalog
  dbname = "bacula"; DB Address = "localhost"; dbuser = "bacula"; dbpassword = "qwerty12345"
}


Messages {
  Name = Standard



  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e of %c %l\" %r"
  operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: Intervention needed for %j\" %r"
  mail = root = all, !skipped
  operator = root = mount
  console = all, !skipped, !saved


...skipping 1 line

  append = "/var/log/bacula/bacula.log" = all, !skipped
  catalog = all
}


Messages {
  Name = Daemon
  mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula daemon message\" %r"
  mail = root = all, !skipped
  console = all, !skipped, !saved
  append = "/var/log/bacula/bacula.log" = all, !skipped
}

Pool {
  Name = Default
  Pool Type = Backup
  Recycle = yes                       # Bacula can automatically recycle Volumes
  AutoPrune = yes                     # Prune expired volumes
  Volume Retention = 365 days         # one year
  Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  Maximum Volumes = 100               # Limit number of Volumes in Pool
}

Pool {

...skipping 1 line
  Pool Type = Backup
  Recycle = yes                       # Bacula can automatically recycle Volumes
  AutoPrune = yes                     # Prune expired volumes
  Volume Retention = 365 days         # one year
  Maximum Volume Bytes = 50G          # Limit Volume size to something reasonable
  Maximum Volumes = 100               # Limit number of Volumes in Pool
  Label Format = "Vol-"               # Auto label
}


Pool {
  Name = Scratch
  Pool Type = Backup
}


Console {
  Name = hw-10-4-n1-mon
  Password = "qwerty12345"
  CommandACL = status, .status
}

```

---

### Задание 3

Установите программное обеспечении Rsync. Настройте синхронизацию на двух нодах. Протестируйте работу сервиса.

*Пришлите рабочую конфигурацию сервера и клиента Rsync.*

---

### Задание со звёздочкой*
Это задание дополнительное. Его можно не выполнять. На зачёт это не повлияет. Вы можете его выполнить, если хотите глубже разобраться в материале.

---

### Задание 4*

Настройте резервное копирование двумя или более методами, используя одну из рассмотренных команд для папки /etc/default. Проверьте резервное копирование.

*Пришлите рабочую конфигурацию выбранного сервиса по поставленной задаче.*

