# COMPONENTS

This document will explain the necessary steps to configure the oni-oa component.

## Folder Structure

        data                            -> Database engine configuration python module
                                            | engine.json           -> Configuration file to setup db engine and node

        reputation                      -> Reputation services python module
                                            | fb                        -> GTI sub-module for facebook ThreatExchange
                                            | gti                       -> GTI sub-module for McAfee GTI
                                            | gti/gti_cat_codes.csv     -> GTI category and family names 
                                            | reputation_config.json    -> Configuration file for the reputation services module
                                            
        iana                            -> Internet Assigned Numbers Authority codes translation python module
                                            | dns-qclass.csv   -> Iana DNS classes
                                            | dns-qtype.csv    -> Iana DNS types
                                            | dns-rcode.csv    -> Iana DNS rcodes
                                            | http-rcode.csv    -> Iana HTTP rcodes 
                                            | iana_config.json -> Configuration file for Iana module
        nc                              -> Network Context python module
                                            | nc_config.json   -> Configuration file for Network Context module

        geoloc                          -> Geolocation module
                                            | geoloc   -> Module to assign geolocation to every IP


###Data
Data source module

This module needs to be configured correctly to avoid errors during the oni-oa execution. Here you need to select the correct database engine to obtain the correct results while creating additional details files.
Currently oni-oa is prepared to work with Impala, but you can always configure any other database engine and make the corresponding updates in the code.

**Configuration**

You need to update the _engine.json_ file accordingly:

        {
            "oa_data_engine":"<database engine>",
            
            "impala":{
                "impala_daemon":"<node>"
            },
            "hive":{}
        }

Where:
- database engine: Whichever database engine you have installed and configured in your cluster to work with ONI. i.e. "Impala" or "Hive".
For this key, the value you enter needs to match exactly with one of the following keys, where you'll need to add the corresponding node name. 
- node: The node name in your cluster where you have the database service running. 

Example:

    {
        "oa_data_engine":"impala",        
        "impala":{
            "impala_daemon":"workernode07"
        },
        "hive":{}
    }


###Reputation
Reputation check module.

This module is called during oni-oa execution to check the reputation for any given IP, URL or URI (depending on the pipeline). The reputation module makes use of two third-party services, McAfee GTI and Facebook ThreatExchange. 
Each of these services are represented by a sub-module in this project, McAfee GTI is implemented by sub-module gti and Facebook ThreatExchange by sub-module fb. For more information see Folder Structure section.

**Pre-requisites**

- McAfee GTI client and credentials. McAfee GTI client is not included in this project. To get a copy of their [rest client](https://secure.mcafee.com/apps/downloads/my-products/login.aspx?region=us) and credentials 
(server, user, password) get in touch with a McAfee® representative at Licensing@McAfee.com. *If you are not interested on using McAfee GTI or you are not McAfee customer you can always disable McAfee GTI reputation check.*

- Facebook API Key. For Facebook ThreatExchange it is required to obtain first an API Key. To learn more about how to get an API Key go to [Facebook Developers](https://developers.facebook.com/). *If you are not interested on using Facebook ThreatExchange you can always disable ThreatExchange reputation check.*

**Enable/Disable GTI service**

It's possible to disable any of the reputation services mentioned above, all it takes is to remove the configuration for the undesired service in gti_config.json. To learn more about this see the next section.

**Configuration**

- reputation_config.json: Stores a list of reputation services to call during oni-oa execution. Also it contains a list of columns to check
 reputation. gti_config.json looks like this:

        {
            "gti":{
                "server" : "<server>",
                "user" : "<user>",
                "password" : "<password>",
                "ci" : "{\"ci\":{\"cliid\":\"<cliid>\", \"prn\":\"<prn>\", \"sdkv\":\"1.0\", \"pv\":\"1.0.0\", \"pev\":1, \"rid\":1, \"affid\":\"0\"},\"q\":[###QUERY###]}",
                "refclient" : "<refclient location>",
                "category_file":"<gti_cat_codes file location>"
            },
            "fb":{
                "app_id" : "<app_id>",
                "app_secret" : "<app_secret>"
            }
        }
    Where
    - gti: McAfee GTI service connection details. Please note that these values have to be provided by McAfee upon contracting the GTI service.
        - server: string with McAfee GTI server.
        - user: string with McAfee GTI user.
        - password: string with McAfee GTI password.
        - ci: request body for GTI service. This string parameter is formatted as json content,
        be careful with removing scape characters for single quotes and double quotes.

        Do not remove, replace or modify the label ###QUERY###, it's a special placeholder and is required for
        reputation.py to work.
        - prn: you can give a name to your Oni instance like "CompanyX-Oni" or you can just leave as is.
        - refclient: absolute path for restclient, which will be provided by McAfee upon contracting the GTI service. It should be the path where the *restclient* file is
        located without ending backslash i.e.

                /home/solution-user/refclient/restclient

        - category_file: absolute path for the gti category and functional names catalog. This file is also provided by McAfee upon contracting the GTI service.
        This file should be a comma separated text file, including headers for the following columns:
            - Category code: int
            - Category name: string
            - Functional group: string

    - fb: facebook [ThreatExchange](https://developers.facebook.com/products/threat-exchange) service connection details.
        - app_id: App id to connect to ThreatExchange service.
        - app_secret: App secret to connect to ThreatExchange service.

###IANA
Internet Assigned Numbers Authority codes translation module.

**Configuration**

- iana_config.json:  Stores the location of codes translation files. By default, oni-oa iana module comes with 4 csv files
with different code types for DNS and HTTP queries and usually they will be located in oa/components/iana/. These files can be
moved to any preferred location.

To set this up, replace each value with the absolute path for each IANA codes catalog, for example if IANA code files are in the
default location, your configuration file should look like this:

		{
		    "IANA": {
	             "dns_qry_class":"/home/solution-user/oni-oa/oa/components/iana/dns-qclass.csv",
	             "dns_qry_type":"/home/solution-user/oni-oa/oa/components/iana/dns-qtype.csv",
	              dns_qry_rcode":"/home/solution-user/oni-oa/oa/components/iana/dns-rcode.csv",
	              http_qry_rcode":"/home/solution-user/oni-oa/oa/components/iana/http-rcode.csv"
	          }
		 }


###Network Context (nc)
Network Context module.

**Pre-requisites**

Before start working with network context it's required to generate a comma separated file with the corresponding network context information.

This file can be placed anywhere in the system running oni-oa, although we suggest you place it inside the _context_ folder to keep uniformity.
The following schema is expected:
-   IP: string
-   Description: String

Example:

        10.192.180.150, NOT IN DNS
        10.192.180.1, Machine name 

**Configuration**

- nc_config.json: replace the value of network_context with the absolute path of networkcontext.csv file, the
configuration file should look like this:

		{
		    NC" : {
                        "network_context_dns" : "/home/solution-user/oni-oa/context/networkcontext.csv"
			   }
		 }

         
###Geoloc
Geolocation module.

This is an optional functionality you can enable / disable depending on your preferences.

**Pre-requisites**
To start using this module, you need to include a comma separated file containing the geolocation for most (or all) IPs.
To learn more about the expected schema for this file or where to find a full geolocation db, please refer 
to the [_context_](https://github.com/Open-Network-Insight/oni-oa/blob/1.1/oa/context/README.md)
 documentation  

 
**Configuration**
Oni-oa is preconfigured to look for the geolocation file at the _~/context/_ path.

Example:

        /home/solution-user/oni-oa/context/iploc.csv