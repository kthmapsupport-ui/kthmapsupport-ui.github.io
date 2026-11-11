Campus Map: data derived from OpenStreetMap
============================================

Contains information from OpenStreetMap, which is made available here under the Open Database
License (ODbL) 1.0: https://opendatacommons.org/licenses/odbl/1-0/
(c) OpenStreetMap contributors, https://www.openstreetmap.org/copyright

Some buildings of the street maps were added from Overture Maps (https://overturemaps.org), a database that is also
under the ODbL; its contributors include Esri Community Maps contributors (CC BY 4.0), Microsoft (ODbL) and Google
(CC BY 4.0).

What this is: the files below are the derived databases that the Campus Map app carries inside it (map shapes,
walking network, entrances, places, addresses). You may copy, distribute and adapt them under the ODbL: credit
OpenStreetMap, say that you changed them if you did, and offer what you publish from them under the same licence.

What this is NOT: the rooms, floor plans and the names taken from KTH Places belong to KTH and are not part of this
offer. In gyms.json and scooter_parking.json the field "sources" says where each kind of entry comes from; only
the OpenStreetMap part is offered under the ODbL here, the other sources keep their own terms.

Coordinates are integers (degrees x 1,000,000) in most files. The format of each file is described below.
Questions: kthmap.support@gmail.com

FILES
-----
vector_map.json: Street map of the main campus: buildings, roads, water, green
vector_region.json: Coarse street map of greater Stockholm around the main campus
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
     the walking network (a path or a street), marked as a guess.
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

Only spots near a campus are kept. Usage:
    python tools/build_scooter_parking.py [--city-file esp.json]   (without it the city layer is downloaded)

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
