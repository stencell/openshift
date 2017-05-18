
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

	

Run oc get node to ensure that the new nodes are added and available. Run any other basic tests on cluster that you deem necessary

Update /etc/ansible/hosts to remove reference to new_nodes. The 3 support nodes should now just be in [nodes] group

On bastion host:
	register host with RHN
	-or-
	get someone in opentlc to reposync the rh-gluster-3-for-rhel-7-server-rpms to make available via repos.d file

	subscription-manager repos --disable=* --enable=rh-gluster-3-for-rhel-7-server-rpms

	yum install cns-deploy heketi-client

	create and run the following playbook:

		---
		- name: update iptables
		  hosts: nodes
		  become: true
		  tasks:
		    - name: update iptables
		      lineinfile:
		        insertafter: ^-A OS_FIREWALL
		        regexp: -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport {{ item }} -j ACCEPT
		        dest: /etc/sysconfig/iptables
		        line: -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport {{ item }} -j ACCEPT
		      with_items:
		        - 24007
		        - 24008
		        - 2222

		    - lineinfile:
		        insertafter: ^-A OS_FIREWALL
		        regexp: -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m multiport --dports 49152:49664 -j ACCEPT
		        dest: /etc/sysconfig/iptables
		        line: -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m multiport --dports 49152:49664 -j ACCEPT

		    - shell: systemctl reload iptables

	Check to make sure dm_thin_pool module is loaded (this should already be done)
		ansible nodes -m shell -a 'lsmod | grep dm_thin_pool'

	oc new-project storage-project

	oadm policy add-scc-to-user privileged -z storage-project
	oadm policy add-scc-to-user privileged -z default

	Create a topology file for gluster. see example file

	cns-deploy -n storage-project -g gluster-topology.json

	oc get pod -n storage-project -w to observe status

	export HEKETI_CLI_SERVER=http://$(oc get route -n storage-project | grep heketi | awk '{print $2}')

	heketi-cli topology info

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
