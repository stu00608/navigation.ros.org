.. _behavior_tree_nav_to_pose_and_replan_if_path_invalid:

Navigate To Pose and Replan Only if Path Invalid
################################################

This behavior tree implements a significantly more mature version of the behavior tree on :ref:`behavior_trees`.
It navigates from a starting point to a single point goal in freespace.
It contains both use of custom recoveries in specific sub-contexts as well as a global recovery subtree for system-level failures.
It also provides the opportunity for users to retry tasks multiple times before returning a failed state.

The ``ComputePathToPose`` and ``FollowPath`` BT nodes both also specify their algorithms to utilize.
By convention we name these by the style of algorithms that they are (e.g. not ``DWB`` but rather ``FollowPath``) such that a behavior tree or application developer need not worry about the technical specifics. They just want to use a path following controller.

In this behavior tree, we attempt to retry the entire navigation task 6 times before returning to the caller that the task has failed.
This allows the navigation system ample opportunity to try to recovery from failure conditions or wait for transient issues to pass, such as crowding from people or a temporary sensor failure.

In nominal execution, this will replan only if the previous path is invalid and pass a new path onto the controller.
However, this time, if the planner fails, it will trigger contextually aware recoveries in its subtree, clearing the global costmap.
Additional recoveries can be added here for additional context-specific recoveries, such as trying another algorithm.

Similarly, the controller has similar logic. If it fails, it also attempts a costmap clearing of the local costmap impacting the controller.
It is worth noting the ``GoalUpdated`` node in the reactive fallback.
This allows us to exit recovery conditions when a new goal has been passed to the navigation system through a preemption.
This ensures that the navigation system will be very responsive immediately when a new goal is issued, even when the last goal was in an attempted recovery.

If these contextual recoveries fail, this behavior tree enters the recovery subtree.
This subtree is reserved for system-level failures to help resolve issues like the robot being stuck or in a bad spot.
This subtree also has the ``GoalUpdated`` BT node it ticks every iteration to ensure responsiveness of new goals.
Next, the recovery subtree will the recoveries: costmap clearing operations, spinning, waiting, and backing up.
After each of the recoveries in the subtree, the main navigation subtree will be reattempted. 
If it continues to fail, the next recovery in the recovery subtree is ticked.

While this behavior tree does not make use of it, the ``PlannerSelector``, ``ControllerSelector``, and ``GoalCheckerSelector`` behavior tree nodes can also be helpful. Rather than hardcoding the algorithm to use (``GridBased`` and ``FollowPath``), these behavior tree nodes will allow a user to dynamically change the algorithm used in the navigation system via a ROS topic. It may be instead advisable to create different subtree contexts using condition nodes with specified algorithms in their most useful and unique situations. However, the selector nodes can be a useful way to change algorithms from an external application rather than via internal behavior tree control flow logic. It is better to implement changes through behavior tree methods, but we understand that many professional users have external applications to dynamically change settings of their navigators.

.. code-block:: xml

	<root main_tree_to_execute="MainTree">
	  <BehaviorTree ID="MainTree">
	    <RecoveryNode number_of_retries="6" name="NavigateRecovery">
	      <PipelineSequence name="NavigateWithReplanning">
		<RateController hz="1.0">
		  <RecoveryNode number_of_retries="1" name="ComputePathToPose">
		    <Fallback> 
		      <ReactiveSequence>
		        <Inverter>
		          <GlobalUpdatedGoal/>
		        </Inverter>
		        <IsPathValid path="{path}"/>
		      </ReactiveSequence>
		      <ComputePathToPose goal="{goal}" path="{path}" planner_id="GridBased"/>
		    </Fallback>
		    <ClearEntireCostmap name="ClearGlobalCostmap-Context" service_name="global_costmap/clear_entirely_global_costmap"/>
		  </RecoveryNode>
		</RateController>
		<RecoveryNode number_of_retries="1" name="FollowPath">
		  <FollowPath path="{path}" controller_id="FollowPath"/>
		  <ClearEntireCostmap name="ClearLocalCostmap-Context" service_name="local_costmap/clear_entirely_local_costmap"/>
		</RecoveryNode>
	      </PipelineSequence>
	      <ReactiveFallback name="RecoveryFallback">
		<GoalUpdated/>
		<RoundRobin name="RecoveryActions">
		  <Sequence name="ClearingActions">
		    <ClearEntireCostmap name="ClearLocalCostmap-Subtree" service_name="local_costmap/clear_entirely_local_costmap"/>
		    <ClearEntireCostmap name="ClearGlobalCostmap-Subtree" service_name="global_costmap/clear_entirely_global_costmap"/>
		  </Sequence>
		  <Spin spin_dist="1.57"/>
		  <Wait wait_duration="5"/>
		  <BackUp backup_dist="0.30" backup_speed="0.05"/>
		</RoundRobin>
	      </ReactiveFallback>
	    </RecoveryNode>
	  </BehaviorTree>
	</root>


