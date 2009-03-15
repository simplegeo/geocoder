.. _lookup:

===================================
Geocoder.us Address Lookup Strategy
===================================

:Author: Schuyler Erle
:Contact: schuyler at geocoder dot us
:Created: 2009/03/13
:Edited: 2009/03/14

Definitions
-----------

Edge
  Database representation of a street segment, consisting of a linestring
  geometry and an edge ID. Edges relate to many ranges and many features
  through its ID.

Feature
  Database representation of a named street, consisting of street name
  and modifier elements, a reference ZIP code, and a primary/alternate flag.

Range
  Database representation of a range of address numbers on a given
  street, consisting of range start and end numbers, an optional prefix
  ending with a non-numeric character, and a delivery ZIP code for that
  range.

Place
  Database representation of a ZIP code, consisting of a city name,
  state abbreviation, a ZIP code, and a primary/alternate flag.

Address record
  A set consisting of exactly one edge, one feature, and one range, related
  through the edge ID.

Address query
  An ordered set of {Number Prefix, Number, Directional Prefix, Type Prefix,
  Qualifier Prefix, Street Name, Qualifier Suffix, Type Suffix, Directional
  Suffix, City, State, ZIP}. All of the elements are optional except Number and
  Street Name. Either ZIP or City must also be present. The State element
  and all of the prefix and suffix elements are assumed to be normalized to
  standard postal abbreviations.

Address string
  A string including some or all of the elements of an address.

Address Lookup Strategy
-----------------------

1. Given a an address query, initialize an empty set of candidate places,
   and an empty set of candidate address records.

#. If a ZIP was given, look up the place from the ZIP, and add the
   place, if any, to the candidate place set.

#. If a city was given, look up all the places matching the metaphone hash
   of the city name, and add them, if any, to the candidate place set.

#. Generate a unique set of ZIPs from the set of candidate places, since a ZIP
   may have one or more names associated with it.

#. Generate a list of candidate address records by fetching all the street
   features matching the metaphone hash of the street name and one of the ZIPs
   in the query set, along with the ranges matching the edge ID of each
   feature, where the given number is in the range. The edge does not
   need to be fetched yet.

#. If the look up generates no results, optionally look up all the street
   features matching the metaphone hash of the street name, along with the
   ranges matching the edge ID of each feature, where the given number is
   in the range. This may be a very time consuming database query, because
   some street names are quite common.

#. Score each of the candidate records as follows:

   a. Score one point for every provided element of the address query that it
      matches exactly. 
   #. Optionally, compute the scaled Damerau-Levenshtein distance (or
      alternately the simple Levenshtein distance) between each provided
      element of the address query and the corresponding element in the
      candidate. Score one minus the scaled distance, which yields a fraction
      of a point.
   #. Score one point if the starting range number matches the parity of the
      queried address number.
   #. Note that the maximum possible score is equal to the number of provided
      elements in the address query. Divide the score against the maximum
      possible. This is the confidence value of the candidate.

#. Sort the candidate address range records by confidence. Retain only the
   records that share the highest confidence as candidates.

#. Fetch all of the edges matching the remaining the candidate address ranges.

#. For each remaining candidate range:

   a. Fetch all of the ranges for the edge ID of the candidate, sorted by
      starting number.
   #. Compute the sum of the differences of the starting and ending house
      number for each range. This is the total number width of the edge.
   #. Take the difference between the candidate starting number and the lowest
      starting number, add the difference between the queried number and the
      candidate starting number, and divide by the total number width. This is
      the interpolation distance.
   #. Optionally, find the local UTM zone and project the edge into it.
   #. Find the point along the line at the interpolation distance.
   #. If the edge was projected, unproject the point.
   #. Assign the point as the geocoded location of the query to the candidate
      record.

#. Construct a set of result ZIPs from the remaining candidates, and look up
   the primary name and state for each ZIP in the set. Assign the primary city
   and state to each candidate.

#. Return the set of candidate records as the result of the query.
 