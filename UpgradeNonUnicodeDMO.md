- [Creating the Non Unicode DMO DB](#creating-the-non-unicode-dmo-db)
    - [Plan](#plan)
    - [Clearing Upgrade Image](#clearing-upgrade-image)
    - [Fresh Install of Upgrade for DMO DB](#fresh-install-of-upgrade-for-dmo-db)
    - [Setting Up the Install Workstation](#setting-up-the-install-workstation)
    - [Creating a database manually on UNIX](#creating-a-database-manually-on-unix)
    - [Generating Scripts to run on DMO](#generating-scripts-to-run-on-dmo)
    - [Checking the Database](#checking-the-database)
    - [TODO](#todo)
    - [Errors](#errors)
        - [Installation Error](#installation-error)
        - [Scripts Generation Error](#scripts-generation-error)

# Creating the Non Unicode DMO DB

## Plan

* Clear the Upgrade Image from the Server to make sure we get a clean install. Follow steps in the following document `PeopleSoft_9.2_Application_Installation_on_Oracle_PeopleSoft_PeopleTools_8.56_102017.pdf` Chapter 2, Task 2-7. 

* Once the Server is clean perform a mid-tier fresh install using the e Upgrade DPK by following instruction from `E-DPK: How to Install DPK Non-Unicode Upgrade Source Image (Doc ID 2380490.1)` 

>Note: Oracle said not to use full tier as it can has issues later in the upgrade process

* Install Application, Tools and Patch on an AIX machine, create the web server, app server and process scheduler and connect to the DEMO DB to access the PIA


## Clearing Upgrade Image

* Open Command Prompt as Admin and go to the `DPK_INSTALL\setup` directory. 
* Run the following command `psft-dpk-setup.bat --cleanup`
* Delete the Base_DIR `C:\app\psft`

> Note: Avoid the below steps as in the mid-tier installation, Puppet is not installed, hence you need Puppet to be there 

* Uninstall Puppet Agent, Control Panel->Uninstall Programs-Puppet Agent(64-bit)->Uninstall
* Search for Puppet installation folders, such as `C:\Program Files\Puppet Labs` and Delete
* Check the Windows Registry for the following by entering `regedit` in Run and delete `HKEY_LOCAL_MACHINE > SOFTWARE > Wow6432Node > Puppet Labs`

## Fresh Install of Upgrade for DMO DB

* Using the following Documents
    1. `E-DPK: How to Install DPK Non-Unicode Upgrade Source Image (Doc ID 2380490.1)`
    2. `PeopleTools_8.56_Installation_for_Oracle_062017.pdf` From Chapter 11 and
    3. `PeopleSoft_9.2_Application_Installation_on_Oracle_PeopleSoft_PeopleTools_8.56_102017.pdf` From Chapter 5

* Download the 11 Zip files for Native OS DPK Upgrade Image, Extract Here the 1st Zip file
* Open the cmd as Admin and navigate to the DPK_INSTALL/setup folder and run the following command `psft-dpk-setup.bat --env_type midtier --deploy_only --deploy_type all` This command extracts the Zip Files
* Verifying Puppet exist 

>Note: If it doesn't exist then this type of install is failing, so puppet should be pre installed before trying this step

*  People Soft base folder `C:\app\psft`
* Enter the PeopleSoft database platform `[ORACLE]`
* Is the PeopleSoft database Unicode: `n`
* Do you want Multi Lan Support in PeopleSoft Database: `y`
*  Are you happy with your answers: `y`
*  When prompted with
* Do you want to continue with the default initialization process? [y|n]: enter **N** to exit

```shell
Do you want to continue with the default initialization process? [y|n]: n

You have decided not to continue with the default PeopleSoft environment
setup process. Any customizations to the PeopleSoft environment should be
done in the Hiera YAML file 'psft_customizations.yaml' and place it under
[C:\app\psft\dpk\puppet\production\data] folder. After making the
necessary customizations, run the following commands to continue with the
setup of PeopleSoft environment.

cd /d C:\app\psft\dpk\puppet\production\manifests
"C:\Program Files\Puppet Labs\Puppet\bin\puppet.bat" apply --confdir=C:\app\p
sft\dpk\puppet site.pp --debug --trace

Exiting the PeopleSoft environment setup process.

The PeopleSoft Environment Setup Process Ended.
```

* create a text file called `C:\app\psft\dpk\puppet\production\data\psft_customizations.yaml`with the following contents: Verify that the format is correct in `yamllint.com`

```yaml

---
ps_home:
  db_type: "%{hiera('db_platform')}"
  unicode_db: false
  location: "%{hiera('ps_home_location')}"

```
* Open the cmd as Admin and navigate to `C:\app\psft\dpk\puppet\production\manifests`
* `set TEMP=c:/temp`
* `set TMP=c:/temp`
* `echo %TEMP%` and `echo %TMP%` to verify the variables are set properly
* Run the following command `"C:\Program Files\Puppet Labs\Puppet\bin\puppet.bat" apply --confdir=C:\app\psft\dpk\puppet site.pp --debug --trace --logdest C:\temp\install2.log`
* Read More about error here [Installation Error](###Installation-Error)


## Setting Up the Install Workstation

* Setup config manager by running `pscfg.exe` from `C:\app\psft\pt\ps_home8.56.05\bin\client\winx86`
 
```
Database Type: Oracle
DB Name: CMPSDMO
USER ID: PS
Connect ID: people
Connect Password: ******
```
* Profile-->Process Schduler

![](processSchedulerConfigManager.png)

* Profile-->Common

![](commonTabConfigManager.png)

* Install WorkStation, Go to Config Manager-->Client Setup-->Check App Designer, Config Manager, Data Mover and Install Work Station and Click Apply. You will get a pop up stating Workstation Installation Complete.


## Creating a database manually on UNIX


On the AIX make sure Oracle RDBMS Software is installed and `ORACLE_HOME` and `ORACLE_HOME/BIN` is setup


Below are the steps which we need to run for CMPSDMO. A side note, in the PeopleSoft Docs, there are two type of DB. The below is for non-Multitenant.)

*	Verify that the init.ora has the following two. If not you will have to edit the file and restart DB
```

    DB_BLOCK_SIZE > 8192 
    NLS_LENGTH_SEMANTICS = BYTE
```
Also, below is from the Oracle Documentation, verify we have the two variables

```
If you are creating pluggable databases, set <SID> to the database name for the CDB (root database). This
documentation uses PDB_SERVICE_NAME to refer to the PDB.
Create an init<SID>.ora file as described in Creating an INIT<SID>.ORA File, and append the following line in
the init<SID>.ora:
enable_pluggable_database=true
Note. It is strongly advisable to set the value of the parameter MEMORY_TARGET, otherwise you may get the
error "ORA-04031: unable to allocate string bytes of shared memory" (where string represents the memory size)
while creating the database. For information on MEMORY_TARGET, see the Oracle Database documentation.
```

Have to run the below scripts as sysdba in the following order

*	Run CREATEDB.SQL (You will have to edit the SID, Logfile and datafile locations) as sysdba
*	UTLSPACE.SQL (SID and Path Changes)
*	DBOWNER.SQL (No edit required)
*	PTDDL.sql (SID and Path Changes)
*	CSDDL.SQL (SID and Path Changes)

```
Note. Compare the sizes of the PeopleTools tablespaces in CSDDL.SQL with the tablespaces in PTDDL.SQL. If
the tablespace sizes in PTDDL.SQL are larger, increase the PeopleTools tablespace sizes in CSDDL.SQL to be at least as large as those in PTDDL.SQL.
```
```
Note. For multilanguage installs, you need to increase the size of the PTTBL, PSIMAGE, and PSINDEX
tablespaces. Refer to the comments in the DDL scripts for further details regarding the incremental increase for
each additional language.( I INSTALLED UPGRADE IMAGE WITH MULTILANGUAGES ENABLED)
```

*	PSSROLES.SQL(NO EDIT REQUIRED)

Run the below scripts as System User (system/manager)

*	RUN PSADMIN.SQL(NO EDIT REQUIRED) AS system
*	CONNECT.SQL (No edit required) 
*	NLS_LANG = AMERICAN_AMERICAN.UTF8 (Not sure if this is important, I’ve made this change in Patra)

Once the above steps are completed. I’ll start with the Data Mover to generate some other scripts.

Work with DBA to execute the above Scripts

## Generating Scripts to run on DMO

* Open Data Mover
* Login using access ID `sysadm/***`
* File->DB Setup
* Select `Oracle`, DB Type `Non-unicode` Select Character Set `Western European ISO 8859-1`
* Select Database Type `Demo` and add `CS- US English`

Application Server ID/Server Password = PS/**
disabled Global Password

![](dbSetupParameters.png)


* Script running, approximate time 4 hours
* Scripts Completed Successfully. 5 Logs created. `csengl`, `csengs`, `encrypt`, `triggers` and `views`
* Received an error during this, refer - [Scripts Generation Error](###Scripts-Generation-Error)

## Checking the Database

Need to run ddaudit.sqr and sysaudit.sqr

* Go to `C:\app\psft\product\pt856\bin\sqr\ora\BINW` and run sqrw.exe
* Select the `ddaudit.sqr/sysaudit.sqr` from `C:\app\psft\product\pt856\sqr`
* Add the require details
* Add 
```
-ZIFC:\app\psft\product\pt856\sqr\pssqr.ini -iC:\app\psft\product\pt856\sqr -fC:\temp2\dddaudit.htm -keep -printer:ht -OC:\temp\sqr.log
```
* Do the above for `setpsace.sqr`
* Make sure all results are OK in the htm file located in `C:\temp2`

The database is created. Make sure you are able to login using App Designer via `PS` and in sqlplus using `sysadm`

## TODO

1. Installing App and Tools in AIX
2. Creating App Server, Web servers and Process Schedulers
3. Creating PIA to connect to DMO

## Errors

### Installation Error

```
Error: Evaluation Error: Error while evaluating a Function Call, Could not find 
class ::pt_role::pt_apptools_deployment for patra.scsd.nevada.edu at C:/app/psft 
/dpk/puppet/production/manifests/site.pp:22:3 on node patra.scsd.nevada.edu 
```

Opened a SR `Severity 3		SR 3-17656527821 : Upgrade Issue: Installing DPK Non-Unicode Upgrade Source Image, erroring out`

Got on a call with Oracle, the issue was in either the pdf command format. It was taking some extra characters which was corrupting files somewhere without telling us. Or it had to do with the psft_customizations.yaml. I copied the code from SR, but it was advised I paste the code in `yamllint.com` and then copy the verified code. 

**Solution:**
* Cleaned Up the DPK install
* Copied all the commands to notepad before running 
* Copied psft_customizations.yaml from the above link

### Scripts Generation Error
```
File: E:\pt85607c-retail\peopletools\src\psdmt\DMTSCR.cppSQL error. Stmt #: 845  Error Position: 0  Return: 805 - ORA-00001: unique constraint (PS.PS_PSDBOWNER) violated 
Failed SQL stmt: INSERT INTO PS.PSDBOWNER VALUES('CMPSDMO', 'SYSADM')
```

**Solution**

DBA recreated CMPSDMO

Copied grant from PRD

![](dbSetupParameters2.png)

Scripts Completed Successfully. 5 Logs created. `csengl`, `csengs`, `encrypt`, `triggers` and `views`