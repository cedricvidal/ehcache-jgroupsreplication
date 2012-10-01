Ehcache JGroups EHCJGRP-9 git-svn fork
--------------------------------------

"Synchronous" JGroups replication is not really synchronous. This fork tracks the fix proposed in [EHCJGRP-9](https://jira.terracotta.org/jira/browse/EHCJGRP-9) until some solution is included in the trunk code base.

# Git svn fork repository structure

###### [EHCJGRP-9](https://github.com/cedricvidal/ehcache-jgroupsreplication/tree/EHCJGRP-9) branch
Contains only the patch, to easily track differences with the remotes/git-svn branch

###### [master](https://github.com/cedricvidal/ehcache-jgroupsreplication/tree/master) branch
Commits to EHCJGRP-9 are cherry picked to master in order to build releases of this fork. It also contains this README and the pom.xml version is modified to reflect the fork releases.

###### [remotes/git-svn](https://github.com/cedricvidal/ehcache-jgroupsreplication/tree/remotes/git-svn) branch
Original fork of the SVN repository.

# Explanation of this fork

As stated in the JGroups documentation, in order for messages to be sent synchronously, you need two things:
- The RSVP protocol declared in the stack above the GMS protocol and under the UNICAST one.
- You also need to set the RSVP flag on the message to send synchrously

The first condition is only met by proper configuration of the JGroups stack in the JGroups XML configuration file. Note that you need to give your own properly configured file because the default JGroups udp.xml file puts the RSVP protocol too low in the stack to receive view change events from the GMS protocol required to properly wait for all cluster members acknowledges.

A udp.xml based JGroups configuration file which fixes the RSVP protocol position is included below.

The second one needs to be carried by the code and the jgroups-replication module doesn't set that flag.

This simple patch just modifies the `JGroupsCachePeer` so that it sets the RSVP flag on the message when the cache level `JGroupsCacheReplicatorFactory` `replicateAsynchronously` property is set to `false`.

# JGroups udp.xml based configuration fixing RSVP

	<!--
	  This configuration file is from the JGroups 3.1.0.Final release package. The only changes are:
	   * Add mcast_addr attribute to UDP
	   * Change mcast_port attribute to UDP
	   * Change mcast_ttl attribute to UDP
	-->

	<!--
	  Default stack using IP multicasting. It is similar to the "udp"
	  stack in stacks.xml, but doesn't use streaming state transfer and flushing
	  author: Bela Ban
	-->

	<config xmlns="urn:org:jgroups"
			xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
			xsi:schemaLocation="urn:org:jgroups http://www.jgroups.org/schema/JGroups-3.1.xsd">
		<UDP
			 mcast_addr="231.12.21.132"
			 mcast_port="45566"
			 tos="8"
			 ucast_recv_buf_size="20M"
			 ucast_send_buf_size="640K"
			 mcast_recv_buf_size="25M"
			 mcast_send_buf_size="640K"
			 loopback="true"
			 discard_incompatible_packets="true"
			 max_bundle_size="64K"
			 max_bundle_timeout="30"
			 ip_ttl="0"
			 enable_bundling="true"
			 enable_diagnostics="true"
			 thread_naming_pattern="cl"

			 timer_type="new"
			 timer.min_threads="4"
			 timer.max_threads="10"
			 timer.keep_alive_time="3000"
			 timer.queue_max_size="500"

			 thread_pool.enabled="true"
			 thread_pool.min_threads="2"
			 thread_pool.max_threads="8"
			 thread_pool.keep_alive_time="5000"
			 thread_pool.queue_enabled="true"
			 thread_pool.queue_max_size="10000"
			 thread_pool.rejection_policy="discard"

			 oob_thread_pool.enabled="true"
			 oob_thread_pool.min_threads="1"
			 oob_thread_pool.max_threads="8"
			 oob_thread_pool.keep_alive_time="5000"
			 oob_thread_pool.queue_enabled="false"
			 oob_thread_pool.queue_max_size="100"
			 oob_thread_pool.rejection_policy="Run"/>

		<PING timeout="2000"
				num_initial_members="20"/>
		<MERGE2 max_interval="30000"
				min_interval="10000"/>
		<FD_SOCK/>
		<FD_ALL/>
		<VERIFY_SUSPECT timeout="1500"  />
		<BARRIER />
		<pbcast.NAKACK2 xmit_interval="1000"
						xmit_table_num_rows="100"
						xmit_table_msgs_per_row="2000"
						xmit_table_max_compaction_time="30000"
						max_msg_batch_size="500"
						use_mcast_xmit="false"
						discard_delivered_msgs="true"/>
		<UNICAST  xmit_interval="2000"
				  xmit_table_num_rows="100"
				  xmit_table_msgs_per_row="2000"
				  xmit_table_max_compaction_time="60000"
				  conn_expiry_timeout="60000"
				  max_msg_batch_size="500"/>

		<!--
			RSVP protocol needs to be positionned between GMS and UNICAST protocols
			in order for RSVP flagged messages to be correctly sent synchronously.
			
			Cedric Vidal (cedric at vidal dot biz) 28/09/2012
		-->
		<RSVP resend_interval="2000" timeout="10000"/>

		<pbcast.STABLE stability_delay="1000" desired_avg_gossip="50000"
					   max_bytes="4M"/>
		<pbcast.GMS print_local_addr="true" join_timeout="3000"
					view_bundling="true"/>
		<UFC max_credits="2M"
			 min_threshold="0.4"/>
		<MFC max_credits="2M"
			 min_threshold="0.4"/>
		<FRAG2 frag_size="60K"  />
		<pbcast.STATE_TRANSFER />
		<!-- pbcast.FLUSH  /-->
	</config>

Regards,

Cedric Vidal
http://blog.proxiad.com
