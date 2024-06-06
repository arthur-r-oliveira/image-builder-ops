# Title 

**Is there support for NVIDIA drivers and RHEL for Edge (rpm-ostree) systems?**

# Issue

- Is there support for NVIDIA drivers and RHEL for Edge `rpm-ostree` systems?
- How to build RHEL for Edge images with NVIDIA GPU support?

# Environment 

- Red Hat Device Edge
- Red Hat Build of MicroShift

# Resolution

**Disclaimer:** _Links contained herein to external website(s) are provided for convenience only. Red Hat has not reviewed the links and is not responsible for the content or its availability. The inclusion of any link to an external website does not imply endorsement by Red Hat of the website or their entities, products or services. You agree that Red Hat is not responsible or liable for any loss or expenses that may result due to your use of (or reliance on) the external site or content._

- As initial reference, to enable workloads to use NVIDIA GPUs on an x86 system running Red Hat Device Edge, see NVIDIA's [external documentation, "Accelerating workloads with NVIDIA GPUs with Red Hat Device Edge"](https://docs.nvidia.com/datacenter/cloud-native/edge/latest/nvidia-gpu-with-device-edge.html#introduction). This documentation is still based on RPM systems, but can be used as referece to later build a RHEL for Edge (`rpm-ostree`) image.
  - For Red Hat Device Edge deployments with MicroShift and Podman, ensure that the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) is installed and configured properly. 
    - With MicroShift deployments, [configure CRI-O](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuring-cri-o): 

      ~~~
      # mv /etc/crio/crio.conf.d/microshift.conf /etc/crio/crio.conf.d/10-microshift.conf
      # nvidia-ctk runtime configure --runtime=crio --set-as-default --config=/etc/crio/crio.conf.d/99-nvidia.conf
      ~~~

    - With Podman deployments, NVIDIA recommends using [CDI](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html) for accessing NVIDIA devices in containers: 

      ~~~
      # nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
      ~~~

  - For Red Hat Device Edge deployments with MicroShift, ensure that [NVIDIA Device Plugin](https://docs.nvidia.com/datacenter/cloud-native/edge/latest/nvidia-gpu-with-device-edge.html#installing-the-nvidia-device-plugin) is installed.


- To build RHEL for Edge (`rpm-ostree`) images, see the following samples of blueprints and ignition file that could be used with  NVIDIA's [external documentation, "Accelerating workloads with NVIDIA GPUs with Red Hat Device Edge"](https://docs.nvidia.com/datacenter/cloud-native/edge/latest/nvidia-gpu-with-device-edge.html#introduction) as reference.

  - first blueprint`# cat base_blueprint_with_ignition.toml`:

      ~~~
      name = "rhel-9.3-microshift-4.15-edge-raw"
      description = "Blueprint for Edge Raw image, RHEL 9.3 with MicroShift 4.15 and Configuration through ignition"
      version = "0.0.1"
      modules = []
      groups = []
      distro = "rhel-93"
      
      [[packages]]
      name = "microshift"
      version = "*"
      
      [[packages]]
      name = "microshift-greenboot"
      version = "*"
      
      [[packages]]
      name = "microshift-networking"
      version = "*"
      
      [[packages]]
      name = "microshift-selinux"
      version = "*"
        ### NVIDIA packages https://docs.nvidia.com/datacenter/cloud-native/edge/latest/nvidia-gpu-with-device-edge.html#installing-the-nvidia-gpu-driver
      
      [[packages]]
      name = "nvidia-driver"
      version = "*"
      
      [[packages]]
      name = "nvidia-fabric-manager"
      version = "*"
      
      [[packages]]
      name = "libnvidia-nscq-550"
      version = "*"
      
      [[packages]]
      name = "nvidia-container-toolkit"
      version = "*"
      
      [[packages]]
      name = "container-selinux.noarch"
      version = "*"
      
      [[packages]]
      name = "nvidia-driver-cuda"
      version = "*"
      
      ### 
      ## If using azure VMs with NVIDIA GPUs, https://learn.microsoft.com/en-us/azure/virtual-machines/nct4-v3-series
      ## According to https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd#hyper-v-manager
      ### Install the Azure Linux Agent, cloud-init and other necessary utilities 
      
      [[packages]]
      name = "WALinuxAgent"
      version = "*"
      
      [[packages]]
      name = "cloud-init"
      version = "*"
      
      [[packages]]
      name = "cloud-utils-growpart"
      version = "*"
      
      [[packages]]
      name = "gdisk"
      version = "*"
      
      [[packages]]
      name = "hyperv-daemons"
      version = "*"
      
      [customizations.ignition.firstboot]
      url = "http://20.115.57.36/azure-config.ig"
      ~~~ 

  - See a sample of the second blueprint, where there is additional customizations, on top of the previous one, according to specific target platforms. `# cat min_blueprint_with_ignition.toml`:

      ~~~ 
      name = "rhel-9.3-microshift-4.15-edge-raw-azure-nvidia"
      description = "Extended rhel-9.3-microshift-4.15-edge-raw-gpu with kernel parms for Microsoft Azure compatibility"
      version = "0.0.1"
      modules = []
      groups = []
      distro = "rhel-93"
      
      [customizations]
      partitioning_mode = "lvm"
      
      [customizations.kernel]
      append = "loglevel=3 crashkernel=auto console=tty1 console=ttyS0 earlyprintk=ttyS0 rootdelay=300"
      
      [customizations.ignition.firstboot]
      url = "http://20.115.57.36/azure-config.ig"
      ~~~   

  - `# cat azure-ignition.bu`

      ~~~
      variant: r4e
      version: 1.1.0
      passwd:
        users:
          - name: core
            groups:
              - wheel
            system: true
            ssh_authorized_keys:
              - "ssh-rsa <your ssh key>"
      storage:
        files:
          - path: /etc/waagent.conf
            overwrite: true
            contents:
              inline: |
                #
                # Azure Linux Agent Configuration	
                #
                Role.StateConsumer=None 
                Role.ConfigurationConsumer=None 
                Role.TopologyConsumer=None
                Provisioning.Enabled=y
                Provisioning.Agent=cloud-init
                Provisioning.DeleteRootPassword=n
                Provisioning.RegenerateSshHostKeyPair=y
                Provisioning.SshHostKeyPairType=rsa
                Provisioning.MonitorHostName=y
                ResourceDisk.Format=n
                ResourceDisk.Filesystem=ext4
                ResourceDisk.MountPoint=/mnt/resource 
                ResourceDisk.EnableSwap=n 
                ResourceDisk.SwapSizeMB=0
                LBProbeResponder=y
                Logs.Verbose=n
                OS.RootDeviceScsiTimeout=300
                OS.OpensslPath=None
            mode: 0644
          - path: /etc/cloud/cloud.cfg.d/91-azure_datasource.cfg
            overwrite: true
            contents:
              inline: |
                datasource_list: [ Azure ]
                datasource:
                    Azure:
                        apply_network_config: False
            mode: 0644
          - path: /etc/modprobe.d/blocklist.conf
            overwrite: true
            contents:
              inline: |
                blacklist nouveau
                blacklist lbm-nouveau
                blacklist floppy
                blacklist amdgpu
                blacklist skx_edac
                blacklist intel_cstate
            mode: 0644
          - path: /etc/cloud/cloud.cfg.d/05_logging.cfg
            overwrite: true
            contents:
              inline: |
                output: {all: '| tee -a /var/log/cloud-init-output.log'}
            mode: 0644
          - path: /etc/udev/rules.d/68-azure-sriov-nm-unmanaged.rules
            overwrite: true
            contents:
              inline: |
                SUBSYSTEM=="net", DRIVERS=="hv_pci", ACTION=="add", ENV{NM_UNMANAGED}="1"
            mode: 0644
          - path: /etc/sudoers.d/core
            contents:
              inline: |
                core ALL=(ALL) NOPASSWD: ALL
            mode: 0440        
          - path: /etc/crio/openshift-pull-secret
            overwrite: true
            contents:
              inline: |
                <your pull secrets>
            mode: 0600
          - path: /etc/microshift/manifests/nvidia-device-plugin-time-slice.yaml
            overwrite: true
            contents:
              inline: |
                ---
                apiVersion: v1
                kind: Namespace
                metadata:
                  labels:
                    pod-security.kubernetes.io/enforce: privileged
                  name: nvidia-device-plugin
                ---
                apiVersion: rbac.authorization.k8s.io/v1
                kind: Role
                metadata:
                  name: nvidia-device-plugin
                  namespace: nvidia-device-plugin
                rules:
                  - apiGroups:
                      - ""
                    resources:
                      - nodes
                    verbs:
                      - get
                      - list
                      - watch
                  - apiGroups:
                      - security.openshift.io
                    resourceNames:
                      - privileged
                    resources:
                      - securitycontextconstraints
                    verbs:
                      - use
                ---
                apiVersion: v1
                kind: ServiceAccount
                metadata:
                  name: nvidia-device-plugin
                  namespace: nvidia-device-plugin
                ---
                apiVersion: v1
                data:
                  nvidia-plugin-configs: |-
                    version: v1
                    sharing:
                      timeSlicing:
                        resources:
                        - name: nvidia.com/gpu
                          replicas: 4
                kind: ConfigMap
                metadata:
                  labels:
                    app.kubernetes.io/name: nvidia-device-plugin
                    app.kubernetes.io/version: 0.15.0
                  name: nvidia-device-plugin-configs
                  namespace: nvidia-device-plugin
                ---
                apiVersion: rbac.authorization.k8s.io/v1
                kind: RoleBinding
                metadata:
                  name: nvidia-device-plugin
                  namespace: nvidia-device-plugin
                roleRef:
                  apiGroup: rbac.authorization.k8s.io
                  kind: Role
                  name: nvidia-device-plugin
                subjects:
                  - kind: ServiceAccount
                    name: nvidia-device-plugin
                    namespace: nvidia-device-plugin
                ---
                apiVersion: rbac.authorization.k8s.io/v1
                kind: ClusterRole
                metadata:
                  labels:
                    app.kubernetes.io/name: nvidia-device-plugin
                    app.kubernetes.io/version: 0.15.0
                  name: nvidia-device-plugin-clusterrole
                rules:
                - apiGroups:
                  - ""
                  resources:
                  - nodes
                  verbs:
                  - get
                  - list
                  - watch
                ---
                apiVersion: rbac.authorization.k8s.io/v1
                kind: ClusterRoleBinding
                metadata:
                  name: nvidia-device-plugin-clusterrolebinding
                roleRef:
                  apiGroup: rbac.authorization.k8s.io
                  kind: ClusterRole
                  name: nvidia-device-plugin-clusterrole
                subjects:
                - kind: ServiceAccount
                  name: nvidia-device-plugin
                  namespace: nvidia-device-plugin
                ---
                apiVersion: apps/v1
                kind: DaemonSet
                metadata:
                  name: nvidia-device-plugin-daemonset
                  namespace: nvidia-device-plugin
                spec:
                  selector:
                    matchLabels:
                      name: nvidia-device-plugin-ds
                  updateStrategy:
                    type: RollingUpdate
                  template:
                    metadata:
                      labels:
                        name: nvidia-device-plugin-ds
                    spec:
                      containers:
                      - command:
                        - config-manager
                        env:
                        - name: ONESHOT
                          value: "false"
                        - name: KUBECONFIG
                        - name: NODE_NAME
                          valueFrom:
                            fieldRef:
                              apiVersion: v1
                              fieldPath: spec.nodeName
                        - name: NODE_LABEL
                          value: nvidia.com/device-plugin.config
                        - name: CONFIG_FILE_SRCDIR
                          value: /available-configs
                        - name: CONFIG_FILE_DST
                          value: /config/config.yaml
                        - name: DEFAULT_CONFIG
                          value: nvidia-plugin-configs
                        - name: FALLBACK_STRATEGIES
                          value: named,single
                        - name: SEND_SIGNAL
                          value: "true"
                        - name: SIGNAL
                          value: "1"
                        - name: PROCESS_TO_SIGNAL
                          value: nvidia-device-plugin
                        image: nvcr.io/nvidia/k8s-device-plugin:v0.15.0
                        imagePullPolicy: IfNotPresent
                        name: nvidia-device-plugin-sidecar
                        resources: {}
                        securityContext:
                          privileged: true
                        terminationMessagePath: /dev/termination-log
                        terminationMessagePolicy: File
                        volumeMounts:
                        - mountPath: /available-configs
                          name: available-configs
                        - mountPath: /config
                          name: config
                      - command:
                        - nvidia-device-plugin
                        env:
                        - name: MPS_ROOT
                          value: /run/nvidia/mps
                        - name: CONFIG_FILE
                          value: /config/config.yaml
                        - name: DEFAULT_CONFIG
                          value: nvidia-plugin-configs
                        - name: NVIDIA_MIG_MONITOR_DEVICES
                          value: all
                        - name: NVIDIA_VISIBLE_DEVICES
                          value: all
                        - name: NVIDIA_DRIVER_CAPABILITIES
                          value: compute,utility
                        image: nvcr.io/nvidia/k8s-device-plugin:v0.15.0
                        imagePullPolicy: IfNotPresent
                        name: nvidia-device-plugin-ctr
                        resources: {}
                        securityContext:
                          privileged: true
                        terminationMessagePath: /dev/termination-log
                        terminationMessagePolicy: File
                        volumeMounts:
                        - mountPath: /var/lib/kubelet/device-plugins
                          name: device-plugin
                        - mountPath: /dev/shm
                          name: mps-shm
                        - mountPath: /mps
                          name: mps-root
                        - mountPath: /var/run/cdi
                          name: cdi-root
                        - mountPath: /available-configs
                          name: available-configs
                        - mountPath: /config
                          name: config
                      dnsPolicy: ClusterFirst
                      initContainers:
                      - command:
                        - config-manager
                        env:
                        - name: ONESHOT
                          value: "true"
                        - name: KUBECONFIG
                          value: ""
                        - name: NODE_NAME
                          valueFrom:
                            fieldRef:
                              apiVersion: v1
                              fieldPath: spec.nodeName
                        - name: NODE_LABEL
                          value: nvidia.com/device-plugin.config
                        - name: CONFIG_FILE_SRCDIR
                          value: /available-configs
                        - name: DEFAULT_CONFIG
                          value: nvidia-plugin-configs
                        - name: CONFIG_FILE_DST
                          value: /config/config.yaml
                        - name: FALLBACK_STRATEGIES
                          value: named,single
                        - name: SEND_SIGNAL
                          value: "false"
                        - name: SIGNAL
                          value: ""
                        - name: PROCESS_TO_SIGNAL
                          value: ""
                        volumeMounts:
                          - name: available-configs
                            mountPath: /available-configs
                          - name: config
                            mountPath: /config
                        image: nvcr.io/nvidia/k8s-device-plugin:v0.15.0
                        imagePullPolicy: IfNotPresent
                        name: nvidia-device-plugin-init
                        resources: {}
                        terminationMessagePath: /dev/termination-log
                        terminationMessagePolicy: File
                      priorityClassName: system-node-critical
                      restartPolicy: Always
                      schedulerName: default-scheduler
                      securityContext:
                        privileged: true
                      serviceAccount: nvidia-device-plugin
                      serviceAccountName: nvidia-device-plugin
                      shareProcessNamespace: true
                      terminationGracePeriodSeconds: 30
                      tolerations:
                      - key: CriticalAddonsOnly
                        operator: Exists
                      - effect: NoSchedule
                        key: nvidia.com/gpu
                        operator: Exists
                      volumes:
                      - hostPath:
                          path: /var/lib/kubelet/device-plugins
                          type: ""
                        name: device-plugin
                      - hostPath:
                          path: /run/nvidia/mps
                          type: DirectoryOrCreate
                        name: mps-root
                      - hostPath:
                          path: /run/nvidia/mps/shm
                          type: ""
                        name: mps-shm
                      - hostPath:
                          path: /var/run/cdi
                          type: DirectoryOrCreate
                        name: cdi-root
                      - configMap:
                          defaultMode: 420
                          name: nvidia-device-plugin-configs
                        name: available-configs
                      - emptyDir: {}
                        name: config   
            mode: 0640
          - path: /etc/microshift/manifests/kustomization.yaml
            overwrite: true
            contents:
              inline: |
                apiVersion: kustomize.config.k8s.io/v1beta1
                kind: Kustomization
                namespace: nvidia-device-plugin
                resources:
                  - nvidia-device-plugin-time-slice.yaml    
            mode: 0640
          - path: /etc/microshift/nvidia-container-microshift.te
            overwrite: true
            contents:
              inline: |
                module nvidia-container-microshift 1.0;
                
                require {
                              type xserver_misc_device_t;
                              type container_t;
                              class chr_file { map read write };
                }
                # ============= container_t ==============
                allow container_t xserver_misc_device_t:chr_file map;
            mode: 0600
          - path: /usr/local/bin/config-firewall-and-services.sh
            overwrite: true
            contents:
              inline: |
                #!/bin/bash -x
                firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
                firewall-cmd --permanent --zone=trusted --add-source=169.254.169.1
                firewall-cmd --reload
                rm -f /etc/udev/rules.d/70-persistent-net.rules
                rm -f /etc/udev/rules.d/75-persistent-net-generator.rules
                rm -f /etc/udev/rules.d/80-net-name-slot-rules
                setsebool -P container_use_devices on
                checkmodule -m -M -o /etc/microshift/nvidia-container-microshift.mod /etc/microshift/nvidia-container-microshift.te
                semodule_package --outfile /etc/microshift/nvidia-container-microshift.pp --module /etc/microshift/nvidia-container-microshift.mod
                semodule -i /etc/nvidia-container-microshift.pp
                mv /etc/crio/crio.conf.d/microshift.conf /etc/crio/crio.conf.d/10-microshift.conf
                nvidia-ctk runtime configure --runtime=crio --set-as-default --config=/etc/crio/crio.conf.d/99-nvidia.conf
                nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
                systemctl restart crio
                systemctl restart microshift
                #rm -f /usr/local/bin/config-firewall-and-services.sh
                systemctl stop first-boot.service
                systemctl disable first-boot.service
                sudo systemctl daemon-reload
            mode: 0750
      systemd:
        units:
          - name: microshift.service
            enabled: true
          - name: NetworkManager.service
            enabled: true
          - name: nvidia-fabricmanager.service
            enabled: true
          - name: nvidia-persistenced.service
            enabled: true
          - name: first-boot.service
            enabled: true
            contents: |
              [Unit]
              Description=Setup host at first boot
              After=network.target microshift.service firewalld.service
              ConditionPathExists=!/etc/first-boot-done
              [Service]
              Type=oneshot
              RemainAfterExit=yes
              ExecStart=/usr/local/bin/config-firewall-and-services.sh
              ExecStartPost=/bin/systemctl disable --now first-boot.service
              ExecStartPost=/bin/touch /etc/first-boot-done
              [Install]
              WantedBy=multi-user.target
          - name: waagent.service
            enabled: true
          - name: cloud-init.service
            enabled: true
      ~~~

       - `# butane azure-ignition.bu > /var/www/html/ostree/azure-config.ig`

  - Creating an ostree commit using the first blueprint as reference, where the NVIDIA packages will be integrated: 

      ~~~
      # composer-cli compose start rhel-9.3-microshift-4.15-edge-raw edge-commit
      Compose 7cde21d9-243a-46fc-b507-f364b94942f5 added to the queue
      ~~~

  - Export the ostree commit as http repo. You can optionally refer to this configuration as a [container repo](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_installing_and_managing_rhel_for_edge_images/installing-rpm-ostree-images_composing-installing-managing-rhel-for-edge-images#doc-wrapper). 


      ~~~
      # composer-cli compose image 31c913a1-3499-4e96-bd30-447098e102de 
      31c913a1-3499-4e96-bd30-447098e102de-commit.tar
      # ll
      total 1093952
      -rw-------. 1 root root 1120204800 May 30 06:02 31c913a1-3499-4e96-bd30-447098e102de-commit.tar
      # tar xf 31c913a1-3499-4e96-bd30-447098e102de-commit.tar 
      # ostree summary --repo=repo -u
      # ostree summary --repo=repo -v
      OT: using fuse: 0
      * rhel/9/x86_64/edge
          Latest Commit (18.3 kB):
            33c8e7a812c51fdddc0533517c6604276f01ba714f132fc70419ebcc285c31b0
          Version (ostree.commit.version): 9.3
          Timestamp (ostree.commit.timestamp): 2024-05-30T05:42:44-04
      
      Repository Mode (ostree.summary.mode): archive-z2
      Last-Modified (ostree.summary.last-modified): 2024-05-30T06:02:57-04
      Has Tombstone Commits (ostree.summary.tombstone-commits): No
      ostree.summary.indexed-deltas: true
      ~~~


  - Create as `edge-raw-image`, as a sample to exented the initial commit.

      ~~~
      # composer-cli compose start-ostree rhel-9.3-microshift-4.15-edge-raw-azure edge-raw-image --url http://10.26.2.44:8989/ostree/raw-commit/repo
      Compose 91738544-763c-42f9-bd4e-32eb6618ff1c added to the queue
      ~~~

# Root Cause

- Although built and updated [using different concepts and procedures](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index#edge-difference-between-rhel-rpm-images-and-rhel-for-edge-images_introducing-rhel-for-edge-images), `rpm-ostree` systems are still composed using RPM Packages. Meaning that, in additional to Red Hat's repositories, third-party RPM packages, like ones offered for [NVIDIA® Data Center GPU Drivers for Linux](https://docs.nvidia.com/datacenter/tesla/index.html), can be also [integrated](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index#proc_adding-third-party-repositories-with-specific-distributions_managing-repositories) within `rpm-ostree` systems using [RHEL Image builder](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index#setting-up-image-builder_composing-installing-managing-rhel-for-edge-images). For RPM packages that may need additional customization to be full integrated, as [blueprint customizations](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_installing_and_managing_rhel_for_edge_images/composing-a-rhel-for-edge-image-using-image-builder-command-line_composing-installing-managing-rhel-for-edge-images#specifying_customized_files_in_the_blueprint) are not supported by  `OSTree commits`, such as `edge-raw-image`, `edge-installer`, and `edge-simplified-installer`, leverage [`Ignition configuration files`](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index#proc_creating-an-ignition-configuration-file_assembly_using-the-ignition-tool-for-the-rhel-for-edge-simplified-installer-images)with your composed images.