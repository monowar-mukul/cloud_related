Deploying Oracle DB Dashboard with Grafana Mimir, Alloy, and OracleDB Exporter
Oracle → Grafana Mimir Observability (via Grafana Alloy & OracleDB Exporter)
This guide assumes:
•	Oracle Database is already installed and running on Linux 9
•	You want to monitor Oracle using Prometheus + Grafana + oracledb_exporter
•	You have root or sudo privileges
Scope: Stand up an end to end metrics pipeline for Oracle databases using:
Component	Purpose
Oracle Exporter	Pulls Oracle metrics in Prometheus format
Grafana Agent / Alloy	Scrapes exporter & pushes to Mimir
Grafana Mimir	Stores metrics
Grafana	Dashboards & alerts
________________________________________
Architecture (at a glance)
OracleDB → OracleDB Exporter (:9162 /metrics) → Grafana Alloy (prometheus.scrape → prometheus.remote_write) → Grafana Mimir (long term storage + PromQL API) → Grafana Dashboards\ {panel}
Notes:
•	OracleDB Exporter (v0.5+) uses a pure Go driver; requires a URL style DSN and URL encoding special characters.
•	Alloy’s prometheus.remote_write sends metrics to Mimir using the Prometheus remote_write protocol.
•	Mimir’s remote_write ingestion endpoint is /api/v1/push; multi tenancy relies on the X Scope OrgID header.
________________________________________
Prerequisites
Oracle access from the exporter host and a dedicated monitoring user (e.g., @GRAFANAU@).
A running Grafana Mimir (PoC: monolithic; Prod: microservices) reachable at:
•	remote_write: @ http(s)://:9009/api/v1/push 
•	query (Prometheus API): @ /api/v1/query
Grafana instance able to connect to Mimir as a Prometheus data source.
________________________________________
Step 1 — Install and Run the OracleDB Exporter
•	If your password contains special characters (e.g., #), URL encode them in the DSN (# → %23) and remember that in systemd unit files the percent sign must be written as %% to pass a literal % to the process.
•	Exporter ships with defaults; you can customize later.
1.1 Install Prometheus Oracle Exporter
cd /etc
sudo wget https://github.com/iamseth/oracledb_exporter/releases/download/v0.6.0/oracledb_exporter-0.6.0.linux-amd64.tar.gz
sudo tar -xvf oracledb_exporter-0.6.0.linux-amd64.tar.gz
sudo mv oracledb_exporter-0.6.0.linux-amd64 oracledb_exporter

# ls -lt /usr/local/bin/oracledb_exporter
-rwxr-xr-x. 1 root root 26185988 Jan 28 15:46 /usr/local/bin/oracledb_exporter
-- Make oracle as owner for oracledb_exporter
# chown -R oracle: /etc/oracledb_exporter
# chmod 600 /etc/oracledb_exporter
•	Place the exporter binary at /usr/local/bin/oracledb_exporter and the metrics file at /etc/oracledb_exporter/default-metrics.toml.
•	In this case we are using Oracle user for exporter
1.2 Create system user folders & metrics file
Place the exporter binary at /usr/local/bin/oracledb_exporter and the metrics file at /etc/oracledb_exporter/default-metrics.toml.
1.3 Systemd unit (copy paste)
# /etc/systemd/system/oracledb-exporter.service
[Unit]
Description=Prometheus OracleDB Exporter
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=oracle
Group=oracle

Environment="ORACLE_HOME=/a01/app/oracle/product/19.0/db"
Environment="LD_LIBRARY_PATH=/a01/app/oracle/product/19.0/db/lib"
Environment="PATH=/a01/app/oracle/product/19.0/db/bin:/usr/local/bin:/usr/bin:/bin"

# DSN with new password
Environment="DATA_SOURCE_NAME=oracle://GRAFANAU:<Password>@<hostname>:1521/<pdb_service_name>"

ExecStart=/usr/local/bin/oracledb_exporter \
  --web.listen-address=":9162" \
  --default.metrics="/etc/oracledb_exporter/default-metrics.toml" \
  --log.level=debug

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
Validate database connection is fine which is using for DATA_SOURCE_NAME
sqlplus GRAFANAU/<Password>@//<hostname>:1521/<pdb_service_name>

SQL*Plus: Release 19.0.0.0.0 - Production on Thu Jan 29 17:02:26 2026
Version 19.27.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.27.0.0.0

SQL>
1.4 Start & verify
Create Systemd Service
[root@ ~]# cd /etc/systemd/system/
[root@]# cat oracledb-exporter.service
[Unit]
Description=Prometheus OracleDB Exporter
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=oracle
Group=oracle

Environment="ORACLE_HOME=/a01/app/oracle/product/19.0/db"
Environment="LD_LIBRARY_PATH=/a01/app/oracle/product/19.0/db/lib"
Environment="PATH=/a01/app/oracle/product/19.0/db/bin:/usr/local/bin:/usr/bin:/bin"

# DSN with new password
Environment="DATA_SOURCE_NAME=oracle://GRAFANAU:<Password>@<host name>:1521/<pdb service name>"


ExecStart=/usr/local/bin/oracledb_exporter \
  --web.listen-address=":9162" \
  --default.metrics="/etc/oracledb_exporter/default-metrics.toml" \
  --log.level=debug

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
Reload, enable, start and check service
systemctl daemon-reload
systemctl enable --now oracledb_exporter
systemctl restart oracledb_exporter
systemctl status oracledb_exporter
 
Exporter exposes
Test metrics endpoint
curl -s http://localhost:9162/metrics | head
[root@ ~]# curl -s http://localhost:9162/metrics | head
# HELP go_build_info Build information about the main Go module.
# TYPE go_build_info gauge
go_build_info{checksum="",path="github.com/iamseth/oracledb_exporter",version="(devel)"} 1
# HELP go_gc_duration_seconds A summary of the wall-time pause (stop-the-world) duration in garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 2.4258e-05
go_gc_duration_seconds{quantile="0.25"} 3.0503e-05
go_gc_duration_seconds{quantile="0.5"} 3.9459e-05
go_gc_duration_seconds{quantile="0.75"} 5.1603e-05
go_gc_duration_seconds{quantile="1"} 0.000913725
[root@ system]# curl http://md-ora2-rh9v:9162/metrics | grep oracledb*
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 14848    0 14848    0     0  17975      0 --:--:-- --:--:-- --:--:-- 17975
[root@ ~]# curl -s http://localhost:9162/metrics | grep -E 'oracledb_*'
go_build_info{checksum="",path="github.com/iamseth/oracledb_exporter",version="(devel)"} 1
# HELP oracledb_activity_execute_count Generic counter metric from v$sysstat view in Oracle.
# TYPE oracledb_activity_execute_count gauge
oracledb_activity_execute_count 1.80092775e+08
# HELP oracledb_activity_parse_count_total Generic counter metric from v$sysstat view in Oracle.
# TYPE oracledb_activity_parse_count_total gauge
oracledb_activity_parse_count_total 6.692955e+07
# HELP oracledb_activity_user_commits Generic counter metric from v$sysstat view in Oracle.
# TYPE oracledb_activity_user_commits gauge
oracledb_activity_user_commits 1.387658e+06
# HELP oracledb_activity_user_rollbacks Generic counter metric from v$sysstat view in Oracle.
# TYPE oracledb_activity_user_rollbacks gauge
oracledb_activity_user_rollbacks 161783
# HELP oracledb_asm_diskgroup_free Free space available on ASM disk group.
# TYPE oracledb_asm_diskgroup_free gauge
oracledb_asm_diskgroup_free{name="CSS_DG"} 5.3347352576e+10
oracledb_asm_diskgroup_free{name="MD_DATA_DG"} 1.563818983424e+12
oracledb_asm_diskgroup_free{name="MD_FRA_DG"} 7.74468796416e+11
# HELP oracledb_asm_diskgroup_total Total size of ASM disk group.
# TYPE oracledb_asm_diskgroup_total gauge
oracledb_asm_diskgroup_total{name="CSS_DG"} 5.36870912e+10
oracledb_asm_diskgroup_total{name="MD_DATA_DG"} 8.01011400704e+12
oracledb_asm_diskgroup_total{name="MD_FRA_DG"} 8.01011400704e+11
# HELP oracledb_exporter_last_scrape_duration_seconds Duration of the last scrape of metrics from Oracle DB.
# TYPE oracledb_exporter_last_scrape_duration_seconds gauge
oracledb_exporter_last_scrape_duration_seconds 1.029847763
# HELP oracledb_exporter_last_scrape_error Whether the last scrape of metrics from Oracle DB resulted in an error (1 for error, 0 for success).
# TYPE oracledb_exporter_last_scrape_error gauge
oracledb_exporter_last_scrape_error 0
# HELP oracledb_exporter_scrapes_total Total number of times Oracle DB was scraped for metrics.
# TYPE oracledb_exporter_scrapes_total counter
oracledb_exporter_scrapes_total 8
# HELP oracledb_process_count Gauge metric with count of processes.
# TYPE oracledb_process_count gauge
oracledb_process_count 105
# HELP oracledb_sessions_value Gauge metric with count of sessions by status and type.
# TYPE oracledb_sessions_value gauge
oracledb_sessions_value{status="ACTIVE",type="BACKGROUND"} 88
oracledb_sessions_value{status="ACTIVE",type="USER"} 7
oracledb_sessions_value{status="INACTIVE",type="USER"} 3
# HELP oracledb_tablespace_bytes Generic counter metric of tablespaces bytes in Oracle.
# TYPE oracledb_tablespace_bytes gauge
oracledb_tablespace_bytes{tablespace="AUDITDATA",type="PERMANENT"} 9.74782464e+08
oracledb_tablespace_bytes{tablespace="DIUSERS",type="PERMANENT"} 4.380622848e+10
oracledb_tablespace_bytes{tablespace="EDDATA",type="PERMANENT"} 2.02008690688e+11
oracledb_tablespace_bytes{tablespace="SSBIDATA",type="PERMANENT"} 6.51374821376e+11
oracledb_tablespace_bytes{tablespace="SYSAUX",type="PERMANENT"} 1.673199616e+10
oracledb_tablespace_bytes{tablespace="SYSTEM",type="PERMANENT"} 8.39385088e+08
oracledb_tablespace_bytes{tablespace="TEMP",type="TEMPORARY"} 0
oracledb_tablespace_bytes{tablespace="UNDOTBS1",type="UNDO"} 7.589134336e+09
oracledb_tablespace_bytes{tablespace="UNDOTBS2",type="UNDO"} 1.7959616512e+10
oracledb_tablespace_bytes{tablespace="USERS",type="PERMANENT"} 1.179648e+06
# HELP oracledb_tablespace_free_bytes Generic counter metric of tablespaces free bytes in Oracle.
# TYPE oracledb_tablespace_free_bytes gauge
oracledb_tablespace_free_bytes{tablespace="AUDITDATA",type="PERMANENT"} 1.02101286912e+11
oracledb_tablespace_free_bytes{tablespace="DIUSERS",type="PERMANENT"} 2.9978066944e+11
oracledb_tablespace_free_bytes{tablespace="EDDATA",type="PERMANENT"} 3.8502137856e+10
oracledb_tablespace_free_bytes{tablespace="SSBIDATA",type="PERMANENT"} 2.7632115712e+11
oracledb_tablespace_free_bytes{tablespace="SYSAUX",type="PERMANENT"} 1.7626693632e+10
oracledb_tablespace_free_bytes{tablespace="SYSTEM",type="PERMANENT"} 3.3519304704e+10
oracledb_tablespace_free_bytes{tablespace="TEMP",type="TEMPORARY"} 1.7179344896e+11
oracledb_tablespace_free_bytes{tablespace="UNDOTBS1",type="UNDO"} 1.64204314624e+11
oracledb_tablespace_free_bytes{tablespace="UNDOTBS2",type="UNDO"} 1.53833832448e+11
oracledb_tablespace_free_bytes{tablespace="USERS",type="PERMANENT"} 3.4357510144e+10
# HELP oracledb_tablespace_max_bytes Generic counter metric of tablespaces max bytes in Oracle.
# TYPE oracledb_tablespace_max_bytes gauge
oracledb_tablespace_max_bytes{tablespace="AUDITDATA",type="PERMANENT"} 1.03076069376e+11
oracledb_tablespace_max_bytes{tablespace="DIUSERS",type="PERMANENT"} 3.4358689792e+11
oracledb_tablespace_max_bytes{tablespace="EDDATA",type="PERMANENT"} 2.40510828544e+11
oracledb_tablespace_max_bytes{tablespace="SSBIDATA",type="PERMANENT"} 9.27695978496e+11
oracledb_tablespace_max_bytes{tablespace="SYSAUX",type="PERMANENT"} 3.4358689792e+10
oracledb_tablespace_max_bytes{tablespace="SYSTEM",type="PERMANENT"} 3.4358689792e+10
oracledb_tablespace_max_bytes{tablespace="TEMP",type="TEMPORARY"} 1.7179344896e+11
oracledb_tablespace_max_bytes{tablespace="UNDOTBS1",type="UNDO"} 1.7179344896e+11
oracledb_tablespace_max_bytes{tablespace="UNDOTBS2",type="UNDO"} 1.7179344896e+11
oracledb_tablespace_max_bytes{tablespace="USERS",type="PERMANENT"} 3.4358689792e+10
# HELP oracledb_tablespace_used_percent Gauge metric showing as a percentage of how much of the tablespace has been used.
# TYPE oracledb_tablespace_used_percent gauge
oracledb_tablespace_used_percent{tablespace="AUDITDATA",type="PERMANENT"} 0.945692312387463
oracledb_tablespace_used_percent{tablespace="DIUSERS",type="PERMANENT"} 12.749679555650502
oracledb_tablespace_used_percent{tablespace="EDDATA",type="PERMANENT"} 83.99151585436567
oracledb_tablespace_used_percent{tablespace="SSBIDATA",type="PERMANENT"} 70.21425515199736
oracledb_tablespace_used_percent{tablespace="SYSAUX",type="PERMANENT"} 48.69800408948027
oracledb_tablespace_used_percent{tablespace="SYSTEM",type="PERMANENT"} 2.443006683553575
oracledb_tablespace_used_percent{tablespace="TEMP",type="TEMPORARY"} 0
oracledb_tablespace_used_percent{tablespace="UNDOTBS1",type="UNDO"} 4.417592394787438
oracledb_tablespace_used_percent{tablespace="UNDOTBS2",type="UNDO"} 10.454191717276528
oracledb_tablespace_used_percent{tablespace="USERS",type="PERMANENT"} 0.003433332316049684
# HELP oracledb_up Whether the Oracle database server is up.
# TYPE oracledb_up gauge
oracledb_up 1
•	oracledb_exporter_last_scrape_error → 1 = failed
•	oracledb_up → 0 = Oracle is DOWN or unreachable
________________________________________
Step 2 — Grant Oracle Privileges for Monitoring
Minimum privileges (execute in the same PDB that the service/DSN targets):
Create monitoring user and grant privileges
  -- Create the monitoring user "grafanau"
  CREATE USER grafanau IDENTIFIED BY <YOUR-PASSWORD>;

  -- Grant the "grafanau" user the required permissions
  -- resource → No metrics found while parsing  
  -- wait_time → No metrics found while parsing  
  -- oracledb_exporter_last_scrape_error 1
  GRANT CONNECT TO grafanau;
  GRANT SELECT ON SYS.GV_$RESOURCE_LIMIT to grafanau;
  GRANT SELECT ON SYS.V_$SESSION to grafanau;
  GRANT SELECT ON SYS.V_$WAITCLASSMETRIC to grafanau;
  GRANT SELECT ON SYS.GV_$PROCESS to grafanau;
  GRANT SELECT ON SYS.GV_$SYSSTAT to grafanau;
  GRANT SELECT ON SYS.V_$DATAFILE to grafanau;
  GRANT SELECT ON SYS.V_$ASM_DISKGROUP_STAT to grafanau;
  GRANT SELECT ON SYS.V_$SYSTEM_WAIT_CLASS to grafanau;
  GRANT SELECT ON SYS.DBA_TABLESPACE_USAGE_METRICS to grafanau;
  GRANT SELECT ON SYS.DBA_TABLESPACES to grafanau;
  GRANT SELECT ON SYS.GLOBAL_NAME to grafanau;
Additional privileges
[oracle@  ~]$ . oraenv
ORACLE_SID = [oracle] ? edtstc2
The Oracle base has been set to /a01/app/oracle
[oracle@  ~]$
SQL> sho pdbs

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         4 EDTST                          READ WRITE NO
alter session set container=EDTST;
alter user GRAFANAU identified by PassSCU2026;
GRANT CREATE SESSION TO GRAFANAU;
GRANT SELECT ANY DICTIONARY TO GRAFANAU;
GRANT SELECT_CATALOG_ROLE TO GRAFANAU;

CREATE ROLE ORACLE_EXPORTER;
GRANT SELECT ANY DICTIONARY TO ORACLE_EXPORTER;
GRANT CREATE SESSION TO ORACLE_EXPORTER;
GRANT SELECT_CATALOG_ROLE TO ORACLE_EXPORTER;
GRANT ORACLE_EXPORTER TO GRAFANAU;
V$ views are public synonyms; you grant on the underlying SYS.V_$… objects. This is a frequent gotcha and required for non DBA users to select those views
Multitenant: @V$RESOURCE_LIMIT@ is global (CDB); in a PDB it can legitimately return 0 rows. If you want the exporter’s default “resource” context to work as is, connect the exporter to a CDB root service; otherwise comment/remove that block from the default metrics file.
Validation (as GRAFANAU via the same service):
SELECT COUNT(*) FROM dba_tablespaces;
SELECT COUNT(*) FROM v$sysstat;
SELECT COUNT(*) FROM v$resource_limit;
Step 3 — Confirm Exporter Health
# curl -s http://localhost:9162/metrics | grep -E 'oracledb_up|oracledb_exporter_last_scrape_error'
# HELP oracledb_exporter_last_scrape_error Whether the last scrape of metrics from Oracle DB resulted in an error (1 for error, 0 for success).
# TYPE oracledb_exporter_last_scrape_error gauge
oracledb_exporter_last_scrape_error 0
# HELP oracledb_up Whether the Oracle database server is up.
# TYPE oracledb_up gauge
oracledb_up 1
# expect: oracledbup 1  and  oracledbexporterlastscrapeerror 0
If errors persist, enable debug once: add --log.level=debug (only once, do not duplicate) to ExecStart, restart, and read journalctl.
-- Turn on debug logs (temporarily) to see the exact failing context
sudo systemctl stop oracledb-exporter
sudo sed -i 's|oracledb_exporter |oracledb_exporter --log.level=debug |' /etc/systemd/system/oracledb-exporter.service
sudo systemctl daemon-reload
sudo systemctl start oracledb-exporter
journalctl -u oracledb-exporter -o cat -n 200
________________________________________
Step 4 — Install Grafana Alloy (Grafana Agent) & Configure Scrape + Remote Write
4.1 Alloy config
Step 1: Install Required System Packages - Only if not installed
## Verify installed or not
dnf list installed | grep grafana
alloy.x86_64                                     1.9.2-1                          @grafana
## If need to install -- get the correct tar file and installed
dnf install -y wget tar unzip
# /etc/alloy/config.alloy
[root@ ~]# cat /etc/alloy/config.alloy
remotecfg {
    url = "https://fleet-management-***.grafana.net"
    basic_auth {
        username      = "<username>"
        password      = "XXXXXXXXXXXX=="
    }

    id             = constants.hostname
    attributes     = {"env" = "dev"}
    poll_frequency = "30m"
}
Configure Fleet Management
 
// ---- Scrape the local OracleDB Exporter ----
prometheus.scrape "EDTSTC_MD" {
	targets = [
		{
			__address__ = "localhost:9162",
			job         = "oracledb_exporter",
		},
	]
	forward_to = [prometheus.relabel.integrations_node_exporter.receiver]
}

//scrape_interval = "30s"
//scrape_timeout  = "10s"

prometheus.relabel "integrations_node_exporter" {
	forward_to = [prometheus.remote_write.metrics_service.receiver]

	rule {
		target_label = "instance"
		replacement  = constants.hostname
	}

	rule {
		target_label = "job"
		replacement  = "integrations/orabledb_exporter"
	}
}

// ---- Send metrics to Grafana Mimir ----
prometheus.remote_write "metrics_service" {
	endpoint {
		url = "https://mimir-prd.grafana.scu.edu.au/api/v1/push"

		queue_config {
			min_backoff = "10s"
			max_backoff = "2m"
		}

		basic_auth {
			username = "username"
			password = sys.env("AZURE_MIMIR_PASSWORD")
		}

		tls_config {
			min_version = "TLS12"
			ca_pem      = "-----BEGIN CERTIFICATE-----\wxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxSAwP\n-----END CERTIFICATE-----\n"
		}
	}
}
Why this works: prometheus.scrape collects the exporter’s metrics and feeds them to prometheus.remote_write, which buffers to a WAL and streams to Mimir at /api/v1/push. The component and endpoint syntax are defined in Alloy docs; Mimir’s ingestion path is confirmed in its HTTP API.
4.2 Enable/Start Alloy
# If installed as a service:
sudo systemctl enable --now grafana-alloy
systemctl restart alloy
systemctl status alloy
[root@ ~]# systemctl status alloy
● alloy.service - Vendor-agnostic OpenTelemetry Collector distribution with programmable pipelines
     Loaded: loaded (/usr/lib/systemd/system/alloy.service; enabled; preset: disabled)
     Active: active (running) since Fri 2026-01-30 14:17:19 AEDT; 53min ago
       Docs: https://grafana.com/docs/alloy
   Main PID: 3507067 (alloy)
      Tasks: 12 (limit: 820560)
     Memory: 103.9M
        CPU: 16.026s
     CGroup: /system.slice/alloy.service
             └─3507067 /usr/bin/alloy run --storage.path=/var/lib/alloy/data /etc/alloy/config.alloy

Jan 30 14:17:19 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:19.9461821Z level=info msg="finished node evaluation" controller_path=/ controller_id=remotecfg trace_id=078ebd65559a5d8b300c7274ae348429>
Jan 30 14:17:19 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:19.946190673Z level=info msg="finished node evaluation" controller_path=/ controller_id=remotecfg trace_id=078ebd65559a5d8b300c7274ae3484>
Jan 30 14:17:19 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:19.946197357Z level=info msg="finished complete graph evaluation" controller_path=/ controller_id=remotecfg trace_id=078ebd65559a5d8b300c>
Jan 30 14:17:19 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:19.981517881Z level=info msg="scheduling loaded components and services"
Jan 30 14:17:19 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:19.981594207Z level=info msg="scheduling loaded components and services"
Jan 30 14:17:19 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:19.981619056Z level=info msg="scheduling loaded components and services"
Jan 30 14:17:19 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:19.981927049Z level=info msg="scheduling loaded components and services"
Jan 30 14:17:37 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:37.148237725Z level=info msg="Done replaying WAL" component_path=/remotecfg/dev_system_metrics_exporter_linux.default component_id=promet>
Jan 30 14:17:49 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:17:49.896270405Z level=info msg="Done replaying WAL" component_path=/remotecfg/OracleDB_Metrics.default component_id=prometheus.remote_write>
Jan 30 14:18:06 md-ora2-rh9v alloy[3507067]: ts=2026-01-30T03:18:06.378005416Z level=info msg="Done replaying WAL" component_path=/remotecfg/dev_self_monitoring_metrics.default component_id=prometheus.r>
lines 1-21/21 (END)
________________________________________
Step 5 — Verify Mimir Ingestion & Query
Ingestion: confirm Alloy logs show remote_write HTTP 200 to Mimir; Mimir exposes status and service endpoints (build info, readiness, user limits).
Query: Mimir is Prometheus compatible for queries. Use the query endpoint under your configured.
  
Click “SAVE”.
 
Click “Save”
________________________________________
Step 6 — Configure Grafana (Data Source + Dashboard)
6.1 Add Prometheus data source pointing at Mimir
•	In Grafana → Connections → Data sources → Prometheus
•	Set URL to your Mimir query base (Prometheus API). This prefix depends on your deployment; Mimir documents all query endpoints and compatible paths.
•	If multi tenant, add the X Scope OrgID header (or configure auth).
•	Save & Test → “Data source is working.”
6.2 Quick sanity in Explore
Run these in Grafana Explore (Prometheus/Mimir data source):
oracledbup
 
  
________________________________________
Troubleshooting Appendix
•	Exporter won’t start / exits immediately: Most common cause is duplicate flags (e.g., --log.level repeated). Check journalctl; the binary rejects duplicate flags.
•	DSN issues: v0.5+ requires oracle://user:pass\@host:port/service. URL encode special characters; in systemd unit files, use %% to pass a literal %.
•	ORA 00942 / empty metrics: Grant on SYS.V_$… not on V$ synonyms; validate selects as the monitoring user in the same PDB/service.
•	PDB vs CDB metrics: @V$RESOURCE_LIMIT@ may be empty in a PDB; query in root or remove/replace that metric block.
•	Alloy → Mimir path: Mimir remote_write endpoint is @/api/v1/push@; add X Scope OrgID for multi tenant.
•	Grafana no data: Verify Prometheus (Mimir) data source points to correct query prefix and Explore returns data for oracledb_up.
________________________________________
Reference Links
•	Oracle DB Exporter (archived README; flags, DSN, metrics, systemd example): GitHub – iamseth/oracledb_exporter
•	Grants for V$ views via underlying SYS.V_$ objects: TechGoEasy – Grant access to V$ views
•	V$RESOURCE_LIMIT is global; PDBs can show 0 rows: Oracle 19c Database Reference – V$RESOURCE_LIMIT
•	Grafana Alloy: collect Prometheus metrics & remote_write: Alloy – Collect Prometheus metrics, Alloy – prometheus.remote_write
•	Grafana Mimir HTTP API (remote_write /api/v1/push, Prometheus compatible query endpoints): Mimir HTTP API
•	Grafana Mimir product docs (getting started/overview): Mimir docs
________________________________________
Copy ready Files (as attachments or code blocks)
A) Systemd service — OracleDB Exporter
# /etc/systemd/system/oracledb-exporter.service
[root@ ~]# cat /etc/systemd/system/oracledb-exporter.service
[Unit]
Description=Prometheus OracleDB Exporter
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=oracle
Group=oracle

Environment="ORACLE_HOME=/a01/app/oracle/product/19.0/db"
Environment="LD_LIBRARY_PATH=/a01/app/oracle/product/19.0/db/lib"
Environment="PATH=/a01/app/oracle/product/19.0/db/bin:/usr/local/bin:/usr/bin:/bin"

# DSN with new password
Environment="DATA_SOURCE_NAME=oracle://GRAFANAU:PassSCU2026@<host name>:1521/<pdb service name>"

ExecStart=/usr/local/bin/oracledb_exporter \
  --web.listen-address=":9162" \
  --default.metrics="/etc/oracledb_exporter/default-metrics.toml" \
  --log.level=debug

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
B) Grafana Alloy config
[root@ ~]# cat /etc/systemd/system/oracledb-exporter.service
[Unit]
Description=Prometheus OracleDB Exporter
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=oracle
Group=oracle

Environment="ORACLE_HOME=/a01/app/oracle/product/19.0/db"
Environment="LD_LIBRARY_PATH=/a01/app/oracle/product/19.0/db/lib"
Environment="PATH=/a01/app/oracle/product/19.0/db/bin:/usr/local/bin:/usr/bin:/bin"

# DSN with new password
Environment="DATA_SOURCE_NAME=oracle://GRAFANAU:Passffff6@<host name>:1521/<pdb service name>"

ExecStart=/usr/local/bin/oracledb_exporter \
  --web.listen-address=":9162" \
  --default.metrics="/etc/oracledb_exporter/default-metrics.toml" \
  --log.level=debug

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
[root@ ~]# cat  /etc/alloy/config.alloy
remotecfg {
    url = "https://fleet-management-prod-017.grafana.net"
    basic_auth {
        username      = "12345678"
        password      = "xxxxxxxxxxxxxxxxxxxxxxxxxx3V0aGVhc3QtMSJ9fQ=="
    }

    id             = constants.hostname
    attributes     = {"env" = "dev"}
    poll_frequency = "30m"
}

C) vi /etc/oracledb_exporter/default-metrics.toml
stay on the PDB and remove/replace the failing metric(s)
If you must remain connected to the PDB service, modify the metrics so the exporter stops failing when V$RESOURCE_LIMIT (and sometimes V$WAITCLASSMETRIC) return nothing in a PDB:
1) Comment out the “resource” (and optionally “wait_time”) blocks
Open metrics file: vi /etc/oracledb_exporter/default-metrics.toml > Find the sections > Comment those entire blocks (prepend # to each line), save, and restart the exporter
 
________________________________________
Verification Checklist
•	[ ] Exporter healthy: oracledb_up=1, oracledb_exporter_last_scrape_error=0.
•	[ ] Alloy remote_write 200s to Mimir; no WAL backlog.
•	[ ] Mimir query API returns data for oracledb_up.
•	[ ] Grafana data source “working” and panels populated.
