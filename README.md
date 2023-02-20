# T-FLIP Implementation Guide

The following production has been developed using Intersystems technology and Postgres for database management, if an alternative DBMS is to be used then the queries will have to be updated. This guide assumes the staging tables have been already created on postgres. If not, go to page 8 on the following guide: https://docs.google.com/document/d/1dzqWKm4k0zYn-9zFc2bqY-Ohqk3n1bqvXnD8PMobWKw/edit# and use the create scripts. It is recommended to create a new namespace to hold this production. Alternatively step 2 can be skipped.

The following steps can be followed to implement T-FLIP:

1. Save Production XML file to the trust server

2. Create new namespace:
Open Intersystems Management portal -> System Administration -> Configuration -> System Configuration -> Namespaces -> Create new namespace -> Enter a suitable name for the namespace 'FLIP' -> Create new database -> choose suitable directory to hold the IRIS database

More details can found here: https://docs.intersystems.com/iris20222/csp/docbook/Doc.View.cls?KEY=GORIENT_ch_enviro

3. Deploy production 

Interoperability -> Manage -> Deployment changes -> Deploy -> Open deployment -> find production XML file -> Select target production -> deploy

More details of this can be found here: https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=EGDV_deploying

4. Check to see if production components are visible 

Interoperability -> Configure -> Production -> If not then the press open -> Select Production

5. Set up OBDC connection to database if it does not already exist on the server

6. Download Apache Tika from the following link:https://tika.apache.org/download.html (version 2.6 was used for testing)

7. Update the following T-FLIP components: 
- DSN for 'Doc_InvokeHistoricIngestion','Img_InvokeHistoricIngestion' and 'ExecuteQuery' with the name of the OBDC that has been set up 
-  Port numbers on components 'TestHL7In' and 'TestMDMIn' for receiving ORU and MDM HL7 message - These components can be replaced for appropriate names for Test labs or document systems
- 'TempFileConvertorFolder' on components Doc_ProcessHistoricData and Doc_ProcessMDM with an appropriate location to temporarily store files during the text extraction process
- 'TikaFilePath' on components Doc_ProcessHistoricData and Doc_ProcessMDM with the location of .jar file downloaded
- 'URL' and 'FLIPAuthorisation' on HTTP_SendToFlip - the endpoint and Authorisation details will be sent seperately

# T-FLIP Interface Components

***Doc_InvokeHistoricIngestion***

This component  will query the document metadata staging table (tflip.usdoc_metadata) and select the number of rows available that have yet to be processed will then send a SQL snapshot of this number to Doc_ProcessHistoricData

***Img_InvokeHistoricIngestion***

This component  will query the imaging results staging table (tflip.imaging_results) and select the number of rows available that have yet to be processed

***Doc_ProcessHistoricData***

Component receives a SQL snapshot from Doc_Invoke historic ingestion with the number of rows to be processed, will check this number against a value in the lookup table “FLIP.Configuration” to check if there is an appropriate number of records to send in the batch. If not, then the process will quit.
The process will then prepare the query and retrieve the rows from the database. For each row in the table a check is made to see if the document is base 64 encrypted, if so then the document is decrypted and saved in a temporary folder. Apache tika is then used to convert the document to a text file and the text is then extracted.  A query is then prepared to update the staging table so the records are not reprocessed

***Img_ProcessHistoricData***

Component receives a SQL snapshot from Doc_Invoke historic ingestion with the number of rows to be processed, will check this number against a value in the lookup table “FLIP.Configuration” to check if there is an appropriate number of records to send in the batch. If not, then the process will quit. The process will then prepare the query and retrieve the rows from the database, a query is then prepared to update the staging table so the records are not reprocessed.

***Doc_ProcessMDM***

Receives a HL7 MDM message and extracts the required data. If the message contains an encrypted document then the document will be decrypted and then saved into a temporary folder. Apache tika is then used to convert the document into a text file and the text is then extracted

***Doc_ProcessORU***

Receives HL7 ORU message and extracts the required data

***FHIRProcessor***

Common Business process that will receive data from all four streams and convert into the relevant FHIR bundle. The bundle will then be placed inside a stream and passed on to the HTTPOperation

***ExecuteQuery***

Common Business Operation that will be handle the communication with the Postgres database.

***HTTP_SendToFlip***

Common Business Operation for sending HTTP requests to the FLIP endpoint


