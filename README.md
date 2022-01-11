## IBM API Connect v10.0.x on OCP: Postgres pod failures (adding space to PVCs)
This is a specific issue on APIC v10.0.x where Postgres pods are failing due to insufficient pvc space.  
Therefore this article goes through analyzing the pods logs, adding pvc space, and refeshing the pods.  

### Analyzing the pod logs  
1. Log into the bastion host and into the OCP cluster to look at the postgres pods that in error status:  
``` oc get pods ```  

![image](https://user-images.githubusercontent.com/66093865/148870555-18ba9de1-2f69-48e9-aedd-113316aac6da.png)  

2. Check the pods logs with ```oc logs <postgres-pods-name-that-is-in-error-state>```  
Should look something like this (contain time out or connection refused errors):
```  
# oc logs backrest-backup-management-pa-postgres-2mrhd
time="2022-01-10T21:52:47Z" level=info msg="pgo-backrest starts"
time="2022-01-10T21:52:47Z" level=info msg="debug flag set to false"
time="2022-01-10T21:52:47Z" level=info msg="backrest backup command requested"
time="2022-01-10T21:52:47Z" level=info msg="command to execute is [pgbackrest backup  --db-host=x.x.x.x --db-path=/pgdata/management-pa-postgres]"
time="2022-01-10T21:52:47Z" level=info msg="command is pgbackrest backup  --db-host=x.x.x.x --db-path=/pgdata/management-pa-postgres "
time="2022-01-10T21:54:05Z" level=error msg="command terminated with exit code 82"
time="2022-01-10T21:54:05Z" level=info msg="output=[]"
time="2022-01-10T21:54:05Z" level=info msg="stderr=[WARN: option 'repo1-retention-full' is not set for 'repo1-retention-full-type=count', the repository may run out of space\n      HINT: to retain full backups indefinitely (without warning), set option 'repo1-retention-full' to the maximum.\nWARN: a timeline switch has occurred since the 20210904-161556F_20220103-064836I backup, enabling delta checksum\n      HINT: this is normal after restoring from backup or promoting a standby.\nWARN: resumable backup 20210904-161556F_20220103-094413I of same type exists -- remove invalid files and resume\nERROR: [082]: WAL segment 000002A0000001DB000000A1 was not archived before the 60000ms timeout\n       HINT: check the archive_command to ensure that all options are correct (especially --stanza).\n       HINT: check the PostgreSQL server log for errors.\n]"
time="2022-01-10T21:54:05Z" level=error msg="command terminated with exit code 82"
```  

```  
# oc logs backrest-backup-management-pa-postgres-lz7xd
time="2022-01-10T22:19:48Z" level=info msg="pgo-backrest starts"
time="2022-01-10T22:19:48Z" level=info msg="debug flag set to false"
time="2022-01-10T22:19:48Z" level=info msg="backrest backup command requested"
time="2022-01-10T22:19:48Z" level=info msg="command to execute is [pgbackrest backup  --db-host=x.x.x.x --db-path=/pgdata/management-pa-postgres]"
time="2022-01-10T22:19:48Z" level=info msg="command is pgbackrest backup  --db-host=x.x.x.x --db-path=/pgdata/management-pa-postgres "
time="2022-01-10T22:19:48Z" level=error msg="error dialing backend: dial tcp x.x.x.x:10250: connect: connection refused"
time="2022-01-10T22:19:48Z" level=info msg="output=[]"
time="2022-01-10T22:19:48Z" level=info msg="stderr=[]"
time="2022-01-10T22:19:48Z" level=error msg="error dialing backend: dial tcp x.x.x.x:10250: connect: connection refused"
```  

### Adding space to the PVC  
3. Get the list of PVCs with ```oc get pvc``` then locate the pvc that contains **postgres-wal**, and issue the following command to edit the yml ```oc edit <management-postgres-wal>```  
4. Update the stoage in the spec section to a higher value to accomodate more of the space.  
![image](https://user-images.githubusercontent.com/66093865/148871659-009ff0d5-f2ec-439b-9b97-088f3ef93707.png)  

5.	Once saved, refresh all the postgres pods:  
```# oc get pods ```  
```# oc delete <backrest-backup…postgres..> <etc> <etc> ```  
![image](https://user-images.githubusercontent.com/66093865/148871968-c1842051-601e-4650-949a-f44a1abffc57.png)  

### Restarting the node associated to any error pods  
6.	If errors persist, locate the node that the error postgres pod is on, and reboot it with the following commands:  
```# oc get pods -owide```  
![image](https://user-images.githubusercontent.com/66093865/148872004-deb767fd-ffa6-401b-9136-2a194955ad22.png)  

```# oc debug node/<nodename> ```  
```# chroot /host ```  
```# shutdown –r now ```  
 
7.	Once the node is reboot you may need to refresh any pods that are in error state, and that should completely remove any stale pods in the cluster. You may watch the pods with the following command:  
```# watch oc get pods ```  


Thank you Felix Rafael Gonzalez Vargas for the details (IBM Slack@felixg)


