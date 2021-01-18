package sg.com.dao;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;
import javax.sql.DataSource;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.skife.jdbi.v2.DBI;
import org.skife.jdbi.v2.Handle;
import org.skife.jdbi.v2.Query;
import org.skife.jdbi.v2.Update;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import sg.com.dto.ConcurrentWorkflowInstance;
import sg.com.dto.WorkItem;



@Component("WorkItemTWPROCDAO")
public class WorkItemTWPROCDAOImpl implements WorkItemTWPROCDAO {


	private static Log logger=LogFactory.getLog(WorkItemTWPROCDAOImpl.class);

	@Resource(name="dataSourceTWPROC")
	private DataSource dataSource;

	private JdbcTemplate jdbcTemplate; 

	
	@Override
	public WorkItem getLatestActiveTask(String instanceId) {
		
		WorkItem result = null;
		String query = "SELECT task_id, activity_name from lsw_task where bpd_instance_id=:instanceId and status=12";
		
		try{
			DBI dbi = new DBI(dataSource);
			Handle handle = dbi.open();
			Query sqlQuery = handle.createQuery(query);
			sqlQuery.bind("instanceId", instanceId);
			List<Map<String, Object>> rows = sqlQuery.list();
			if (rows.size() > 0){
				result = new WorkItem();
				BigDecimal latestTaskId = (BigDecimal)rows.get(0).get("task_id");
				String latestTaskActivityName = (String)rows.get(0).get("activity_name");
				logger.debug("latest active task is " + latestTaskId);
				result.setTask_id(latestTaskId);
				result.setTask_activity_name(latestTaskActivityName);
			}
		}catch (Exception e){
			e.printStackTrace();
		}
		return result;
	}
	
	@Override
	public List<ConcurrentWorkflowInstance> getConcurrentCases() {
		
		logger.debug("inside getConcurrentCases()");
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <ConcurrentWorkflowInstance> instanceList = new ArrayList <ConcurrentWorkflowInstance>();

		String query = "SELECT BPD_INSTANCE_ID, INSTANCE_NAME, EXECUTION_STATUS, CREATE_DATETIME FROM LSW_BPD_INSTANCE WHERE INSTANCE_NAME LIKE 'Concurent Process%' and EXECUTION_STATUS in (1,6) ORDER BY CREATE_DATETIME";
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			List<Map<String, Object>> rows = sqlQuery.list();
			for (Map row : rows) {
				ConcurrentWorkflowInstance instance = new ConcurrentWorkflowInstance ();
				instance.setInstanceId(row.get("BPD_INSTANCE_ID").toString());
				instance.setInstanceName((String)row.get("INSTANCE_NAME"));
				instance.setExecutionStatus((String)row.get("EXECUTION_STATUS").toString());
				instance.setCreated((String)row.get("CREATE_DATETIME").toString());
				instanceList.add(instance);
			}
		}catch (Exception e){
			e.printStackTrace();
		}
		
		return instanceList;
	}
	
	@Override
	public List<WorkItem> getStuckConcurrentCases() {
		List <WorkItem> workitems = new ArrayList <WorkItem>();
		
		String query="SELECT a.bpd_instance_id, a.activity_name, a.status, a.rcvd_datetime, " +
					"round(a.dif) DaysStuck, b.string_value POLICYNO, c.name,d.name,f.string_value CONCURRENT_POLICY FROM " +
					"(select  status, (sysdate - rcvd_datetime) dif, bpd_instance_id, rcvd_datetime,activity_name from lsw_task " +
					"where activity_name like '%Wait%') a, " +
					"(select bpd_instance_id,string_value from lsw_bpd_instance_variables where alias='POLICYNO') b, " +
					"lsw_snapshot c, lsw_project d,lsw_bpd_instance e, " +
					"(select bpd_instance_id,string_value from lsw_bpd_instance_variables where alias='CONCURRENTPOLICY') f " +
					"where dif > 0 " +
					"and a.bpd_instance_id = b.bpd_instance_id(+) " +
					"and a.bpd_instance_id = f.bpd_instance_id(+) " +
					"and a.bpd_instance_id = e.bpd_instance_id " +
					"and e.snapshot_id = c.snapshot_id " +
					"and d.project_id = c.project_id " +
					"and a.status = 12 " +
					"order by daysstuck desc";
		
		jdbcTemplate = new JdbcTemplate(dataSource);		
				
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);
		for (Map row : rows) {
			WorkItem workItem = new WorkItem();
			workItem.setInstanceid((BigDecimal)row.get("bpd_instance_id"));
			workItem.setPolicy_no((String)row.get("POLICYNO")); 
			workItem.setTask_activity_name((String)row.get("activity_name"));
			workItem.setTask_rcvd_datetime((Date)row.get("rcvd_datetime"));
			workItem.setConcurrent_policy((String)row.get("CONCURRENT_POLICY"));
			workitems.add(workItem);
		}			
		
		return workitems;
	}


	@Override
	public int getPendingTrackingGroupRecordTransfer() {
		// TODO Auto-generated method stub
		
		String query = "SELECT count (*) as pending_count from LSW_PERF_DATA_TRANSFER ";
		int pendingCount=0;
		jdbcTemplate = new JdbcTemplate(dataSource);		
		
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);		
		
		for (Map row : rows) {
			
			BigDecimal pCount = (BigDecimal)row.get("pending_count");
			pendingCount = pCount.intValue();
		}	
		
		return pendingCount;
	}
	
	
	@Override
	public List<WorkItem> getErrorCases() {
		// TODO Auto-generated method stub
		List <WorkItem> workitems = new ArrayList <WorkItem>();
		String query = "SELECT distinct lsw_bpd_instance_variables.string_value POLICYNO, LSW_BPD_INSTANCE.INSTANCE_NAME, lsw_task.rcvd_datetime FROM lsw_task, LSW_BPD_INSTANCE, lsw_bpd_instance_variables WHERE subject LIKE '%Error:%' AND "
				+ "LSW_BPD_INSTANCE.bpd_instance_id IS NOT NULL "
				+ "AND LSW_TASK.BPD_INSTANCE_ID = LSW_BPD_INSTANCE.BPD_INSTANCE_ID "
				+ "and lsw_bpd_instance_variables.ALIAS = 'POLICYNO' "
				+ "and lsw_bpd_instance_variables.bpd_instance_id = LSW_BPD_INSTANCE.BPD_INSTANCE_ID ";
		
		jdbcTemplate = new JdbcTemplate(dataSource);		
				
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);
		for (Map row : rows) {
			WorkItem workItem = new WorkItem();
			workItem.setPolicy_no((String)row.get("POLICYNO")); 
			workItem.setInstance_name((String)row.get("INSTANCE_NAME"));
			workItem.setTask_rcvd_datetime((Date)row.get("RCVD_DATETIME"));
			workitems.add(workItem);
		}		

		return workitems;
	}

	@Override
	public List<WorkItem> getFailedToSuspend() {
		
		// TODO Auto-generated method stub
		List <WorkItem> workitems = new ArrayList <WorkItem>();
		String query = "Select  *  From Lombardi.Lsw_Bpd_Instance Lbi , Lombardi.Lsw_Task Lt , "
				+ " Lsw_Usr_Grp_Xref Lugx, Lombardi.Lsw_Bpd_Instance_Variables Lbiv00, "
				+ " Lombardi.Lsw_Bpd_Instance_Variables Lbiv01 Where Lbiv00.Bpd_Instance_Id(+) = Lbi.Bpd_Instance_Id "
				+ " And Lbi.Bpd_Instance_Id = Lt.Bpd_Instance_Id And Lugx.Group_Id = Lt.Group_Id   "
				+ "And Lbiv00.Alias(+) = 'POLICYNO'  And Lbiv01.Bpd_Instance_Id(+) = Lbi.Bpd_Instance_Id     "
				+ "  And Lbiv01.Alias(+) = 'LOCATION' And  Lt.Activity_Name    IN   ('GSUWH_Followup Suspend',"
				+ "'GSPY_Followup Suspend','GSH_Followup Suspend','GSUW_Followup Suspend','GS_Followup Suspend',"
				+ "'GSPYH_Followup Suspend', 'Pending Issuance', 'Further Requirements', 'GSPY_PendCheque Suspend', "
				+ "'APS PostPone', 'Suspend', 'REQ Suspend', 'GSPY_ Rgnl Screening Suspend', 'PLA Reopen Suspend')"
				+ " And Lt.User_Id = -1 And Lt.Status = 12 And Lugx.Description = 'All Users' and Lbi.Execution_Status = 1  "
				+ "And Rownum <=  2000	";
		jdbcTemplate = new JdbcTemplate(dataSource);		
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);
		for (Map row : rows) {
			WorkItem workItem = new WorkItem();
			workItem.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
			workItem.setInstance_name((String)row.get("INSTANCE_NAME"));
			workItem.setTask_subject((String)row.get("SUBJECT"));
			workItem.setExecution_status((BigDecimal)row.get("EXECUTION_STATUS"));
			workItem.setTask_rcvd_datetime((Date)row.get("RCVD_DATETIME"));
		
			workitems.add(workItem);
		}		

		return workitems;	
	
	}
	
	
	@Override
	public List<WorkItem> getLatestTaskHistory(String policyno, int maxtasks) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <WorkItem> taskHistory = new ArrayList <WorkItem>();
		String query = "select b.bpd_instance_id, b.instance_name, a.subject, a.rcvd_datetime  "
				+ "from lsw_task a, lsw_bpd_instance b "
				+ "where a.bpd_instance_id = b.bpd_instance_id "
				+ "and instance_name like :policyno " 
				+ "order by a.rcvd_datetime desc ";
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("policyno", "%"+policyno+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	
		
			int taskCount=0;
			
			for (Map row : rows) {
				WorkItem task = new WorkItem();
				task.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
				task.setInstance_name((String)row.get("INSTANCE_NAME"));
				task.setTask_subject((String)row.get("SUBJECT"));
				task.setTask_rcvd_datetime((Date)row.get("RCVD_DATETIME"));
		
				taskHistory.add(task);
				taskCount++;
				if (taskCount>maxtasks)
					break;
			}	
		}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return taskHistory;
	}
	
	//new method added on 1-Mar-2018
	@Override
	public List<WorkItem> getInstanceLastTaskByInstanceId(String instanceid) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <WorkItem> taskRecords = new ArrayList <WorkItem>();
		String query = "Select b.Bpd_Instance_Id, b.Instance_Name, a.Task_Id,a.Subject,a.Status,b.Create_Datetime, a.Rcvd_Datetime,a.Close_Datetime "
				+ " From Lsw_Task a, Lsw_Bpd_Instance b"
				+ " where a.Bpd_Instance_id like :instanceid and a.Bpd_Instance_Id = b.Bpd_Instance_id"
				+ " and a.Status='12' order by b.Bpd_Instance_Id";

		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("instanceid", "%"+instanceid+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	

			for (Map row : rows) {
			
				WorkItem tracker = new WorkItem ();
				tracker.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
				tracker.setInstance_name((String)row.get("INSTANCE_NAME"));
				tracker.setTask_id((BigDecimal)row.get("TASK_ID"));
				tracker.setTask_subject((String)row.get("SUBJECT"));
				tracker.setStatus((String)row.get("STATUS"));
				tracker.setCreate_datetime((Date)row.get("CREATE_DATETIME"));
				tracker.setTask_rcvd_datetime((Date)row.get("RCVD_DATETIME"));
				tracker.setTask_closed_datetime((Date)row.get("CLOSE_DATETIME"));
				taskRecords.add(tracker);
				}
			}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return taskRecords;
		
	}
	
	//new method added on 1-Mar-2018
	@Override
	public List<WorkItem> getInstanceLastTaskByPolicyNo(String policyno) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <WorkItem> taskRecords = new ArrayList <WorkItem>();
		String query = "Select b.Bpd_Instance_Id, b.Instance_Name, a.Task_Id,a.Subject,a.Status,b.Create_Datetime, a.Rcvd_Datetime,a.Close_Datetime "
				+ " From Lsw_Task a, Lsw_Bpd_Instance b"
				+ " where b.Instance_Name like :policyno and a.Bpd_Instance_Id = b.Bpd_Instance_id"
				+ " and a.Status='12' order by b.Bpd_Instance_Id";
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("policyno", "%"+policyno+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	

			for (Map row : rows) {
				WorkItem tracker = new WorkItem ();
				tracker.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
				tracker.setInstance_name((String)row.get("INSTANCE_NAME"));
				tracker.setTask_id((BigDecimal)row.get("TASK_ID"));
				tracker.setTask_subject((String)row.get("SUBJECT"));
				tracker.setStatus((String)row.get("STATUS"));
				tracker.setCreate_datetime((Date)row.get("CREATE_DATETIME"));
				tracker.setTask_rcvd_datetime((Date)row.get("RCVD_DATETIME"));
				tracker.setTask_closed_datetime((Date)row.get("CLOSE_DATETIME"));
				taskRecords.add(tracker);
			}
		}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return taskRecords;
		
	}
	
	//new method added on 1-Mar-2018
	@Override
	public List<WorkItem> getInstanceVariables(String instanceid) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
        
		List <WorkItem> varRecords = new ArrayList <WorkItem>();
		String query = "Select Variable_Name,String_Value From Lsw_Bpd_Instance_Variables "
				+ "where Bpd_Instance_id like :instanceid order by Variable_Name";
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("instanceid", "%"+instanceid+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	

			for (Map row : rows) {
				WorkItem tracker = new WorkItem ();
				tracker.setVariable_name((String)row.get("VARIABLE_NAME"));
				tracker.setVariable_str_value((String)row.get("STRING_VALUE"));
				varRecords.add(tracker);
			}
		}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return varRecords;
		
	}
	
	@Override
	public List<WorkItem> getActiveInstance(String policyNo) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <WorkItem> varRecords = new ArrayList <WorkItem>();

		String query = "SELECT BPD_INSTANCE_ID, INSTANCE_NAME, EXECUTION_STATUS" +
				" FROM LSW_BPD_INSTANCE" +
				" WHERE INSTANCE_NAME LIKE :policyNo and EXECUTION_STATUS in (1,6)";	
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("policyNo", "%"+policyNo+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	

			for (Map row : rows) {
				WorkItem tracker = new WorkItem ();
				tracker.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
				tracker.setInstance_name((String)row.get("INSTANCE_NAME"));
				tracker.setExecution_status((BigDecimal)row.get("EXECUTION_STATUS"));
				varRecords.add(tracker);
			}
		}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return varRecords;
		
	}
	
	public String getActiveInstanceId(String policyNo) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		String instanceId;
		WorkItem tracker = null;
		List <WorkItem> varRecords = new ArrayList <WorkItem>();
		
		String query = "SELECT BPD_INSTANCE_ID FROM LSW_BPD_INSTANCE" +
				" WHERE INSTANCE_NAME LIKE :policyNo and EXECUTION_STATUS in (1,6)";	
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("policyNo", "%"+policyNo+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	
			
			for (Map row : rows) {
				tracker = new WorkItem ();
				tracker.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
				varRecords.add(tracker);
				}
			}catch (Exception ex){
			ex.printStackTrace();
			}
		
		instanceId = tracker.getInstanceid().toString();
		
		return instanceId;
		
	}
	
	public String getBPMGroup() {
		// TODO Auto-generated method stub
		String result;
		List varRecords = new ArrayList();
		StringBuffer sb = new StringBuffer();

		String query = "select group_name from lsw_usr_grp_xref where group_type = 3";	

		jdbcTemplate = new JdbcTemplate(dataSource);	
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);

		for (Map row : rows) {
			varRecords.add(row.get("group_name"));
		}
		
		for (int i=0;i<varRecords.size();i++){
			sb.append(varRecords.get(i));
			if(i<varRecords.size()){
				sb.append(",");
			}
		}
		
		result = sb.toString();
		
		return result;
		
	}
	
	
	public int failedCaseMarkResolved(String instanceId) { 
		// TODO Auto-generated method stub
		
        Handle handle = null;
        int updCount = 0;
        DBI dbi = new DBI(dataSource);
        
		String query = "update lsw_bpd_instance  set EXECUTION_STATUS=2 where EXECUTION_STATUS=3 and bpd_instance_id =:instanceId";
		
		try{
			handle = dbi.open();
			Update sqlUpdate = handle.createStatement(query);
			sqlUpdate.bind("instanceId", instanceId);
			updCount = sqlUpdate.execute();
		}catch (Exception ex){
			ex.printStackTrace();
		}finally{
			handle.close();
		}
		
		return updCount;
				
	}
	
	
	@Override
	public List<WorkItem> trackPolicyInstance(String instanceid) {
		// TODO Auto-generated method stub
		List <WorkItem> varRecords = new ArrayList <WorkItem>();
		
		Handle handle = null;
        int recCount = 0;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		String query = "select a.TASK_ID, a.SUBJECT, c.group_name, " +
		"b.USER_NAME, to_char(a.RCVD_DATETIME,'DD-MON-YYYY HH:MI:SS') RCVD_DATETIME, " +
		"to_char(a.CLOSE_DATETIME, 'DD-MON-YYYY HH:MI:SS') CLOSED_DATETIME, " +
		"case " +
		"when a.STATUS = '12' then 'Received' " +
		"when a.STATUS = '32' then 'Closed' " +
		"end STATUS " +
		"from lsw_task a, " +
		"(select user_id, user_name from lsw_usr_xref) b, " +
		"(select group_id, group_name from lsw_usr_grp_xref) c " +
		"where a.USER_ID = b.USER_ID(+) " +
		"and a.group_id = c.group_id(+) " +
		"and a.BPD_INSTANCE_ID = :instanceId " +
		"order by a.TASK_ID";

		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("instanceId", instanceid);
			List<Map<String, Object>> rows = sqlQuery.list();

		for (Map row : rows) {
			
			WorkItem tracker = new WorkItem ();
			tracker.setTask_id((BigDecimal)row.get("TASK_ID"));
			tracker.setTask_subject((String)row.get("SUBJECT"));
			tracker.setTask_assigned_group((String)row.get("GROUP_NAME"));
			tracker.setTask_assigned_user((String)row.get("USER_NAME"));
			tracker.setRcvd_datetime((String)row.get("RCVD_DATETIME"));
			tracker.setClosed_datetime((String)row.get("CLOSED_DATETIME"));
			tracker.setStatus((String)row.get("STATUS"));
			
			varRecords.add(tracker);
		}
		}catch(Exception ex){
			ex.printStackTrace();
		}finally{
			handle.close();
		}
		
		return varRecords;
		
	}
}
------------------------------------------------------------------------------



package sg.com.dao;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;
import javax.sql.DataSource;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import sg.com.dto.WorkItem;
@Component("workitemdao")
public class WorkItemDAOImpl implements WorkItemDAO {

	private static Log logger=LogFactory.getLog(BORequestTrackerDAOImpl.class);

	@Resource(name="dataSourceTWPROC")
	private DataSource dataSource;
	

	private JdbcTemplate jdbcTemplate; 
	
	@Override
	public List<WorkItem> getFailedCasesList() {
		List <WorkItem> workitems = new ArrayList <WorkItem>();
		String query = "SELECT lbi.bpd_instance_id, lbi.instance_name, lbi.execution_status from  lsw_bpd_instance lbi where LBI.EXECUTION_STATUS = 3";
		
		jdbcTemplate = new JdbcTemplate(dataSource);		
				
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);
		for (Map row : rows) {
			WorkItem workItem = new WorkItem();
			workItem.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
			workItem.setInstance_name((String)row.get("INSTANCE_NAME"));
			workItem.setExecution_status((BigDecimal)row.get("EXECUTION_STATUS"));
		
			workitems.add(workItem);
		}

		return workitems;
	}

	@Override
	public List<WorkItem> getStuckCases() {

		List <WorkItem> workitems = new ArrayList <WorkItem>();

		String query = "SELECT bpd_instance_id, instance_name, execution_status, rcvd_datetime, activity_name FROM "
				+ "(Select lbi.Bpd_Instance_Id,  lbi.instance_name,  lbi.execution_status,  lt.Task_Id,  lt.activity_name, lt.rcvd_datetime,"
				+ "lt.Close_Datetime, lt.status, ROW_NUMBER () OVER (PARTITION BY  lt.bpd_instance_id ORDER BY  lt.close_datetime "
				+ "DESC NULLS FIRST) maxrownum  FROM  lsw_bpd_instance lbi,  lsw_snapshot ls , lsw_task lt "
				+ "WHERE lbi.bpd_instance_id = lt.bpd_instance_id "
				+ "AND ls.snapshot_id = lbi.snapshot_id AND lbi.execution_status In (1 , 6)) tbl "
				+ "WHERE maxrownum = 1 AND tbl.status = 32"
				+ "    And Tbl.Activity_Name != 'Read Concurent Data'"
				+ "    And Tbl.Close_Datetime < Sysdate-10/1440";

		jdbcTemplate = new JdbcTemplate(dataSource);		
				
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);
		for (Map row : rows) {
			WorkItem workItem = new WorkItem();
			workItem.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
			workItem.setInstance_name((String)row.get("INSTANCE_NAME"));
			workItem.setExecution_status((BigDecimal)row.get("EXECUTION_STATUS"));
			workItem.setTask_activity_name((String)row.get("activity_name"));
			workItem.setTask_rcvd_datetime((Date)row.get("rcvd_datetime"));
		
			workitems.add(workItem);
		}		
		return workitems;
	}

	@Override
	public List<WorkItem> getExceptionCases() {
		// TODO Auto-generated method stub
		List <WorkItem> workitems = new ArrayList <WorkItem>();
		
		String query = "SELECT lbi.bpd_instance_id, lbi.instance_name, lt.activity_name, lbi.execution_status, "
		+ "to_char(lt.rcvd_datetime,'YYYY-MM-DD HH24:MI:SS') as RCVD_DATETIME from lsw_bpd_instance lbi ,  "
		+ "lsw_task lt WHERE lbi.bpd_instance_id = lt.bpd_instance_id  "
		+ " AND lbi.execution_status = 1   and lt.status  IN (11,12) and lt.activity_name "
		+ " like ('%Exception Manager%') and lt.activity_name <> 'DMU Exception Manager' order by lt.rcvd_datetime desc";		
		jdbcTemplate = new JdbcTemplate(dataSource);		
				
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);
		for (Map row : rows) {
			WorkItem workItem = new WorkItem();
			workItem.setInstanceid((BigDecimal)row.get("BPD_INSTANCE_ID"));
			workItem.setInstance_name((String)row.get("INSTANCE_NAME"));
			workItem.setTask_activity_name((String)row.get("ACTIVITY_NAME"));
			workItem.setExecution_status((BigDecimal)row.get("EXECUTION_STATUS"));
			workItem.setRcvd_datetime((String)row.get("RCVD_DATETIME"));
			workitems.add(workItem);
		}		
		
		return workitems;
	}

}
-------------------------------------------------------

package sg.com.dao;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;
import javax.sql.DataSource;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.skife.jdbi.v2.DBI;
import org.skife.jdbi.v2.Handle;
import org.skife.jdbi.v2.Query;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import sg.com.dto.User;
import sg.com.dto.WorkItem;

@Component("userdao")
public class UserDAOImpl implements UserDAO 

{
	
	@Resource(name="dataSourceDSCONFIG")
	DataSource datasourceDSCONFIG;
	
	@Resource(name="dataSourceTWPROC")
	DataSource dataSource;
	
	
	
	
	private static Log logger=LogFactory.getLog(UserDAOImpl.class);


	@Override
	public List <User> selectUserByUserID(String userID)
	{

		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <User> userdetails = new ArrayList <User>();
		String query = "select a.user_name, a.full_name, b.group_name, a.user_id "
				+ "from lsw_usr_xref a,lsw_usr_grp_xref b,lsw_usr_grp_mem_xref c "
				+ "where "
				+ "a.user_id = c.user_id And C.Group_Id = B.Group_Id "
				+ "And A.User_Name In (:userID) and b.group_name !='tw_allusers' Order By A.User_Name,B.Group_Name";
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("userID", userID);
			List<Map<String, Object>> rows = sqlQuery.list();	
		
			for (Map row : rows) {
				User userdtl = new User();
				userdtl.setUserid((BigDecimal)row.get("USER_ID"));
				userdtl.setUserName((String)row.get("USER_NAME"));
				userdtl.setDeptName((String)row.get("GROUP_NAME"));
				userdtl.setFullName((String)row.get("FULL_NAME"));
				userdetails.add(userdtl);
				}
		}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return userdetails;
		
	}


	@Override
	public List <User> selectUserByFullName(String fullName) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <User> userdetails = new ArrayList <User>();
		String query = "select a.user_name, a.full_name, b.group_name, a.user_id "
				+ "from lsw_usr_xref a,lsw_usr_grp_xref b,lsw_usr_grp_mem_xref c "
				+ "where "
				+ "a.user_id = c.user_id And C.Group_Id = B.Group_Id "
				+ "And A.Full_Name like :fullName Order By A.User_Name,B.Group_Name";
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("fullName",  "%"+fullName+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	
		
			for (Map row : rows) {
			User userdtl = new User();
			userdtl.setUserid((BigDecimal)row.get("USER_ID"));
			userdtl.setUserName((String)row.get("USER_NAME"));
			userdtl.setDeptName((String)row.get("GROUP_NAME"));
			userdtl.setFullName((String)row.get("FULL_NAME"));
		
			userdetails.add(userdtl);
			}
		}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return userdetails;
	}
		
	@Override
	public List <User> getUserRoleThresholds (String userid) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
        
		JdbcTemplate jdbcTemplate = new JdbcTemplate(datasourceDSCONFIG);
		
		List <User> userdetails = new ArrayList <User>();
		
		String query = "SELECT a.USERNAME, b.ROLENAME, c.MINTHRESHOLD, c.MAXTHRESHOLD "
				+ "FROM TDUSERMASTER a, TDROLEMASTER b, TDUSERROLECONFIG c "
				+ "WHERE a.USERID = c.USERID "
				+ "AND b.ROLEID = c.ROLEID "
				+ "AND a.USERNAME = :userid "
				+ "ORDER BY b.ROLENAME";
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("userid", userid);
			List<Map<String, Object>> rows = sqlQuery.list();	
		
		for (Map row : rows) {
			User userdtl = new User();
			userdtl.setUserName((String)row.get("USERNAME"));
			userdtl.setDeptName((String)row.get("ROLENAME"));
			userdtl.setMinThreshold((BigDecimal)row.get("MINTHRESHOLD"));
			userdtl.setMaxThreshold((BigDecimal)row.get("MAXTHRESHOLD"));
			userdetails.add(userdtl);
			}
		}catch (Exception ex){
			ex.printStackTrace();
		}
		
		return userdetails;
	}  


	
}
----------------------------------------

package sg.com.dao;

import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.Date;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;
import javax.sql.DataSource;

import org.skife.jdbi.v2.DBI;
import org.skife.jdbi.v2.Handle;
import org.skife.jdbi.v2.Query;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import sg.com.dto.WorkItem;

@Component("TWPROCXDAO")
public class TWPROCXDAOImpl implements TWPROCXDAO {

	@Resource(name="dataSourceTWPROCX")
	DataSource dataSource;
	
	JdbcTemplate jdbcTemplate;
	
	@Override
	public List<WorkItem> getPolicyHistory(String policyno) {
		// TODO Auto-generated method stub
		
		Handle handle = null;
        Query sqlQuery = null;
        DBI dbi = new DBI(dataSource);
		
		List <WorkItem> taskHistoryRecords = new ArrayList <WorkItem>();
		String query = "Select Instance_Name,Task_Id,Task_Activity_Name,Cmtimestamp,Channel,Snapshot_Name,"
				+ "Assigned_To_User,Documenttype,Docid,Instance_Status,Docid,Task_Rcvd_Date,Task_Closed_Date,"
				+ "Task_Status From Prl_V_Task_Search "
				+ "where  instance_name like :policyno order by  instance_name,task_rcvd_date, task_closed_date";
		
		try{
			handle = dbi.open();
			sqlQuery = handle.createQuery(query);
			sqlQuery.bind("policyno", "%"+policyno+"%");
			List<Map<String, Object>> rows = sqlQuery.list();	
			
			for (Map row : rows) {
				WorkItem tracker = new WorkItem ();
				tracker.setInstance_name((String)row.get("INSTANCE_NAME"));
				tracker.setTask_subject((String)row.get("TASK_ACTIVITY_NAME"));
				tracker.setTask_assigned_user((String)row.get("ASSIGNED_TO_USER"));
				tracker.setDocid((String)row.get("DOCID"));
				tracker.setCmtimestamp((String)row.get("CMTIMESTAMP"));
				tracker.setInstance_status((String)row.get("instance_status"));
				tracker.setTask_rcvd_datetime((Date)row.get("task_rcvd_date"));
				tracker.setTask_closed_datetime((Date)row.get("task_closed_date"));
			
				taskHistoryRecords.add(tracker);
				}
			}catch(Exception ex){
				ex.printStackTrace();
			}
		
		return taskHistoryRecords;
		
	}
	
	@Override
	public List<WorkItem> getMissedOutTWPROCXArchive() {
		// TODO Auto-generated method stub
		List twprocxArchiveList = new ArrayList();
		List <WorkItem> missedOutArchiveList = new ArrayList <WorkItem>();
		String query="select to_char(datetime_stamp,'YYYY-MM-DD') as ARCHIVE_DATE, count(Instance_Id) as ARCHIVE_COUNT from aviva_instance "
					+"where to_char(datetime_stamp,'YYYY-MM-DD') >= to_char(sysdate-7,'YYYY-MM-DD') "
					+"and to_char(datetime_stamp,'YYYY-MM-DD') <= to_char(sysdate,'YYYY-MM-DD') "
					+"group by to_char(datetime_stamp,'YYYY-MM-DD') "
					+"order by to_char(datetime_stamp,'YYYY-MM-DD') desc";

		jdbcTemplate = new JdbcTemplate(dataSource);		
		
		List<Map<String, Object>> rows = jdbcTemplate.queryForList(query);
		WorkItem missedOutWi = new WorkItem();
		
		DateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
		
		for (Map row : rows) {
			twprocxArchiveList.add(row.get("ARCHIVE_DATE").toString());
		}
		
		for (int x=0;x<8;x++){
			
			boolean dateFound = false;
			Calendar cal = Calendar.getInstance();
			cal.add(Calendar.DATE, -x);
			String dateString=dateFormat.format(cal.getTime());
		
			for(int a=0;a<twprocxArchiveList.size();a++){
				
				if(twprocxArchiveList.get(a).equals(dateString)){
					dateFound = true;
					break;
				}
			}
			
			if (!dateFound){
				missedOutWi.setArchive_date(dateString);
				missedOutWi.setArchive_count(0);
				missedOutArchiveList.add(missedOutWi);
			}
		}
		
		return missedOutArchiveList;
		
	}

}
---------------------------------------------
