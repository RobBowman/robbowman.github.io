---
title: BizTalk DateTime vs Oracle
category: BizTalk
tags:
    - BizTalk
    - Oracle
---
# BizTalk DateTime vs Oracle
I have an orchestration that received a data file and inserted contents into an Oracle database as part of its processing.

I'm using BizTalk 2013r2 sp with VS2013. I "Add Generated Items" on my schema project and select "Consume Adapter Service". From here I choose to generate a schema for "Insert" into the required table of my target Oracle database. This causes a target schema to be created. Now from here, my next step would usually be to create a BizTalk map with the newly created schema as my target. However, in this case the business logic for the mapping is such that I need to use a C# helper. So, my next step is to generate a C# representation of the new schema, for this I use the excellent https://xsd2code.codeplex.com/

One of the fields in the target table is called IMPORT_DATE and is of type DATE. The has been mapped as follows by VS / XSD2Code:

    public partial class IMPORT_DATE__COMPLEX_TYPE {
        
        private string inlineValueField;
        
        private System.DateTime valueField;
        
        private static System.Xml.Serialization.XmlSerializer serializer;
        
        [System.Xml.Serialization.XmlAttributeAttribute()]
        public string InlineValue {
            get {
                return this.inlineValueField;
            }
            set {
                this.inlineValueField = value;
            }
        }
        
        [System.Xml.Serialization.XmlTextAttribute()]
        public System.DateTime Value {
            get {
                return this.valueField;
            }
            set {
                this.valueField = value;
            }
        }

Into this field I simply need to place the current date time. My first effort was to use the following:

    IMPORT_DATE = new Generated.DW_ST_SUPPLIER_STOCK.IMPORT_DATE__COMPLEX_TYPE() { Value = DateTime.Now }

This did not work, the adapter raised the following error message:

    Error details: Microsoft.ServiceModel.Channels.Common.XmlReaderParsingException: Value for the field "IMPORT_DATE" is invalid. DateTime.Kind must be DateTimeKind.Unspecified. Ensure that there is no TimeZone or TimeZoneOffset contained in the DateTime value

My resolution was the following simple helper function:

    private DateTime GetOracleDate(DateTime dt)
        {
            string dtString = dt.ToString("yyyy-MM-dd hh:mm:ss");
            return Convert.ToDateTime(dtString);
        }

I then changed the calling code to the following:

    IMPORT_DATE = new Generated.DW_ST_SUPPLIER_STOCK.IMPORT_DATE__COMPLEX_TYPE() { Value = GetOracleDate(DateTime.Now) }

