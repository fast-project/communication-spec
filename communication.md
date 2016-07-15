Umsetzung der Schnittstellen mit MQTT
=====================================

Grundlagen
----------

### Annahmen
* Scheduler kennt die Hostnamen aller Knoten.


### Aufbau einer MQTT Nachricht
Eine MQTT Nachricht besteht aus zwei Teilen

* Topic
* Payload in YAML


### Ablauf der Kommunikation
Grundsätzlich ist das ein einfaches Publisher-Subscriber Pattern.
Hier eine grobe Übersicht.

1. Ein Teilnehmer verbinden sich mit einem Server.
  * Ein Teilnehmer melden sich für N Topics an.
  * Der Server schickt alle Nachrichten für die sich ein Teilnehmer angemeldet
    hat.

2. Ein Teilnehmer meldet sich ab.

### Diskussion
* Eine Nachricht pro gestartete VM oder eine Nachricht mit allen gestarteten VMs
  an Scheduler? (siehe Kommentar unter "vm started")
  //J: Macht das einen relevanten Unterschied?
* Einheitlicher result status: "status: success | error" (Eventuell "details:
  <error description>" hinzufügen für mehr Infos.)
  //J: Finde ich gut, sollte in der FASTLib auch eine Klasse dafür geben.
* Einheitlicher task/result Name: "task: <start|stop|migrate> vm", "result: vm
  <started|stopped|migrated>"
  //J: Wichtiger wäre wohl jeder Nachricht eine eindeutige ID zu geben, die beim
       Antworten wieder mitgeschickt wird.

### Online YAML parser
http://yaml-online-parser.appspot.com/


MQTT Topics
-----------

### Struktur
```
fast
+-- migfra
|   +-- <hostname>
|   |   +-- task
|   |   +-- result
|   +-- ...
|   +-- <hostname>
+-- pscom
|   +-- <hostname>
|   |   +-- <procID>
|   |   |   +-- request
|   |   |   +-- response
|   |   |-- ...
|   |   |-- <procID>
|   +-- ...
|   +-- <hostname>
+-- agent
|   +-- <hostname>
|   |    +-- mmbwmon
|   |    |   +-- request
|   |    |   +-- response
|   |	 +-- status
|   |	 +-- task
|   +-- ...
|   +-- <hostname>
+-- application
    +-- <slurm_jobid>
        +-- status
        +-- KPIs
```

### Subscriber
* Scheduler
  * fast/agent/+/status
    Ueberwachung der Statusänderungen *aller* Agenten
  * fast/migfra/+/result
    Ergebnisse von Anfragen des Schedulers an eine der
    Migrationsframework-Instanzen
  * fast/application/+/status
  * fast/application/+/KPIs
* Migration-Framework
  * fast/migfra/\<hostname\>/task
    Entgegennehmen von Anfragen des Schedulers
  * fast/pscom/\<hostname\>/\<procID\>/response
    Antwort vom entsprechenden Prozess, dass alle Verbindungen herunter gefahren wurden
* Agent
  * fast/migfra/\<hostname\>/result
    Aenderungen im Schedule des betreffenden nodes (z.B. neuer Job)
  * fast/agent/\<hostname\>/task
    Anfragen des Schedulers an den entsprechenden Agenten (z.B. start/stop
    monitoring)
* pscom
  * fast/pscom/\<hostname\>/\<procID\>/request
    Entgegennehmen von Anfragen des Migrationsframeworks
* Anwendungen
  //J: Von mir, aber noch nicht ganz klar ob das so passt
  * fast/migfra/\<hostname\>/task
    Überwacht um mögliche Migration festzustellen <- so aber keine Möglichkeit einzugreifen
  * fast/application/\<hostname\>/\<pid\>/task <- \<pid\> nicht über VMs hinweg identisch / \<hostname\>=VM name?
    Entgegennehmen der Anfragen des Agenten / Schedulers / MigFra

### Publisher
* Scheduler
  * fast/migfra/\<hostname\>/task
    Starten, Stoppen und Migrieren von VMs
  * fast/agent/\<hostname\>/task
    starten/stoppen des Monitorings; Konfiguration der KPIs
* Migration-Framework
  * fast/migfra/\<hostname\>/result
    VM gestartet/gestoppt/migriert
  * fast/pscom/\<hostname\>/\<procID\>/request
    Herunterfahren von pscom Verbindungen
* Agent
  * fast/agent/\<hostname\>/status
    Anmeldung des Agenten beim Scheduler
* pscom
  * fast/pscom/\<hostname\>/\<procID\>/response
    Information an Migrationsframework, dass Verbindungen herunter gefahren wurden
* Anwendungen
  //J: Von mir, aber noch nicht ganz klar ob das so passt
  //* fast/application/\<hostname\>/\<pid\>/status <- \<pid\> nicht über VMs hinweg identisch / \<hostname\>=VM name?

  * fast/applicaiton/\<slurm_jobid\>/status
    Status Informationen der Anwendung
  * fast/application/\<slurm_jobid\>/KPIs

Nachrichtenformat
-----------------
Im Folgenden werden die Nachrichten, welche über die oben definierten Topics
verteilt werden, definiert. Hierbei handelt es sich jeweils um Nachrichten im
YAML Format. Die unterschiedlichen Nachrichten sind nach ihrer Quelle sortiert.

### Scheduler
#### Agenten Initialisieren
Der Scheduler meldet sich beim Agenten und liefert eine initiale Konfiguration.
* topic: fast/agent/\<hostname\>/task/init_agent
* Payload

```
task: init agent
KPI:
  categories:
    - Application/VM memory usage : < value >
    - Application/VM Cpu usage: <value>   //(100  is all cores used):
    - Node memory usage : < value >
    - Node memory bandwidth : <value>
    - Node Cpu usage: <value>   //(100  is all cores used):
    - Node Network Interconnect bandwidth:  <value>
    - communication intensity (network): <high,medium,low>
    - expected runtime: <high,medium,low>
  repeat: <number in seconds how often the KPIs are reported>
```

* Erwartetes Verhalten:
  Der Agent merkt sich die gesendete Konfiguration und reagiert
  entsprechend. Der für den Knoten zuständige Scheduler empfängt die
  Nachrichten über das angegebene Topic.
* Antwort: Default result status

#### Monitoring/Tuning beenden
Der Agent soll aufhören die Anwendung zu überwachen.
* topic: fast/agent/\<hostname\>/task/stop_monitoring
* Payload
  ```
  task: stop monitoring
  job-description:
    job-id: <job id>
    process-id: <process id of the vm>
  ```
* Erwartetes Verhalten:
  Agent hört auf den Prozess zu überwachen.
* Antwort: Default result status

#### VMs starten
Anfrage des Schedulers an die entsprechende Migrationsframework-Instanz
eine oder mehrere VMs zu starten.
Diskussion: VM vorbereiten in Scheduler Skripten oder von Migfra?
* topic: fast/migfra/\<hostname\>/task
* Payload

```
host: <string>
task: start vm
id: <uuid>
vm-configurations:
  - vm-name: <string>
    memory: <unsigned long (in kiB)>
    vcpus: <Anzahl>
  - xml: <XML string>
    pci-ids:
      - vendor: <vendor-id>
        device: <device-id>
      - ..
  - ..
```
* id: Wird bei result Nachricht mit zurück geschickt, um die Zugehörigkeit zwischen task/result erfassen zu können.
* VM kann entweder per "name" gestartet werden oder per "xml"
* vm-name: Es wird eine schon definierte VM gesucht (virDomainLookupByName)
* xml: Es wird eine VM anhand des XML-Strings definiert (virDomainDefineXML)
* overlay-image & base-image sind im XML definiert.
* memory: Setzt memory und maxmemory. (optional)
* vcpus: Setzt vcpus und maxvcpus. (optional)
* pci-ids: Ermöglicht eine Liste von PCI-IDs anzugeben, um je ein PCI-Device mit dieser ID zu attachen.
  PCI-IDs geben den Geräte-Typ an und können mithilfe von "lspci -nn" leicht herausgefunden werden.
  Sie bestehen aus vendor und device ID.
  Bsp.:
```
  - vendor: 0x15b3
    device: 0x1004
```
* Erwartetes Verhalten:
  VMs werden auf dem entsprechenden Host gestartet.
  Es wird gewartet bis die VMs bereit (erreichbar mit ssh) sind bevor result geschickt wird.
* Antwort: Default result status? Oder doch lieber modifiziert mit eindeutige IDs für die VMs?

#### VM stoppen
Annahme: VM wird gestoppt wenn die Anwendung fertig ist / beendet werden soll.
* topic: fast/migfra/\<hostname\>/task
* Payload

```
host: <string>
task: stop vm
id: <uuid>
list:
  - vm-name: <vm name>
  - vm-name: <vm name>
    force: true
  - ..
```
* force: Ermöglicht es optional die VM unmittelbar zu beenden (virDomainDestroy statt virDomainShutdown).
  Ist "false", wenn weggelassen.
* Erwartetes Verhalten:
  VM wird gestoppt.
* Antwort: Default result status

#### Migration starten
Diese Nachricht informiert die zuständige Migrationsframework-Instanz darüber,
dass die Anwendung vom Quellknoten auf den Zielknoten migriert werden soll.
* topic: fast/migfra/\<hostname\>/task
* Payload

```
host: <string>
task: migrate vm
id: <uuid>
vm-name: <vm name>
destination: <destination hostname>
time-measurement: true
parameter:
  retry-counter: <counter>
  migration-type: live | warm | offline
  rdma-migration: true | false
  pscom-hook-procs: <Anzahl der Prozesse>
```
* time-measurement: Gibt Informationen über die Dauer einzelner Phasen im result zurück. (Optional)
* pscom-hook-procs: Anzahl der Prozesse deren pscom Schicht unterbrochen werden muss. (Optional)
* Erwartetes Verhalten:
  VM wird vom Migrationsframework gestartet und anschließend wird eine
  entsprechende Statusinformation über den 'scheduler' channel gechickt.
* Antwort: Default result status

### Migration-Framework
#### VM gestartet
Nachdem die VM gestartet ist und bereit ist eine Anwendung auszuführen
informiert die entsprechende Migrationsframework-Instanz den Schduler darüber.
* topic: fast/migfra/\<hostname\>/status
* Payload:

```
scheduler: <hostname/global>
result: vm started
id: <uuid>
list:
  - vm-name: <vm-hostname>
    status: success | error
    details: <string>
    process-id: <process id of the vm>
  - vm-name: <vm-hostname>
    status: success | error
    process-id: <process id of the vm>
  - ..
```
* details: Ermöglicht detailierte Fehlerinformationen zurückzugeben.
* Erwartetes Verhalten:
  Der für den Knoten zuständige Scheduler empfängt die Nachricht und
  startet die Anwendung in der VM.
* Implementierung:
  Über mqtt_publish in Startup Skript der VM. Hierbei gibt es die
  folgenden Möglichkeiten:
	1. VM sendet an migfra "vm ready", migfra sendet dies gebündelt an
	   scheduler mit "vm started"
	2. Migfra wartet bis alle VMs der task "start vm" per SSH erreichbar
	   sind und schickt gebündelt eine Liste der Stati an den zuständigen
	   scheduler.

#### VM gestoppt
Informiert den zuständigen Scheduler, dass die VM gestoppt ist.
* topic: fast/migfra/\<hostname\>/status
* Payload

```
result: vm stopped
id: <uuid>
list:
  - vm-name: <vm-hostname>
    status: success | error
    details: <string>
  - vm-name: <vm-hostname>
    status: success | error
  - ..
```

* details: Ermöglicht detailierte Fehlerinformationen zurückzugeben.
* Erwartetes Verhalten:
  VM aufräumen? Log files für den Nutzer rauskopieren?

#### Migration abgeschlossen
Meldung an den Scheduler dass die Migration fertig ist.
* topic: fast/migfra/\<hostname\>/status
* Payload

```
result: vm migrated
id: <uuid>
vm-name: <vm name>
status: <success | error>
details: <retries | error-string>
process-id: <process id of the vm>
time-measurement:
  - <tag>: <duration in sec>
  - ..
```
* details: Ermöglicht detailierte Fehlerinformationen oder bei "success" die Anzahl der Versuche zurückzugeben.
* time-measurement: Falls Zeitmessungen im task aktiviert wurden, wird hier eine Liste von Tags mit Zeitdauern zurückgegeben.
* Erwartetes Verhalten:
  Scheduler markiert ursprüngliche Ressource als frei.

#### Verbindungen abbauen
Meldung an die pscom-Schicht, dass die Verbindungen abgebaut werden sollen.
* topic: fast/pscom/\<hostname\>/\<pid\>/request
* Payload

```
task: suspend
```

* Erwartetes Verhalten:
  pscom faehr suspend-Protokoll ab

#### Verbindungen aufbauen
Meldung an die pscom-Schicht, dass die Verbindungen wieder hergestellt werden
koennen.
* topic: fast/pscom/\<hostname\>/\<pid\>/request
* Payload

```
task: resume
```

* Erwartetes Verhalten:
  pscom gibt Verbindungen frei

### Agent
#### Agenten anmelden
Knoten wird gestartet. Meldet und meldet sich beim Scheduler an.
* topic: fast/agent/\<hostname\>/status
* payload:

```
task: init
source: <hostname>
```

* Erwartetes Verhalten:
  Der für den Knoten zuständige Scheduler empfängt die Nachricht und
  nimmt den Knoten in sein Scheduling mit auf.
* Implementierung:
  Agent wird automatisch bei Systemstart gestartet und verschickt die
  o.g. Nachricht.

#### Messwerte zurückschicken
Agend resoponds to the configruation proposed by the scheduler.
* topic: fast/agent/<hostname>/status
* Payload

```
task: configuration
source: <hostname>
status: accepted|denied
```
* Erwartetes Verhalten:
The agent should send this message to confirm or the deny the configuration proposed by the scheduler.

#### KPIs
* topic: fast/agent/<hostname>/status
* Payload

```
task: KPI
source: <hostname>
KPIS:
  - Application/VM memory usage : < value >
  - Application/VM Cpu usage: <value>   //(100  is all cores used):
  - Node memory usage : < value >
  - Node memory bandwidth : <value>
  - Node Cpu usage: <value>   //(100  is all cores used):
  - Node Network Interconnect bandwidth:  <value>
```

### MMBWMON (Agent zur Speicherbandbreitemessung)
#### Anfrage Main Memory Bandwidth
Anfrage an den Agenten eine Speicher Bandbreitenmessung anzustoßen.
* topic: fast/agent/\<hostname\>/mmbwmon/request
* payload:

```
task: mmbwmon request
cores: <list of CPU cores>
```

* Aktuelles Verhalten von MMBWMON:
  Die verfügbare Speicherbandbreite für die unter 'cores' angegebenen CPU Kernen wird gemessen und anschließend
  zurückgesendet (siehe nächsten Punkt). VMs/Prozesse die u.U. auf den CPU Kernen laufen müssen vorher angehalten werden.

#### Rückmeldung Main Memory Bandwidth
Antwort des Agentens mit der aktuell verfügbaren Speicherbandbreite auf den gemessenen Kernen.
* topic: fast/agent/\<hostname\>/mmbwmon/response
* payload:

```
task: mmbwmon response
cores: <list of cores>
response: <value between 0.33 and 1>
```

* Aktuelles Verhalten von MMBWMON:
  Die verfügbare Speicherbandbreite für die unter 'cores' angegebenen CPU Kernen wird zurückgeliefert.
  1    == CPU Kerne erhalten aktuell maximale Speicherbandbreite
  0.33 == CPU Kerne erreichen nur 33% der theoretisch verfügbaren Bandbreite. Minimalwert.


### Anwendungen
TODO?
#### Application status
* topic: fast/application/\<slurm_jobid\>/status
* payload:

```
task: status
source: <slurm_jobid>
migration: yes | no
supported_KPI :
  - memory usage
  - memory bandwidth
  - ..
```

* Erwartetes Verhalten:
  Application is migratable be default.
  However this can be changed by setting 'migration' to 'no' in the status message.
  Also application at start up can declare the list of KPIs that it is welling to provide.
#### Application KPI
* topic: fast/application/\<slurm_jobid\>/KPIs
* payload:

```
task: KPIs
source: <slurm_jobid>
supported_KPI :
    - memory usage : < value >
    - memory bandwidth : <value>
    - ..
```

* Erwartetes Verhalten:
  The scheduler should receive the application's KPIs and take them into account.
