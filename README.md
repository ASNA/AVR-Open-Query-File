##Use IBM i's Open Query File with ASNA Visual RPG

Finding records where a field contains a given value is an awkward thing to do with RPG's record-level access. In either green-screen RPG or ASNA Visual RPG (AVR), the ugly solution is simple loop over every record in the file checking each field as you go. This works, but have something nearby to keep you occupied while you wait.
 
Intermediate to advanced coders may turn to embedded SQL with green-screen RPG or use ADO.NET with AVR. This is a good solution, but n the case of AVR requires troublesome setup and configuration for the .NET managed provider required&mdash;and SQL knowledge. 

####The Open Query File alternative

However, a good alternative does lurk in the form of AVR's support of Open Query File. The IBM i's Open Query File (OQF) is a rich and deep set of features. On the IBM i side of things you can do lots with OQF. However, IBM i programmers have pretty much relegated OQF to the sidelines with the advent of IBM i's superior SQL-based options. AVR provides support for a subset of OQF facilities which generally encompass OQF's QRYSLT and KEYFLD keywords. Don't expect to be able to do with AVR all that you can ILE RPG with OQF, but even with limited power, using OQF with AVR can provide some excellent results. As always, test, test, test. If you do something cool with IBM i's OQF and AVR, please let us know. Read more about IBM i's [Open Query File here](https://www-01.ibm.com/support/knowledgecenter/ssw_ibm_i_72/cl/opnqryf.htm) 

Open Query File is pretty smart and does a good job brute forcing its way through files, even when queries are specified that don't have a corresponding previously-created access path. That said, test your queries carefully. Throw a sloppy enough query at OQF and you might make smoke come out of you IBM i.      

AVR doesn't support OQF on DataGate for Windows (AKA the local ASNA database) and results with DataGate for SQL Server are limited. It's best to consider what this article offers as an DataGate for IBM i-only solution. 

####Get to it! 

This simple example shows how to extract records from a file searching the customer name for a given string anywhere in the customer name. The input file is an indexed logical file keyed on customer name and customer number. A code narrative follows the code below. 
 
    BegClass Test
    
        DclDB pgmDB DBName( "*Public/Cypress" )
    
        DclDiskFile  Cust                 +
            Type( *Input )                +
            Org( *Indexed )               +
            Prefix( Cust_ )               +
            File( "Examples/CMastNewL1" )   +
            DB( pgmDB )                   +
            ImpOpen( *No )
    
    
        BegSr Run Access(*Public)        
            DclFld QueryMask  Type(*String) 
            DclFld QueryValue Type(*String) 
            DclFld Query      Type(*String) 
    
            Connect pgmDB 
    
            Cust.QryKeyFlds = "CMName *ASCEND CMCustNo *ASCEND"
            QueryValue = "CANADA"
            
            QueryMask = "%XLATE(CMNAME qsystrntbl) *CT '{0}'"
            Query = String.Format(QueryMask, QueryValue)         
    
            Cust.QrySelect = Query
    
            Open Cust 
    
            Read Cust 
            DoWhile NOT Cust.IsEof
                Console.WriteLine(Cust_CMName)
    
                Read Cust 
            EndDo 
    
            Close *All
            Disconnect pgmDB         
        EndSr
    EndClass   

**Lines 1 - 9.** 

Declare the database object and the disk file. The `ImpOpen(*No)` requires the file must be explicitly opened in the code. This is an OQF requirement.  

**Line 18.** 

Specify the fields to include in the query output and optionally their sorting sequence by assigning them to the `Cust` file's QryKeyFields property.

**Line 19.** 

The `QueryValue` field is the query value that, although hard coded here, would normally be provided by the end user at runtime. In this case, we're looking for all customer names that contain 'AND SONS'. 

**Lines 21 - 22.**

The `QueryMask` field provides the base value to be passed as AVR's OQF's QrySelect value. Using the String class's static Format method interpolates the `QueryValue` with the `QueryMask` field (which, using `ToUpper()` is converted to uppercase before the interpolation is performed).This results in the `Query` field having the value:
       
    %XLATE(CMNAME qsystrntbl) *CT 'AND SONS'

This query is using some IBM i Open Query File magic that I learned way back in my formative years. Break it into two three pieces to understand it: 

1. `%XLATE(CMNAME qsystrntbl)` passes the `CMNAME` field to the `%XLATE` which uses the IBM i translation table `qsystrntbl` to return the uppercase value of the `CMNAME` field.  
2. *CT is Open Query File's `Contains` operator. It looks for the occurrence of the query value anywhere in the `CMNAME` field value. 
3. `'AND SONS'` is the query value. Because Step 1 converted the `CMNames` value to uppercase, specifying this value in uppercase (via the `ToUpper()` method during the string interpolation ensures the query is case insensitive. 

**Line 24.** 

Assign the resolved `Query` field to the `Cust` file's `QrySelect` property.

**Line 26.**

OQF is in play because the `Cust` file's `QryKeyFlds` and `QrySelect` properties have values. When OQF is in play, opening `Cust` doesn't open the underlying physical or logical file, but rather opens the result set returned by the query. The query occurs, on the server side, immediately after opening the file. 

The rest of the code is pretty obvious. Loop over the result set and do something with it. Remember that the only fields available are those specified by the `QryKeyFlds` value. 

### Rounding out your programming kitbag

This isn't a technique you'll use everywhere, but it's the kind of technique that when you do it need you'll be very glad it's in your AVR programming kit bag.      

