
# CNS Setup

### This is meant to be run in the OpenShift advanced training lab, but you can use these steps more generally in any OpenShift environment.

*Unless specified, the steps should all be run from the bastion server or whatever server has your /etc/ansible/hosts file.*

1. Add new_nodes under [OSEv3:children] group in /etc/ansible/hosts
2. Add 3 support nodes to [new_nodes] group in /etc/ansible/hosts
3. Add label for these nodes 'env=storage'
4. Clean up storage nodes:
  * Get rid of /etc/fstab entry
  * Unmount any leftovers:  
    `ansible new_nodes -a 'umount /srv/nfs'`
  * Get rid of LVM configs:  
    `ansible new_nodes -m shell -a 'lvremove -f /dev/nfsvg/nfsmount;vgremove -f nfsvg;pvremove -f /dev/xvdb'`
5. Prep storage nodes:  
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
16. Copy the [gluster-topology.json](./gluster-topology.json) file to your host:  
   `curl https://raw.githubusercontent.com/stencell/openshift/master/cns/gluster-topology.json > /root/gluster-topology.json`
17. Deploy your CNS magic:  
   `cns-deploy -n storage-project -g gluster-topology.json`
18. Check the status of your new pods however you like:  
   `oc get pod -n storage-project -w`
19. Set your environment variable to talk to heketi:  
   `export HEKETI_CLI_SERVER=http://$(oc get route -n storage-project | grep heketi | awk '{print $2}')`
20. Check your topology to make sure everything looks pretty:  
   `heketi-cli topology info`



	optional to do this...i rolled with existing
	create endpoints & svc:
		oc create -f gluster-endpoints.yaml
		oc create -f gluster-service.yamml

	heketi-cli volume create --size=7 --persistent-volume-file=gluster-pv.json

	ansible localhost -m replace -a 'dest=/root/gluster-pv.json regexp="TYPE ENDPOINT HERE" replace=heketi-storage-endpoints'

	oc create -f gluster-pv.json

	oc label pv $(oc get pv | grep gluster | awk '{print $1}') storage-tier=gluster

	oc get pv --show-labels

	oc create -f gluster-pvc.yaml

	oc get pv
	oc get pvc

	oc create -f app.yaml

	oc rsh busybox

	df -hT
