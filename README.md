

For general requirements and installation instructions, please consult the corresponding installation guides:

- [Installation Guidance: SAP Edge Integration Cell on OpenShift](https://access.redhat.com/articles/7085063)

## 1. OpenShift Container Platform validation version matrix {#validation-version-matrix}

The following version combinations of SAP Edge Integration Cell (EIC), OpenShift Container Platform (OCP), and NetApp Trident have been validated:

SAP Product                   | OpenShift Container Platform | Infrastructure and (Storage)
------------                  |------------------------------| ----------------------------
SAP EIC **8.22**              | **4.16**                     | Bare Metal (NetApp Trident 24.06 (ONTAP SAN))

> **Note:** ONTAP can be used with various tools depending on your setup:
~~~
 NFS Tools: Install NFS tools if using ontap-nas, ontap-nas-economy, ontap-nas-flexgroup, azure-netapp-files, or gcp-cvs.
 iSCSI Tools: Install iSCSI tools if using ontap-san, ontap-san-economy, or solidfire-san.
 NVMe Tools: Install NVMe tools if using ontap-san for non-volatile memory express (NVMe) over TCP (NVMe/TCP) protocol.
~~~
> For more information, see the [NetApp documentation](https://docs.netapp.com/us-en/trident/trident-use/worker-node-prep.html).

**Important:** Using NFS is **NOT** recommended, as Solace component inside Edge Integration Cell advises against NFS usage. In this article, we will use iSCSI tools as an example. You can also configure NVMe tools in a similar way.

### 1.1. Supportability Note {#supportability-note}

The validation of NetApp Trident storage is not conducted by Red Hat. Red Hat does not directly support NetApp Trident software or NetApp hardware.

If you encounter issues related to Trident or its integration with NetApp storage solutions, existing NetApp customers can submit a support case via the [NetApp Support Portal](https://mysupport.netapp.com/). For architecture or pre-sales inquiries, please contact your NetApp account manager.

## 2. Requirements {#requirements}

### 2.1. Hardware/VM and OS Requirements {#hw-os-requirements}

For OpenShift Container Platform (OCP) and SAP Edge Integration Cell (EIC) requirements, please refer to the [Prerequisites for Installing SAP Integration Suite Edge Integration Cell](https://access.redhat.com/articles/7085063#sap-integration-suite-edge-integration-cell-on-openshift).

We will assume that a management host is available.

**NetApp Requirements:**

Please refer to the Astra Trident [Requirements](https://docs.netapp.com/us-en/trident/trident-get-started/requirements.html#critical-information-about-astra-trident) section for detailed information.

**Important Notice:**

~~~
Astra Trident strictly enforces the use of a multipathing configuration in SAN environments, with a recommended value of `find_multipaths: no` in the `multipath.conf` file.

Using a non-multipathing configuration or setting `find_multipaths: yes` or `find_multipaths: smart` in the `multipath.conf` file will result in mount failures. Astra Trident has recommended the use of `find_multipaths: no` since the 21.07 release.
~~~

In this case, we need to use OpenShift MachineConfig to configure the relevant nodes to run NetApp correctly. We will provide an example in the installation guidance section.

### 2.2. Software Requirements {#sw-requirements}

An OpenShift Container Platform (OCP) cluster must be installed and configured.

## 3. NetApp Trident Installation {#install-trident}

The installation must be performed from the management host.

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

2. Create an [initiator group](https://library.netapp.com/ecmdocs/ECMP1196995/html/GUID-CF01DCCD-2C24-4519-A23B-7FEF55A0D9A3.html) composed of the initiators on the NetApp ONTAP System:

    ```bash
    # ssh admin@<ClusterIP>
    grenada::> igroup create -vserver iscsi -igroup trident -protocol iscsi -ostype linux -initiator iqn.1994-05.com.redhat:12636781b54
    grenada::> igroup add trident -vserver iscsi -initiator iqn.1994-05.com.redhat:5db94959f iqn.1994-05.com.redhat:9ae4359f9e30
    grenada::> igroup show -vserver iscsi
    ```

   Example output:
    ```
    Vserver   Igroup       Protocol OS Type  Initiators
    --------- ------------ -------- -------- ------------------------------------
    iscsi     worker-0-5f663f29-7369-4062-ba08-83a495ebba5f
                               iscsi    linux    iqn.1994-05.com.redhat:12636781b54
    iscsi     worker-1-5f663f29-7369-4062-ba08-83a495ebba5f
                               iscsi    linux    iqn.1994-05.com.redhat:5db94959f
    iscsi     worker-2-5f663f29-7369-4062-ba08-83a495ebba5f
                               iscsi    linux    iqn.1994-05.com.redhat:9ae4359f9e30
    3 entries were displayed.
    ```

3. Verify that the previous operation succeeded by [performing a discovery](https://library.netapp.com/ecmdocs/ECMP1217221/html/GUID-2A8546C7-347A-40B0-B937-4B31DAAA16DA.html#GUID-2A8546C7-347A-40B0-B937-4B31DAAA16DA) on the OCP nodes. In this example, `192.168.10.17` is the logical interface that offers the iSCSI protocol and therefore the iSCSI LUNs on the NetApp system.

    ```bash
    # TARGET=192.168.10.17
    # oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | while read -r node; do oc debug node/"$node" -- bash -c "chroot /host iscsiadm -m discovery -t st -p '$TARGET'"; done
    ```

   Example output:
    ```
    Starting pod/worker-0-debug-sdjwm ...
    To use host binaries, run `chroot /host`
    192.168.8.41:3260,1037 iqn.1992-08.com.netapp:sn.7789a7fb84af11efae39d039ea12dd29:vs.17
    192.168.8.42:3260,1038 iqn.1992-08.com.netapp:sn.7789a7fb84af11efae39d039ea12dd29:vs.17

    Removing debug pod ...
    Starting pod/worker-1-debug-cbd2n ...
    To use host binaries, run `chroot /host`
    192.168.8.41:3260,1037 iqn.1992-08.com.netapp:sn.7789a7fb84af11efae39d039ea12dd29:vs.17
    192.168.8.42:3260,1038 iqn.1992-08.com.netapp:sn.7789a7fb84af11efae39d039ea12dd29:vs.17

    Removing debug pod ...
    Starting pod/worker-2-debug-psjbk ...
    To use host binaries, run `chroot /host`
    192.168.8.41:3260,1037 iqn.1992-08.com.netapp:sn.7789a7fb84af11efae39d039ea12dd29:vs.17
    192.168.8.42:3260,1038 iqn.1992-08.com.netapp:sn.7789a7fb84af11efae39d039ea12dd29:vs.17

    Removing debug pod ...
    ```

#### 3.1.1. Configure Nodes with MachineConfig {#node-configure-mc}

We need to use MachineConfig to configure `/etc/multipath.conf` to meet the prerequisites for multipath configuration. For detailed information, please refer to [Red Hat Solution 5607891](https://access.redhat.com/solutions/5607891).

Here is an example configuration for the worker nodes' `/etc/multipath.conf`:

~~~
defaults {
    user_friendly_names yes
    find_multipaths no
}
~~~

Below is an example of a MachineConfig that sets up the required configuration:

~~~
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 98-worker-multipath-config
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            compression: ""
            source: data:,defaults%20%7B%0A%20%20user_friendly_names%20yes%0A%20%20find_multipaths%20no%0A%7D%0A
          mode: 420
          overwrite: true
          path: /etc/multipath.conf
~~~

#### 3.1.2. Verify ONTAP Cluster {#trident-verify-ontap}

Verify that the OCP nodes are registered on the ONTAP Cluster by executing the following command:

        rhfiler::> igroup show -vserver iscsi -igroup worker-1-5f663f29-7369-4062-ba08-83a495ebba5f
        Vserver Name: iscsi
        Igroup Name: worker-1-5f663f29-7369-4062-ba08-83a495ebba5f
        Protocol: iscsi
        OS Type: linux
        Portset Binding Igroup: -
        Igroup UUID: 7aad864c-870e-11ef-ae39-d039ea12dd29
        ALUA: true
        Initiators: iqn.1994-05.com.redhat:5db94959f (logged in)


### 3.2. Install NetApp Trident {#install-trident-tridentctl}

The following is an alternative to the [official installation method](https://docs.netapp.com/us-en/trident/trident-get-started/kubernetes-deploy-operator.html). Please consult the official documentation if you encounter any issues.

1. In OpenShift, you can use the OperatorHub to install Astra Trident. From the UI, navigate to OperatorHub, search for "Astra Trident," and initiate the installation from there. This method allows you to easily manage upgrades for the installer operator directly from OpenShift OperatorHub. Once the installation is complete, you can continue with [Trident Installation Step 4: Create the TridentOrchestrator and install Trident](https://docs.netapp.com/us-en/trident/trident-get-started/kubernetes-deploy-operator.html#step-4-create-the-tridentorchestrator-and-install-trident).

2. After the operator installation is complete and the TridentOrchestrator is in the "installed" status, verify the deployment by running the following command:

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


4. You can also check the Trident version using the `tridentctl` command line:

    ```bash
    ./tridentctl -n trident version
    ```

   **Expected output:**

    ```
    +----------------+----------------+
    | SERVER VERSION | CLIENT VERSION |
    +----------------+----------------+
    | 20.10.1        | 20.10.1        |
    +----------------+----------------+
    ```

5. Create the Trident SAN backend:

        # cat >jsons/backend_nas.json <<EOF
        {
          "version": 1,
          "storageDriverName": "ontap-nas",
          "backendName": "<Backend Name, z.B. svm-sap01-nas>",
          "managementLIF": "<Cluster Management LIF>",
          "dataLIF": "<Data LIF>",
          "svm": "<SVM Name, z.B. svm-sap01>",
          "username": "<Cluster Admin User>",
          "password": "<Cluster Admin User Passwort>"
        }
        EOF
        # ./tridentctl -n trident create backend -f jsons/backend_nas.json
        +---------------+----------------+--------------------------------------+--------+---------+
        | NAME          | STORAGE DRIVER | UUID                                 | STATE  | VOLUMES |
        +---------------+----------------+--------------------------------------+--------+---------+
        | svm-sap01-nas | ontap-nas      | ccdac84f-84fa-47b1-9ca4-2b9798c7554d | online | 0       |
        +---------------+----------------+--------------------------------------+--------+---------+

6. Create the Trident iSCSI backend:

   - Please note that this version is configured with Multipath enabled (in this case, the `dataLIF` key is missing):

    ```bash
    # cat jsons/backend_iscsi.json
    {
      "version": 1,
      "storageDriverName": "ontap-san",
      "backendName": "<Backend Name, e.g., svm-sap01-san>",
      "managementLIF": "<Cluster Management LIF>",
      "svm": "<SVM Name, e.g., svm-sap01>",
      "igroupName": "<Name of the created igroup, e.g., trident>",
      "username": "<Cluster Admin User>",
      "password": "<Cluster Admin User Password>"
    }
    ```

   Then, create the iSCSI backend of your choice:

    ```bash
    # ./tridentctl -n trident create backend -f jsons/backend_iscsi.jsona
    ```

   **Expected output:**

    ```
    +---------------+----------------+--------------------------------------+--------+------------+---------+
    |     NAME      | STORAGE DRIVER |                 UUID                 | STATE  | USER-STATE | VOLUMES |
    +---------------+----------------+--------------------------------------+--------+------------+---------+
    | san-iscsi-ocp | ontap-san      | 15063068-48c0-4f19-b4be-40519ba79c84 | online | normal     |       0 |
    +---------------+----------------+--------------------------------------+--------+------------+---------+
    ```

7. Verify the successful deployment of the backends:

    ```bash
    # ./tridentctl -n trident get backend
    ```

8. Create the iSCSI storage class:

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

9. (*optional*) Mark the new storage classes as the default:

    ```bash
    # Clean existing default flag if any
    # oc annotate sc --all storageclass.kubernetes.io/is-default-class- storageclass.beta.kubernetes.io/is-default-class-
    # oc annotate sc/san-iscsi-ocp storageclass.kubernetes.io/is-default-class=true
    ```

11. Verify that the storage classes have been created:

    ```bash
    # oc get sc
    NAME                                   PROVISIONER                   AGE
    san-iscsi-ocp (default)               csi.trident.netapp.io         17s
    ```

## 4. Continue with SAP ELM/EIC {#next-steps}

Proceed with the preparation for the SAP Edge Integration Cell installation.

