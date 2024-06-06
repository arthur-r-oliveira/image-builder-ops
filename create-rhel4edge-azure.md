- There is an RFE (Request for Enhancement) recently created asking for a similar feature, [RHEL-26438 - Image Builder support for RHEL for Edge on Azure](https://issues.redhat.com/browse/RHEL-26438). If this feature is required for your business, please, do [open a support case](https://access.redhat.com/support/cases/#/case/new/open-case?caseCreate=true) on the Red Hat Customer Portal referring to this solution.

- As **workaround**, `RHEL Image Builder` can produce `RHEL for Edge Raw` file which can later converted into `vhd` using `qemu-img convert`. See more about it with [RHEL9 Documentation, Deploying RHEL 9 on Microsoft Azure](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/deploying_rhel_9_on_microsoft_azure/assembly_deploying-a-rhel-image-as-a-virtual-machine-on-microsoft-azure_cloud-content-azure#convert-the-image_deploying-a-virtual-machine-on-microsoft-azure). See the example below as reference. 

  - 1.1 [Setup the RHEL Image Builder host](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_installing_and_managing_rhel_for_edge_images/setting-up-image-builder_composing-installing-managing-rhel-for-edge-images), [configure the require repositories](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_installing_and_managing_rhel_for_edge_images/managing-repositories_composing-installing-managing-rhel-for-edge-images) for the target RHEL images and, optionally, [setup a HTTP Server](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/deploying_web_servers_and_reverse_proxies/setting-apache-http-server_deploying-web-servers-and-reverse-proxies#setting-up-a-single-instance-apache-http-server_setting-apache-http-server) to later host the ostree commit repo and the [needed ignition file](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index#assembly_using-the-ignition-tool-for-the-rhel-for-edge-simplified-installer-images_composing-installing-managing-rhel-for-edge-images).
  - 1.2 Create two [blueprints](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_installing_and_managing_rhel_for_edge_images/composing-a-rhel-for-edge-image-using-image-builder-command-line_composing-installing-managing-rhel-for-edge-images#image-customizations_composing-a-rhel-for-edge-image-using-image-builder-command-line). The first one to be used to create the first `edge-commit` wihtout kernel options, as base `edge-commit` doesn't support those parms. The second blueprint will extend the first one and add the required kernel options for [compatibility with Microsoft Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd#rhel-8-using-hyper-v-manager) and will be later used to build the `edge-raw-image` image. Any needed  customization in additional to the fields Ignition, Kernel, User and Group, it must be integrated through an [Ignition file](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index#proc_creating-an-ignition-configuration-file_assembly_using-the-ignition-tool-for-the-rhel-for-edge-simplified-installer-images) to run the [pre-requisites](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd#hyper-v-manager), like packages, services and configuration files for Microsoft Azure. While building the solution, ensure that `customizations.ignition.firstboot.url` is a host reachable from your Microsoft Azure target ResourceGroup or, when fully disconnected installation is needed, embed the ignition with `customizations.ignition.embedded` [field](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/composing_installing_and_managing_rhel_for_edge_images/index#proc_creating-a-blueprint-with-support-to-ignition-using-the-cli_assembly_using-the-ignition-tool-for-the-rhel-for-edge-simplified-installer-images).

       - See a sample of the first blueprint`# cat base_blueprint_with_ignition.toml`:

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
          url = "http://10.26.2.44:8989/ostree/azure-config.ig"
          ~~~  

       - See a sample of the second blueprint`# cat microshift_blueprint_with_ignition.toml`:

          ~~~
          name = "rhel-9.3-microshift-4.15-edge-raw-azure"
          description = "Extended rhel-9.3-microshift-4.15-edge-raw with kernel parms for Microsoft Azure compatibility"
          version = "0.0.1"
          modules = []
          groups = []
          distro = "rhel-93"
                 
          [customizations]
          partitioning_mode = "lvm"

          [customizations.kernel]
          append = "loglevel=3 crashkernel=auto console=tty1 console=ttyS0 earlyprintk=ttyS0 rootdelay=300"

          [customizations.ignition.firstboot]
          url = "http://10.26.2.44:8989/ostree/azure-config.ig"
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
                ssh_authorized_keys:
                  - "ssh-rsa <YOUR SSH pub key>"
          storage:
            files:
              - path: /etc/waagent.conf
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
                contents:
                  inline: |
                    datasource_list: [ Azure ]
                    datasource:
                        Azure:
                            apply_network_config: False
                mode: 0644
              - path: /etc/cloud/cloud.cfg.d/05_logging.cfg
                contents:
                  inline: |
                    output: {all: '| tee -a /var/log/cloud-init-output.log'}
                mode: 0644
              - path: /etc/sudoers.d/core
                contents:
                  inline: |
                    core ALL=(ALL) NOPASSWD: ALL
                mode: 0440                
              - path: /etc/crio/openshift-pull-secret
                contents:
                  inline: |
                    <add your secrets>
                mode: 0600
              - path: /usr/local/bin/config-firewall-and-services.sh
                contents:
                  inline: |
                    firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
                    firewall-cmd --permanent --zone=trusted --add-source=169.254.169.1
                    firewall-cmd --reload
                mode: 0750
          systemd:
            units:
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
              - name: microshift.service
                enabled: true
              - name: waagent.service
                enabled: true
              - name: cloud-init.service
                enabled: true
              - name: NetworkManager.service
                enabled: true
          ~~~

       - `# butane azure-ignition.bu > /var/www/html/ostree/azure-config.ig`

  - 1.3 Creating an ostree commit: 

        ~~~
        # composer-cli compose start rhel-9.3-microshift-4.15-edge-raw edge-commit
        Compose 7cde21d9-243a-46fc-b507-f364b94942f5 added to the queue
        ~~~

  - 1.4 Export the ostree commit as http repo. You can optionally refer to this configuration as a [container repo](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/composing_installing_and_managing_rhel_for_edge_images/installing-rpm-ostree-images_composing-installing-managing-rhel-for-edge-images#doc-wrapper). 


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


  - 1.5 create the `edge-raw-image`

        ~~~
        # composer-cli compose start-ostree rhel-9.3-microshift-4.15-edge-raw-azure edge-raw-image --url http://10.26.2.44:8989/ostree/raw-commit/repo
        Compose 91738544-763c-42f9-bd4e-32eb6618ff1c added to the queue
        ~~~

  - 1.6 Convert the image, using the procedures documented at [RHEL9 Documentation, Deploying RHEL9 on Microsoft Azure](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/deploying_rhel_9_on_microsoft_azure/assembly_deploying-a-rhel-image-as-a-virtual-machine-on-microsoft-azure_cloud-content-azure#convert-the-image_deploying-a-virtual-machine-on-microsoft-azure) as reference. 
    - 1.6.1 First, download the `edge-raw-image` and [align it](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/deploying_rhel_9_on_microsoft_azure/assembly_deploying-a-rhel-image-as-a-virtual-machine-on-microsoft-azure_cloud-content-azure#convert-the-image_deploying-a-virtual-machine-on-microsoft-azure): 

        ~~~
        # composer-cli compose image 91738544-763c-42f9-bd4e-32eb6618ff1c
        91738544-763c-42f9-bd4e-32eb6618ff1c-image.raw.xz
        # unxz 91738544-763c-42f9-bd4e-32eb6618ff1c-image.raw.xz
        # ll 91738544-763c-42f9-bd4e-32eb6618ff1c-image.raw
        -rw-------. 1 root root 10737418240 May 30 06:09 91738544-763c-42f9-bd4e-32eb6618ff1c-image.raw
        # sh align.sh 91738544-763c-42f9-bd4e-32eb6618ff1c-image.raw
        Your image is already aligned. You do not need to resize.
        ~~~

    - 1.6.2 Convert the `edge-raw-image` into a `vhd` format: 

        ~~~
        # qemu-img convert -f raw -o subformat=fixed,force_size -O vpc 91738544-763c-42f9-bd4e-32eb6618ff1c-image.raw rhel4edge.vhd -p
        (100.00/100%)
        ~~~
  - 1.7 [push the vhd file to azure and create a custom azure image](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/deploying_rhel_9_on_microsoft_azure/assembly_deploying-a-rhel-image-as-a-virtual-machine-on-microsoft-azure_cloud-content-azure#create-azure-resources_deploying-a-virtual-machine-on-microsoft-azure).
  - 1.8 [Create the VM](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/deploying_rhel_9_on_microsoft_azure/assembly_deploying-a-rhel-image-as-a-virtual-machine-on-microsoft-azure_cloud-content-azure#create-start-the-VM_deploying-a-virtual-machine-on-microsoft-azure).