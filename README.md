# neuron_in_postgis
concept about storing and querying large amounts of neuron in PostGIS and PostGres, possibly using PGPointcloud


2017-01-17 22:43 GMT+01:00 Tom Kazimiers <tom@voodoo-arts.net>:

    Hi all,

    I am using Postgres 9.6 with PostGIS 2.3 and want to improve the performance of a bounding box intersection query on a large set of lines. I asked the same questions two month ago on gis.stackexchange.com, but didn't get any reply. Therefore, please excuse me for cross-posting, but I figured this list would be a good place to ask. This is the original post:

     https://gis.stackexchange.com/questions/215762

    Problem: We store about 6 million lines/edges in a 3D space and use an axis
    aligned bounding box to find all intersecting (and included) lines. This works
    well, but we want to improve the query time for larger point sets.

    Context: Tree-like 3D structures represent neurons, each node has a parent node
    or it is the root. At the moment we deal with about 15 million nodes, grouped
    into 150000 trees (many > 10000 nodes). I want to improve existing performance
    bottlenecks with bigger result sets plus I plan to scale this setup to 10-100x
    the nodes. Typical query bounding boxes are rather slices than boxes, i.e. they expand any range in XY, but only a short distance in Z, but it could in principal any dimension that is "flat".

    Setup: The table storing edges looks like this:

     =>\d+ treenode_edge
                       Tabelle »public.treenode_edge«
             Spalte   |          Typ          | Attribute  ------------+-----------------------+-----------
      id         | bigint                | not null
      project_id | integer               | not null
      edge       | geometry(LineStringZ) |  Indexe:
              "treenode_edge_pkey" PRIMARY KEY, btree (id)
              "treenode_edge_gix" gist (edge gist_geometry_ops_nd)
              "treenode_edge_project_id_index" btree (project_id)

    Note that there is a 3D index for the edge column in place (treenode_edge_gix).

    Current timing: To request 2000 nodes with an typical bounding box size I use
    the following query:

     SELECT te.id
     FROM treenode_edge te
                                                                     -- left bottom z2, right top z1    WHERE te.edge &&& 'LINESTRINGZ( -537284.0  699984.0 84770.0,
                                                                      1456444.0 -128432.0 84735.0)'
                                       -- left top halfz, right top halfz,
                                       -- right bottom halfz, left bottom halfz,
                                       -- left top halfz; halfz (distance)
     AND ST_3DDWithin(te.edge, ST_MakePolygon(ST_GeomFromText(
              'LINESTRING( -537284.0 -128432.0 84752.5, 1456444.0 -128432.0 84752.5,
                                       1456444.0  699984.0 84752.5, -537284.0  699984.0 84752.5,
                                       -537284.0 -128432.0 84752.5)')), 17.5)
     AND te.project_id = 1
     LIMIT 2000;

    This takes about 900ms, statistics are up-to-date. This is not so bad, but I look for strategies to make this much faster still. The &&& operator filters already most of the existing edges by bounding box test (using index) so that ST_3DWithin only needs to check the edges that are most likely part of the result.

    The query plan looks like this:

     Limit  (cost=48.26..4311.24 rows=70 width=8) (actual time=856.261..864.208 rows=2000 loops=1)
             Buffers: shared hit=20470
             ->  Bitmap Heap Scan on treenode_edge te  (cost=48.26..4311.24 rows=70 width=8) (actual time=856.257..863.974 rows=2000 loops=1)
                       Recheck Cond: (edge &&& '01020000800200000000000000886520C100000000A05C25410000000020B2F440000000003C39364100000000005BFFC000000000F0AFF440'::geometry)
                       Filter: ((edge && '0103000080010000000500000000000000AB6520C100000000185CFFC000000000F0AFF44000000000AB6520C100000000C35C254100000000F0AFF440000000804D39364100000000C35C25410000000020B2F440000000804D39364100000000185CFFC00000000020B2F44000000000AB6520C100000000185CFFC000000000F0AFF440'::geometry) AND (project_id = 1) AND ('0103000080010000000500000000000000886520C100000000005BFFC00000000008B1F440000000003C39364100000000005BFFC00000000008B1F440000000003C39364100000000A05C25410000000008B1F44000000000886520C100000000A05C25410000000008B1F44000000000886520C100000000005BFFC00000000008B1F440'::geometry && st_expand(edge, '17.5'::double precision)) AND _st_3ddwithin(edge, '0103000080010000000500000000000000886520C100000000005BFFC00000000008B1F440000000003C39364100000000005BFFC00000000008B1F440000000003C39364100000000A05C25410000000008B1F44000000000886520C100000000A05C25410000000008B1F44000000000886520C100000000005BFFC00000000008B1F440'::geometry, '17.5'::double precision))
                       Heap Blocks: exact=1816
                       Buffers: shared hit=20470
                       ->  Bitmap Index Scan on treenode_edge_gix  (cost=0.00..48.25 rows=1044 width=0) (actual time=855.795..855.795 rows=2856 loops=1)
                                     Index Cond: (edge &&& '01020000800200000000000000886520C100000000A05C25410000000020B2F440000000003C39364100000000005BFFC000000000F0AFF440'::geometry)
                                     Buffers: shared hit=18654
      Planning time: 3.467 ms
      Execution time: 864.614 ms

    This shows clearly that the big bounding box test on all existing edges is the
    problem, though it already makes use of the GiST index. Are there maybe other indices available that can discard many far-away bounding boxes without looking at them, i.e. that useses a hierarchy like an octree? So far I didn't find any, but it looked like the SP-GiST interface might be useful if one were to create a new one.

    Alternatives: There is one approach that I currently implement to improve this,
    but I'd like to hear if someone has alternative, possibly better or other
    suggestions?

    This is what I tried: create a new table that would partition the space
    into many cubes and store for each all intersecting edges. New, updated or
    deleted nodes would then lead to an update of the relevant cubes, something
    that would probably have to run asynchronously (didn't implement the update yet). Queries then pre-filtered the space by only looking at relevant cubes instead of using the &&& operator with the bounding box of each edge. For a range of cubse sizes this improved query time quite a bit, the above query would run in about a third of thet time above. However, I realize cell sizes will behave differently for different dataset, but I didn't try to implement a dynamic cube size update, yet.  Thinking about it, it makes me wonder if it wouldn't be best if I would implement a octree---table based or as an extension.

    Now I also saw the pointcloud extension and its indexing looked very
    interesting (octree), but I couldn't really see how it could work in my use case. But I would appreciate ideas if you see a way to make it work.

    Best,
    Tom
	
	
	
	

Hey,
This is an interesting problem for sure.
AS always in the "slow query" topic,
improvement can come from a number of places
 - hardware : are your tables on SSD, does your index fit in RAM
 - postgres tuning : you have customised postgresql.conf to fit your hardware
 - query writing : your query is optimally written (looks fine here)
 - data model (detailed after)

IF you want fast results, you need a good indexing, and this is possible if you exploit the sparsness of your data.
So where is the sparsness?


 * Scenario A.1: your usual slices affects a majority of trees
  - you don't really need an index on X,Y, but mainly on Z.
    The idea is to separate your index into index(X,Y) + index(Z)
     Try adding an index on GIST(numrange(ST_Zmin,ST_ZMax)) (you could add it anyway)
     and an index on GIST(geom) (aka only 2D).
 * Scenario A.2 : your usual slices affect few trees
  - you could easily perform a 2 part query, where the first part use an index on tree bounding box, and the second perform the query on edge. This require to create and maintain a tree_bounding_box table with appropriate indexes, which is quite easy with triggers Ex (not optimised):
   WITH filtering_by_tree AS (
     SELECT distinct tree_id
     FROM my_sync_table_of_tree_bounding_boxes as tree
     WHERE tree.bbox &&& your_slice
   ) SELECT edge_id
     FROM treenode_edge  AS te
     WHERE te.geom &&& your_slice
      AND EXISTS (SELECT 1 FROM filtering_by_tree As ft WHERE ft.tree_id = te.tree_id)
  - you could use pgpointcloud to similar effect by construction a patch per tree.

Now something is very ambiguous in your data, and it could have major consequences :
 * Are the edges between nodes that represent neuron __only logical__ (they are straight lines, always the same Zmax-Zmin, and represent a conceptual graph), or are they polylines that represent the __actual__ neurone path?

 * Scenario B.1.1 : edges are only logical and have a constant Zmax-Zmin, i.e. nodes have quantified Z :
  - you don't need edges at all for indexing, you should work only on nodes (i.e. index nodes IN Z + (X,Y), find nodes, then find relevant edges). The great advantage in this case is that you can use the full power of pgpointcloud to create meaningfull patches (of variables size for instance, see this, 3.1, Adapt patch grouping rules ), which means scalling in the range of billions of points possible.

  - you could try to think of using the dual of your current graph (edges become nodes, nodes become edges).

 * Scenario B.1.2 : edges are only logical, but nodes don't have a constant Zmax-Zmin. : you could insert (virtual) nodes so you are again in the same case as scenario B.1.1
 
 * Scenario B.2 : edges are actual neurone path.
  - you could construct intermediary objects (groups of neurone) of variable size (those within cubes), so as to have again a 2 steps query : first look for groups of neuron, then for neurons inside

 * Scenario C : your data don't have obvious sparsity.
  - use the same strategy you currently use, but scale by partitionning your data into numerous tables.
   You can see the partitionning entry of the postgres manual.
   This would be much easier to create and maintain if you could partition on only Z dimension.
   Note that you will need to duplicate (and then de-duplicate) some edges.


Cheers,
Rémi-C


 




