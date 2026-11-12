Campus Map: data derived from OpenStreetMap
============================================

Contains information from OpenStreetMap, which is made available here under the Open Database
License (ODbL) 1.0: https://opendatacommons.org/licenses/odbl/1-0/
(c) OpenStreetMap contributors, https://www.openstreetmap.org/copyright

Some buildings of the street maps were added from Overture Maps (https://overturemaps.org), a database that is also
under the ODbL; its contributors include Esri Community Maps contributors (CC BY 4.0), Microsoft (ODbL) and Google
(CC BY 4.0).

What this is: the files below are the derived databases that the Campus Map app carries inside it (map shapes,
walking network, bicycle network, entrances, places, addresses). You may copy, distribute and adapt them under the ODbL: credit
OpenStreetMap, say that you changed them if you did, and offer what you publish from them under the same licence.

What this is NOT: the rooms, floor plans and the names taken from KTH Places belong to KTH and are not part of this
offer. In gyms.json and scooter_parking.json the field "sources" says where each kind of entry comes from; only
the OpenStreetMap part is offered under the ODbL here, the other sources keep their own terms.

Coordinates are integers (degrees x 1,000,000) in most files, and in bike_graph.bin (a binary file: the format is the
docstring of bike_graph_codec.py below, which has a reader and a writer). The format of each file is described below.
Questions: kthmap.support@gmail.com

FILES
-----
vector_map.json: Street map of the main campus: buildings, roads, water, green
vector_region.json: Coarse street map of greater Stockholm around the main campus
vector_outer.json: Coarser street map of the ring around greater Stockholm, so that every campus has map all around it
vector_flemingsberg.json: Street map of the Flemingsberg campus
vector_haninge.json: Street map of the Haninge campus
vector_kista.json: Street map of the Kista campus
vector_sodertalje.json: Street map of the Sodertalje campus
vector_solna.json: Street map of the Solna campus
walk_graph.json: Walking network of the campuses (nodes and edges)
doors.json: Entrances of the buildings (OpenStreetMap entrance nodes; a guess where there is none)
campus.json: Places near the campuses (food, shops, services...)
addresses.json: Address points (street and house number)
gyms.json: Gyms and outdoor gyms near the campuses (OpenStreetMap places plus facts named in the file)
scooter_parking.json: Scooter parking places (open data of Stockholm city, OpenStreetMap; sources named in the file)
access.json: Benches, accessible toilets, parking for disabled people, lifts and step-free entrances, and the wheelchair tags of places
bike_parking.json: Places to leave a bike, an e-bike or an e-scooter near the campuses, with what OpenStreetMap says about each one (roof, lockers, how many bikes, a charger for e-bikes, a fee, who may use it)
bike_graph.bin: Bicycle network of greater Stockholm, Sodertalje and Haninge: cycle ways, paths, streets and what a rider can do on them, as nodes and edges (binary)

FORMAT NOTES (from the scripts that build the files)
---------------------------------------------------

[build_vector_map.py]
Build the bundled street map of the campus area from OpenStreetMap (ODbL).

    python tools/build_vector_map.py        # writes assets/data/vector_map.json

One free Overpass query at build time. The app draws these shapes itself (lib/services/vectormap.dart),
so the street map needs no tile server, no key and works offline. (c) OpenStreetMap contributors.

Water and green (with their holes) and the sea come from the raw region data of tools/fetch_region_raw.py,
assembled into proper rings with shapely, so run that first and use tools/.venv/Scripts/python.

File format (coordinates are integers: degrees * q):
  q          1000000
  bbox       [south, west, north, east] in degrees
  green      [[ring, hole, ...], ...]               parks, grass, forest; flat [lat, lng, ...] rings
  water      [[ring, hole, ...], ...]               lakes and sea
  buildings  [[...], ...]
  roads      [[class, lat, lng, lat, lng, ...], ...]   class: 0 major, 1 street, 2 service, 3 path, 4 steps
  rail       [[...], ...]
  names      [[name, lat, lng], ...]               street names (one label per street)

[build_vector_region.py]
Build the coarse street map of greater Stockholm that surrounds the detailed campus map.

    python tools/fetch_region_raw.py           # once: raw OpenStreetMap shapes (cached)
    tools/.venv/Scripts/python tools/build_vector_region.py

Writes assets/data/vector_region.json. Everything is simplified (a few metres) and the campus box is cut
out of the areas and left out of the roads, because assets/data/vector_map.json draws it in detail.
(c) OpenStreetMap contributors, ODbL.

Format (integers are degrees * q):
  q, bbox [south, west, north, east], hole [south, west, north, east] (the campus box)
  water, green  [[ring, hole, ...], ...]  flat [lat, lng, ...] rings, biggest area first
  roads         [[class, lat, lng, ...], ...]   class 0 major road, 1 street
  rail          [[lat, lng, ...], ...]

[build_vector_outer.py]
Build the OUTER street map: the coarse ring around greater Stockholm that makes the covered area reach well past every campus.

    tools/.venv/Scripts/python tools/fetch_outer_raw.py     # once: the raw OpenStreetMap shapes (cached in tools/cache/outer)
    tools/.venv/Scripts/python tools/build_vector_outer.py

Writes assets/data/vector_outer.json, in the format of vector_region.json (see build_vector_region.py). It covers OUT (outer_geom.py)
except the region (vector_region.json draws that) and the detailed map of every campus outside the region (their own files draw
them). Around those campuses (MID_METERS) it is as fine as the region; further out it is coarse: main roads, the railways, and the
water and green areas that are big enough to be seen on a map of the whole area.  (c) OpenStreetMap contributors, ODbL.

[build_campuses.py]
Detailed maps, walking network and places for the campuses outside Valhallavägen.

    tools/.venv/Scripts/python tools/build_campuses.py [--refetch] [id ...]

For every campus of tools/campuses.py except the main one, one Overpass query for the shapes (buildings, roads, rail,
water, green) and one for places and address points; the answers are cached in tools/cache/ so the build can be repeated.
Writes assets/data/vector_<id>.json (the same format as vector_map.json), merges the walking network of every
campus into assets/data/walk_graph.json, the places into assets/data/campus.json and the campus list into
assets/data/campuses.json. (c) OpenStreetMap contributors, ODbL.

[build_walk_graph.py]
Build the on-device walking graph of the campus area from OpenStreetMap (ODbL).

    python tools/build_walk_graph.py       # writes assets/data/walk_graph.json

One-off, free: a single Overpass query at build time. The app then routes on the phone (no routing
server, works offline). Data (c) OpenStreetMap contributors, ODbL: keep the attribution in the app.

File format (compact, integers):
  bbox   [south, west, north, east] in degrees
  nodes  flat list [lat1e6, lng1e6, ...]
  edges  flat list [a, b, lengthDm, flags, ...]   a/b = node indexes, length in decimetres,
                                                  flags = 1 for stairs, else the surface, wheelchair, incline and trail bits
                                                  of tools/walk_access.py

[build_osm_doors.py]
The entrances of every building of every campus: assets/data/doors.json.

    tools/.venv/Scripts/python tools/build_osm_doors.py [--refetch]

Sources, for each building drawn on the map (ways, buildings with a courtyard and the ones added from Overture):
  1. the entrance nodes of OpenStreetMap (entrance=main|yes|home|residential|staircase|shop|service|garage...)
     within 6 m of its outline: real doors;
  2. a building with no entrance in OpenStreetMap gets the most likely one: the point of its outline nearest to
     the walking network (a path or a street), marked as a guess; a point that lies inside another building (by more than
     INSIDE_OTHER_M: a building drawn inside the outline of a bigger one) is not taken while a better one is left.
Entrances that are private (access=private|no), emergency exits and plain exits are left out.

File: {"q": 1000000, "b": [[lat, lng, areaM2, kth, ring, doors], ...]} in 1/q degrees; ring = flat [lat, lng, ...];
doors = [[lat, lng, kind], ...] with kind 0 main, 1 door, 2 service, 3 guessed; kth = 1 when the building is one of
the KTH buildings with floor plans (the app uses the entrances of its own catalogue for those).

[build_campus_snapshot.py]
Bundle a snapshot of the OpenStreetMap campus data (same query and same
conversion as CampusData in lib/services/data.dart) so the map works offline
and on first launch even when Overpass is slow or rate limited.

[fetch_osm_addr.py]
Download OpenStreetMap address points (street + house number) around the KTH campuses: in Swedish
data they sit at the entrance of the address, which tells where a building's main door is.
Free Overpass query (ODbL); cached in tools/cache/osm_addr.json.

[build_gyms.py]
Builds assets/data/gyms.json: gyms and outdoor gyms (utegym) near the KTH campuses.

Sources:
  osm   OpenStreetMap (ODbL, (c) OpenStreetMap contributors): leisure=fitness_centre, leisure=fitness_station,
        amenity=gym, sport=calisthenics/crossfit/fitness; within a radius of each campus (see CAMPUSES).
  kth   Facts taken from published pages and placed by hand:
        - KTH-Hallen is a gym of Nordic Wellness in a hall owned by the KTH sports foundation (Brinellvagen 38).
        - Akademiska Hus' property map of KTH Campus (Elsparkcyklar.pdf, the same drawing as the scooter places) shows an
          "UTEGYM" symbol north of KTH-Hallen; its position is placed on the real map with the same fitted transform
          (about 15 m of error).
        - Akademiska Hus (24 March 2022): the Albano outdoor gym is at Bjornasplatsen, where Roslagsbanan crosses Greta
          Arwidssons vag, and is open to everyone (the OpenStreetMap station there takes its name).

Record: [lat, lng, kind (0 gym, 1 outdoor gym), name, access (0 unknown, 1 free, 2 paid, 3 members), website, source (0 osm, 1 kth)]

WHAT IS IN THE FILE (10 Nov 2026): only places CONFIRMED against a page of their operator or owner that is current (CONFIRMED below,
with the evidence of each; the list of what was dropped and why is docs/GYMS_VERIFICATION.md). OpenStreetMap is only used to FIND
candidates: the ones that are not confirmed are printed and are NOT in the app (gyms close and move: of the 53 places this script
once took from OpenStreetMap, 20 could not be confirmed, 5 of them are gone for sure, FLX Fitness among them). To add a place,
check it on its own page and add a line to CONFIRMED.

Usage: python tools/build_gyms.py

[build_scooter_parking.py]
Builds assets/data/scooter_parking.json: where an electric scooter may be parked near the KTH campuses.

Sources (all free to reuse):
  city  Stockholms stad, Trafikkontoret, open data "Elsparkcykelplats" (WFS od_gis:Elsparkcykelplats): the places the
        city has marked for shared e-scooters. Reuse is free and licence-free (https://openstreetgs.stockholm.se).
  kth   The five areas Akademiska Hus set up at KTH Campus (Valhallavagen) for e-scooters, from the map "Elsparkcyklar.pdf"
        of their news post "Ny parkering for elsparkcyklar" (17 Oct 2022). The map is a drawing turned about 53 degrees
        from north: its numbered arrows were placed on the real map with a similarity transform fitted on eight buildings
        (about 15 m of error), so their positions are approximate.
  osm   OpenStreetMap, amenity=scooter_parking / kick-scooter_parking (ODbL, (c) OpenStreetMap contributors).

Only spots near a campus are kept. Two places closer than SAME_PLACE_M with the same address are one place (the city's layer has
two or three points for one marked area): the newest month is kept. Usage:
    python tools/build_scooter_parking.py [--city-file esp.json]   (without it the city layer is downloaded)
    python tools/build_scooter_parking.py --merge-only             (no download: the same-place merge on the file as it is)

[build_access.py]
Builds assets/data/access.json: what OpenStreetMap knows that helps people with reduced mobility, in the boxes of the campuses.

    python tools/build_access.py            # fetch (cached in tools/cache/access_<id>.json) and write the file
    python tools/build_access.py --refetch  # ask Overpass again

What it keeps (one Overpass query per campus box, free):
  spots   [[lat, lng, kind, status, name], ...]  kind:
            bench               amenity=bench: a place to rest
            accessible_toilets  amenity=toilets with wheelchair=yes|designated or toilets:wheelchair=yes
            disabled_parking    parking for disabled people (capacity:disabled, parking_space=disabled)
            elevator            highway=elevator (a lift outdoors, at a station or a footbridge)
            accessible_entrance an entrance or a door with wheelchair=yes|designated, or an automatic door
          status  the wheelchair tag of any other place or building that has one: {"node/123": "yes" | "limited" | "no"}, so that
                  a place the app already shows can say whether it is accessible.
The file says what the data says: a place with no tag is NOT in `status` (unknown is not the same as not accessible).
Data (c) OpenStreetMap contributors, ODbL: the file is offered in the open data zip (tools/pack_osm_data.py).

[build_bike_parking.py]
Builds assets/data/bike_parking.json: where a bike, an e-bike or an e-scooter can be left near the KTH campuses, with what
OpenStreetMap says about each place (a roof, lockers, how many bikes, a charger for e-bikes, a fee, who may use it).

Source: OpenStreetMap, amenity=bicycle_parking (nodes, and ways and relations as their centre) and the charging stations for
bikes (amenity=charging_station with bicycle=yes). (c) OpenStreetMap contributors, Open Database Licence (ODbL) 1.0. The
places for e-scooters are not here: they stay in assets/data/scooter_parking.json (tools/build_scooter_parking.py).

Area: every campus box of assets/data/campuses.json plus 800 m on every side (a campus inside another one, AlbaNova, is in the
box of the other). One Overpass query per box, one at a time, with a pause between them and growing waits when the server is
busy (it often answers 502): `--max-wait-minutes` (default 60) is how long one query is retried before giving up on it. If a
box cannot be had, `--cache-dir` (a folder of earlier Overpass answers, any file with a list of elements or {"elements": [...]})
and, last, the plain places of assets/data/campus.json (position only, no tags) are used for that box, and the file says so in
"sources" and "fallback". User-Agent: campusmap-build/0.1. Usage:

    python tools/build_bike_parking.py [--out FILE] [--raw-dir DIR] [--max-wait-minutes N] [--cache-dir DIR] [--self-test]

THE FORMAT of assets/data/bike_parking.json (version 1):

    {"v": 1, "built": "YYYY-MM-DD", "sources": {...}, "fallback": [...],
     "spots": [[lat, lng, kind, flags, capacity, "n123", "label"], ...],
     "private": ["n456", ...]}

  lat, lng   degrees, six decimals (about 0.1 m)
  kind       the index of ParkingKind (lib/services/parking.dart): 0 a rack in the open (stands, wall loops, bollards...),
             1 a rack under a roof (a shelter: covered=yes, bicycle_parking=shed), 2 lockers (boxes), 3 inside a building or
             underground (bicycle_parking=building, indoor=yes, a bike garage). The places marked for e-scooters are another file
  flags      a sum of: 1 covered (under a roof, or inside), 2 secure (lockers, cameras, watched), 4 charging (a place to charge an
             e-bike), 8 fee (paid), 16 restricted (customers, permit holders, residents: not a place for everybody), 32 roof known
             (OpenStreetMap says yes or no about a roof: without it "not covered" only means "not known" and the app says nothing)
  capacity   how many bikes, 0 when OpenStreetMap does not say
  id         "n123" is node 123, "w456" way 456, "r789" relation 789 of OpenStreetMap (the app joins it to the same place of
             campus.json, whose id is "node/123")
  label      the name when the place has one that says more than "bicycle parking"; "" otherwise
  private    the ids (as above) of the racks OpenStreetMap says are private: they are not in "spots", and the app removes a rack
             with one of these ids from the plain racks of campus.json (which has them as racks like any other)

What is left out: access=private (private for sure, unless bicycle=yes or designated says bikes are welcome; listed in "private"). What is kept and
flagged 16: access=customers, permit, no, delivery, destination, residents, members, employees. Two places closer than 3 m are
one place (the richer one stays, its flags and its largest capacity too). A charging station for bikes within 10 m of a rack is
a charger of that rack (flag 4); alone it is a rack of its own.

[bike_graph_codec.py]
The file format of the bike ways bundled in the app (assets/data/bike_graph.bin): the reference encoder and decoder.

The app reads it with lib/services/bikegraph.dart (BikeGraph.decode); both are tied to this file by the fixtures of test/fixtures
(bike_graph_mini*.bin and .json, made by `python tools/bike_graph_codec.py --fixtures`). docs/RIDING_MODES.md has the meaning of every
number. All integers are little endian; a "varint" is an unsigned LEB128; "zigzag" maps a signed integer n to (n << 1) ^ (n >> 63)
and then writes it as a varint. Coordinates are integer micro-degrees (degrees times 1,000,000).

    header   u32 magic 0x47424D43 ("CMBG" as bytes 43 4D 42 47) | u16 version = 1 | u16 flags (bit 0: every edge has a grade)
             | u32 nodeCount | u32 edgeCount | u32 shapeCount | u16 boxCount | u16 reserved = 0
             | boxCount x 4 x i32: south, west, north, east of each area the file covers
    sections four of them, each a u32 byte length followed by that many bytes:
      NODES      nodeCount x [zigzag dLat, zigzag dLng]            deltas from the previous node (the first from 0, 0)
      NODEFLAGS  nodeCount x u8                                    bit 0 traffic lights, bit 1 marked crossing, bit 2 barrier
      EDGES      edgeCount x [zigzag (a - previous a), zigzag (b - a), varint length in decimetres, u8 class, u8 attrs,
                              (i8 grade, only when the header says so), varint shape points of this edge]
      SHAPE      shapeCount x [zigzag dLat, zigzag dLng]           the points between the ends of each edge, in order, each as
                                                                   the delta from the one before it (the first from node a)
    attrs      bits 0-2 surface class, bits 3-4 direction (0 both ways, 1 only a to b, 2 only b to a), bit 5 lit, bits 6-7 zero.
    class      0 cycleway, 1 shared path, 2 street with a bike lane or track, 3 quiet street, 4 street, 5 busy road,
               6 main road, 7 push (walk with the machine by hand)
    surface    0 unknown, 1 smooth, 2 paving stones or compacted, 3 gravel, 4 cobblestones, 5 unpaved, 6 and 7 not used
    grade      percent of climb going from a to b (a descent is negative), clamped to -30..30; going from b to a it is the opposite

[bike_classify.py]
The rules of docs/RIDING_MODES.md, section 4.2, as pure functions: what a rider can do on an OpenStreetMap way or node.

    classify_way(tags)  -> [Edge, ...]   the directed edges a way gives a bike, in the direction of the way (its first node to its last):
                                         [] when a bike may not use it, one edge when both ways are alike, two when they differ
                                         (a one-way street, a lane on one side only)
    exclusion(tags)     -> str | None    why a bike may not use the way (None when it may)
    node_flags(tags)    -> int           bit 0 traffic lights, bit 1 marked crossing, bit 2 barrier (the rider gets off)
    surface_class(tags) -> int           0 unknown ... 5 unpaved, as the file

An Edge is (cls, surf, dir, lit, incline): the class and surface numbers of tools/bike_graph_codec.py, dir BOTH / FORWARD (first node
to last) / BACKWARD, lit a bool, incline the percent of climb going from the first node to the last when OpenStreetMap says it, else None.
Nothing here reads a file or the network: the builder (build_bike_graph.py) calls these for every way and node, and
test_bike_classify.py pins every rule.

Where the spec leaves a gap this file decides, and says so next to the decision (the report lists them):
  * a street with a lane on a road above tertiary (secondary, primary, trunk) is not class 2; a separated track is, a painted lane
    takes the road one class down, a shared-lane marking changes nothing;
  * a tertiary street with no speed given is class 4, a tertiary street at 30 or less too (not a quiet street);
  * a cycle street (bicycle_road=yes, cyclestreet=yes) is class 2;
  * a path or track that is only for people on foot (foot=designated, no word about bicycles) or where bicycles are "discouraged" is
    class 7, like a footway;
  * a primary or trunk road with 90 km/h or more and no word that bicycles may use it is not used at all (it is as good as a motorway).
(c) OpenStreetMap contributors, ODbL.

[build_bike_graph.py]
Build the bike network of the app (assets/data/bike_graph.bin, format of tools/bike_graph_codec.py) from OpenStreetMap (ODbL).

    tools/.venv/Scripts/python tools/fetch_bike_raw.py        # once: the raw data (cached in tools/cache/bike, resumable)
    tools/.venv/Scripts/python tools/build_bike_graph.py      # writes assets/data/bike_graph.bin and prints the figures
    tools/.venv/Scripts/python tools/bike_graph_check.py      # the validation of docs/RIDING_MODES.md section 4.6

Sources: tools/cache/region_roads.json (the streets of greater Stockholm) and the files of fetch_bike_raw.py (cycleways, paths,
footways, tracks, service roads; and every way of Sodertalje and Haninge), plus the nodes with tags (traffic lights, crossings,
barriers). The rules of what a bike may do on a way are in bike_classify.py; the areas and the zones in bike_areas.py.

What it does, in order:
  1. every way is classified (bike_classify.classify_way); a way a bike may not use is left out; ways are cut at the edge of the areas
  2. the graph is made of the OpenStreetMap nodes the ways SHARE (an intersection), the ends of ways and the nodes with a flag: a way is
     cut there into units; nothing connects two ways that do not share a node, except the short gaps repaired in step 4
  3. a stretch of a footway or a path that lies on a flight of steps (the steps drawn twice) is dropped; contraction: a node where exactly
     two units of the same attributes meet, with no flag, is not a node (the shape keeps the geometry)
  4. repairs of the data: a way that ends within a few metres of another way it was meant to join gets a link (a mapper's gap; a cycleway
     that stops up to 12 m short, a shared path 8 m, gets a link to walk); never where steps are, across a building (campus maps) or a
     railway; a loop is cut in two, two ways on the same nodes become one. With --near-links (not in the shipped file) also two nodes up
     to 12 m apart, one of them on a cycle way and the other on a street or a cycle way, that no way of 150 m or less joins in both
     directions (link_near_nodes)
  5. components under 40 nodes are dropped unless a campus is in them
  6. one-way pockets (a few ways a rider can enter and not leave, or the other way round) get the cheapest ways out and in as a second
     edge of class 7 against the traffic, never on a busy or a main road, a bridge or a tunnel
  7. size (4.3): when the file is over the limit, outside the zones of the campuses the unneeded service ways, tracks and pushed
     footways are dropped, in this order, each step measured (then the unmarked paths, dead ends of pushed footways and a coarser
     shape if it is still too big); nothing that connects two bike ways is dropped (a way goes only when its ends stay joined by other
     ways that are not much longer)
  8. nodes are ordered along a Hilbert curve (so the deltas of the file are small), shapes are simplified to 3 m, and the file is written

The numbers are printed and kept in tools/cache/bike/build_report.json (docs/BIKE_GRAPH_REPORT.md has the last run).
