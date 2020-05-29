![](Slides/Slide1.JPG?raw=true)

![](Slides/Slide2.JPG?raw=true)

![](Slides/Slide3.JPG?raw=true)

![](Slides/Slide4.JPG?raw=true)

![](Slides/Slide5.JPG?raw=true)

T-SQL showing table distributions by database file
#### NOTE: This is not a fast operation!
``` sql
select  tableName   = object_name(a.object_id)
    ,   file_id     = a.allocated_page_file_id
    ,   allocations = sum(1)
from    sys.dm_db_database_page_allocations(db_id(), null, 1 , null , 'limited') as a
inner   join
    (
        select  object_id 
        from    sys.objects
        where   is_ms_shipped = 0
    )   as o
    on  a.object_id = o.object_id
group   by
        object_name(a.object_id)
    ,   allocated_page_file_id
order   by
        object_name(a.object_id)
    ,   allocated_page_file_id;
```
