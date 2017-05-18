
# CNS Setup

### This is meant to be run in the OpenShift advanced training lab, but you can use these steps more generally in any OpenShift environment. This assumes you have already deployed your initial OpenShift cluster.

*Unless specified, the steps should all be run from the bastion server or whatever server has your /etc/ansible/hosts file.*

#### Basic Setup
1. Add new_nodes under [OSEv3:children] group in /etc/ansible/hosts
2. Add 3 support nodes to [new_nodes] group in /etc/ansible/hosts
3. Add label for these nodes 'env=storage'
4. Clean up storage nodes:
  * Get rid of /etc/fstab entry
  * Unmount any leftovers:  
    `ansible new_nodes -a 'umount /srv/nfs'`
  * Get rid of LVM configs:  
    `ansible new_nodes -m shell -a 'lvremove -f /dev/nfsvg/nfsmount;vgremove -f nfsvg;pvremove -f /dev/xvdb'`
5. Prep storage nodes by installing docker:  
   `ansible new_nodes -m yum -a 'name=docker'`
6. Run the scaleup playbook to add the new nodes:  
   `ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-node/scaleup.yml`
7. Check the status of your new nodes and any other health checks you want to run:  
   `oc get nodes`
8. Update /etc/ansible/hosts to remove reference to new_nodes. The 3 support nodes should now just be in [node] group.
9. Add the `rh-gluster-3-for-rhel-7-server-rpms` repo to the bastion host. You can do this by registering with RHN or using a local repo clone in OpenTLC.
10. *If using subscription-manager*:  
   `subscription-manager repos --disable=* --enable=rh-gluster-3-for-rhel-7-server-rpms`
11. Install packages:  
   `yum install cns-deploy heketi-client`
12. Copy the [gluster-iptables.yml](./gluster-iptables.yml) playbook to your host and run it:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/gluster-iptables.yml > /root/gluster-iptables.yml`  
   `ansible-playbook gluster-iptables.yml`
13. Check to make sure dm_thin_pool module is loaded:  
   `ansible nodes -m shell -a 'lsmod | grep dm_thin_pool'`
14. Create a new project for Gluster:  
   `oc new-project storage-project`
15. Add a couple of users to privileged SCC:  
   `oadm policy add-scc-to-user privileged -z storage-project`  
   `oadm policy add-scc-to-user privileged -z default`

#### Gluster/CNS deploy
16. Copy the [gluster-topology.json](./gluster-topology.json) file to your host:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/gluster-topology.json > /root/gluster-topology.json`
17. Update the gluster-topology.json file to reference the correct node IPs for your environment.
18. Deploy your CNS magic:  
   `cns-deploy -n storage-project -g gluster-topology.json`
19. Check the status of your new pods however you like:  
   `oc get pod -n storage-project -w`
20. Set your environment variable to talk to heketi:  
   `export HEKETI_CLI_SERVER=http://$(oc get route -n storage-project | grep heketi | awk '{print $2}')`
21. Check your topology to make sure everything looks pretty:  
   `heketi-cli topology info`

#### Create PV/PVC & Test Pod
22. Create new gluster volume and get pv template output:  
   `heketi-cli volume create --size=7 --persistent-volume-file=gluster-pv.json`
23. Update the output gluster-pv.json file to point it to the correct endpoints:  
   `ansible localhost -m replace -a 'dest=/root/gluster-pv.json regexp="TYPE ENDPOINT HERE" replace=heketi-storage-endpoints'`
24. Create the new PV:  
   `oc create -f gluster-pv.json`
25. Label the new PV to better target with PVC:  
   `oc label pv $(oc get pv | grep gluster | awk '{print $1}') storage-tier=gluster`
26. Make sure PV looks good and includes proper label:  
   ` oc get pv --show-labels`
27. Download [gluster-pvc.yml](./gluster-pvc.yml) and create your PVC in the storage-project project:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/gluster-pvc.yml > /root/gluster-pvc.yml`
   `oc create -f gluster-pvc.yml -n storage-project`
28. Verify that PV & PVC are created and bound:  
   `oc get pvc`  
   `oc get pv`
29. Download [app.yml](./app.yml) and create your test app:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/app.yml > /root/app.yml`  
   `oc create -f app.yml`
30. Access your newly created pod and see the mount at /usr/share/busybox:  
   `oc rsh busybox`  
   `df -hT`

*The above steps will make gluster volumes usable within the storage-projects project. If you want to enable this in other projects, you will need to create the gluster endpoints & service in that project. See the example *.yml files in this repo.*

#### Create dynamic storage
1. Download [gluster-storageclass.yml](./gluster-storageclass.yml), modify it, and create in OpenShift:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/gluster-storageclass.yml > /root/gluster-storageclass.yml`  
   `vi gluster-storageclass.yml`  
   `oc create -f gluster-storageclass.yml`
2. Download [gluster-pvc-dynamic.yml](./gluster-pvc-dynamic.yml) and create:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/gluster-pvc-dynamic.yml > /root/gluster-pvc-dynamic.yml`  
   `oc create -f gluster-pvc-dynamic.yml`
3. Make sure you have new PV, PVC, endpoint, and service:  
   `oc get pv,pvc,endpoints,svc`
4. Spin up a new app and test:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/app-dynamic.yml > /root/app-dynamic.yml`  
   `oc create -f app-dynamic.yml`  
   `oc rsh busybox-dynamic`  
   `df -hT`






