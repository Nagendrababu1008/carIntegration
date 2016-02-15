# carIntegration
package td.bdops.executionPoint;


import java.io.File;
import java.io.IOException;
import java.util.*;

import org.apache.log4j.Logger;

import td.bdops.DAO.DAO;
import td.bdops.DAO.UpdateOrderInfo;
import td.bdops.DAO.UpdateStatusInRespectiveFlow;
import td.bdops.EmptySignal.CreateEmptySignal;
import td.bdops.UtilsClass.ExceptionHandling;
import td.bdops.UtilsClass.ExtractFile;
import td.bdops.UtilsClass.LogFile;
import td.bdops.UtilsClass.SendMail;
import td.bdops.UtilsClass.ToolTerminate;
import td.bdops.UtilsClass.UtilMethod;
import td.bdops.assesment.Assesment;
import td.bdops.contentList.ContentList_Main;
import td.bdops.cor.Cor;
import td.bdops.database.DataBase;
import td.bdops.databaseNew.DbUtil;
import td.bdops.mail.MailNotify;
import td.bdops.ordertracker.ZipFileTracker;
import td.bdops.resupply.Resupply;
import td.bdops.runVtoolProcess.Vtool;
import td.bdops.suppSolvedProblem.SPSO_CL;
import td.bdops.suppSolvedProblem.SolvedProblem;
import td.bdops.tool.issue.ELS.ELS_POJO;
import td.bdops.tool.mix.BMC.BMC_POJO;
import td.bdops.tool.mix.BPG.BPG;
import td.bdops.tool.mix.BPG.BPG_POJO;
import td.bdops.tool.mix.SEG.SEG_POJO;

public class CLELE_Extraction
{
	public static Logger infoLogger = Logger.getLogger("infolog");
	public static String flow;
	public static ArrayList<Object> zipFile;
	public static ArrayList<Object> xmlArrayList;
	public static String incomingPath;
	public static boolean AIPFlag;
	public static String zip_path;
//	public static ArrayList<Object> issnArrayList;
	public static String dbTable;
	public static String zipFileName;
	public static String orderZipName;
	public static String clPath;
	public static Properties prop=null;
	public static String paths;
	public static String propertiesFilePath;
	public static boolean oni;
	public static boolean checkCarResup=true;
	public static boolean checkCarSolPro=true;
	public static String findingType;
	public static  boolean updateFlag=false;
	public static boolean terminate=false;
	public static String orderxmlPath="";
	public static String outputXmlPath="";
	public static String outputXml="";
	public static String deleleTable="";
	public static String tableName="";
	public static String checkNORMAL="YES";
	
	public static void main(String[] args)
	{
		try {
			System.out.println("Extraction Jar Started Processing");
			LivePath livepath=new LivePath();
			
			
			if(args !=null && args.length>0)
			{
				livepath.runLivePath(args);
			}
			else
			{
				livepath.runLocalPath();
			}
			
			livepath.setVersion();
				
			infoLogger.info("************  INPUT PATH  : "+paths+" ***********");
			
			//start cl
			clPath=prop.getProperty("CL_Production_Path");
			checkNORMAL=prop.getProperty("RUN_FOR_CONFERENCE").trim();
			//	checkNORMAL="NO";
			UtilMethod utilMethod=new UtilMethod();
			

			utilMethod.checkAip();
		
			String zipname=utilMethod.getZipName(paths);
			orderZipName=zipname;
			System.out.println("orderZipName----->"+orderZipName);//----->On 14-10-2015
			zipname=zipname.substring(0,zipname.lastIndexOf("."));
		
			if(new DAO().check_D_orderInfo(orderZipName))
			{
				new UpdateOrderInfo().updateDate();
				
				infoLogger.info(orderZipName +" is extracting... Please wait...");
				boolean	errorUnzip=new ExtractFile().unZip(paths);
				String	paths1=paths.substring(0,paths.lastIndexOf("."));
				//System.out.println("paths---------->"+paths1);//----->On 14-10-2015
		
				if(errorUnzip)
				{
					new UpdateOrderInfo().updateRemark("ZIP FILE CORRUPT");
					new SendMail().errorMessage("ZIP FILE CORRUPT : "+LogicClass.orderId, LogicClass.orderId, CLELE_Extraction.paths, "ZIP FILE HAVE NOT PROPERLY EXTRACT  : "+CLELE_Extraction.paths,CLELE_Extraction.flow);
					new ToolTerminate().terminate();
				}
				else
				{
				zip_path=(paths1.substring(0,paths1.lastIndexOf("\\"))).trim();							
				orderxmlPath =zip_path+"\\"+zipname+"\\order\\order.xml";
				
				//infoLogger.info("order.xml path----->"+orderxmlPath);
				
				infoLogger.info("Reading the order.xml file");
				ReadOrderXml read= new ReadOrderXml();
				findingType=read.readValueFromOrderXml(orderxmlPath);
				infoLogger.info("Successfully read the order.xml");
 				infoLogger.info("Process type ---->  "+findingType);
				
				infoLogger.info("Going to update duedate");
				new UpdateOrderInfo().updateDueDate();
				infoLogger.info("duedate updated");
				
				
				LogicClass.projectName=read.getCarBuldingValue();
				
				
				if(LogicClass.orderType.contains("SUPPLIER-PROBLEM-SOLVED"))
				{
					MailNotify.MailNote("Supplier problem solved order recieved. </BR></BR>ORDER ID :"+LogicClass.orderId+"</BR></BR>FLOW :"+CLELE_Extraction.flow+"</BR></BR>PATH :"+CLELE_Extraction.paths, "BDOPS : SUPPLIER PROBLEM SOLVED RECIEVED");
				}
				
				/*
				 * Code for delete existing record form cl_loginfo table except conference, mix and bookseries
				 * 
				 */
				
				if(!(findingType.contains("CAR") || findingType.equals("COR") || findingType.equals("ASSESSMENT") || findingType.equals("INDEX")))
				{
					infoLogger.info("Going to delete records from cl_loginfo table");
					new UpdateOrderInfo().deleteLogValues();
					infoLogger.info("Record deleted from cl_loginfo table");
				}
				
				
				 //------------------------CAR AND CAR  RESUPPLY -----------------------------------------
				if(findingType.contains("CAR"))
				{
					LogicClass.carsuppUnitId = LogicClass.carsuppUnitId.replace("_DPR", "");
					
					if(findingType.equals("CAR-RESUPPLY"))
					{
						new Car_Run().execute(orderxmlPath);
					}
					if(findingType.equals("CAR"))
					{
						new Car_Run().execute(orderxmlPath);
					}
				}
				
				 //------------------------  COR  -----------------------------------------
				else if(findingType.equals("COR"))
				{
					outputXmlPath=zip_path+"\\"+zipname+"\\ItemFile";
					infoLogger.info("*_output_1.xml path----->"+outputXmlPath);
					
					infoLogger.info("Finding output xml for COR order....");
					File f=new File(outputXmlPath);
					File list[]=f.listFiles();
					for(int i=0;i<list.length;i++)
					{
						if(list[i].getName().contains("output")&&list[i].getName().endsWith(".xml"))
							outputXml=list[i].getAbsolutePath();
						
					}
					infoLogger.info("Output xml path----->"+outputXml);
					new Cor(orderxmlPath,outputXml);
					
				} 
				//------------------------  ASSESMENT  -----------------------------------------
				else if(findingType.equals("ASSESSMENT"))
				{
					new Assesment(orderxmlPath);
				}
				else if(findingType.equals("INDEX"))
				{
				//	new CtcIndex().intregateCtcIndex();
				}
				//------------------------  CONTENT LIST  -----------------------------------------
				else
				{
					if(findingType.equalsIgnoreCase("RESUPPLY"))
					{
						SendMail.ForResupply("Resupply order received", LogicClass.orderId, CLELE_Extraction.paths,"","We Recevied the resupply order");
						//new Resupply(); Commented on 12-10-2015
						new Resupply(orderxmlPath);
					}
					
					else if(findingType.contains("SUPPLIER-PROBLEM-SOLVED"))
			 		{
			 			String type = findingType.substring(0,findingType.indexOf("-"));
			 			if(type.equalsIgnoreCase("CAR"))
			 			{
			 				CLELE_Extraction.flow="SPSO";
			 				new SolvedProblem(orderxmlPath);
			 			}
			 			else if(type.equalsIgnoreCase("CONTENTLIST"))
			 			{
			 				new SPSO_CL();                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  
			 			}
			 		}
					
					
					new CommonMethod().check_ItemFileNotFound(zipname, zip_path);
					
					
					/**
					 * checkCarResup=true
					 * 	then resupply is for the content list creation
					 * false=it is car
					 * */
					
					if(checkCarResup || checkCarSolPro)
					{
						String orderItemFile =zip_path+"\\"+zipname+"\\ItemFile";
						infoLogger.info(orderItemFile +" are extracting .........");
						boolean errorUnzip1	= ExtractFile.unZip2(orderItemFile);
				//		boolean errorUnzip1	= false;
						if(errorUnzip1)
						{
							new UpdateOrderInfo().updateRemark("ZIP FILE CORRUPT : "+orderItemFile );
							new ToolTerminate().terminate();
						}
						else
						{
							//orderItemFile="D:\\OpsBank\\ORDERS\\PROCESS\\36696_1\\ItemFile\\";
							new ExtractFile().extractFileIncoming(orderItemFile);
			 				new ContentList_Run().execute(utilMethod,orderItemFile);
			 				
			 				if(!(flow.equalsIgnoreCase("INTRES")))
			 				{
				 				infoLogger.info("*********   ORDER MATERIAL TRACKING STARTED  **********");
				 				ZipFileTracker.callZipFIleTracker(CLELE_Extraction.incomingPath, LogicClass.orderId);
				 				infoLogger.info("*********   ORDER MATERIAL TRACKING FINISHED  **********");
			 				}
			 				
			 				infoLogger.info("*********   CONTENT LIST CREATION  **********");
			 				
			 				if(!(CommonVariable.missingFileList.size()==0))
			 				{
			 					if(CLELE_Extraction.flow.equals("SEG") || CLELE_Extraction.flow.equals("OUP") || CLELE_Extraction.flow.equals("BMC") || CLELE_Extraction.flow.equals("SPR") || CLELE_Extraction.flow.equals("BPG")  || CLELE_Extraction.flow.equals("APA"))
			 					{
			 						String pathSrc=CLELE_Extraction.zip_path+"\\"+LogicClass.orderId+"_"+"MissingFile.log";
				 					new LogFile().createLogFile(pathSrc,CommonVariable.missingFileList);
				 					new UpdateOrderInfo().updateRemark_WithAppend("File Missing,");
			 					}
			 					else{
			 						String pathSrc=CLELE_Extraction.zip_path+"\\"+LogicClass.orderId+"_"+"MissingFile.log";
				 					new LogFile().createLogFile(pathSrc,CommonVariable.missingFileList);
				 					new UpdateOrderInfo().updateRemark_WithAppend("File Missing,");
				 					new ToolTerminate().terminate();
			 					}
			 				}
			 				
			 				
			 				
			 				new	CreateEmptySignal().createSignal();
			 				
			 				
			 				String pathSrc11=CLELE_Extraction.zip_path+"\\"+LogicClass.orderId+"_"+"Report.log";
			 				String value="Issn not found "+CommonVariable.totalIssnNotPresent+" : Source id not found : "+CommonVariable.totalsrcIdNotPresent+": Total article : "+CommonVariable.totalIssue;
			 				new LogFile().createLogFileReport(pathSrc11, value);
			 				
			 				new ContentList_Main();
			 			
			 				if(!(CommonVariable.missingFileList.size()==0))
			 				{
			 					if(CLELE_Extraction.flow.equals("SEG") || CLELE_Extraction.flow.equals("OUP") || CLELE_Extraction.flow.equals("BMC") || CLELE_Extraction.flow.equals("SPR") || CLELE_Extraction.flow.equals("BPG") || CLELE_Extraction.flow.equals("APA"))
			 					{
			 						/*String pathSrc=CLELE_Extraction.zip_path+"\\"+LogicClass.orderId+"_"+"MissingFile.log";
				 					new LogFile().createLogFile(pathSrc,CommonVariable.missingFileList);
				 					new UpdateOrderInfo().updateRemark_WithAppend("File Missing,");*/
			 					}else{
			 						String pathSrc=CLELE_Extraction.zip_path+"\\"+LogicClass.orderId+"_"+"MissingFile.log";
				 					new LogFile().createLogFile(pathSrc,CommonVariable.missingFileList);
				 					new UpdateOrderInfo().updateRemark_WithAppend("File Missing,");
				 					new ToolTerminate().terminate();
			 					}
			 				}
			 				
			 				new Vtool().runVtoolMain();
			 				
			 				if(updateFlag)
			 				{
			 					UpdateStatusInRespectiveFlow UpdateStatus=new UpdateStatusInRespectiveFlow();
			 					if((BPG_POJO.issueHai)&&(BPG_POJO.articleHai))
			 					{
			 						BPG_POJO.mix="YES";
			 					}
			 					if(ELS_POJO.issueHai && ELS_POJO.articleHai)
			 					{
			 						ELS_POJO.mix="YES";
			 					}
			 					if((SEG_POJO.issIssue)&&(SEG_POJO.issArticle))
			 					{
			 						SEG_POJO.mix="YES";
			 					}
			 					if((BMC_POJO.issueHai)&&(BMC_POJO.articleHai))
			 					{
			 						BMC_POJO.mix="YES";
			 					}
			 					if(CLELE_Extraction.flow.equals("BPG"))
			 					{
			 						UpdateStatus.UpdateCLCraetedStatus("ABPGARTICLEINFO" , BPG_POJO.mix);
			 						UpdateStatus.UpdateCLCraetedStatus2("BPGVOLISSINFO");
			 					}else if(CLELE_Extraction.flow.equals("ELS"))
			 					{
			 						UpdateStatus.UpdateCLCraetedStatus("ABPELARTICLEINFO" , ELS_POJO.mix);
			 						UpdateStatus.UpdateCLCraetedStatus2("ELSVOLISSINFO");
			 					}
			 					else if(CLELE_Extraction.flow.equals("SEG"))
			 					{
			 						UpdateStatus.UpdateCLCraetedStatus("SEGARTICLEINFO" , SEG_POJO.mix);
			 						UpdateStatus.UpdateCLCraetedStatus2("SEGVOLISSINFO");
			 					}
			 					else if(CLELE_Extraction.flow.equals("BMC"))
			 					{
			 						UpdateStatus.UpdateCLCraetedStatus("BMCARTICLEINFO" , BMC_POJO.mix);
			 						UpdateStatus.UpdateCLCraetedStatus2("BMCVOLISSINFO");
			 					}
			 					else{
			 						UpdateStatus.UpdateCLCraetedStatus();
			 					}
			 					if(findingType.equals("RESUPPLY"))
			 					{
			 						if(!CLELE_Extraction.flow.equalsIgnoreCase("INTRES"))
			 						{
			 							UpdateStatus.updateResupplyStatus();
			 						}
			 						if(AIPFlag)
			 						{
			 							//UpdateStatus.updateResupplyDoiInAIP();
			 							UpdateStatus.updateDoi();
			 						}
			 					}
			 				UpdateOrderInfo.updateOrderInfo();
			 					
			 					//add on 11-02
			 					
			 					if(BPG.chkAip)
			 					{
			 						String markerFile=CLELE_Extraction.prop.getProperty("AIP_MIX_MARKER");
			 						File markerFileDi=new File(markerFile);
			 						if(!markerFileDi.exists())
			 						{
			 							new File(markerFile).mkdirs();
			 						}
			 						
			 						String markerFileName=CLELE_Extraction.paths.replace(":","@").replace("\\", "~").replace(".zip", "");
			 						File marker=new File(markerFile+"\\"+markerFileName+".txt");
			 						if(!marker.exists())
			 						{
			 							try {
			 								marker.createNewFile();
			 							} catch (IOException e) {
			 							new ExceptionHandling().IO_Exception(e,"");
			 							//e.printStackTrace();
			 							CLELE_Extraction.infoLogger.error(e.getMessage());
			 							}
			 						}
			 					}
			 				}
			 			}
					}
				}
			}
		}else
		{
			CLELE_Extraction.infoLogger.info(orderZipName+" : ALREADY PROCESSED OR RECORD NOT FOUND IN ORDERINFO ----> CHECK THE ORDERINFO TABLE...");
			new ToolTerminate().terminate();
		}
		} catch (NullPointerException exception) {
			new ExceptionHandling().NULLPointer_Exception(exception," ");
		} catch (Exception exception) {
			new ExceptionHandling().Normal_Exception(exception," ");
		} catch (Throwable exception) {
			new ExceptionHandling().Throwable_Exception(exception," ");
		}
		finally{
			/*if(DataBase.connection != null)
			{
				try {
					DataBase.connection.close();
				} catch (SQLException e) {
					e.printStackTrace();
				}
			}*/
			DbUtil.close(DataBase.connection);
		}
     		
	
     		
		}
	
	
	
	
}
-------------------------------------------car_Run---------------------------------
package td.bdops.executionPoint;

import java.io.File;
import java.sql.SQLException;

import td.bdops.DAO.UpdateOrderInfo;
import td.bdops.UtilsClass.ExtractFile;
import td.bdops.UtilsClass.RunJar;
import td.bdops.UtilsClass.SendMail;
import td.bdops.UtilsClass.ToolTerminate;
import td.bdops.car.Car;
import td.bdops.carBulding.CarBulding;
import td.bdops.emBase.EMBASE;
import td.bdops.pss.PSS;
import td.bdops.cochrane.Cochrane;
import td.bdops.resupply.Resupply;
import td.bdops.resupply.Resupply_DAO;

public class Car_Run {

	public void execute(String orderxmlPath) throws SQLException
	{
			
			//System.out.println("<------------Executing for scopus resupply--------------------->");//Checking for scopus resupply
			/*if(LogicClass.orderType.contains("RESUPPLY"))
			{
				SendMail.ForResupply("CAR Resupply order received", LogicClass.orderId, CLELE_Extraction.paths,"","We Recevied the resupply order");
				new Resupply(orderxmlPath);
			}else{
				new Car();
			}*/
		if(LogicClass.orderType.contains("CAR-RESUPPLY"))
		{
			
			Resupply_DAO dao=new Resupply_DAO();
			boolean	check_duplicate=dao.check_duplicate_inResTab();
			
			if(!check_duplicate)
			{
				int rs=dao.insertInResTab();
				if(rs>0)
				{
					CLELE_Extraction.infoLogger.info("Record are Successfully inserted in the Resupply table");
				}
				else{
					CLELE_Extraction.infoLogger.info("Query not execute");
				}
					
			}else{
				CLELE_Extraction.infoLogger.info(LogicClass.orderId + ": IS ALREADY UPDATED IN RESUPPLY TABLE");
			}
			//remove by sandeep on 16-11
	//		dao.updateTrackingTable();
			CLELE_Extraction.checkCarResup=false;
			CLELE_Extraction.checkCarSolPro=false;
			
			
		}
		if(LogicClass.scopusCar)
		{
			CLELE_Extraction.flow="SCOPUS";
			getIsbn(CLELE_Extraction.zip_path+"\\"+CLELE_Extraction.orderZipName.substring(0,CLELE_Extraction.orderZipName.lastIndexOf("."))+"\\ItemFile");
			if(LogicClass.scopusIsbn == null || LogicClass.scopusIsbn.length()==0)
			{
				new UpdateOrderInfo().updateRemark_WithAppend("Scopus CAR recieved but ISBN is null or empty");
				new ToolTerminate().terminate();
			}
		}
				
		
		/* String cmd="";
		 String moveFileJar=CLELE_Extraction.prop.getProperty("MoveFileJar");
		 CLELE_Extraction.infoLogger.info(" ******* Processing for  Move to Prouction path  **********");
		 if(LogicClass.orderType.contains("RESUPPLY"))
		 {
			 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+" "+"RESUPPLY"; 
			 CLELE_Extraction.infoLogger.info("Resupply cmd :  "+cmd);
			 new RunJar().runJarMove(cmd);
			 CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
			//UpdateOrderInfo.updateOrderInfo();
		 }else{
			 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+"  78";
			 CLELE_Extraction.infoLogger.info("Normal cmd :  "+cmd);
			 new RunJar().runJarMove(cmd);
			 CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
			 //UpdateOrderInfo.updateOrderInfo();
		 }*/
		
		
		   new Car();
		
		
			if(LogicClass.check_CarBulding)
			{
				new CarBulding().addInCarBulding();
			}
			if(LogicClass.check_Embase)
			{
				System.out.println(LogicClass.check_Embase);//----->On 14-10-2015
				new EMBASE().addEmbaseTable();
			}
			if(LogicClass.embaseConfAbsFlag)
			{
				System.out.println(LogicClass.check_Embase);//----->On 14-10-2015
				new EMBASE().insertEmbaseConfAbs();
			}
			if(LogicClass.check_Pss)
			{
				PSS pss=new PSS();
				pss.addPSSTable();
				pss.copyFile();
			}
			
			/************************* 12.02.2016 cochrane****************************/
			
			
			if(LogicClass.cochraneFlag)
			{
				System.out.println("Found the order as Cochrane...");
				new Cochrane().insertIntoCochrane();
				
			}
			
			/************************* 12.02.2016 cochrane****************************/
			
			 if(LogicClass.orderType.contains("RESUPPLY"))
			 {
				UpdateOrderInfo.updateOrderInfo();//updates as processing
			 }else
			 {
			 }
			 UpdateOrderInfo.updateOrderInfo();//updates as processing
			
			
			
			
			 String cmd="";
			 String moveFileJar=CLELE_Extraction.prop.getProperty("MoveFileJar");
			 CLELE_Extraction.infoLogger.info(" ******* Processing for  Move to Prouction path  **********");
			 if(LogicClass.orderType.contains("RESUPPLY"))
			 {
				 System.out.println("Running for resupply and updating resupply table...");
				 //UpdateOrderInfo.updateOrderInfo();;//updates as processing
				 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+" "+"RESUPPLY";
				 CLELE_Extraction.infoLogger.info("Resupply cmd :  "+cmd);
				 new RunJar().runJarMove(cmd);
				 new Resupply_DAO().updateResupply();
				 //CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
				//UpdateOrderInfo.updateOrderInfo1();//updates as CLELE_Extraction.flow+created
			 }else{
				//UpdateOrderInfo.updateOrderInfo();//updates as processing
				 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+"  78";
				 CLELE_Extraction.infoLogger.info("Normal cmd :  "+cmd);
				new RunJar().runJarMove(cmd);
				 //CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
				 //UpdateOrderInfo.updateOrderInfo1();//updates as CLELE_Extraction.flow+created
			 }
	}

	private void getIsbn(String isbnPath) {
		File[] files = new File(isbnPath).listFiles();
		//System.out.println("Problem found in "+isbnPath);
		for(File f : files)
		{
			if(f.getName().endsWith(".txt"))
			{
				//System.out.println(f.getName().indexOf("-"));
				//System.exit(0);
				LogicClass.scopusIsbn=f.getName().substring(0, f.getName().indexOf("-"));
						System.out.println(LogicClass.scopusIsbn);
			}
			else if(f.getName().toLowerCase().endsWith(".zip"))
			{
				String paths=f.getAbsolutePath();
				boolean	errorUnzip=new ExtractFile().unZip(paths);
				
				String	paths1=paths.substring(0,paths.lastIndexOf("."));
				
				if(errorUnzip)
				{
					new UpdateOrderInfo().updateRemark("Problem in extracting zip file. Please check manually");
					new ToolTerminate().terminate();
				}
				
				
				getIsbn(paths1);
			}
			else if(f.isDirectory())
			{
				getIsbn(f.getAbsolutePath());
			}
		}

	}
}
------------------------------Readxml
package td.bdops.executionPoint;

import java.io.File;
import java.sql.SQLException;

import td.bdops.DAO.UpdateOrderInfo;
import td.bdops.UtilsClass.ExtractFile;
import td.bdops.UtilsClass.RunJar;
import td.bdops.UtilsClass.SendMail;
import td.bdops.UtilsClass.ToolTerminate;
import td.bdops.car.Car;
import td.bdops.carBulding.CarBulding;
import td.bdops.emBase.EMBASE;
import td.bdops.pss.PSS;
import td.bdops.cochrane.Cochrane;
import td.bdops.resupply.Resupply;
import td.bdops.resupply.Resupply_DAO;

public class Car_Run {

	public void execute(String orderxmlPath) throws SQLException
	{
			
			//System.out.println("<------------Executing for scopus resupply--------------------->");//Checking for scopus resupply
			/*if(LogicClass.orderType.contains("RESUPPLY"))
			{
				SendMail.ForResupply("CAR Resupply order received", LogicClass.orderId, CLELE_Extraction.paths,"","We Recevied the resupply order");
				new Resupply(orderxmlPath);
			}else{
				new Car();
			}*/
		if(LogicClass.orderType.contains("CAR-RESUPPLY"))
		{
			
			Resupply_DAO dao=new Resupply_DAO();
			boolean	check_duplicate=dao.check_duplicate_inResTab();
			
			if(!check_duplicate)
			{
				int rs=dao.insertInResTab();
				if(rs>0)
				{
					CLELE_Extraction.infoLogger.info("Record are Successfully inserted in the Resupply table");
				}
				else{
					CLELE_Extraction.infoLogger.info("Query not execute");
				}
					
			}else{
				CLELE_Extraction.infoLogger.info(LogicClass.orderId + ": IS ALREADY UPDATED IN RESUPPLY TABLE");
			}
			//remove by sandeep on 16-11
	//		dao.updateTrackingTable();
			CLELE_Extraction.checkCarResup=false;
			CLELE_Extraction.checkCarSolPro=false;
			
			
		}
		if(LogicClass.scopusCar)
		{
			CLELE_Extraction.flow="SCOPUS";
			getIsbn(CLELE_Extraction.zip_path+"\\"+CLELE_Extraction.orderZipName.substring(0,CLELE_Extraction.orderZipName.lastIndexOf("."))+"\\ItemFile");
			if(LogicClass.scopusIsbn == null || LogicClass.scopusIsbn.length()==0)
			{
				new UpdateOrderInfo().updateRemark_WithAppend("Scopus CAR recieved but ISBN is null or empty");
				new ToolTerminate().terminate();
			}
		}
				
		
		/* String cmd="";
		 String moveFileJar=CLELE_Extraction.prop.getProperty("MoveFileJar");
		 CLELE_Extraction.infoLogger.info(" ******* Processing for  Move to Prouction path  **********");
		 if(LogicClass.orderType.contains("RESUPPLY"))
		 {
			 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+" "+"RESUPPLY"; 
			 CLELE_Extraction.infoLogger.info("Resupply cmd :  "+cmd);
			 new RunJar().runJarMove(cmd);
			 CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
			//UpdateOrderInfo.updateOrderInfo();
		 }else{
			 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+"  78";
			 CLELE_Extraction.infoLogger.info("Normal cmd :  "+cmd);
			 new RunJar().runJarMove(cmd);
			 CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
			 //UpdateOrderInfo.updateOrderInfo();
		 }*/
		
		
		   new Car();
		
		
			if(LogicClass.check_CarBulding)
			{
				new CarBulding().addInCarBulding();
			}
			if(LogicClass.check_Embase)
			{
				System.out.println(LogicClass.check_Embase);//----->On 14-10-2015
				new EMBASE().addEmbaseTable();
			}
			if(LogicClass.embaseConfAbsFlag)
			{
				System.out.println(LogicClass.check_Embase);//----->On 14-10-2015
				new EMBASE().insertEmbaseConfAbs();
			}
			if(LogicClass.check_Pss)
			{
				PSS pss=new PSS();
				pss.addPSSTable();
				pss.copyFile();
			}
			
			/************************* 12.02.2016 cochrane****************************/
			
			
			if(LogicClass.cochraneFlag)
			{
				System.out.println("Found the order as Cochrane...");
				new Cochrane().insertIntoCochrane();
				
			}
			
			/************************* 12.02.2016 cochrane****************************/
			
			 if(LogicClass.orderType.contains("RESUPPLY"))
			 {
				UpdateOrderInfo.updateOrderInfo();//updates as processing
			 }else
			 {
			 }
			 UpdateOrderInfo.updateOrderInfo();//updates as processing
			
			
			
			
			 String cmd="";
			 String moveFileJar=CLELE_Extraction.prop.getProperty("MoveFileJar");
			 CLELE_Extraction.infoLogger.info(" ******* Processing for  Move to Prouction path  **********");
			 if(LogicClass.orderType.contains("RESUPPLY"))
			 {
				 System.out.println("Running for resupply and updating resupply table...");
				 //UpdateOrderInfo.updateOrderInfo();;//updates as processing
				 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+" "+"RESUPPLY";
				 CLELE_Extraction.infoLogger.info("Resupply cmd :  "+cmd);
				 new RunJar().runJarMove(cmd);
				 new Resupply_DAO().updateResupply();
				 //CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
				//UpdateOrderInfo.updateOrderInfo1();//updates as CLELE_Extraction.flow+created
			 }else{
				//UpdateOrderInfo.updateOrderInfo();//updates as processing
				 cmd="java -jar "+moveFileJar+" "+CLELE_Extraction.paths+" "+CLELE_Extraction.propertiesFilePath+"  78";
				 CLELE_Extraction.infoLogger.info("Normal cmd :  "+cmd);
				new RunJar().runJarMove(cmd);
				 //CLELE_Extraction.infoLogger.info("Move To Production FINISHED");
				 //UpdateOrderInfo.updateOrderInfo1();//updates as CLELE_Extraction.flow+created
			 }
	}

	private void getIsbn(String isbnPath) {
		File[] files = new File(isbnPath).listFiles();
		//System.out.println("Problem found in "+isbnPath);
		for(File f : files)
		{
			if(f.getName().endsWith(".txt"))
			{
				//System.out.println(f.getName().indexOf("-"));
				//System.exit(0);
				LogicClass.scopusIsbn=f.getName().substring(0, f.getName().indexOf("-"));
						System.out.println(LogicClass.scopusIsbn);
			}
			else if(f.getName().toLowerCase().endsWith(".zip"))
			{
				String paths=f.getAbsolutePath();
				boolean	errorUnzip=new ExtractFile().unZip(paths);
				
				String	paths1=paths.substring(0,paths.lastIndexOf("."));
				
				if(errorUnzip)
				{
					new UpdateOrderInfo().updateRemark("Problem in extracting zip file. Please check manually");
					new ToolTerminate().terminate();
				}
				
				
				getIsbn(paths1);
			}
			else if(f.isDirectory())
			{
				getIsbn(f.getAbsolutePath());
			}
		}

	}
}
----------------------------------------Logic--------------------------
package td.bdops.executionPoint;

import java.util.ArrayList;






public class LogicClass
{
	public static String orderId;
	public static String SupplierDatasetId;
	public static String unitId;
	public static String SourceType;
	public static String carsuppUnitId="";
	public static String orderType;
	public static String parcelId;
	public static String orderInstruction="";
	public static String executorId;
	public static String executorType;
	public static String projectName="";
	public static String issn;
	public static String xmlFilePath;
	public static String supplierUnitId="";
	public static String tableName;
	public static String status;
	public static String carSourceId;
	public static String sourceType;
	public static String batch;
	public static String supplierCode;
	public static String carSourceTitle;
	public static String timeStampDate="";
	public static String dueDate;
	public static String spso="";
	public static String weekValue="";
	public static String originalId;
	public static String carVol;
	public static String carIssue;
	public static String carIssn;
	public static StringBuffer remark=new StringBuffer();
	public static ArrayList<Object> listPathname=new ArrayList<Object>();
	public static String tempsource = "";
	public static String originalIdResupply;
	public static String carDueDate;
	public static boolean check_CarBulding=false;
	public static boolean check_Embase=false;
	public static boolean check_Car=false;
	public static boolean check_Pss=false;
	public static boolean scopusCar=false;
	
	//ctc INDEX
	
	public static String carId="";
	public static String isbn="";
	public static String resupplySupplierUnitId="";
	public static String scopusIsbn="";
	public static boolean embaseConfAbsFlag=false;//26.12.2015
	public static String embaseConfAbsInst="";//26.12.2015
	
	public static boolean cochraneFlag=false;//12.02.2016
	public static String cochraneInst="";//12.02.2016


}
--------------Live path-----------------------------
package td.bdops.executionPoint;

import java.io.FileInputStream;
import java.util.Properties;

import org.apache.log4j.PropertyConfigurator;

import td.bdops.UtilsClass.ToolTerminate;
import td.bdops.mail.MailNotify;

public class LivePath {

	public void runLivePath(String[] args)
	{
		CLELE_Extraction.paths=args[0];
		CLELE_Extraction.propertiesFilePath=args[1];
		CLELE_Extraction.prop = new Properties();
		try {
			CLELE_Extraction.prop.load(new FileInputStream(CLELE_Extraction.propertiesFilePath));
		}catch (Exception e) 
		{
			CLELE_Extraction.infoLogger.error("Problem in loading property file ..Please check Property file is available or not  "+e);
			MailNotify.exceptionSendMail("Problem in loading "+CLELE_Extraction.propertiesFilePath  +" file ..Please check Property file is available or not " , "BDOPS: Signal creation - Property file not found");
			new ToolTerminate().terminate();
		}
		PropertyConfigurator.configure(CLELE_Extraction.prop.getProperty("LOG_FILE"));
	}
	
	
	public void runLocalPath()
	{
		CLELE_Extraction.prop = new Properties();
		
		 String properties="D:\\OPSBANK-II\\PROPERTIES\\CreateCLELE.properties";
		 CLELE_Extraction.propertiesFilePath=properties;
		//String properties="D:\\OPSBANK-II\\PROPERTIES\\CreateCLELE.properties";   //24-10

		try {
			CLELE_Extraction.prop.load(new FileInputStream(properties));
		} catch (Exception e)
		{
			CLELE_Extraction.infoLogger.error("Problem in loading property file ..Please check Property file is available or not  "+e);
			MailNotify.MailNote("Problem in loading property file ..Please check Property file is available or not " , "Problem in loading property file");
			new ToolTerminate().terminate();
		}
		PropertyConfigurator.configure(CLELE_Extraction.prop.getProperty("LOG_FILE"));
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\ABPEL\\19-03-2015\\1955026_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\ELS\\2015-06-03\\3383033_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\INTRES\\2015-06-04\\3445686_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\COR\\2015-06-04\\3458969_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CAR\\20-05-2015\\3183955_1.zip";
		//CLELE_Extraction.paths= "D:\\OPSBANK-II\\ORDERS\\CL\\ABPEL\\2015-06-09\\3445239_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\ABPEL\\2015-06-09\\3536745_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\ABPEL\\IssuesAbsenceKuldeep\\ORDER\\3509710_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CAR\\kuldeepabsence\\3586711_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\kuldeeepabsence\\BPG\\3567427_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\kuldeeepabsence\\ELS\\heapSpaceResupply\\3283849_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\kuldeeepabsence\\ELS\\heapspaceCL\\3525410_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\kuldeeepabsence\\ABPEL\\sqlproblem\\3570927_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\kuldeeepabsence\\ABPEL\\sqlproblem\\others2\\3477368_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\OUP\\2015-05-25\\3295265_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\ELS\\18-05-2015\\3026221_1.zip";
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\Scopus-Resupply\\5184618_1.zip";//Scopus Resupply
	//	CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\ABPEL\\5331777_1.zip";//For abpel
	//	CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\ABPG\\5371943_1.zip";//abpg
		//CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\WBPG\\5417025_1.zip";//wpbg
		//CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\ELS\\5462317_1.zip";//els
		
	//	CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\OUP\\5372136_1.zip";//oup
		//CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\SPR\\5487132_1.zip";//spr
		//CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\COR\\5403847_1.zip";//Cor
		//CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\ASSESSMENT\\5324821_1.zip";//ass
		//CLELE_Extraction.paths="D:\\ExtractionTool_Orders\\RESUPPLY\\5336351_1.zip";//resupply
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\Embase\\5269484_1.zip";//Embase Supply13-10
		//CLELE_Extraction.paths="D:\\OPSBANK-II\\ORDERS\\CL\\BMC\\1954981_1.zip";
	}


	/*public void setVersion()
	{
		CLELE_Extraction.infoLogger.info("********************************************************");
		CLELE_Extraction.infoLogger.info("***********		Version 07-07-2015(1)             						 ****");
		CLELE_Extraction.infoLogger.info("***********		UPDATED FOR : OUP APLIT/CPX   				   *****");
		CLELE_Extraction.infoLogger.info("***********  								     									 ********");
		CLELE_Extraction.infoLogger.info("***********         EXTRACTION  TOOL            					********");
		CLELE_Extraction.infoLogger.info("***********         [07-JULY-15] [11 : 30 AM]   ********");
		CLELE_Extraction.infoLogger.info("***********                                     ********");
		CLELE_Extraction.infoLogger.info("***********                                     									 ********");
		CLELE_Extraction.infoLogger.info("********************************************************");
	}*/
	public void setVersion()
	{
		CLELE_Extraction.infoLogger.info("***********************************************************");
		CLELE_Extraction.infoLogger.info("**********		CAR_EXTRACTION TOOL	(1)			*********");
		CLELE_Extraction.infoLogger.info("********** 		UPDATED   ON  03.02.2016		*********");
		CLELE_Extraction.infoLogger.info("********** 	FOR EMBASE CONF ABS AND SCOPUS		*********");
		CLELE_Extraction.infoLogger.info("**********	 	Version 03-02-2016 WEDNESDAY	*********");
		CLELE_Extraction.infoLogger.info("********** 										*********");
		CLELE_Extraction.infoLogger.info("***********************************************************");
	}

}
--------------------contentLIST_RUN----------------------------------
package td.bdops.executionPoint;

import td.bdops.DAO.UpdateOrderInfo;
import td.bdops.UtilsClass.ContentProvider;
import td.bdops.UtilsClass.ToolTerminate;
import td.bdops.UtilsClass.UtilMethod;
import td.bdops.ordertracker.ZipFileTracker_old;

public class ContentList_Run {

	public void execute(UtilMethod util,String orderItemFile)
	{
		CLELE_Extraction.flow=new ContentProvider().getContentProviderName(orderItemFile);
		if(CLELE_Extraction.flow==null)
		{
			CLELE_Extraction.infoLogger.info("Flow VALUE IS NULL : CONTENT PROVIDER NOT HANDLE");
			new UpdateOrderInfo().updateRemark("NEW CONTENT PROVIDER");
			new ToolTerminate().terminate();
		}else{
			CLELE_Extraction.infoLogger.info("Content Provider   : "+CLELE_Extraction.flow);
		}
			util.getAllZipFile(orderItemFile);
			CLELE_Extraction.infoLogger.info("Zip File Name for Extracting ...   : "+CLELE_Extraction.zipFileName);
			
			util.getIncomingPath(orderItemFile);
			CLELE_Extraction.infoLogger.info("Incoming path.....   : "+CLELE_Extraction.incomingPath);
			
			new GetFLow();
	}
	
}



