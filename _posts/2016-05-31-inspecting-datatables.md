---
layout: post
title: "Inspecting DataTables with WinDbg"
date: 2016-06-07 
comments: true
tags: ["WinDbg", "trick"]
published: true 
---

DataTables are great structures in .Net... they store a lot of 
data with a lot of different natures. Typed data, untyped, with lots of
columns, with lots of rows... sometimes they can be used to cache data
and do some local processing...

... and sometimes they get out of control.
<!-- More -->

The problem with DataTables is that they can store big amounts of data, 
and this can become a problem in two ways:

- CPU usage can increase when we do processing on big DataTables (think 
on "foreach row" loops)
- RAM usage can be huge and lots of problems can happen if they get big 
enough to go to the LOH (Large Object Heap... I promise I will talk on 
this on further posts).

OK, let's go.
We will use *SOSEX* on this recipe

```
.load sosex
```

## Prerequisites of this recipe

Well, first thing to do is to find the DataTable we want to look at.
Basically, there are two ways to find a specific object in memory

### Get a reference on the stack of a thread.

Remember: 
```
!mk -p
```
will show all the parameters of selected thread's call stack. If we are
lucky enough, perhaps we will find a reference to the DataTable we are
looking for.

### Find the object on the heap

The hard way to find an object is to look for it on the heap. The heap
is the place where all the reference types are stored... so, almost 
everything is on the heap. My advise to find an object: if you are looking
for long living objects, start by looking at GEN2 heap. This can be
achieved with SOSEX this way:

```
0:012> !dumpgen 2 -type System.Data.DataTable
Object MT Size Name
---------------------------------------------------
0140ddb8 65244990 36 System.Data.DataTableCollection
014156e8 042f3a48 12 System.Collections.Generic.ObjectEqualityComparer`1[[System.Data.DataTable, System.Data]]
01418960 65242cc8 296 System.Data.DataTable
01526204 65244990 36 System.Data.DataTableCollection
01526e28 65242cc8 296 System.Data.DataTable
01532324 65242cc8 296 System.Data.DataTable
01533928 65242cc8 296 System.Data.DataTable
0169b4f4 65244990 36 System.Data.DataTableCollection
0169b570 65242cc8 296 System.Data.DataTable
0169cc90 652466e4 72 System.Data.DataTablePropertyDescriptor
01781718 65244990 36 System.Data.DataTableCollection
01781794 65242cc8 296 System.Data.DataTable
01781e90 652466e4 72 System.Data.DataTablePropertyDescriptor
017cddb8 65244990 36 System.Data.DataTableCollection
017ce3ec 65242cc8 296 System.Data.DataTable
017e02a4 65244990 36 System.Data.DataTableCollection
017e0320 65242cc8 296 System.Data.DataTable
017e0a1c 652466e4 72 System.Data.DataTablePropertyDescriptor
18 objects, 2,812 bytes
```
If you cant find your object here, look at GEN1 and the GEN0 heaps.

> Tip: If we follow good programming practices, GEN2 should have few objects,
> as we should tend to minimize long living objects

Another way to find a DataTable is to look for an unusual string you
know it is stored on this DataTableCollection

```
!strings /m:<Very_Unusual_String>
```

> Tip: You can look for a string in GEN2 using 
>```
>!strings /g:2 /m:<Very_Unusual_String>
>```

Then, you can use 
```
!refs <addr>
```
to look for objects that reference your string. Following the chain of
references, if you are lucky and the chain is not broken, you could find
your desired DataTable.

In any case... these are only prerequisites. Let's start with our real
business

## Dump a given DataTable

Given the address of a DataTable, we will start showing its properties.

```
0:012> !mdt 01526e28
This command may not work correctly without full memory info.
01526e28 (System.Data.DataTable)
site:NULL (System.ComponentModel.ISite)
events:NULL (System.ComponentModel.EventHandlerList)
dataSet:01526198 (System.Data.DataSet)
defaultView:NULL (System.Data.DataView)
nextRowID:0xb (System.Int32)
rowCollection:015270a4 (System.Data.DataRowCollection)
columnCollection:01526fe4 (System.Data.DataColumnCollection)
constraintCollection:0152706c (System.Data.ConstraintCollection)
elementColumnCount:0x1a (System.Int32)
parentRelationsCollection:NULL (System.Data.DataRelationCollection)
childRelationsCollection:015275e4 (System.Data.DataRelationCollection+DataTableRelationCollection)
recordManager:01526fac (System.Data.RecordManager)
indexes:Unable to determine MT for generic field. Error=0x0488caec.
shadowIndexes:Unable to determine MT for generic field. Error=0x0488caec.
shadowCount:0x0 (System.Int32)
extendedProperties:NULL (System.Data.PropertyCollection)
tableName:01526280 (System.String) Length=9, String="Workplace"
tableNamespace:NULL (System.String)
tablePrefix:01341198 (System.String) 
displayExpression:NULL (System.Data.DataExpression)
fNestedInDataset:true (System.Boolean)
_culture:0134264c (System.Globalization.CultureInfo)
_cultureUserSet:true (System.Boolean)
_compareInfo:01347848 (System.Globalization.CompareInfo)
_compareFlags:0x19 (System.Globalization.CompareOptions)
_formatProvider:0134264c (System.Globalization.CultureInfo)
_hashCodeProvider:01527734 (System.CultureAwareComparer)
_caseSensitive:false (System.Boolean)
_caseSensitiveUserSet:false (System.Boolean)
encodedTableName:01526280 (System.String) Length=9, String="Workplace"
xmlText:NULL (System.Data.DataColumn)
_colUnique:NULL (System.Data.DataColumn)
textOnly:false (System.Boolean)
minOccurs:(System.Decimal) 1 VALTYPE (MT=79307c84, ADDR=01526f2c)
maxOccurs:(System.Decimal) 1 VALTYPE (MT=79307c84, ADDR=01526f3c)
repeatableElement:false (System.Boolean)
typeName:013dfc7c (System.Xml.XmlQualifiedName)
primaryKey:01531b90 (System.Data.UniqueConstraint)
_primaryIndex:01532310 (System.Data.IndexField[], Elements: 1, ElementMT=65245148) expand
delayedSetPrimaryKey:NULL (System.Object[])
loadIndex:NULL (System.Data.Index)
loadIndexwithOriginalAdded:NULL (System.Data.Index)
loadIndexwithCurrentDeleted:NULL (System.Data.Index)
_suspendIndexEvents:0x0 (System.Int32)
savedEnforceConstraints:false (System.Boolean)
inDataLoad:false (System.Boolean)
initialLoad:false (System.Boolean)
schemaLoading:false (System.Boolean)
enforceConstraints:true (System.Boolean)
_suspendEnforceConstraints:false (System.Boolean)
fInitInProgress:false (System.Boolean)
inLoad:false (System.Boolean)
fInLoadDiffgram:false (System.Boolean)
_isTypedDataTable:0x02 (System.Byte)
EmptyDataRowArray:NULL (System.Object[])
propertyDescriptorCollectionCache:NULL (System.ComponentModel.PropertyDescriptorCollection)
_nestedParentRelations:0136a4c8 (System.Data.DataRelation[], Elements: 0) expand
dependentColumns:Unable to determine MT for generic field. Error=0x0488caec.
mergingData:false (System.Boolean)
onRowChangedDelegate:NULL (System.Data.DataRowChangeEventHandler)
onRowChangingDelegate:NULL (System.Data.DataRowChangeEventHandler)
onRowDeletingDelegate:NULL (System.Data.DataRowChangeEventHandler)
onRowDeletedDelegate:NULL (System.Data.DataRowChangeEventHandler)
onColumnChangedDelegate:NULL (System.Data.DataColumnChangeEventHandler)
onColumnChangingDelegate:NULL (System.Data.DataColumnChangeEventHandler)
onTableClearingDelegate:NULL (System.Data.DataTableClearEventHandler)
onTableClearedDelegate:NULL (System.Data.DataTableClearEventHandler)
onTableNewRowDelegate:NULL (System.Data.DataTableNewRowEventHandler)
onPropertyChangingDelegate:NULL (System.ComponentModel.PropertyChangedEventHandler)
onInitialized:NULL (System.EventHandler)
rowBuilder:015275d4 (System.Data.DataRowBuilder)
delayedViews:Unable to determine MT for generic field. Error=0x0488caec.
_dataViewListeners:Unable to determine MT for generic field. Error=0x0488caec.
rowDiffId:NULL (System.Collections.Hashtable)
indexesLock:01526f80 (System.Threading.ReaderWriterLock)
ukColumnPositionForInference:0xffffffff (System.Int32)
_remotingFormat:0x0 (Xml) (System.Data.SerializationFormat)
_objectID:0x4 (System.Int32)

```
Wow! Those are a lot of properties! Probably you have noticed that properties seen here
are very different from those in the C# reference of what a DataTable is. This is because
What you are seeing here is not exactly C#... it is IL (intermediate language). You are also
seeing every field, regardless its visibility.

One of the best things of SOSEX is that it supports links... so click on columnCollection and 
SOSEX will execute automatically a mdt command.

```
0:012> !mdt 01526fe4
This command may not work correctly without full memory info.
01526fe4 (System.Data.DataColumnCollection)
table:01526e28 (System.Data.DataTable)
_list:0152701c (System.Collections.ArrayList)
defaultNameIndex:0x1 (System.Int32)
delayedAddRangeColumns:NULL (System.Object[])
columnFromName:01527034 (System.Collections.Hashtable)
onCollectionChangedDelegate:NULL (System.ComponentModel.CollectionChangeEventHandler)
onCollectionChangingDelegate:NULL (System.ComponentModel.CollectionChangeEventHandler)
onColumnPropertyChangedDelegate:NULL (System.ComponentModel.CollectionChangeEventHandler)
fInClear:false (System.Boolean)
columnsImplementingIChangeTracking:0136a49c (System.Data.DataColumn[], Elements: 0) expand
nColumnsImplementingIChangeTracking:0x0 (System.Int32)
nColumnsImplementingIRevertibleChangeTracking:0x0 (System.Int32)

```

Now click on _list

```
0:012> !mdt 0152701c
This command may not work correctly without full memory info.
0152701c (System.Collections.ArrayList) expand
_items:015282c4 (System.Object[], Elements: 32) expand
_size:0x1a (System.Int32)
_version:0x1a (System.Int32)
_syncRoot:NULL (System.Object)
```

Look at this! columns are stored in an ArrayList. Lets expand it..
click on _items

```
0:012> !mdt -e:1 015282c4
This command may not work correctly without full memory info.
015282c4 (System.Object[], Elements: 32) expand
[0] 015276a0 (System.Data.DataColumn)
[1] 01527744 (System.Data.DataColumn)
[2] 015277d8 (System.Data.DataColumn)
[3] 0152786c (System.Data.DataColumn)
[4] 01527900 (System.Data.DataColumn)
[5] 01527994 (System.Data.DataColumn)
[6] 01527a28 (System.Data.DataColumn)
[7] 01527abc (System.Data.DataColumn)
[8] 01527b50 (System.Data.DataColumn)
[9] 01527be4 (System.Data.DataColumn)
[10] 01527c78 (System.Data.DataColumn)
[11] 01527d0c (System.Data.DataColumn)
[12] 01527da0 (System.Data.DataColumn)
[13] 01527e34 (System.Data.DataColumn)
[14] 01527ec8 (System.Data.DataColumn)
[15] 01527f5c (System.Data.DataColumn)
[16] 01527ff0 (System.Data.DataColumn)
[17] 01528354 (System.Data.DataColumn)
[18] 015283e8 (System.Data.DataColumn)
[19] 0152847c (System.Data.DataColumn)
[20] 01528510 (System.Data.DataColumn)
[21] 015285a4 (System.Data.DataColumn)
[22] 01528638 (System.Data.DataColumn)
[23] 015286cc (System.Data.DataColumn)
[24] 01528760 (System.Data.DataColumn)
[25] 0152cce8 (System.Data.DataColumn)
[26] null
[27] null
[28] null
[29] null
[30] null
[31] null
```

Now you are seeing some internals of the ArrayList. As you can see, ArrayList expands its capacity in powers of two.
Some of the positions of the inner array are empty... so, you have data till you see the first null value (26 columns in this case)

Now click on a DataColumn...

```
0:012> !mdt 01528760
This command may not work correctly without full memory info.
01528760 (System.Data.DataColumn)
site:NULL (System.ComponentModel.ISite)
events:NULL (System.ComponentModel.EventHandlerList)
allowNull:true (System.Boolean)
autoIncrement:false (System.Boolean)
autoIncrementStep:0x1 (System.Int64)
autoIncrementSeed:0x0 (System.Int64)
caption:NULL (System.String)
_columnName:01526854 (System.String) Length=14, String="imputationDate"
dataType:013646f8 (System.RuntimeType)
defaultValue:01341f38 (System.DBNull)
_dateTimeMode:0x2 (Unspecified) (System.Data.DataSetDateTime)
expression:NULL (System.Data.DataExpression)
maxLength:0xffffffff (System.Int32)
_ordinal:0x18 (System.Int32)
readOnly:false (System.Boolean)
sortIndex:NULL (System.Data.Index)
table:01526e28 (System.Data.DataTable)
unique:false (System.Boolean)
columnMapping:0x1 (Element) (System.Data.MappingType)
_hashCode:0xcea0f08c (System.Int32)
errors:0x0 (System.Int32)
isSqlType:false (System.Boolean)
implementsINullable:false (System.Boolean)
implementsIChangeTracking:false (System.Boolean)
implementsIRevertibleChangeTracking:false (System.Boolean)
implementsIXMLSerializable:false (System.Boolean)
defaultValueIsNull:true (System.Boolean)
dependentColumns:Unable to determine MT for generic field. Error=0x0488caec.
extendedProperties:NULL (System.Data.PropertyCollection)
onPropertyChangingDelegate:NULL (System.ComponentModel.PropertyChangedEventHandler)
_storage:01535028 (System.Data.Common.DateTimeStorage)
autoIncrementCurrent:0x0 (System.Int64)
_columnUri:NULL (System.String)
_columnPrefix:01341198 (System.String) 
encodedColumnName:01526854 (System.String) Length=14, String="imputationDate"
description:01341198 (System.String) 
dttype:015268b0 (System.String) Length=8, String="dateTime"
simpleType:NULL (System.Data.SimpleType)
_objectID:0x35 (System.Int32)
```

Mmm... perhaps you were expecting the data to be stored on rows (the intuitive way), but data is really stored in columns.

Now click on _storage.

```
0:012> !mdt 01535028
This command may not work correctly without full memory info.
01535028 (System.Data.Common.DateTimeStorage)
Column:01528760 (System.Data.DataColumn)
Table:01526e28 (System.Data.DataTable)
DataType:013646f8 (System.RuntimeType)
StorageTypeCode:0x10 (DateTime) (System.Data.Common.StorageType)
dbNullBits:015375e4 (System.Collections.BitArray)
DefaultValue:01535054 (BOXED System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01535058)
NullValue:01341f38 (System.DBNull)
IsCloneable:false (System.Boolean)
IsCustomDefinedType:false (System.Boolean)
IsStringType:false (System.Boolean)
IsValueType:true (System.Boolean)
values:015371d8 (System.DateTime[], Elements: 128, ElementMT=79307f4c) expand
```

Click on the "expand" link next to values

```
0:012> !mdt -e:1 015371d8
This command may not work correctly without full memory info.
015371d8 (System.DateTime[], Elements: 32, ElementMT=79307f4c) expand
[0] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015371e0)
[1] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015371e8)
[2] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015371f0)
[3] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015371f8)
[4] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537200)
[5] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537208)
[6] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537210)
[7] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537218)
[8] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537220)
[9] (System.DateTime) 2016/05/12 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537228)
[10] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537230)
[11] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537238)
[12] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537240)
[13] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537248)
[14] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537250)
[15] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537258)
[16] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537260)
[17] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537268)
[18] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537270)
[19] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537278)
[20] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537280)
[21] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537288)
[22] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537290)
[23] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=01537298)
[24] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372a0)
[25] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372a8)
[26] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372b0)
[27] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372b8)
[28] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372c0)
[29] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372c8)
[30] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372d0)
[31] (System.DateTime) 0001/01/01 00:00:00.000 VALTYPE (MT=79307f4c, ADDR=015372d8)
```

We have arrived to the end of the journey. Here you have the data of the column. 
Again, you have an inner ArrayList that grows in powers of two, so you have to read the
data till you see a value initialized with the default value of its DataType.
In this case, we have 10 DateTimes assigned till row 9.

This process of expanding DataColumns must be repeated for all the DataColumns of the DataTable.
Use a spreadsheet and you will have a quite good representation of the "in memory" DataTable.