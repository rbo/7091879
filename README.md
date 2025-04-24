

For general requirements and installation instructions, please consult the corresponding installation guides:

- [Installation Guidance: SAP Edge Integration Cell on OpenShift](https://access.redhat.com/articles/7085063)

## 1. OpenShift Container Platform validation version matrix {#validation-version-matrix}

The following version combinations of SAP Edge Integration Cell (EIC), OpenShift Container Platform (OCP), and NetApp Trident have been validated:

SAP Product                   | OpenShift Container Platform | Infrastructure and (Storage)
------------                  |------------------------------| ----------------------------
SAP EIC **8.22**              | **4.16**                     | Bare Metal (NetApp Trident 25.02.1 (Backend Type: ontap-san))

> **Note:** Trident can be used with various protocols depending on your setup:
~~~
 NFS: supported with the Backend types: ontap-nas, ontap-nas-economy, ontap-nas-flexgroup, azure-netapp-files, or gcp-cvs.
 iSCSI: supported with the Backend types: ontap-san, ontap-san-economy, or solidfire-san.  
 FC: supported with the Backend types: ontap-san.
 NVMe/TCP: supported with the Backend types: ontap-san.
~~~
It is important to ensure that the worker nodes are prepared for the used protocol. For more information, see the [NetApp documentation](https://docs.netapp.com/us-en/trident/trident-use/worker-node-prep.html).

**Important:** Using NFS is **NOT** recommended, as Solace component inside Edge Integration Cell advises against NFS usage. In this article, we will use iSCSI as an example. You can also configure NVMe in a similar way.

### 1.1. Supportability Note {#supportability-note}

The validation of NetApp Trident storage is not conducted by Red Hat. Red Hat does not directly support NetApp Trident software or NetApp hardware.

If you encounter issues related to Trident or its integration with NetApp storage solutions, existing NetApp customers can submit a support case via the [NetApp Support Portal](https://mysupport.netapp.com/). For architecture or pre-sales inquiries, please contact your NetApp account manager.

## 2. Requirements {#requirements}

### 2.1. Hardware/VM and OS Requirements {#hw-os-requirements}

For OpenShift Container Platform (OCP) and SAP Edge Integration Cell (EIC) requirements, please refer to the [Prerequisites for Installing SAP Integration Suite Edge Integration Cell](https://access.redhat.com/articles/7085063#sap-integration-suite-edge-integration-cell-on-openshift).

We will assume that a management host is available.

**NetApp Requirements:**

Please refer to the Trident [Requirements](https://docs.netapp.com/us-en/trident/trident-get-started/requirements.html) section for detailed information.

**Important Notice:**

~~~
Trident strictly enforces the use of a multipathing configuration in SAN environments, with a recommended value of `find_multipaths: no` in the `multipath.conf` file.

Using a non-multipathing configuration or setting `find_multipaths: yes` or `find_multipaths: smart` in the `multipath.conf` file will result in mount failures. Trident is enforcing the use of `find_multipaths: no` since the 21.07 release.
~~~
This can either be done by using the node prep functionality in the Trident Orchestrator configuration or by an OpenShift MachineConfig. Both will be provided as an example below.

### 2.2. Software Requirements {#sw-requirements}

An OpenShift Container Platform (OCP) cluster must be installed and configured.

## 3. NetApp Trident Installation {#install-trident}

The installation can be done by a Certified Operator from the OperatorHub.

### 3.1. OCP Nodes Preparation {#prepare-ocp-nodes}

1. In Red Hat OpenShift Container Platform (OCP) 4.16, the `iscsid` service is enabled by default. Therefore, iSCSI Initiator IDs need be determined for all OCP nodes.

    ```bash
    # No need to modify the command - copy and paste should work
    oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | while read -r node; do oc debug node/"$node" -- bash -c "cat /etc/iscsi/initiatorname.iscsi"; done
    ```

   Example output:
    ```
    InitiatorName=iqn.1994-05.com.redhat:12636781b54
    InitiatorName=iqn.1994-05.com.redhat:5db94959f
    InitiatorName=iqn.1994-05.com.redhat:9ae4359f9e30
    ```
    Ensure that the IQN is unique for every host in your cluster.
#### 3.1.1. Variant A: Configure Nodes leveraging the node prep functionality of Trident {#node-configure-mc}

Ensure that your Trident Orchestrator configuration contains the following lines in the spec section:
```yaml
spec:
  nodePrep:
  - iscsi
```

#### 3.1.1. Variant B: Configure Nodes with MachineConfig {#node-configure-mc}

We need to use MachineConfig to configure `/etc/multipath.conf` and `/etc/iscsi/iscsid.conf` to meet the prerequisites listed in the worker nodes preperation section of Trident. It's easier to create this files, utilizing [Butane]([https://docs.netapp.com/us-en/trident/trident-get-started/requirements.html](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installation_configuration/installing-customizing#installation-special-config-butane_installing-customizing)) 

Here is an example butane configuration to modify both files:

```yaml
variant: openshift
version: 4.14.0
metadata:
  name: 99-worker-iscsi
  labels:
    machineconfiguration.openshift.io/role: worker

systemd:
  units:
    - name: iscsid.service
      enabled: true
    - name: iscsi.service
      enabled: true
    - name: multipathd.service
      enabled: true

storage:
  files:
    - path: /etc/multipath.conf
      mode: 0420
      overwrite: true
      contents:
        inline: |

            defaults {
                user_friendly_names no
                find_multipaths no
            }

            blacklist {
                device {
                    vendor “!NETAPP”
                    product “!LUN”
                }
            }

            blacklist_exceptions {
                device {
                    vendor “NETAPP”
                    product “LUN”
                }
            }
    - path: /etc/iscsi/iscsid.conf
      mode: 0600
      overwrite: true
      contents:
        inline: |

            #
            # Open-iSCSI default configuration.
            # Could be located at /etc/iscsi/iscsid.conf or ~/.iscsid.conf
            #
            # Note: To set any of these values for a specific node/session run
            # the iscsiadm --mode node --op command for the value. See the README
            # and man page for iscsiadm for details on the --op command.
            #

            ######################
            # iscsid daemon config
            ######################
            #
            # If you want iscsid to start the first time an iscsi tool
            # needs to access it, instead of starting it when the init
            # scripts run, set the iscsid startup command here. This
            # should normally only need to be done by distro package
            # maintainers. If you leave the iscsid daemon running all
            # the time then leave this attribute commented out.
            #
            # Default for Fedora and RHEL. (uncomment to activate).
            iscsid.startup = /bin/systemctl start iscsid.socket iscsiuio.socket
            # 
            # Default if you are not using systemd (uncomment to activate)
            # iscsid.startup = /usr/bin/service start iscsid

            # Check for active mounts on devices reachable through a session
            # and refuse to logout if there are any.  Defaults to "No".
            # iscsid.safe_logout = Yes

            #############################
            # NIC/HBA and driver settings
            #############################
            # open-iscsi can create a session and bind it to a NIC/HBA.
            # To set this up see the example iface config file.

            #*****************
            # Startup settings
            #*****************

            # To request that the iscsi initd scripts startup a session set to "automatic".
            # node.startup = automatic
            #
            # To manually startup the session set to "manual". The default is automatic.
            node.startup = automatic

            # For "automatic" startup nodes, setting this to "Yes" will try logins on each
            # available iface until one succeeds, and then stop.  The default "No" will try
            # logins on all available ifaces simultaneously.
            node.leading_login = No

            # *************
            # CHAP Settings
            # *************

            # To enable CHAP authentication set node.session.auth.authmethod
            # to CHAP. The default is None.
            #node.session.auth.authmethod = CHAP

            # To configure which CHAP algorithms to enable set
            # node.session.auth.chap_algs to a comma seperated list.
            # The algorithms should be listen with most prefered first.
            # Valid values are MD5, SHA1, SHA256, and SHA3-256.
            # The default is MD5.
            #node.session.auth.chap_algs = SHA3-256,SHA256,SHA1,MD5

            # To set a CHAP username and password for initiator
            # authentication by the target(s), uncomment the following lines:
            #node.session.auth.username = username
            #node.session.auth.password = password

            # To set a CHAP username and password for target(s)
            # authentication by the initiator, uncomment the following lines:
            #node.session.auth.username_in = username_in
            #node.session.auth.password_in = password_in

            # To enable CHAP authentication for a discovery session to the target
            # set discovery.sendtargets.auth.authmethod to CHAP. The default is None.
            #discovery.sendtargets.auth.authmethod = CHAP

            # To set a discovery session CHAP username and password for the initiator
            # authentication by the target(s), uncomment the following lines:
            #discovery.sendtargets.auth.username = username
            #discovery.sendtargets.auth.password = password

            # To set a discovery session CHAP username and password for target(s)
            # authentication by the initiator, uncomment the following lines:
            #discovery.sendtargets.auth.username_in = username_in
            #discovery.sendtargets.auth.password_in = password_in

            # ********
            # Timeouts
            # ********
            #
            # See the iSCSI README's Advanced Configuration section for tips
            # on setting timeouts when using multipath or doing root over iSCSI.
            #
            # To specify the length of time to wait for session re-establishment
            # before failing SCSI commands back to the application when running
            # the Linux SCSI Layer error handler, edit the line.
            # The value is in seconds and the default is 120 seconds.
            # Special values:
            # - If the value is 0, IO will be failed immediately.
            # - If the value is less than 0, IO will remain queued until the session
            # is logged back in, or until the user runs the logout command.
            node.session.timeo.replacement_timeout = 5

            # To specify the time to wait for login to complete, edit the line.
            # The value is in seconds and the default is 15 seconds.
            node.conn[0].timeo.login_timeout = 15

            # To specify the time to wait for logout to complete, edit the line.
            # The value is in seconds and the default is 15 seconds.
            node.conn[0].timeo.logout_timeout = 15

            # Time interval to wait for on connection before sending a ping.
            node.conn[0].timeo.noop_out_interval = 5

            # To specify the time to wait for a Nop-out response before failing
            # the connection, edit this line. Failing the connection will
            # cause IO to be failed back to the SCSI layer. If using dm-multipath
            # this will cause the IO to be failed to the multipath layer.
            node.conn[0].timeo.noop_out_timeout = 5

            # To specify the time to wait for abort response before
            # failing the operation and trying a logical unit reset edit the line.
            # The value is in seconds and the default is 15 seconds.
            node.session.err_timeo.abort_timeout = 15

            # To specify the time to wait for a logical unit response
            # before failing the operation and trying session re-establishment
            # edit the line.
            # The value is in seconds and the default is 30 seconds.
            node.session.err_timeo.lu_reset_timeout = 30

            # To specify the time to wait for a target response
            # before failing the operation and trying session re-establishment
            # edit the line.
            # The value is in seconds and the default is 30 seconds.
            node.session.err_timeo.tgt_reset_timeout = 30


            #******
            # Retry
            #******

            # To specify the number of times iscsid should retry a login
            # if the login attempt fails due to the node.conn[0].timeo.login_timeout
            # expiring modify the following line. Note that if the login fails
            # quickly (before node.conn[0].timeo.login_timeout fires) because the network
            # layer or the target returns an error, iscsid may retry the login more than
            # node.session.initial_login_retry_max times.
            #
            # This retry count along with node.conn[0].timeo.login_timeout
            # determines the maximum amount of time iscsid will try to
            # establish the initial login. node.session.initial_login_retry_max is
            # multiplied by the node.conn[0].timeo.login_timeout to determine the
            # maximum amount.
            #
            # The default node.session.initial_login_retry_max is 8 and
            # node.conn[0].timeo.login_timeout is 15 so we have:
            #
            # node.conn[0].timeo.login_timeout * node.session.initial_login_retry_max =
            #								120 seconds
            #
            # Valid values are any integer value. This only
            # affects the initial login. Setting it to a high value can slow
            # down the iscsi service startup. Setting it to a low value can
            # cause a session to not get logged into, if there are distuptions
            # during startup or if the network is not ready at that time.
            node.session.initial_login_retry_max = 8

            ################################
            # session and device queue depth
            ################################

            # To control how many commands the session will queue set
            # node.session.cmds_max to an integer between 2 and 2048 that is also
            # a power of 2. The default is 128.
            node.session.cmds_max = 128

            # To control the device's queue depth set node.session.queue_depth
            # to a value between 1 and 1024. The default is 32.
            node.session.queue_depth = 128

            ##################################
            # MISC SYSTEM PERFORMANCE SETTINGS
            ##################################

            # For software iscsi (iscsi_tcp) and iser (ib_iser) each session
            # has a thread used to transmit or queue data to the hardware. For
            # cxgb3i you will get a thread per host.
            #
            # Setting the thread's priority to a lower value can lead to higher throughput
            # and lower latencies. The lowest value is -20. Setting the priority to
            # a higher value, can lead to reduced IO performance, but if you are seeing
            # the iscsi or scsi threads dominate the use of the CPU then you may want
            # to set this value higher.
            #
            # Note: For cxgb3i you must set all sessions to the same value, or the
            # behavior is not defined.
            #
            # The default value is -20. The setting must be between -20 and 20.
            node.session.xmit_thread_priority = -20


            #***************
            # iSCSI settings
            #***************

            # To enable R2T flow control (i.e., the initiator must wait for an R2T
            # command before sending any data), uncomment the following line:
            #
            #node.session.iscsi.InitialR2T = Yes
            #
            # To disable R2T flow control (i.e., the initiator has an implied
            # initial R2T of "FirstBurstLength" at offset 0), uncomment the following line:
            #
            # The defaults is No.
            node.session.iscsi.InitialR2T = No

            #
            # To disable immediate data (i.e., the initiator does not send
            # unsolicited data with the iSCSI command PDU), uncomment the following line:
            #
            #node.session.iscsi.ImmediateData = No
            #
            # To enable immediate data (i.e., the initiator sends unsolicited data
            # with the iSCSI command packet), uncomment the following line:
            #
            # The default is Yes
            node.session.iscsi.ImmediateData = Yes

            # To specify the maximum number of unsolicited data bytes the initiator
            # can send in an iSCSI PDU to a target, edit the following line.
            #
            # The value is the number of bytes in the range of 512 to (2^24-1) and
            # the default is 262144
            node.session.iscsi.FirstBurstLength = 262144

            # To specify the maximum SCSI payload that the initiator will negotiate
            # with the target for, edit the following line.
            #
            # The value is the number of bytes in the range of 512 to (2^24-1) and
            # the defauls it 16776192
            node.session.iscsi.MaxBurstLength = 16776192

            # To specify the maximum number of data bytes the initiator can receive
            # in an iSCSI PDU from a target, edit the following line.
            #
            # The value is the number of bytes in the range of 512 to (2^24-1) and
            # the default is 262144
            node.conn[0].iscsi.MaxRecvDataSegmentLength = 262144

            # To specify the maximum number of data bytes the initiator will send
            # in an iSCSI PDU to the target, edit the following line.
            #
            # The value is the number of bytes in the range of 512 to (2^24-1).
            # Zero is a special case. If set to zero, the initiator will use
            # the target's MaxRecvDataSegmentLength for the MaxXmitDataSegmentLength.
            # The default is 0.
            node.conn[0].iscsi.MaxXmitDataSegmentLength = 0

            # To specify the maximum number of data bytes the initiator can receive
            # in an iSCSI PDU from a target during a discovery session, edit the
            # following line.
            #
            # The value is the number of bytes in the range of 512 to (2^24-1) and
            # the default is 32768
            # 
            discovery.sendtargets.iscsi.MaxRecvDataSegmentLength = 32768

            # To allow the targets to control the setting of the digest checking,
            # with the initiator requesting a preference of enabling the checking, uncomment
            # the following lines (Data digests are not supported.):
            #node.conn[0].iscsi.HeaderDigest = CRC32C,None

            #
            # To allow the targets to control the setting of the digest checking,
            # with the initiator requesting a preference of disabling the checking,
            # uncomment the following line:
            #node.conn[0].iscsi.HeaderDigest = None,CRC32C
            #
            # To enable CRC32C digest checking for the header and/or data part of
            # iSCSI PDUs, uncomment the following line:
            #node.conn[0].iscsi.HeaderDigest = CRC32C
            #
            # To disable digest checking for the header and/or data part of
            # iSCSI PDUs, uncomment the following line:
            #node.conn[0].iscsi.HeaderDigest = None
            #
            # The default is to never use DataDigests or HeaderDigests.
            #
            node.conn[0].iscsi.HeaderDigest = None

            # For multipath configurations, you may want more than one session to be
            # created on each iface record.  If node.session.nr_sessions is greater
            # than 1, performing a 'login' for that node will ensure that the
            # appropriate number of sessions is created.
            node.session.nr_sessions = 1

            # When iscsid starts up it recovers existing sessions, if possible.
            # If the target for a session has gone away when this occurs, the
            # iscsid daemon normally tries to reestablish each session,
            # in succession, in the background, by trying again every two
            # seconds, until all sessions are restored. This configuration
            # variable can limits the number of retries for each session.
            # For example, setting reopen_max=150 would mean that each session
            # recovery was limited to about five minutes.
            # 
            node.session.reopen_max = 0

            #************
            # Workarounds
            #************

            # Some targets like IET prefer after an initiator has sent a task
            # management function like an ABORT TASK or LOGICAL UNIT RESET, that
            # it does not respond to PDUs like R2Ts. To enable this behavior uncomment
            # the following line (The default behavior is Yes):
            node.session.iscsi.FastAbort = Yes

            # Some targets like Equalogic prefer that after an initiator has sent
            # a task management function like an ABORT TASK or LOGICAL UNIT RESET, that
            # it continue to respond to R2Ts. To enable this uncomment this line
            # node.session.iscsi.FastAbort = No

            # To prevent doing automatic scans that would add unwanted luns to the system
            # we can disable them and have sessions only do manually requested scans.
            # Automatic scans are performed on startup, on login, and on AEN/AER reception
            # on devices supporting it.  For HW drivers all sessions will use the value
            # defined in the configuration file.  This configuration option is independent
            # of scsi_mod scan parameter. (The default behavior is auto):
            node.session.scan = manual
```

### 3.2. Install NetApp Trident {#install-trident-tridentctl}

1. In OpenShift, you can use the OperatorHub to install Astra Trident. From the UI, navigate to OperatorHub, search for "Astra Trident," and initiate the installation from there. This method allows you to easily manage upgrades for the installer operator directly from OpenShift OperatorHub. Once the installation is complete, you can continue with [Trident Installation Step 4: Create the TridentOrchestrator and install Trident](https://docs.netapp.com/us-en/trident/trident-get-started/kubernetes-deploy-operator.html#step-4-create-the-tridentorchestrator-and-install-trident).

Below is an example of a TridentOrchestrator Config:

```yaml
apiVersion: trident.netapp.io/v1
kind: TridentOrchestrator
metadata:
  name: trident
  namespace: openshift-operators 
spec:
  IPv6: false
  debug: true
  nodePrep:
  - iscsi
  imageRegistry: ''
  k8sTimeout: 30
  namespace: trident
  silenceAutosupport: false
```

3. After the operator installation is complete and the TridentOrchestrator is in the "installed" status, verify the deployment by running the following command:

    ```bash
    oc get pods -n trident
    ```

   **Expected output:**

    ```
    NAME                                 READY   STATUS    RESTARTS   AGE
    trident-controller-687bf94d5-6gl2p   6/6     Running   0          6d23h
    trident-node-linux-7jb5w             2/2     Running   0          27d
    trident-node-linux-8rddh             2/2     Running   0          27d
    trident-node-linux-dp5hk             2/2     Running   0          27d
    trident-node-linux-h5zcx             2/2     Running   8          27d
    trident-node-linux-lprb4             2/2     Running   0          27d
    trident-node-linux-m6fxz             2/2     Running   8          27d
    ```

4. Create the Trident SAN backend:

A backend configuration is needed to make trident aware of which backend type and what SVM should be used. Below are two example files to get this done.

Secret to provide the credentials for the SVM:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-svm
  namespace: trident
type: Opaque
stringData:
  username: trident
  password: imverysecretdonttellanybody
```

TridentBackendConfiguration:

```yaml
apiVersion: trident.netapp.io/v1
kind: TridentBackendConfig
metadata:
  name: tbc-ontap-san
  namespace: trident
spec:
  version: 1
  backendName: ontap-san
  storageDriverName: ontap-san
  managementLIF: <ip of svm mgmt lif>
  credentials:
    name: secret-svm
```
Apply both files to the OpenShift cluster.
You can verify whether all is successful with the following command:

```bash
oc get tbc -n trident
```
```bash
NAMESPACE   NAME                BACKEND NAME      BACKEND UUID                           PHASE   STATUS
trident     tbc-ontap-san       ontap-san         17c482e4-6aa7-4a0a-b4f8-26c75eae8a59   Bound   Success
```
If there are any other state than bound or status success, investigate it by
```bash
oc describe tbc <backendname> -n trident
```
7. Create the iSCSI storage class:



    ```bash
    # cat > yamls/storage-class-san.yaml <<EOF
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: san-iscsi-ocp
    mountOptions:
    - discard
    parameters:
      backendType: ontap-san
      fsType: xfs
    provisioner: csi.trident.netapp.io
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    EOF
    ```

   Create the storage class:

    ```bash
    # oc create -f yamls/storage-class-san.yaml
    storageclass.storage.k8s.io/san-iscsi-ocp created
    ```

8. (*optional*) Mark the new storage classes as the default:

    ```bash
    # Clean existing default flag if any
    # oc annotate sc --all storageclass.kubernetes.io/is-default-class- storageclass.beta.kubernetes.io/is-default-class-
    # oc annotate sc/san-iscsi-ocp storageclass.kubernetes.io/is-default-class=true
    ```

9. Verify that the storage classes have been created:

    ```bash
    # oc get sc
    NAME                                   PROVISIONER                   AGE
    san-iscsi-ocp (default)               csi.trident.netapp.io         17s
    ```

## 4. Continue with SAP ELM/EIC {#next-steps}

Proceed with the preparation for the SAP Edge Integration Cell installation.

