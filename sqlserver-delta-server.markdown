---
layout: post
title: SQL Server Delta Server
categories: SBP
parent: data-access-patterns.html
weight: 1300
---


{% tip %}
**Summary:** {% excerpt %}This pattern presents the SQL Server Delta Server allowing the data grid to receive updates from the database conducted by applications that are not using the data grid as their system of record (Non-Aware Data-Grid Clients){% endexcerpt %}<br/>
**Author**:  Ronnie Guha<br/>
**Recently tested with GigaSpaces version**: XAP.NET 9.7.0 x64<br/>
**Last Update:** April 2014<br/>

{% toc minLevel=1|maxLevel=1|type=flat|separator=pipe %}

{% endtip %}



# Overview

Almost every large enterprise system includes legacy applications or backend systems that are communicating with the enterprise main database system for reporting, batch processing, data mining, OLAP and other processing activity. These applications might not need to access the data grid to improve their performance or scalability. They will be using the database directly. Once these systems perform data updates , removing data or add new records to the database, these updates might need to be reflected within the data grid. This synchronization activity ensures the data grid is consistent and up to date with the enterprise main database server.

<!--iframe width="640" height="390" src="//www.youtube.com/embed/EOyDg-mI3z0" frameborder="0" allowfullscreen></iframe-->

The Delta Server described with this pattern responsible for getting notifications from the database for any user activity (excluding data grid updates) and delegate these to the data grid. You may specify the exact data set updates to be delegated to the data grid by specifying a SQL Query that will indicate which records updates / removal / addition should be reflected within the data grid.

# Scenario

We have an In Memory Data Grid (IMDG) that is used for querying information. The initial load of the IMDG was performed from an SQL Server Database. The IMDG is not used as a system of record in this case - in other words, any changes to the objects in the IMDG are not propagated back into the Database and instead Data-Grid Non-Aware Clients are updating the Database. These updates (insert,update and delete) need to be reflected in the IMDG.

{%align center%}
[<img src="/pics/sqlserver-delta-server.png" width="400" height="300">](/pics/sqlserver-delta-server.png)
{%endalign%}

We will show you how you can implement this scenario with XAP. A fully functional example is available on [GitHub](https://github.com/Gigaspaces/xxxxx).


# Change Data Capture (CDC)

Change Data Capture is a new feature in SQL Server 2008 that records insert, update and delete activity in SQL Server tables.  This feature allows one to record and update an external data warehouse or IMDG with any data that has changed in the source systems since the last time the ETL process was run.  Without CDC determining any rows that were physically deleted or determining what was changed and when is quite cumbersome and difficult.  CDC provides a configurable solution that addresses these requirements and more.

Change data capture records insert, update, and delete activity that is applied to a SQL Server table. This makes the details of the changes available in an easily consumed relational format. Column information and the metadata that is required to apply the changes to a target environment is captured for the modified rows and stored in change tables that mirror the column structure of the tracked source tables. Table-valued functions are provided to allow systematic access to the change data by consumers.

For further information please consult the Microsoft [documentation] (http://technet.microsoft.com/en-us/library/bb522489(v=sql.105).aspx).
In our example we will only demonstrate the notifications for INSERT, UPDATE and Delete.

## Change Data Capture (CDC) Setup And Configuration
Note:-Make Sure Sql-Server Agent is enabled. Also, please ensure that you have run
a.	Gs-agent
b.	Created a space (e.g) - gs-cli deploy-space -cluster total_members=1,1 mydatagrid
          Keep this grid name handy as you will be using it later on.
1)	Create a database first (right click and create a new DB)

{%align center%}
[<img src="/pics/pic1.png" width="400" height="300">](/pics/pic1.png)
{%endalign%}

You will see the following:

{%align center%}
[<img src="/pics/pic2.png" width="400" height="300">](/pics/pic2.png)
{%endalign%}

2)	Once a database is created, create a table (for our example, let’s use Person, with columns, ID, Firstname, Lastname, Age):
USE [datagrid]
GO

/****** Object:  Table [dbo].[Person]    Script Date: 4/7/2014 4:03:28 PM ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE [dbo].[Person](
	[Id] [int] NOT NULL,
	[FirstName] [nvarchar](255) NULL,
	[LastName] [nvarchar](255) NULL,
	[Age] [int] NULL,
PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO

3)	CDC must be enabled at the database level (it is disabled by default).  To enable CDC you must be a member of the sysadmin fixed server role.  You can enable CDC ONLY on any user database (not on system databases).  Execute the following T-SQL script in the database of your choice (e.g. datagrid in the following screenshots):
declare @rc int
exec @rc = sys.sp_cdc_enable_db
select @rc
-- you will find that a new column is added to sys.databases: is_cdc_enabled
select name, is_cdc_enabled from sys.databases
  
  
{%align center%}
[<img src="/pics/pic3.png" width="400" height="300">](/pics/pic3.png)
{%endalign%}
  
(notice that datagrid has is_cdc_enabled column = 1)



4)	Now, you have to enable CDC for the table in question (in our case, Person table). Execute the following system stored procedure to enable CDC for the Person table:
exec sys.sp_cdc_enable_table 
    @source_schema = 'dbo', 
    @source_name = 'Person' ,
    @role_name = 'CDCRole',
    @supports_net_changes = 1
5)	Verify that CDC is enabled for that table:
select name, type, type_desc, is_tracked_by_cdc from sys.tables

{%align center%}
[<img src="/pics/pic4.png" width="400" height="300">](/pics/pic4.png)
{%endalign%}

6)	From what you have done so far, enabling CDC at the database and table levels will create certain tables, jobs, stored procedures and functions in the CDC-enabled database. In our case, this is the datagrid table.  
You will see a message that two SQL Agent jobs were created; e.g. cdc.datagrid_capture which scans the database transaction log to handle changes to the tables that have CDC enabled, and cdc.datagrid_cleanup which purges the change tables periodically.  

You can examine the schema objects created by running the following T-SQL script:
select o.name, o.type, o.type_desc from sys.objects o
join sys.schemas  s on s.schema_id = o.schema_id
where s.name = 'cdc'

7)	Now, create another table so we can track the last LSN we (the DeltaServer) are processing:
create table dbo.Person_lsn (last_lsn binary(10))


8)	Now, create a function to get the last person LSN thus:
   create function dbo.get_last_Person_lsn() 
         returns binary(10)
             as
           begin
         declare @last_lsn binary(10)
   select @last_lsn = last_lsn from dbo.Person_lsn
   select @last_lsn = isnull(@last_lsn, sys.fn_cdc_get_min_lsn('dbo_Person'))
      return @last_lsn
      end

What you have done is that you have created a Scalar-valued Function. Double check this function by right clicking on Programmability->Functions->Scalar Valued Functions
 
{%align center%}
[<img src="/pics/pic5.png" width="400" height="300">](/pics/pic5.png)
{%endalign%}


9)	Now, let’s create a stored procedure that will capture as soon as next person changes are executed:

-- ================================================
-- Template generated from Template Explorer using:
-- Create Procedure (New Menu).SQL
--
-- Use the Specify Values for Template Parameters 
-- command (Ctrl-Shift-M) to fill in the parameter 
-- values below.
--
-- This block of comments will not be included in
-- the definition of the procedure.
-- ================================================
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Author,,Name>
-- Create date: <Create Date,,>
-- Description:	<Description,,>
-- =============================================
CREATE PROCEDURE [dbo].[Capture]
AS
BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	-- interfering with SELECT statements.
	SET NOCOUNT ON;
	declare @begin_lsn binary(10), @end_lsn binary(10)
    -- get the next LSN for Person changes
	select @begin_lsn = dbo.get_last_Person_lsn()
	-- get the last LSN for Person changes
	select @end_lsn = sys.fn_cdc_get_max_lsn()


	-- get all individual changes in the range
	select * from cdc.fn_cdc_get_all_changes_dbo_Person(
	 @begin_lsn, @end_lsn, 'all'); 
	-- save the end_lsn in the Person_lsn table
	update dbo.Person_lsn
	set last_lsn = @end_lsn
	if @@ROWCOUNT = 0
	insert into dbo.Person_lsn values(@end_lsn)

END
GO

10)	You can test what you have done by some simple statements like:

insert Person values (1, 'abc', 'md', 99) 
update Person set age = 199 where id = 1 
delete from Person where id = 1 
insert Person values (2, 'xyz', 'de', 59) 
update Person set age = 299 where id = 2
delete from Person where id = 2 
insert Person values (3, 'xox', 'va', 29) 
update Person set age = 399 where id = 3 
delete from Person where id = 3

And then, you can run

declare @begin_lsn binary(10), @end_lsn binary(10)
-- get the first LSN for Person changes
select @begin_lsn = sys.fn_cdc_get_min_lsn('dbo_Person')
-- get the last LSN for Person changes
select @end_lsn = sys.fn_cdc_get_max_lsn()
-- get net changes; group changes in the range by the pk
select * from cdc.fn_cdc_get_net_changes_dbo_Person( @begin_lsn, @end_lsn, 'all'); 
-- get individual changes in the range
select * from cdc.fn_cdc_get_all_changes_dbo_Person(@begin_lsn, @end_lsn, 'all');



{%align center%}
[<img src="/pics/pic6.png" width="400" height="300">](/pics/pic6.png)
{%endalign%}



11)	Now, head on over to Visual Studio and open the project to make some customizations for your project.
In program.cs, you will find values like string _connStr = "Data Source=YOUR-PC\\SQL2012;Initial Catalog=datagrid;Integrated Security=True";
Please change them to reflect values according to your set up. You can easily copy this string from visual studio itself (if you have configured the database connection). Head over to Server explorer in Visual Studio, click on Data Connections->your database connection. On the right hand side, properties window you will see the connection string that you would need to copy/paste in _connStr.

{%align center%}
[<img src="/pics/pic7.png" width="400" height="300">](/pics/pic7.png)
{%endalign%}


12)	Change your group or space connect string if you desire to. Currently, it is set to default
ISpaceProxy spaceProxy = GigaSpacesFactory.FindSpace("jini://*/*/mydatagrid?groups=XAP-9.7.0-ga-NET-4.0.30319-x64");
Please note that this is the same grid name that you defined before (in first step)




## Registering a listener, capturing data therefore
Lets assume we have an Employee represented in a Person table in the database that we want to be notified when a record is inserted, updated or deleted so we can update the IMDG with the changes. Here is an example of an ChangeNotificationListener.

{%highlight java%}
static void ChangeCapture(object sender, ElapsedEventArgs e)
        {
         
           
            ISpaceProxy spaceProxy = GigaSpacesFactory.FindSpace("jini://*/*/mydatagrid?groups=XAP-9.7.0-ga-NET-4.0.30319-x64");

            string _connStr = ConfigurationManager.ConnectionStrings["DBSTRING"].ConnectionString;
            
            SqlConnection con = new SqlConnection(_connStr);
            con.Open();
            SqlCommand cmd = new SqlCommand("Capture", con);
            cmd.CommandType = CommandType.StoredProcedure;
            SqlDataAdapter da = new SqlDataAdapter(cmd);
            // this will query your database and return the result to your datatable
            DataTable dt = new DataTable();
            da.Fill(dt);

            //Insert Query starts
            List<Employee> objEmployee2 = (from item in dt.AsEnumerable()
                                           where item.Field<int>("__$operation") == 2
                                           select new Employee()
                                           {
                                               Id = item.Field<int>("Id"),
                                               FirstName = item.Field<string>("FirstName"),
                                               LastName = item.Field<string>("LastName"),
                                               Age = item.Field<int>("Age")
                                           }).ToList();
            var personarray = new Person[objEmployee2.Count];
            for (int i = 0; i < objEmployee2.Count; i++) // Loop through List with for
            {
                personarray[i] = new Person
                {
                    Id = objEmployee2[i].Id,
                    FirstName = objEmployee2[i].FirstName,
                    LastName = objEmployee2[i].LastName,
                    Age = objEmployee2[i].Age

                };
             
            }
            spaceProxy.WriteMultiple(personarray);
            //Insert Query ends

            //Update Query starts
            List<Employee> objEmployee3 = (from item in dt.AsEnumerable()
                                           where item.Field<int>("__$operation") == 4
                                           select new Employee()
                                           {
                                               Id = item.Field<int>("Id"),
                                               FirstName = item.Field<string>("FirstName"),
                                               LastName = item.Field<string>("LastName"),
                                               Age = item.Field<int>("Age")
                                           }).ToList();


     
            for (int i = 0; i < objEmployee3.Count; i++) // Loop through List with for
            {

                IdQuery<Person> idQuery = new IdQuery<Person>(objEmployee3[i].Id);
                IChangeResult<Person> changeResult =
                    spaceProxy.Change<Person>(idQuery,
                    new ChangeSet().Set("FirstName", objEmployee3[i].FirstName).Set("LastName", objEmployee3[i].LastName).Set("Age", objEmployee3[i].Age));

            }
            //Update Query ends

            //Delete Query starts

            List<Employee> objEmployee1 = (from item in dt.AsEnumerable()
                                           where item.Field<int>("__$operation") == 1
                                           select new Employee()
                                           {
                                               Id = item.Field<int>("Id"),
                                               FirstName = item.Field<string>("FirstName"),
                                               LastName = item.Field<string>("LastName"),
                                               Age = item.Field<int>("Age")
                                           }).ToList();
            var deltearray = new Person[objEmployee1.Count];
            for (int i = 0; i < objEmployee1.Count; i++) // Loop through List with for
            {

                Person template = new Person();
                template.Id = objEmployee1[i].Id;
                spaceProxy.Take<Person>(template);
            }

            //Delete Query ends


            da.Dispose();
            con.Close();

        }
{%endhighlight%}



# Running the example

1. Download the example from [GitHub|https://github.com/Gigaspaces/XXXXXX].
2. Change the Database properties according to your environment:

{%highlight java%}
# App.config
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <connectionStrings>
    <add name="DBSTRING" connectionString="Data Source=YOUR-SERVER\SQL2012;Initial Catalog=datagrid;Integrated Security=True"/>
  </connectionStrings>
</configuration>

{%endhighlight%}

13)	Now, you can build the solution and test it out. For this, you can either run it from Visual Studio (Ctrl+F5) or you can head over to DeltaServer\DeltaServer\bin\Debug and run the DeltaServer.exe. Or if you are creating a release version, you can run it from there. Upon running the program, you should be prompted to insert/update/delete or exit the program

{%align center%}
[<img src="/pics/pic8.png" width="400" height="300">](/pics/pic8.png)
{%endalign%}

14)	As you insert/update/delete, please note such changes in the gs-webgui.

{%align center%}
[<img src="/pics/pic9.png" width="400" height="300">](/pics/pic9.png)
{%endalign%}


15)	If you would like to, you can change the frequency of updates thus (in program.cs)
a.	timer.Interval = 5000; //set interval of polling here

