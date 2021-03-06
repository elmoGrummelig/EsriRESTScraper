EsriRESTScraper
===============

A lightweight Python (tested in Python 3.x and 2.x) module that scrapes ESRI REST endpoints and parses the data into a local or enterprise geodatabase feature class

Updates for Python 3
====================

### Some updates in the latest release
* __Paging support__:
Module now utilizes Esri's REST API pagination support to scrape entire record sets when a single request would not otherwise capture all records (based on the MaxRecordCount property) in the query and the module would throw a TooManyRecords exception.  If the feature service does not support pagination, and the query exceeds the maximum records returned in a single query, then the exception is thrown.  Otherwise, updateFeatureClass makes as many requests as necessary to capture all records in a query.  

* __No ijson dependency__:
Ijson dependency removed - the module relies soley on Python's requests library

* __Support for multipoint geometry__:
Multipoint geometry feature classes/services are now supported.

* __GlobalID bug fixed__:
Scraping no longer fails when feature service contains a global id.  The module simply ignores this field type.

### Dependencies
* __Esri's arcpy library__ - Python 2.x library installed wtih ArcGIS for Desktop (ArcMap) and Python 3.x library installed ArcGIS Pro

Instructions
====================

This class is instantiated with the Esri REST Endpoint of a feature layer inside a map service.  For secured map services, you can include an optional token when instantiating the class. 
<br> e.g. 
```python
import RestCacheClass
earthquakesScraper = RestCacheClass.RestCache("https://earthquake.usgs.gov/arcgis/rest/services/eq/event_30DaySignificant/MapServer/0")
```
## Constructor
```python
def __init__(self, url, token=None, userFields=[], excludeFields=[]):
```

The RestCache object when instantiated scrapes the feature layer page for it's various attributes: fields, wkid, max record count, name, and geometry type.

This class has three primary methods:

1. createFeatureClass
2. updateFeatureClass
3. recreateFeatureClass

## createFeatureClass

This method creates a feature class in a geodabase with the appropriate spatial reference, geometry type, and the appropriate fields (and field lengths, etc).  This method really only needs to be run a single time, then you have the correct feature class locally, and all you need to do is update it.

The name of the feature class is derived from the name in the REST endpoint, although this can be overwritten with an optional parameter.

### Signature

```python
def createFeatureClass(self, location, name, excludeFields=[]):
```

* __location__ (str): Local geodatabase or enterprise geodatabase
* __name__ (str): Name of the feature class to be created.  Defaults to the name of the feature service
* __excludeFields__ (list): List of fields that are in the feature service to not create in the feature class.  

#### Issues:

1.  The method only supports creating a feature class in a geodatabase (enterprise or local), not a shapefile.  If someone wants to modify this to support creating other types of workspaces, please do so!!
2.  Some field types are not supported either, although the most common ones are: text, date, short, long, double, float.

```python
earthquakesFeatureClass = earthquakesScraper.createFeatureClass(r'C:\Geodata\earthquakes.gdb', 'earthquakes')
```

## updateFeatureClass

This method makes one or more REST calls, parses the data, and updates your local geodatabase feature class.  Pretty straightforward.  This method accepts as input the feature class to update, a single query or list of queries (the default is "1=1"), and a Boolean value on whether to append to the existing dataset or overwrite (default is to overwrite since I didn't want to deal with differentials).

The method will gracefully end if there is a schema mismatch between the REST endpoint and the feature class to update.  This is to account for situations when the service definition changes without your knowledge.  However, you can also account for schema differences using the excludeFields and userFields parameters.

The method will capture all records in a query, even if a query would return more records than the MaxRecordCount, if the service supports pagination (most do).  Otherwise, it will fail if the query would return more records than the MaxRecordCount.  

```python
earthquakesScraper.updateFeatureClass(earthquakesFeatureClass, ["magnitude > 4"])
```
### Signature

```python
def updateFeatureClass(self, featureClassDestination, query=["1=1"], append=False, userFields=[], excludeFields=[], debug=False, debugLoc=sys.path[0]):
```
* __featureClassDestination__ (str): Local geodatabase feature class
* __query__ (list): Single query or list of queries for the feature service (defaults to "1=1" which returns all records)
* __append__ (bool): Either appends records to the feature class or deletes all records first
* __userFields__ (list): Fields in your local feature class (but not in the feature service) to ignore when checking for a schema match between feature class and feature service
* __excludeFields__ (list): Fields in the feature service (but not in the feature class) to ignore when checking for a schema match
* __debug__ (bool): Sets debug mode to true or false
* __debugLoc__ (str): Optional override location for log file.  Defaults to execution directory.  

## recreateFeatureClass

This method will essentially recreate the feature class to match the current service definition schema.  It deletes all fields, and re-adds the fields from the service definition.  Useful for a workflow where you catch a SchemaMismatch error, recreate the feature class, then update.  

### Signature

```python
def recreateFeatureClass(self, target, userFields=[], excludeFields=[]):
        """Method to recreate target feature class by recreating fields from REST Endpoint
        Can be invoked if SchemaMismatch error is thrown and caught"""
```

* __target__ (str): Local geodatabase or enterprise geodatabase feature class
* __userFields__ (list): List of fields in the feature service to retain in your feature class
* __excludeFields__ (list): List of fields that are in the feature service to not create in the feature class.  
 
Please let me know if you have any questions!

### Previous version updates  

* __Debug Fixes__:
I removed the Debug class and just incorporated Python's native logging class for debug operations.  Set debug=True in the updateFeatureClass method and the execution will write to a log file in the same directory as your script.  If you rely heavily on this, you will want to add custom log messages to the code.   

* __Append option during update__:
You can now set an append parameter to true when calling the updateFeatureClass method and the class won't delete the features in the destination feature class before adding entries from the REST endpoint.  I've used this when two endpoints have the same schema and I don't want to maintain two feature classes.  

* __Ability to incorporate custom fields__:
When running the updateFeatureClass method, it checks to ensure the feature service endpoint has the same schema as the destination feature class.  However, some of my workflows required additional fields in my destination feature class, (e.g. fields whose values are calculated based on other fields whose attributes are scraped from REST endpoints).  So I added a userFields parameter to the updateFeatureClass method that allows a user to specify custom fields that shouldn't be included in the schema match check (or the update).  So after updating the feature class, you can now add additional methods to calculate values of those extra fields.  

* __Ability to exclude specific Feature Service fields__:
Achieving a similar effect to the userFields parameter.  This excludes updating specific fields if you choose not to include them in your local feature class.  So if a FeatureService schema changes, you can add whatever additional fields are now breaking your workflow to the excludeFields parameter without recreating your feature class with the new fields. 

* __Recreate Feature Class__:
However, if needed, I included a function to just recreate the feature class.  It deletes all fields, then re-adds them.  

### Some less important updates

* Features are not deleted until records from REST endpoint for the first query are successfully returned.  So if the endpoint is unavailable or if an error occurs, you're previous features won't be deleted

* Error handling for some edge cases in older versions of ArcGIS Server.  Still not perfect, but there.  

* Error handling for null Point geometries

* \__getEsriRESTJSON function will attempt it's operation 5 times, waiting 5 seconds between each try, before erring out. Some public ArcGIS servers have been error prone, but will often have a successful respond on a successive query.  I've found that my workflows most often err out when running the \_\_getNumRecordsFromQuery method

* Better code documentation and flow






