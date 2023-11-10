# Dataflow schema mapping and sampling a few rows
Power BI dataflow workspace documentation of the dataflows and the entity schemas
## Use case 
Imagine you have a Power BI workspace with dozens of dataflows created by your team. Some of these dataflows might be using sensitive data that should be restricted for the wider audience. You would like to have a high level overview of all the dataflows in that particular workspace. The overview should be able to show all the dataflows in that workspace, the entities in each of the dataflow, the columns in each of the entity , the data type of those columns and finally a few rows of the actual data from all the tables.
## Solution
#### Create a Power BI report that gives you this overview.
I will collect the data in 2 different queries - one for the table schemas and the second one for sampling a few rows from the tables
- Use PowerPlatform as the datasource

#### Schema querying

 ![image](https://github.com/datawings/DataflowSchemaSampling/assets/61468624/0f59c0cd-789f-4383-839a-fe7e09e3efda)
- In the GetData window, select the workspace you would like to analyse and click Create
- Retain only the dataflowName and Data columns
- Expand the Data column with Data and entityName as selection
  ![image](https://github.com/datawings/DataflowSchemaSampling/assets/61468624/f0ee19dd-2484-4dc2-b520-c96411d3213f)
- You will find a new column Data.1 after the previous step, which is the entity. Click anywhere in the whitespace near the cell to get a preview of the entity
- Now we will create a new column to get the schema of the enttity. You will not be able to get this via the GUI and so create a Custom column and enter the formula Table.Schema([Data.1])
  ![image](https://github.com/datawings/DataflowSchemaSampling/assets/61468624/92e7e60a-e06a-4e34-b048-5a5b0b42c1a6)
- Now expand the newly created column and choose Name and Kind as columns for expansion. Name is the column name and Kind is the column data type
- Remove unwanted columns and now you have the schema information of all dataflows in the selected workspace
  
  Here is the Power Query that I used . Replace <workspace-id> with the workspace id of yours
  ``` PQ
  let
   Source = PowerPlatform.Dataflows([]),
   #"Navigation 1" = Source{[Id = "Workspaces"]}[Data],
   #"Navigation 2" = #"Navigation 1"{[workspaceId = <workspace-id>]}[Data],
   #"Removed other columns" = Table.SelectColumns(#"Navigation 2", {"dataflowName", "Data"}),
   #"Expanded Data" = Table.ExpandTableColumn(#"Removed other columns", "Data", {"Data", "entityName"}, {"tabledata", "entityName"}),
   #"Added custom" = Table.AddColumn(#"Expanded Data", "ListCols", each Table.Schema([tabledata])),
   #"Expanded ListCols" = Table.ExpandTableColumn(#"Added custom", "ListCols", {"Name", "Kind"}, {"Name", "Kind"}),
   #"Removed columns" = Table.RemoveColumns(#"Expanded ListCols", {"tabledata"}),
   #"Renamed columns" = Table.RenameColumns(#"Removed columns", {{"Name", "columnName"}}),
   #"Changed column type" = Table.TransformColumnTypes(#"Renamed columns", {{"dataflowName", type text}, {"entityName", type text}, {"columnName", type text}, {"Kind", type text}})
  in
   #"Changed column type"
  ```
  
#### Row sampling 
- In this query, we will try to get N number of rows from each of the tables inside the dataflows in the workspace
- Steps to follow is the same as in the previous query, till we get the column [Data.1]
- Once we have this column, create a custom column, I named it as Top . In the formula use Table.FirstN([Data.1],10) . 10 here indicates the number of rows that I want to show as sample in my report. You can choose any integer, but please consider the data load we might encounter
- Remove every other column except dataflowName, entity and Top. You can preview the top n data by clicking on the whitespace in the cell 
- Now, we can have tables with just 2 columns and 100 columns . So  if we expand the columns, by default, Power Query is not going to expand every columns in all of the tables. Moreover, while expanding will hardcode the column names , the report will not be dynamic and will subjected to failures upon column removals in the base dataflows in the future. But if we could expand those column names as rows, we will be able to overcome those issues. So basically we want to unpivot that nested table in the [Top] column as rows.
- To do this, again create a custom column and use the formula Table.UnpivotOtherColumns([Top],{},"Attribute","Value") , where Top is the name of the custom column we created in the previous step, "Attribute" is the name of the column we are about to create which will contain all the column names and "Value" is the name of the column where all that N rows of the data will be there.
 ![image](https://github.com/datawings/DataflowSchemaSampling/assets/61468624/f3809a7c-9feb-4c5c-a12b-7e06e74fb18b)
- Now simple expand the newly created column and choose Attribute and Value as the columns for expansion.
- We now have the table with all the dataflow names, entity names, column names in each of those entities and also N rows from each of those entities
Here is the PowerQuery code for the above steps. Replace <workspace-id> with the workspace id of your choice

```PQ
let
  Source = PowerPlatform.Dataflows([]),
  #"Navigation 1" = Source{[Id = "Workspaces"]}[Data],
  #"Navigation 2" = #"Navigation 1"{[workspaceId = <workspace-d>]}[Data],
  #"Removed other columns" = Table.SelectColumns(#"Navigation 2", {"dataflowName", "Data"}),
  #"Expanded Data" = Table.ExpandTableColumn(#"Removed other columns", "Data", {"entity", "Data"}, {"entity", "Data.1"}),
  #"Added custom" = Table.AddColumn(#"Expanded Data", "Top", each Table.FirstN([Data.1], 10)),
  #"Removed columns" = Table.RemoveColumns(#"Added custom", {"Data.1"}),
  #"Added custom 1" = Table.AddColumn(#"Removed columns", "Custom", each Table.UnpivotOtherColumns([Top], {}, "Atrribute", "Value")),
  #"Expanded Custom" = Table.ExpandTableColumn(#"Added custom 1", "Custom", {"Atrribute", "Value"}, {"Atrribute", "Value"}),
  #"Removed columns 1" = Table.RemoveColumns(#"Expanded Custom", {"Top"}),
  #"Changed column type" = Table.TransformColumnTypes(#"Removed columns 1", {{"dataflowName", type text}, {"entity", type text}, {"Atrribute", type text}, {"Value", type text}})
in
  #"Changed column type"

```

With that, we have the data required for the overview report. Please be aware that this is a high level report only and won't be able to see every single data inside the tables. Instead of using the Table.FirstN(), we could use Table.LastN() - to get the last N rows or use Table.SelectRows() and use a condition to generate the rows. You could very well do some rando msampling by using the index, List.Random() and Table.SelectRows() to select some random rows from the data.
Learn more about the Table Functions from [Microsoft Learn](https://learn.microsoft.com/en-us/powerquery-m/table-functions)
