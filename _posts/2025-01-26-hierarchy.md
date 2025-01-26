## Introduction
[`Hierarchy`](https://help.qlik.com/en-US/sense/November2024/Subsystems/Hub/Content/Sense_Hub/Scripting/ScriptPrefixes/Hierarchy.htm) is a very useful funcion in *Qlik Sense*, that can transform a parent-child hierarchy table. A similar funcionality can be replicated with a recursive CTE in *T-SQL*. A hierarchy where a child node has a single parent can also be transformed using *Parent and Child functions*, such as [`PATH`](https://learn.microsoft.com/en-us/dax/understanding-functions-for-parent-child-hierarchies-in-dax) in *DAX* in *Power BI*, however these functions cannot handle data where a child node might have multiple parents. This is often the case with components and assemblies in a *Bill of Materials*, because a single component might be required for multiple assemblies.

## Qlik Sense
### Hierarchy syntax
```text
Hierarchy (NodeID, ParentID, NodeName, [ParentName, [ParentSource, [PathName, [PathDelimiter, Depth]]]])(loadstatement | selectstatement) 
```

### input
| NodeID | ParentID | NodeName |
| :--- |:--- |:--- |
|	1 |	4 |	London |
|	2 |	3 |	Munich |
|	3 |	5 |	Germany |
|	4 |	5 |	UK |
|	5 |	 |	Europe |

### Qlik function
```text
Hierarchy(NodeID, ParentID, NodeName, ParentName, NodeName, PathName, '\', Depth)
```

### output
| NodeID | ParentID | NodeName | NodeName1 | NodeName2 | NodeName3 | ParentName | PathName | Depth |
| :--- |:--- |:--- |:--- |:--- |:--- |:--- |:--- |:--- |
| 1 |	4	| London | Europe | UK | London | UK | Europe\UK\London | 3 |
| 2	| 3	| Munich | Europe | Germany | Munich | Germany | Europe\Germany\Munich | 3 |
| 3	| 5	| Germany | Europe | Germany | - | Europe | Europe\Germany | 2 |
| 4	| 5	| UK | Europe | UK | - | Europe | Europe\UK | 2 |
| 5	|	  | Europe | Europe | - | - | - | Europe | 1 |

## AdventureWorks
This demonstration utilizes a sample dataset, that is provided by *Microsoft*, and it can be downloaded here: [`AdventureWorks`](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms)
   

### Qlik Sense
- [Qlik Sense script](Qlik_Sense_script.txt)

The solution is *Qlik* is fairly straight-forward as all the work is done by the `Hierarchy` function. The `BOMLevel` is adjusted by `1`, so that it aligns with the `Depth` in the output. `BOMLevel` starts at `0` and `Depth` starts counting the levels in the hierarchy at `1`.
```sql
BOM_Hierarchy:
Hierarchy(ComponentID, ProductAssemblyID, ComponentName, AssemblyName, ComponentName, PathName, '\', Depth)
SQL
SELECT
    BillOfMaterialsID
    , ProductAssemblyID
    , ComponentID
    , Product.Name as ComponentName
    , BOMLevel + 1 as BOMLevel
    , PerAssemblyQty
    , UnitMeasureCode
FROM
    Production.BillOfMaterials
    INNER JOIN Production.Product
        ON BillOfMaterials.ComponentID = Product.ProductID
WHERE
    EndDate IS NULL;
```

### T-SQL
- [T-SQL query](BOM_Hierarchy.sql)

#### 1. Load Data into CTE
The first step is to load it into a CTE. This step can be omitted, but it is helpful to see, that the query is the same as in *Qlik* so far.
```sql
WITH BOM_CTE as (
SELECT
    BillOfMaterialsID
    , ProductAssemblyID
    , ComponentID
    , Product.Name as ComponentName
    , BOMLevel + 1 as BOMLevel
    , PerAssemblyQty
    , UnitMeasureCode
FROM
    Production.BillOfMaterials
    INNER JOIN Production.Product
        ON BillOfMaterials.ComponentID = Product.ProductID
WHERE
    EndDate IS NULL
)
```
#### 2. Recursive_CTE
The recursive CTE is able to flatten the hierarchy. This step adds the name of the parent node - `AssemblyName`, complete path - `PathName`, as well as `Depth`. `PathJson` is a slightly modified step of `PathName`, which then allows us to use the `JSON_VALUE` in the last step.
```sql
, Recursive_CTE as (
SELECT
    *
    , CAST(NULL as varchar(50)) as AssemblyName
    , CAST(ComponentName as varchar(max)) as PathName
    , '"' + CAST(ComponentName as varchar(max)) + '"' as PathJson
    , 1 as Depth
FROM
    BOM_CTE
WHERE
    ProductAssemblyID IS NULL
UNION ALL
SELECT
    t.BillOfMaterialsID
    , t.ProductAssemblyID
    , t.ComponentID
    , t.ComponentName
    , t.BOMLevel
    , t.PerAssemblyQty
    , t.UnitMeasureCode
    , CAST(Recursive_CTE.ComponentName as varchar(50)) as AssemblyName
    , Recursive_CTE.PathName + '\' + CAST(t.ComponentName as varchar(max)) as PathName
    , Recursive_CTE.PathJson + ', "' + CAST(t.ComponentName as varchar(max)) + '"' as PathJson
    , Recursive_CTE.Depth + 1 as Depth
FROM
    BOM_CTE as t
    INNER JOIN Recursive_CTE
        ON t.ProductAssemblyID = Recursive_CTE.ComponentID
)
```
#### 3. Final output table
`T-SQL` lacks proper functionality for splitting of a string by a delimiter. A workaround I found utilizes the `JSON_VALUE`, which is able to extract a value by its index. All that is required is to add double quoutes `"` and a comma `,` between each string, and then encapsulate the result in square brackets `[` `]`. Howerver, with this approach, it is necessary to know the maximum number of levels in the Hierarchy. The function in *Qlik*, on the other hand, can dynamically determine the number of levels and it add the appropriate number of extra columns, one for each level.
```sql
SELECT 
    BillOfMaterialsID
    , ProductAssemblyID
    , ComponentID
    , ComponentName
    , BOMLevel
    , PerAssemblyQty
    , UnitMeasureCode
    , JSON_VALUE('[' + PathJson + ']', '$[0]') as ComponentName1
    , JSON_VALUE('[' + PathJson + ']', '$[1]') as ComponentName2
    , JSON_VALUE('[' + PathJson + ']', '$[2]') as ComponentName3
    , JSON_VALUE('[' + PathJson + ']', '$[3]') as ComponentName4
    , JSON_VALUE('[' + PathJson + ']', '$[4]') as ComponentName5
    , AssemblyName
    , PathName
    , Depth
FROM
    Recursive_CTE;
```
